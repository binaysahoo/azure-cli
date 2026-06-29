# synctl — Architecture Notes

> Notes from studying Azure CLI (`az`) architecture, with recommendations for building **synctl** — a kubectl-style developer CI/CD CLI for Perforce (P4) workflows: shelve → build → unit test → regression → approval → code review → merge → submit.

---

## 1. Azure CLI Architecture (What `az` Actually Is)

Azure CLI is **not** a monolith. It is a layered Python platform with ~100+ pluggable command modules.

### 1.1 High-Level Layers

```
┌─────────────────────────────────────────────────────────────────┐
│  User:  az [group] [subgroup] [command] {flags}                 │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  Entry: azure/cli/__main__.py                                   │
│    → get_default_cli() → cli.invoke(args)                       │
│    → telemetry start/conclude, exit codes, auto-upgrade         │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  Knack Framework (Microsoft's generic CLI framework)            │
│    CLI, CLICommandsLoader, CommandInvoker, Parser, Help, Output │
└────────────────────────────┬────────────────────────────────────┘
                             │
┌────────────────────────────▼────────────────────────────────────┐
│  azure-cli-core (AzCli extends Knack)                           │
│    MainCommandsLoader, AzCommandsLoader, AzCliCommandParser     │
│    Auth (MSAL), Profiles, Cloud config, Error types, AAZ layer  │
└────────────────────────────┬────────────────────────────────────┘
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────────┐
│ Command Modules │ │   Extensions    │ │  azure-cli-telemetry │
│ (built-in)      │ │ (optional WHL)  │ │  (client telemetry)  │
│ vm, storage,    │ │ az extension    │ │                      │
│ network, ...    │ │ install         │ │                      │
└────────┬────────┘ └────────┬────────┘ └─────────────────────┘
         │                   │
         ▼                   ▼
┌─────────────────────────────────────────────────────────────────┐
│  Azure SDK (azure-mgmt-*, azure-core) + REST via AAZ          │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Package Layout (Monorepo)

| Package | Role |
|---------|------|
| `azure-cli` | Ships all built-in command modules (`azure.cli.command_modules.*`) |
| `azure-cli-core` | Framework extensions on Knack: auth, parsing, errors, profiles, AAZ |
| `azure-cli-telemetry` | Client-side usage telemetry → Kusto |
| `azure-cli-testsdk` | Scenario test framework for command modules |
| `tools/` | Build, packaging, index generation |

### 1.3 Core Framework: Knack

Azure CLI 2.x is built on **[Knack](https://github.com/Microsoft/knack)** — Microsoft's Python CLI framework (the Go equivalent would be Cobra + a thin custom core).

Knack provides:
- Command/group registration (`CLICommandsLoader`)
- Argument parsing (`CLICommandParser`)
- Help system
- Output formats (JSON, table, TSV)
- Tab completion (`argcomplete`)
- Logging
- Config file support

Azure extends Knack via `AzCli`:

```python
# src/azure-cli-core/azure/cli/core/__init__.py
AzCli(cli_name='az',
      commands_loader_cls=MainCommandsLoader,
      invocation_cls=AzCliCommandInvoker,
      parser_cls=AzCliCommandParser,
      logging_cls=AzCliLogging,
      output_cls=AzOutputProducer,
      help_cls=AzCliHelp)
```

### 1.4 Command Loading (Lazy + Indexed)

`MainCommandsLoader` is the heart of scalability:

1. **Command index** (`~/.azure/commandIndex.json`) — pre-built map of command → module. Avoids loading all 100+ modules on every invocation.
2. **Module discovery** — `pkgutil.iter_modules('azure.cli.command_modules')` when index rebuilds.
3. **Extensions** — separate index, installed as Python wheels via `az extension install`.
4. **Stub commands** — for fast tab-completion without full module load.

Each module exports:

```python
class MyModCommandsLoader(AzCommandsLoader):
    def load_command_table(self, args): ...   # register commands
    def load_arguments(self, command): ...   # arg metadata

COMMAND_LOADER_CLS = MyModCommandsLoader
```

### 1.5 Command Authoring Pattern

Commands are **plain Python functions**. The framework introspects signatures for required/optional args and help text.

```python
def create_myfoo(cmd, myfoo_name, resource_group_name, location=None):
    client = cf_mymod(cmd.cli_ctx)
    ...
```

Registration:

```python
with self.command_group('myfoo', mymod_custom) as g:
    g.command('create', 'create_myfoo')
```

**Special injected params:**
- `cmd` — command instance, access to `cli_ctx`
- `client` — SDK client from `client_factory`

### 1.6 AAZ (Atomic Azure CLI) — Modern Codegen Layer

Newer commands use **AAZ** — atomic commands mapped 1:1 to REST operations, generated from OpenAPI/Swagger via `aaz-dev-tools`. Benefits: faster dev, consistent style, no SDK release wait.

For synctl: analogous pattern = generate command stubs from your pipeline API OpenAPI spec.

### 1.7 Authentication & Session

| Concern | Azure CLI approach |
|---------|-------------------|
| Identity | MSAL (user, SP, managed identity, device code) |
| Session store | `~/.azure/azureProfile.json` (accounts/subscriptions) |
| Config | `~/.azure/az.json` via Knack config |
| Cloud endpoints | Multi-cloud profiles (AzurePublicCloud, etc.) |
| SDK creds | `CredentialAdaptor` wraps MSAL → Track 2 SDK |

### 1.8 Output & Scriptability

From `doc/command_guidelines.md` — rules synctl should copy:

- **stdout** = structured command output only
- **stderr** = logs, progress, errors
- Support **JSON / table / TSV** (`-o json`)
- **`--query`** with JMESPath for filtering
- Return objects/dicts, never raw strings
- POSIX-friendly (pipe to `jq`, `grep`)
- Defined **exit codes**: 0=success, 1=error, 2=parse error, 3=not found

### 1.9 Error Handling

Layered error types in `azclierror.py`:
- `UserFault` — bad input, auth, not found (user's mistake)
- `ClientError` — CLI internal bug
- `ServiceError` — backend failure

Each maps to telemetry `ActionResult` and consistent user messaging.

### 1.10 Telemetry

- **Client telemetry** — every command, sent at end (`telemetry.start()` / `conclude()`)
- **ARM telemetry** — server-side HTTP logs (separate)
- Opt-out: `az config set core.collect_telemetry=no`
- Fields: command name, params (names only), duration, success/fault, version, OS

### 1.11 Extensions vs Modules

| | Built-in Module | Extension |
|--|-----------------|-----------|
| Install | Ships with `az` | `az extension install` |
| Release | Tied to CLI cadence | Independent velocity |
| Standards | Strict | More experimental OK |
| Use case | GA commands | Preview, niche, fast iteration |

### 1.12 Key Dependencies (azure-cli-core)

```
knack, argcomplete, jmespath, azure-core, azure-mgmt-core,
msal, requests, cryptography, PyJWT, humanfriendly, packaging
```

`azure-cli` itself pins **dozens** of `azure-mgmt-*` SDK packages (one per service).

### 1.13 Testing & Dev Tooling

- **azdev** — dev CLI for style, linter, tests, extension create/publish
- **Scenario tests** — record/replay HTTP (VCR-style) in `tests/latest/`
- **Command index generation** — `azdev latest-index generate/verify`

---

## 2. kubectl Patterns Worth Copying

kubectl is the better mental model for synctl than az (smaller domain, workflow-oriented, watches long-running ops).

| Pattern | kubectl | synctl equivalent |
|---------|---------|-------------------|
| Resource nouns | `pod`, `job`, `pipeline` | `shelve`, `build`, `run`, `review`, `submit` |
| Verbs | `get`, `describe`, `create`, `delete`, `watch` | `list`, `describe`, `start`, `cancel`, `watch`, `approve` |
| Context | `kubectl config current-context` | `synctl context` (P4 server, user, workspace, stream) |
| Long-running | `kubectl wait`, `kubectl logs -f` | `synctl run watch`, `synctl logs -f` |
| Output | `-o json/yaml/wide` | `-o json/table` |
| Plugins | `kubectl-foo` in PATH | `synctl-foo` or Cobra plugin pattern |
| Server mode | kube-apiserver | synctl API server (gRPC/REST) |
| Declarative | YAML manifests | `synctl apply -f pipeline.yaml` |

---

## 3. synctl — Proposed Architecture

### 3.1 Design Principles (borrowed from az + kubectl)

1. **Thin CLI, fat server** — CLI is a client; pipeline orchestration lives server-side.
2. **Noun-verb commands** — `synctl shelve create`, `synctl run watch`, `synctl review approve`.
3. **JSON-first output** — everything scriptable in CI and P4 triggers.
4. **Context-aware** — P4 server, depot, stream, workspace, Jenkins job mapping.
5. **Pluggable stages** — each pipeline step is a registered handler (like az command modules).
6. **Audit trail** — every action logged to Postgres + optional Kafka event stream.
7. **Human dashboard** — web UI reads same API; CLI and UI are peers.

### 3.2 Recommended Stack Mapping

Your planned stack maps cleanly to az/kubectl roles:

| Layer | Your choice | Role | Azure CLI analog |
|-------|-------------|------|------------------|
| CLI framework | **Cobra** | Commands, flags, help | Knack |
| Config | **Viper** | `~/.synctl/config`, env `SYNCTL_*` | Knack config + `az.json` |
| API server | **Gin** (HTTP) + **gRPC** | Orchestration API | ARM + internal services |
| Persistence | **Postgres** | Runs, approvals, audit, state machine | N/A (az is stateless client) |
| Events | **Kafka** | Pipeline events, log aggregation, notifications | Telemetry pipeline |
| CI integration | **Jenkins** | Build/test/regression execution | Azure SDK calls |
| Dashboard | **Web** (React or Go templates) | History, approvals, coverage | Azure Portal (separate) |
| VCS | **P4** | shelve, submit, merge | N/A |

**Note:** Gin and gRPC can coexist — gRPC for CLI ↔ server (fast, typed), Gin for dashboard + webhooks (Jenkins callback, approval links).

### 3.3 Suggested Repo Layout

```
synctl/
├── cmd/
│   └── synctl/              # main entry, Cobra root
├── internal/
│   ├── cli/                 # command groups (like command_modules)
│   │   ├── shelve/
│   │   ├── build/
│   │   ├── run/
│   │   ├── review/
│   │   ├── merge/
│   │   └── context/
│   ├── config/              # Viper: profiles, P4 connection
│   ├── client/              # gRPC client for CLI
│   ├── server/              # gRPC + Gin handlers
│   ├── pipeline/            # state machine: stages, transitions
│   ├── p4/                  # P4 API wrapper (shelve, submit, integrate)
│   ├── jenkins/             # trigger jobs, poll status, fetch logs
│   ├── store/               # Postgres repos (runs, steps, approvals)
│   ├── events/              # Kafka producer/consumer
│   └── auth/                # tokens, RBAC for approvals
├── api/
│   └── proto/               # gRPC definitions
├── web/                     # dashboard SPA or server-rendered
└── plugins/                 # optional synctl-* plugins
```

### 3.4 Pipeline State Machine

```
                    ┌──────────┐
                    │  SHELVED │
                    └────┬─────┘
                         │ synctl run start
                    ┌────▼─────┐
              ┌─────│  QUEUED  │─────┐
              │     └────┬─────┘     │
              │          │           │
         ┌────▼────┐┌────▼────┐ ┌────▼────┐
         │ BUILDING││ UNITTEST│ │REGRESSION│  (parallel or serial)
         └────┬────┘└────┬────┘ └────┬────┘
              │          │           │
              └──────────┼───────────┘
                    ┌────▼─────┐
                    │ COVERAGE │
                    └────┬─────┘
                    ┌────▼─────┐
                    │ APPROVAL │  (human gate)
                    └────┬─────┘
                    ┌────▼─────┐
                    │  REVIEW  │  (code review / voice review hook)
                    └────┬─────┘
                    ┌────▼─────┐
                    │  MERGE   │  (integrate / resolve)
                    └────┬─────┘
                    ┌────▼─────┐
                    │ SUBMITTED│
                    └──────────┘
```

Each transition:
- Persisted in Postgres (`pipeline_runs`, `pipeline_steps`)
- Event published to Kafka (`synctl.pipeline.step.completed`)
- CLI: `synctl run describe <id>` shows current stage + history

### 3.5 Command Surface (Draft)

```bash
# Context (like kubectl config + az account)
synctl context set --p4port p4:1666 --user binay --client my-ws --stream //depot/main
synctl context show

# Shelve workflow
synctl shelve create --desc "feature X"
synctl shelve list
synctl shelve describe <id>

# Pipeline runs
synctl run start --shelve <id> [--pipeline default]
synctl run list [--status running|failed]
synctl run describe <run-id>
synctl run watch <run-id>          # stream status (like kubectl watch)
synctl run cancel <run-id>
synctl logs <run-id> [-f] [--step build|unittest|regression]

# Gates
synctl review list --pending
synctl review approve <run-id> --comment "LGTM"
synctl review reject <run-id>

# Merge & submit
synctl merge start <run-id>
synctl merge status <run-id>
synctl submit <run-id>             # P4 submit after all gates pass

# Coverage
synctl coverage show <run-id>

# Config
synctl config set jenkins.url=https://jenkins.example.com
synctl config set output format=json
```

### 3.6 Server API Split

| Transport | Consumers | Endpoints |
|-----------|-----------|-----------|
| **gRPC** | synctl CLI, automation | `StartRun`, `GetRun`, `WatchRun` (stream), `Approve` |
| **Gin HTTP/REST** | Web dashboard, Jenkins webhooks, approval email links | `GET /api/v1/runs`, `POST /hooks/jenkins` |
| **Kafka** | Log indexer, Slack bot, metrics | consume `synctl.*` topics |

### 3.7 Postgres Schema (Core Tables)

```sql
-- pipeline_runs: one per shelve-triggered workflow
-- pipeline_steps: build, unittest, regression, approval, merge, submit
-- approvals: who, when, comment
-- p4_changes: shelve/cl changelist mapping
-- jenkins_jobs: job name, build number, status, log URL
-- audit_log: all mutations (compliance)
```

### 3.8 Jenkins Integration

- **Trigger:** server calls Jenkins REST API on stage entry
- **Callback:** Gin webhook `POST /hooks/jenkins` updates step status
- **Logs:** fetch console log → store blob or S3; CLI `synctl logs` tails from store
- **Summary:** parse JUnit/coverage artifacts → attach to run record

### 3.9 Kafka Topics (Suggested)

```
synctl.pipeline.run.created
synctl.pipeline.step.started
synctl.pipeline.step.completed
synctl.pipeline.step.failed
synctl.approval.requested
synctl.approval.decided
synctl.p4.submitted
```

Dashboard and future integrations (Slack, email) subscribe without polling Postgres.

### 3.10 Config & Secrets

```
~/.synctl/
├── config.yaml          # Viper: server URL, output format, default pipeline
├── contexts.yaml        # named P4 contexts
└── credentials          # P4 ticket/password (chmod 600) — or use P4 trust + SSO
```

Env prefix: `SYNCTL_` (mirrors `AZURE_` / `AZURE_CLI_` pattern).

### 3.11 Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | General failure (build failed, approval rejected) |
| 2 | Usage/parse error |
| 3 | Resource not found (unknown run-id) |
| 4 | Pipeline still running (for commands that require completion) |

### 3.12 What NOT to Copy from Azure CLI

- **100+ SDK dependencies** — synctl integrates P4 + Jenkins, not 80 cloud APIs
- **Python** — Go is better for single-binary distribution to dev machines
- **Massive command index** — start with ~20 commands; optimize later
- **MSAL auth** — use your org's SSO + P4 tickets or service accounts

---

## 4. Go Library Choices (Refined)

| Concern | Library | Notes |
|---------|---------|-------|
| CLI | `spf13/cobra` | De facto standard; kubectl uses it |
| Config | `spf13/viper` | Files + env + flags |
| HTTP server | `gin-gonic/gin` | Dashboard + webhooks |
| gRPC | `google.golang.org/grpc` + `protobuf` | CLI ↔ server |
| Postgres | `jackc/pgx/v5` or `sqlc` | Type-safe queries with sqlc |
| Migrations | `golang-migrate/migrate` | Schema versioning |
| Kafka | `segmentio/kafka-go` | Simpler than confluent-kafka-go for start |
| P4 | `golang-p4` or subprocess `p4 -G` | Evaluate; many teams wrap `p4` CLI |
| Jenkins | `gojenkins` or raw REST | Trigger + poll |
| Logging | `log/slog` (stdlib) | Structured logs |
| Testing | `testify` + integration tests | Mock Jenkins/P4 in CI |

---

## 5. Phased Delivery

### Phase 1 — MVP (CLI + minimal server)
- Cobra CLI: `context`, `shelve list`, `run start`, `run describe`, `logs`
- gRPC server + Postgres (runs, steps)
- P4 shelve integration
- Jenkins: trigger one build job, webhook callback

### Phase 2 — Full pipeline
- Unit test + regression stages
- Approval gate (`synctl review approve`)
- `synctl run watch`, Kafka events

### Phase 3 — Dashboard + polish
- Web UI (history, approvals, coverage charts)
- Merge tool integration
- `synctl submit` with pre-checks
- Plugins (`synctl-*`)

### Phase 4 — Enterprise
- RBAC, multi-tenant P4 servers
- Voice review / meeting integration hook
- Declarative pipelines (`pipeline.yaml`)

---

## 6. Azure CLI → synctl Cheat Sheet

| Azure CLI concept | synctl equivalent |
|-------------------|-------------------|
| `az` root | `synctl` root |
| Knack | Cobra + small `internal/cli` core |
| Command module | `internal/cli/<group>/` package |
| Extension | `synctl-*` plugin binary |
| `~/.azure/az.json` | `~/.synctl/config.yaml` |
| `azureProfile.json` | `contexts.yaml` + credentials |
| `MainCommandsLoader` | Cobra command tree registration |
| Command index | Not needed initially; lazy load plugins later |
| ARM / SDK call | gRPC call to synctl server |
| Scenario tests | Go integration tests + mocked Jenkins |
| Telemetry | Kafka + Postgres audit (your choice) |
| `az configure` | `synctl config` |
| JMESPath `--query` | `jq` friendly JSON + optional `--query` via go-jmespath |

---

## 7. Open Questions / Decisions

1. **P4 integration:** native API library vs wrap `p4` CLI? CLI wrap is faster to ship; API is better for server deployment.
2. **Merge tool:** P4V merge, custom web merge UI, or delegate to existing code review tool?
3. **Voice review:** manual approval step with recording link, or integration with Teams/Zoom API?
4. **Single binary vs CLI+server:** developers need CLI; server is always remote — ship CLI via brew/internal apt; server as container.
5. **Offline/airgap:** cache Jenkins results? Azure CLI has `doc/use_cli_in_airgapped_clouds.md` — similar config for internal-only Jenkins/P4.

---

## 8. References (Azure CLI repo)

| Topic | Path |
|-------|------|
| Entry point | `src/azure-cli/azure/cli/__main__.py` |
| Core CLI class | `src/azure-cli-core/azure/cli/core/__init__.py` |
| Command loading | `MainCommandsLoader.load_command_table` |
| Authoring modules | `doc/authoring_command_modules/README.md` |
| Command guidelines | `doc/command_guidelines.md` |
| Extensions | `doc/extensions/README.md` |
| Error handling | `doc/error_handling_guidelines.md` |
| Telemetry | `doc/telemetry/README.md` |
| AAZ layer | `src/azure-cli-core/azure/cli/core/aaz/` |
| Knack (external) | https://github.com/Microsoft/knack |

---

*Generated: 2026-06-29 — for synctl planning*
