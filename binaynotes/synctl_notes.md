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

## 8. gRPC Proto Definitions (Draft)

Protos live under `api/proto/synctl/v1/`. Generate Go with:

```bash
buf generate   # or: protoc --go_out=. --go-grpc_out=. api/proto/synctl/v1/*.proto
```

### 8.1 File Layout

```
api/proto/synctl/v1/
├── common.proto          # shared enums + messages
├── context.proto         # P4 context service
├── shelve.proto          # shelve operations
├── pipeline.proto        # runs, steps, watch stream
├── review.proto          # approvals
├── merge.proto           # merge + submit
└── synctl.proto          # imports all; optional single entry
```

### 8.2 `common.proto`

```protobuf
syntax = "proto3";

package synctl.v1;

option go_package = "github.com/yourorg/synctl/api/gen/synctl/v1;synctlv1";

import "google/protobuf/timestamp.proto";

// Pipeline stage identifiers — maps to Postgres pipeline_steps.step_type
enum StepType {
  STEP_TYPE_UNSPECIFIED = 0;
  STEP_TYPE_SHELVE      = 1;
  STEP_TYPE_BUILD       = 2;
  STEP_TYPE_UNITTEST    = 3;
  STEP_TYPE_REGRESSION  = 4;
  STEP_TYPE_COVERAGE    = 5;
  STEP_TYPE_APPROVAL    = 6;
  STEP_TYPE_REVIEW      = 7;
  STEP_TYPE_MERGE       = 8;
  STEP_TYPE_SUBMIT      = 9;
}

enum StepStatus {
  STEP_STATUS_UNSPECIFIED = 0;
  STEP_STATUS_PENDING     = 1;
  STEP_STATUS_RUNNING     = 2;
  STEP_STATUS_SUCCEEDED   = 3;
  STEP_STATUS_FAILED      = 4;
  STEP_STATUS_CANCELLED   = 5;
  STEP_STATUS_WAITING     = 6;  // blocked on human gate
}

enum RunStatus {
  RUN_STATUS_UNSPECIFIED = 0;
  RUN_STATUS_QUEUED      = 1;
  RUN_STATUS_RUNNING     = 2;
  RUN_STATUS_SUCCEEDED   = 3;
  RUN_STATUS_FAILED      = 4;
  RUN_STATUS_CANCELLED   = 5;
}

message PageRequest {
  int32 page_size  = 1;  // default 50, max 200
  string page_token = 2;
}

message PageResponse {
  string next_page_token = 1;
}

message UserRef {
  string id    = 1;  // SSO / LDAP id
  string name  = 2;
  string email = 3;
}

message P4Ref {
  string server   = 1;  // p4:1666
  string user     = 2;
  string client   = 3;  // workspace
  string stream   = 4;  // //depot/main
  string changelist = 5;
}

message ArtifactRef {
  string name = 1;  // junit.xml, coverage.lcov
  string url  = 2;
  int64  size_bytes = 3;
}
```

### 8.3 `context.proto`

```protobuf
syntax = "proto3";

package synctl.v1;

option go_package = "github.com/yourorg/synctl/api/gen/synctl/v1;synctlv1";

import "synctl/v1/common.proto";

message Context {
  string name = 1;           // "default", "fpga-main"
  P4Ref  p4   = 2;
  bool   is_current = 3;
  google.protobuf.Timestamp updated_at = 4;
}

message ListContextsRequest {}
message ListContextsResponse {
  repeated Context contexts = 1;
  string current = 2;  // context name
}

message GetContextRequest {
  string name = 1;
}

message SetContextRequest {
  string name = 1;
  P4Ref  p4   = 2;
  bool   set_current = 3;
}

message UseContextRequest {
  string name = 1;
}

message DeleteContextRequest {
  string name = 1;
}

service ContextService {
  rpc ListContexts(ListContextsRequest) returns (ListContextsResponse);
  rpc GetContext(GetContextRequest) returns (Context);
  rpc SetContext(SetContextRequest) returns (Context);
  rpc UseContext(UseContextRequest) returns (Context);
  rpc DeleteContext(DeleteContextRequest) returns (Context);
}
```

### 8.4 `shelve.proto`

```protobuf
syntax = "proto3";

package synctl.v1;

option go_package = "github.com/yourorg/synctl/api/gen/synctl/v1;synctlv1";

import "synctl/v1/common.proto";
import "google/protobuf/timestamp.proto";

message Shelve {
  string id = 1;              // synctl id (uuid)
  string shelve_cl = 2;       // P4 shelved changelist
  string description = 3;
  P4Ref  p4 = 4;
  UserRef owner = 5;
  repeated string files = 6;
  google.protobuf.Timestamp created_at = 7;
}

message CreateShelveRequest {
  string description = 1;
  repeated string paths = 2;  // optional; default: opened files in workspace
  string context = 3;         // named context; default current
}

message ListShelvesRequest {
  PageRequest page = 1;
  string owner = 2;   // filter
  string context = 3;
}

message ListShelvesResponse {
  repeated Shelve shelves = 1;
  PageResponse page = 2;
}

message GetShelveRequest {
  string id = 1;        // synctl id or shelve_cl
}

message DeleteShelveRequest {
  string id = 1;
}

service ShelveService {
  rpc CreateShelve(CreateShelveRequest) returns (Shelve);
  rpc ListShelves(ListShelvesRequest) returns (ListShelvesResponse);
  rpc GetShelve(GetShelveRequest) returns (Shelve);
  rpc DeleteShelve(DeleteShelveRequest) returns (Shelve);
}
```

### 8.5 `pipeline.proto` (core)

```protobuf
syntax = "proto3";

package synctl.v1;

option go_package = "github.com/yourorg/synctl/api/gen/synctl/v1;synctlv1";

import "synctl/v1/common.proto";
import "google/protobuf/timestamp.proto";

message PipelineStep {
  string id = 1;
  StepType type = 2;
  StepStatus status = 3;
  google.protobuf.Timestamp started_at = 4;
  google.protobuf.Timestamp finished_at = 5;
  string message = 6;           // human-readable status
  string jenkins_build_url = 7;
  int32  jenkins_build_number = 8;
  repeated ArtifactRef artifacts = 9;
  map<string, string> metadata = 10;  // coverage %, test counts, etc.
}

message PipelineRun {
  string id = 1;
  string pipeline_name = 2;     // "default", "nightly-regression"
  RunStatus status = 3;
  string shelve_id = 4;
  string shelve_cl = 5;
  P4Ref  p4 = 6;
  UserRef triggered_by = 7;
  repeated PipelineStep steps = 8;
  google.protobuf.Timestamp created_at = 9;
  google.protobuf.Timestamp updated_at = 10;
}

message StartRunRequest {
  string shelve_id = 1;         // synctl shelve id or shelve_cl
  string pipeline = 2;          // default: "default"
  string context = 3;
  map<string, string> params = 4;  // Jenkins params override
}

message ListRunsRequest {
  PageRequest page = 1;
  RunStatus status = 2;         // filter; UNSPECIFIED = all
  string owner = 3;
  string pipeline = 4;
}

message ListRunsResponse {
  repeated PipelineRun runs = 1;
  PageResponse page = 2;
}

message GetRunRequest {
  string id = 1;
}

message CancelRunRequest {
  string id = 1;
  string reason = 2;
}

// Server-streaming: emits on every step transition (like kubectl watch)
message WatchRunRequest {
  string id = 1;
}

message WatchRunEvent {
  PipelineRun run = 1;
  PipelineStep changed_step = 2;  // which step triggered this event
  string event_type = 3;          // "STEP_STARTED", "STEP_COMPLETED", "RUN_FAILED"
}

message GetLogsRequest {
  string run_id = 1;
  StepType step = 2;    // BUILD, UNITTEST, ...
  int32 tail_lines = 3; // 0 = full log
  int64 offset = 4;     // resume from byte offset
}

message GetLogsResponse {
  bytes chunk = 1;
  int64 next_offset = 2;
  bool done = 3;
}

// Bidirectional stream for `synctl logs -f`
message StreamLogsRequest {
  string run_id = 1;
  StepType step = 2;
  int32 tail_lines = 3;
}

message StreamLogsResponse {
  bytes chunk = 1;
  google.protobuf.Timestamp timestamp = 2;
}

message ListPipelinesRequest {}
message PipelineDefinition {
  string name = 1;
  string description = 2;
  repeated StepType stages = 3;
  bool is_default = 4;
}
message ListPipelinesResponse {
  repeated PipelineDefinition pipelines = 1;
}

service PipelineService {
  rpc StartRun(StartRunRequest) returns (PipelineRun);
  rpc ListRuns(ListRunsRequest) returns (ListRunsResponse);
  rpc GetRun(GetRunRequest) returns (PipelineRun);
  rpc CancelRun(CancelRunRequest) returns (PipelineRun);
  rpc WatchRun(WatchRunRequest) returns (stream WatchRunEvent);
  rpc GetLogs(GetLogsRequest) returns (stream GetLogsResponse);
  rpc StreamLogs(stream StreamLogsRequest) returns (stream StreamLogsResponse);
  rpc ListPipelines(ListPipelinesRequest) returns (ListPipelinesResponse);
}
```

### 8.6 `review.proto`

```protobuf
syntax = "proto3";

package synctl.v1;

option go_package = "github.com/yourorg/synctl/api/gen/synctl/v1;synctlv1";

import "synctl/v1/common.proto";
import "google/protobuf/timestamp.proto";

enum ReviewDecision {
  REVIEW_DECISION_UNSPECIFIED = 0;
  REVIEW_DECISION_APPROVE = 1;
  REVIEW_DECISION_REJECT  = 2;
}

message ReviewRequest {
  string id = 1;
  string run_id = 2;
  UserRef requester = 3;
  repeated UserRef required_reviewers = 4;
  repeated ReviewRecord decisions = 5;
  StepStatus status = 6;
  google.protobuf.Timestamp created_at = 7;
}

message ReviewRecord {
  UserRef reviewer = 1;
  ReviewDecision decision = 2;
  string comment = 3;
  google.protobuf.Timestamp decided_at = 4;
}

message ListPendingReviewsRequest {
  PageRequest page = 1;
  string reviewer = 2;  // filter: reviews assigned to me
}

message ListPendingReviewsResponse {
  repeated ReviewRequest reviews = 1;
  PageResponse page = 2;
}

message ApproveRunRequest {
  string run_id = 1;
  string comment = 2;
}

message RejectRunRequest {
  string run_id = 1;
  string comment = 2;
}

message GetReviewRequest {
  string run_id = 1;
}

service ReviewService {
  rpc ListPendingReviews(ListPendingReviewsRequest) returns (ListPendingReviewsResponse);
  rpc GetReview(GetReviewRequest) returns (ReviewRequest);
  rpc ApproveRun(ApproveRunRequest) returns (ReviewRequest);
  rpc RejectRun(RejectRunRequest) returns (ReviewRequest);
}
```

### 8.7 `merge.proto`

```protobuf
syntax = "proto3";

package synctl.v1;

option go_package = "github.com/yourorg/synctl/api/gen/synctl/v1;synctlv1";

import "synctl/v1/common.proto";
import "google/protobuf/timestamp.proto";

message MergeStatus {
  string run_id = 1;
  StepStatus status = 2;
  string target_stream = 3;
  int32 resolve_remaining = 4;
  string merge_tool_url = 5;   // web merge UI if applicable
  string message = 6;
}

message StartMergeRequest {
  string run_id = 1;
  string target_stream = 2;    // optional override
}

message GetMergeStatusRequest {
  string run_id = 1;
}

message SubmitRequest {
  string run_id = 1;
  string description = 2;      // optional submit description override
  bool dry_run = 3;
}

message SubmitResult {
  string run_id = 1;
  string submitted_cl = 2;
  StepStatus status = 3;
  string message = 4;
}

service MergeService {
  rpc StartMerge(StartMergeRequest) returns (MergeStatus);
  rpc GetMergeStatus(GetMergeStatusRequest) returns (MergeStatus);
  rpc Submit(SubmitRequest) returns (SubmitResult);
}
```

### 8.8 Gin REST Mirror (webhooks + dashboard)

gRPC is CLI-primary; Gin exposes a thin REST layer for browsers and Jenkins:

| REST | gRPC equivalent | Notes |
|------|-----------------|-------|
| `GET /api/v1/runs` | `PipelineService.ListRuns` | dashboard |
| `GET /api/v1/runs/:id` | `PipelineService.GetRun` | |
| `POST /api/v1/runs` | `PipelineService.StartRun` | |
| `POST /api/v1/runs/:id/approve` | `ReviewService.ApproveRun` | approval email links |
| `POST /hooks/jenkins` | internal handler | Jenkins callback payload |
| `GET /api/v1/runs/:id/logs/:step` | `PipelineService.GetLogs` | SSE for browser tail |

Jenkins webhook body (Gin handler normalizes to internal event):

```json
{
  "job_name": "synctl-build",
  "build_number": 42,
  "status": "SUCCESS",
  "run_id": "550e8400-e29b-41d4-a716-446655440000",
  "step": "BUILD",
  "artifacts": [
    {"name": "junit.xml", "url": "https://jenkins/.../artifact/junit.xml"}
  ]
}
```

### 8.9 Error Model (gRPC status codes)

| gRPC code | When | CLI exit code |
|-----------|------|---------------|
| `OK` | success | 0 |
| `INVALID_ARGUMENT` | bad flags / missing shelve | 2 |
| `NOT_FOUND` | unknown run-id | 3 |
| `FAILED_PRECONDITION` | submit before approval | 1 |
| `ABORTED` | run cancelled | 1 |
| `UNAVAILABLE` | server down | 1 |
| `DEADLINE_EXCEEDED` | `watch` timeout | 4 |

Use `google.rpc.Status` details for structured errors:

```protobuf
// optional: api/proto/synctl/v1/errors.proto
message SynctlError {
  string code = 1;       // "PIPELINE_BLOCKED", "P4_AUTH_FAILED"
  string message = 2;
  string run_id = 3;
  StepType failed_step = 4;
}
```

---

## 9. Initial Cobra Command Tree

### 9.1 Command Tree (target UX)

```
synctl
├── context
│   ├── list
│   ├── show
│   ├── set
│   ├── use <name>
│   └── delete <name>
├── shelve
│   ├── create
│   ├── list
│   ├── describe <id>
│   └── delete <id>
├── run
│   ├── start
│   ├── list
│   ├── describe <id>
│   ├── watch <id>
│   └── cancel <id>
├── logs <run-id> [--step build] [-f] [--tail 100]
├── review
│   ├── list
│   ├── describe <run-id>
│   ├── approve <run-id>
│   └── reject <run-id>
├── merge
│   ├── start <run-id>
│   └── status <run-id>
├── submit <run-id> [--dry-run]
├── coverage show <run-id>
├── pipeline list
├── config
│   ├── set <key> <value>
│   ├── get <key>
│   └── list
├── completion [bash|zsh|fish]
└── version
```

### 9.2 Package / File Layout

```
cmd/synctl/main.go

internal/cli/
├── root.go              # root cmd, persistent flags, Viper bootstrap
├── flags.go             # --output, --server, --context, --timeout
├── output.go            # json/table printers (stdout only)
├── client.go            # gRPC dial + auth metadata
├── context/
│   └── context.go
├── shelve/
│   └── shelve.go
├── run/
│   └── run.go
├── logs/
│   └── logs.go
├── review/
│   └── review.go
├── merge/
│   └── merge.go
├── submit/
│   └── submit.go
├── coverage/
│   └── coverage.go
├── pipeline/
│   └── pipeline.go
└── config/
    └── config.go
```

### 9.3 `cmd/synctl/main.go`

```go
package main

import (
    "os"

    "github.com/yourorg/synctl/internal/cli"
)

func main() {
    if err := cli.Execute(); err != nil {
        os.Exit(cli.ExitCode(err))
    }
}
```

### 9.4 `internal/cli/root.go`

```go
package cli

import (
    "fmt"
    "os"
    "time"

    "github.com/spf13/cobra"
    "github.com/spf13/viper"

    "github.com/yourorg/synctl/internal/cli/config"
    "github.com/yourorg/synctl/internal/cli/context"
    "github.com/yourorg/synctl/internal/cli/coverage"
    "github.com/yourorg/synctl/internal/cli/logs"
    "github.com/yourorg/synctl/internal/cli/merge"
    "github.com/yourorg/synctl/internal/cli/pipeline"
    "github.com/yourorg/synctl/internal/cli/review"
    "github.com/yourorg/synctl/internal/cli/run"
    "github.com/yourorg/synctl/internal/cli/shelve"
    "github.com/yourorg/synctl/internal/cli/submit"
)

var (
    outputFormat string
    serverAddr   string
    contextName  string
    timeout      time.Duration
)

var rootCmd = &cobra.Command{
    Use:   "synctl",
    Short: "Developer CI/CD CLI for P4 shelve-to-submit pipelines",
    PersistentPreRunE: func(cmd *cobra.Command, args []string) error {
        return initConfig()
    },
    SilenceUsage:  true,
    SilenceErrors: true,
}

func init() {
    cobra.OnInitialize(func() { _ = initViper() })

    rootCmd.PersistentFlags().StringVarP(&outputFormat, "output", "o", "json",
        "Output format: json|table")
    rootCmd.PersistentFlags().StringVar(&serverAddr, "server", "",
        "synctl API server (default from config)")
    rootCmd.PersistentFlags().StringVar(&contextName, "context", "",
        "P4 context name (default: current)")
    rootCmd.PersistentFlags().DurationVar(&timeout, "timeout", 0,
        "Command timeout (0 = no timeout)")

    _ = viper.BindPFlag("output", rootCmd.PersistentFlags().Lookup("output"))
    _ = viper.BindPFlag("server", rootCmd.PersistentFlags().Lookup("server"))

    rootCmd.AddCommand(
        context.NewCmd(),
        shelve.NewCmd(),
        run.NewCmd(),
        logs.NewCmd(),
        review.NewCmd(),
        merge.NewCmd(),
        submit.NewCmd(),
        coverage.NewCmd(),
        pipeline.NewCmd(),
        config.NewCmd(),
    )
}

func Execute() error {
    return rootCmd.Execute()
}

func initViper() error {
    viper.SetEnvPrefix("SYNCTL")
    viper.AutomaticEnv()
    viper.SetConfigName("config")
    viper.SetConfigType("yaml")
    viper.AddConfigPath("$HOME/.synctl")
    _ = viper.ReadInConfig() // missing file is OK
    return nil
}

func initConfig() error {
    if serverAddr == "" {
        serverAddr = viper.GetString("server.address")
    }
    if serverAddr == "" {
        serverAddr = "localhost:50051"
    }
    if contextName == "" {
        contextName = viper.GetString("context.current")
    }
    return nil
}

// ExitCode maps errors to kubectl/az-style exit codes.
func ExitCode(err error) int {
    if err == nil {
        return 0
    }
    if IsNotFound(err) {
        return 3
    }
    if IsUsageError(err) {
        return 2
    }
    if IsStillRunning(err) {
        return 4
    }
    fmt.Fprintln(os.Stderr, err.Error())
    return 1
}
```

### 9.5 Example: `internal/cli/run/run.go`

```go
package run

import (
    "context"
    "fmt"
    "io"
    "time"

    "github.com/spf13/cobra"
    synctlv1 "github.com/yourorg/synctl/api/gen/synctl/v1"
    "github.com/yourorg/synctl/internal/cli"
)

func NewCmd() *cobra.Command {
    cmd := &cobra.Command{
        Use:   "run",
        Short: "Manage pipeline runs",
    }
    cmd.AddCommand(newStartCmd(), newListCmd(), newDescribeCmd(), newWatchCmd(), newCancelCmd())
    return cmd
}

func newStartCmd() *cobra.Command {
    var pipeline string
    cmd := &cobra.Command{
        Use:   "start --shelve <id>",
        Short: "Start a pipeline run from a shelved changelist",
        RunE: func(cmd *cobra.Command, args []string) error {
            shelveID, _ := cmd.Flags().GetString("shelve")
            c, err := cli.NewClient(cmd.Context())
            if err != nil {
                return err
            }
            defer c.Close()

            run, err := c.Pipeline().StartRun(cmd.Context(), &synctlv1.StartRunRequest{
                ShelveId: shelveID,
                Pipeline: pipeline,
                Context:  cli.CurrentContext(),
            })
            if err != nil {
                return err
            }
            return cli.Print(cmd.OutOrStdout(), run)
        },
    }
    cmd.Flags().String("shelve", "", "Shelve ID or shelved changelist (required)")
    cmd.Flags().StringVar(&pipeline, "pipeline", "default", "Pipeline definition name")
    _ = cmd.MarkFlagRequired("shelve")
    return cmd
}

func newWatchCmd() *cobra.Command {
    var interval time.Duration
    cmd := &cobra.Command{
        Use:   "watch <run-id>",
        Short: "Watch a pipeline run until it completes",
        Args:  cobra.ExactArgs(1),
        RunE: func(cmd *cobra.Command, args []string) error {
            runID := args[0]
            c, err := cli.NewClient(cmd.Context())
            if err != nil {
                return err
            }
            defer c.Close()

            ctx := cmd.Context()
            stream, err := c.Pipeline().WatchRun(ctx, &synctlv1.WatchRunRequest{Id: runID})
            if err != nil {
                return err
            }

            for {
                ev, err := stream.Recv()
                if err == io.EOF {
                    break
                }
                if err != nil {
                    return err
                }
                if err := cli.PrintStatus(cmd.OutOrStdout(), ev); err != nil {
                    return err
                }
                if isTerminal(ev.Run.Status) {
                    if ev.Run.Status == synctlv1.RunStatus_RUN_STATUS_FAILED {
                        return fmt.Errorf("pipeline run %s failed", runID)
                    }
                    return nil
                }
            }
            return nil
        },
    }
    cmd.Flags().DurationVar(&interval, "interval", 2*time.Second, "Poll interval if server falls back to polling")
    return cmd
}

func isTerminal(s synctlv1.RunStatus) bool {
    switch s {
    case synctlv1.RunStatus_RUN_STATUS_SUCCEEDED,
        synctlv1.RunStatus_RUN_STATUS_FAILED,
        synctlv1.RunStatus_RUN_STATUS_CANCELLED:
        return true
    default:
        return false
    }
}
```

### 9.6 Example: `internal/cli/logs/logs.go`

```go
package logs

import (
    "context"
    "io"
    "os"

    "github.com/spf13/cobra"
    synctlv1 "github.com/yourorg/synctl/api/gen/synctl/v1"
    "github.com/yourorg/synctl/internal/cli"
)

func NewCmd() *cobra.Command {
    var (
        stepType  string
        follow    bool
        tailLines int32
    )
    cmd := &cobra.Command{
        Use:   "logs <run-id>",
        Short: "Print logs for a pipeline step",
        Args:  cobra.ExactArgs(1),
        RunE: func(cmd *cobra.Command, args []string) error {
            runID := args[0]
            step := parseStepType(stepType)

            c, err := cli.NewClient(cmd.Context())
            if err != nil {
                return err
            }
            defer c.Close()

            if follow {
                return streamLogs(cmd.Context(), c, runID, step, tailLines, os.Stdout)
            }
            return fetchLogs(cmd.Context(), c, runID, step, tailLines, os.Stdout)
        },
    }
    cmd.Flags().StringVar(&stepType, "step", "build", "Step: build|unittest|regression")
    cmd.Flags().BoolVarP(&follow, "follow", "f", false, "Stream logs")
    cmd.Flags().Int32Var(&tailLines, "tail", 0, "Lines of recent log to show")
    return cmd
}

func fetchLogs(ctx context.Context, c *cli.Client, runID string, step synctlv1.StepType, tail int32, w io.Writer) error {
    stream, err := c.Pipeline().GetLogs(ctx, &synctlv1.GetLogsRequest{
        RunId: runID, Step: step, TailLines: tail,
    })
    if err != nil {
        return err
    }
    for {
        resp, err := stream.Recv()
        if err == io.EOF {
            return nil
        }
        if err != nil {
            return err
        }
        if _, err := w.Write(resp.Chunk); err != nil {
            return err
        }
        if resp.Done {
            return nil
        }
    }
}
```

### 9.7 gRPC Client Wrapper: `internal/cli/client.go`

```go
package cli

import (
    "context"
    "time"

    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
    "google.golang.org/grpc/metadata"

    synctlv1 "github.com/yourorg/synctl/api/gen/synctl/v1"
)

type Client struct {
    conn *grpc.ClientConn
    ctx  synctlv1.ContextServiceClient
    sh   synctlv1.ShelveServiceClient
    pl   synctlv1.PipelineServiceClient
    rv   synctlv1.ReviewServiceClient
    mg   synctlv1.MergeServiceClient
}

func NewClient(ctx context.Context) (*Client, error) {
    addr := serverAddr // from root.go initConfig
    dialCtx, cancel := context.WithTimeout(ctx, 10*time.Second)
    defer cancel()

    conn, err := grpc.DialContext(dialCtx, addr,
        grpc.WithTransportCredentials(insecure.NewCredentials()), // TLS in prod
        grpc.WithUnaryInterceptor(authInterceptor),
    )
    if err != nil {
        return nil, err
    }
    return &Client{
        conn: conn,
        ctx:  synctlv1.NewContextServiceClient(conn),
        sh:   synctlv1.NewShelveServiceClient(conn),
        pl:   synctlv1.NewPipelineServiceClient(conn),
        rv:   synctlv1.NewReviewServiceClient(conn),
        mg:   synctlv1.NewMergeServiceClient(conn),
    }, nil
}

func (c *Client) Pipeline() synctlv1.PipelineServiceClient { return c.pl }
func (c *Client) Close() error                           { return c.conn.Close() }

func authInterceptor(ctx context.Context, method string, req, reply any,
    cc *grpc.ClientConn, invoker grpc.UnaryInvoker, opts ...grpc.CallOption) error {
    token := loadToken() // ~/.synctl/credentials or SSO
    if token != "" {
        ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+token)
    }
    if contextName != "" {
        ctx = metadata.AppendToOutgoingContext(ctx, "x-synctl-context", contextName)
    }
    return invoker(ctx, method, req, reply, cc, opts...)
}
```

### 9.8 Output Helper: `internal/cli/output.go`

```go
package cli

import (
    "fmt"
    "io"
    "tabwriter"

    "google.golang.org/protobuf/encoding/protojson"
    "google.golang.org/protobuf/proto"
)

func Print(w io.Writer, msg proto.Message) error {
    switch outputFormat {
    case "json":
        enc := protojson.MarshalOptions{Indent: "  ", EmitUnpopulated: false}
        b, err := enc.Marshal(msg)
        if err != nil {
            return err
        }
        _, err = w.Write(append(b, '\n'))
        return err
    case "table":
        return printTable(w, msg) // per-type formatters
    default:
        return fmt.Errorf("unknown output format %q", outputFormat)
    }
}
```

### 9.9 Registration Pattern (how to add a new command group)

Mirror Azure CLI's `COMMAND_LOADER_CLS` pattern in Go:

```go
// internal/cli/register.go
type CommandGroup interface {
    Name() string
    NewCmd() *cobra.Command
}

var groups []CommandGroup

func Register(g CommandGroup) {
    groups = append(groups, g)
}

// in root init(): for _, g := range groups { rootCmd.AddCommand(g.NewCmd()) }
```

Each package (`shelve`, `run`, …) implements `CommandGroup` and calls `Register` in `init()`. This keeps `root.go` thin as the CLI grows — same idea as az's per-module `COMMAND_LOADER_CLS`.

### 9.10 MVP Command Subset (Phase 1 only)

Ship these first; stub the rest with `synctl <cmd> — coming soon`:

| Command | gRPC RPC | Priority |
|---------|----------|----------|
| `synctl context list/show/set` | `ContextService.*` | P0 |
| `synctl shelve create/list` | `ShelveService.*` | P0 |
| `synctl run start` | `PipelineService.StartRun` | P0 |
| `synctl run describe` | `PipelineService.GetRun` | P0 |
| `synctl logs <id>` | `PipelineService.GetLogs` | P0 |
| `synctl run watch` | `PipelineService.WatchRun` | P1 |
| `synctl review approve` | `ReviewService.ApproveRun` | P1 |
| `synctl submit` | `MergeService.Submit` | P2 |

---

## 10. References (Azure CLI repo)

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

*Updated: 2026-06-29 — added gRPC protos (§8) and Cobra command tree (§9)*
