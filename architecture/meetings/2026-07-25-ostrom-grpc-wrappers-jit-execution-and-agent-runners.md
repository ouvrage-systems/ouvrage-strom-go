# Meeting Notes: 2026-07-25 (Session 2) — Ostrom: gRPC CLI Wrappers, JIT Payload Execution, and Distributed Agent/Runner Architecture

## Metadata
* **Date**: July 25, 2026 (Afternoon Session)
* **Participants**: `@gpineda` (Lead Architect & Systems Engineer), `@Antigravity` (AI Coding Assistant)
* **Status**: Decided & Documented
* **Location / Repository**: `github.com/ouvrage-systems/ouvrage-strom-go`

---

## 1. Context & Objectives

In this afternoon session, we explored how to integrate legacy/imperative command-line utilities into the declarative Ostrom stream model without code rewrites, and designed a distributed runner architecture (`ostrom-master`/`ostrom-agent`) to support large-scale enterprise deployments.

---

## 2. gRPC CLI Wrappers (Declarative Wrapping of Legacy Tools)

To bring existing command-line utilities (e.g. `ddrescue`, `ffmpeg`, `rsync`) into the declarative stream paradigm, Ostrom leverages **gRPC CLI Wrappers**:
* **The Resource Spec**: The user defines the command parameters declaratively in the resource envelope:
  ```yaml
  apiVersion: ops.ouvrage.io/v1
  kind: DDRescueTask
  spec:
    sourceDevice: "/dev/sdb"
    destinationImage: "/var/backups/disk-sdb.img"
    mapFile: "/var/backups/disk-sdb.map"
  ```
* **The gRPC Plugin**: The plugin receives the envelope, translates the specification into a native CLI command (`ddrescue /dev/sdb ...`), monitors the command execution (parsing logfiles JIT), updates execution progress in the shared Keystore (e.g. `rescue.progress = "45%"`), and appends the final outputs back to the stream as payload resources.

---

## 3. Dynamic JIT Command Generation ($2k+1$ Payloads)

To avoid generating temporary scripts or files on disk during execution, we designed a **Closed-Loop Rendering and Execution pipeline**:
* **Dual Payloads**:
  * **Payload $2k$ (Gabarit / Template)**: The raw command template containing Ocalque annotations (e.g., `# @ocq:Replace(pattern="DEV", with=env.DISK_SOURCE)`).
  * **Payload $2k+1$ (Rendered JIT)**: The final, executable command string generated at runtime.
* **The Workflow**:
  1. **Mutate Phase**: Ostrom calls the Ocalque transform plugin. Ocalque evaluates Payload $2k$ using the context variables from the Keystore and writes the rendered executable command into Payload $2k+1$.
  2. **Evaluate Phase**: The gRPC executor (e.g. `ddrescue` or `bash` wrapper) reads the evaluated command from Payload $2k+1$ and runs it directly, keeping the entire lifecycle in memory.

---

## 4. Distributed Agent/Runner Architecture (`ostrom-master` / `ostrom-agent`)

To replace heavy, insecure CI/CD runners (like GitLab Runners or Ansible agents) in critical and Edge environments, Ostrom introduces a distributed topology:

```text
 ┌────────────────────────────────────────────────────────┐
 │                    OSTROM-MASTER                       │
 │  - Compiles, signs, and dispatches streams             │
 │  - Holds global secrets and aggregates audit logs       │
 └──────────────────────────┬─────────────────────────────┘
                            │ (gRPC Stream / WebSockets)
       ┌────────────────────┼────────────────────┐
       ▼                    ▼                    ▼
 ┌───────────┐        ┌───────────┐        ┌───────────┐
 │   AGENT   │        │   AGENT   │        │   AGENT   │  (Lightweight Go Runners)
 ├───────────┤        ├───────────┤        ├───────────┤
 │ Node: Web │        │ Node: DB  │        │ Node: IoT │
 └───────────┘        └───────────┘        └───────────┘
```

* **Control Plane (`ostrom-master`)**: Compiles, signs (using GPG keys), and dispatches the stream packages. It acts as the central observability console, receiving real-time audit updates.
* **Data Plane (`ostrom-agent` / Runner)**: A lightweight Go daemon (less than 15MB RAM) running on target nodes. It:
  1. Validates the package signature.
  2. Resolves the DAG locally.
  3. Uses **Lazy Loading** to only pull payloads in RAM when executing them.
  4. Streams execution logs and Keystore mutations back to the master in real-time.
* **Security & Sandboxing**: Unlike traditional runners that have full shell access and execute arbitrary scripts, the Ostrom agent is sandboxed: it only executes resources through registered, cryptographically verified plugins.

---

## 5. Ostrom as a Stream Micro-Kernel & Virtual Machine

The synergy between the Ostrom core, the in-memory VFS, and the static/dynamic plugin architecture manifests as a **Stream-Based Virtualization / Dataflow Micro-Kernel**:

### 5.1 The Micro-Kernel Analogy
* **The Core Kernel**: The Ostrom Core (`pkg/ostrom`) acts as a minimal micro-kernel. It is strictly limited to stream serialization, basic phase scheduling, and IPC (Inter-Process Communication).
* **User-Space Servers**: The handlers (Ocalque, Ansible, Terraform) run as decoupled user-space processes (via gRPC) or isolated statically linked plugins. They communicate and share state exclusively through Ostrom’s IPC memory channels: the **Keystore** (KV) and **Artifact Store**.
* **Memory Protection / Paging**: The `ResourceInterface` acts as memory segment protection, preventing unauthorized plugins from accessing variables or artifacts they did not explicitly declare in their contract.

### 5.2 The Virtual Machine Analogy
* The `.ostrom` or `.oclqs` package is the **compiled bytecode bundle**.
* The Ostrom Engine (`pkg/reactive`) is the **VM Interpreter**.
* The Keystore and Artifact Store represent the **virtual heap memory**.
* The plugins act as **virtual hardware controllers** interacting with the host system (disk, network, databases).

---

## 6. AI Integration & Model Context Protocol (MCP)

To position Ostrom at the forefront of autonomous software engineering, the engine natively implements the **Model Context Protocol (MCP)**:

### 6.1 Ostrom as an MCP Server
Ostrom exposes resources and tools directly to LLM-based autonomous agents:
* **MCP Resources**: The AI agent can read stream payloads (`ostrom://stream/{index}`), inspect virtual files (`ostrom://vfs/{path}`), or query active variables (`ostrom://keystore/{key}`).
* **MCP Tools**: The AI agent can validate streams (`ostrom_validate`), simulate rendering (`ostrom_render`), or inject dynamic fixes (`ostrom_inject`).

### 6.2 Autonomous Self-Healing Pipelines
By combining the embedded `IntegrationTest` runner with the MCP interface, Ostrom enables a closed-loop autonomous repair cycle:
1. **Diagnosis**: If the integration tests fail on-site, the AI agent reads the error reports via the MCP resource stream.
2. **Patching**: The agent JIT-injects corrected configuration values or AST code directly into the stream in-memory.
3. **Validation**: The agent runs `ostrom_validate` to ensure DAG integrity, and re-runs the tests. Once successful, the finalized, working stream is committed.
4. **Sandboxing the AI**: Because the Ostrom engine strictly enforces `ResourceInterface` ACLs, the autonomous AI agent is naturally sandboxed. It cannot access unauthorized host network ports or execute destructive commands unless explicitly permitted by the verified stream schema.

---

## 7. Cross-Platform Block Recovery: `odrescue` Go Library

To support secure data recovery and automated backups natively across platforms without external unix utility dependencies (such as GNU `ddrescue`), Ostrom introduces the **`odrescue` Go Library**:

### 7.1 Cross-Platform Direct Device Access
Go standard library (`os` and `sys`) wrappers are used to open raw block devices directly as read-only streams:
* **Linux / macOS**: Opens `/dev/sdb` or `/dev/disk1` directly.
* **Windows**: Opens raw physical drives using Win32 namespaces (e.g. `\\.\PhysicalDrive1`) bypassing OS caching via `sys/windows` API flags (`FILE_FLAG_NO_BUFFERING`).

### 7.2 Single-Threaded Resilient Copying Algorithm
* **Physical Constraint (Single-Threaded)**: To prevent physical read-head thrashing on failing mechanical drives (which accelerates hardware degradation), the copy loop runs strictly sequentially in a single thread. Goroutines are only used in background threads to push progress updates or write structured logs.
* **Skipping & Mapfile Tracking**: The copying loop reads block-by-block. Upon encountering an I/O read timeout/error, it skips the failing block, marks its offset status in a text-based Mapfile (e.g. `?` for untested, `-` for bad, `+` for rescued), and resumes copying further down. A second pass (trimming/scraping) isolates and attempts to recover bad sectors.
* **Resume Capability**: If interrupted, the copy process reads the Mapfile and resumes copying from the last uncopied offset.

### 7.3 Positioning & Use-Cases
`odrescue` is designed as a **Go Library first, and CLI second**. It serves as an API-driven component for automated forensic pipelines (SOAR), predictive hardware monitoring on Edge gateways, and unified backup agents.

---

## 8. Declarative Sandbox Isolation: FreeBSD Jails & Firecracker Abstraction

The `kind: Sandbox` template is designed to abstract low-level, host-specific virtualization and sandboxing runtimes:

```text
       [ Declarative Sandbox Specification (YAML) ]
                            │
              (Ostrom local agent translates JIT)
                            │
         ┌──────────────────┼──────────────────┐
         ▼ (Linux)          ▼ (FreeBSD)        ▼ (Cloud / AWS)
  [ Cgroups & NetNS ]    [ FreeBSD Jails ]   [ Firecracker MicroVMs ]
```

* **Device Whitelisting**: Sandboxes can restrict raw physical block device access dynamically:
  ```yaml
  apiVersion: ostrom.ouvrage.io/v1
  kind: Sandbox
  metadata:
    name: read-only-disk-sdb
  spec:
    devices:
      - path: "/dev/sdb"
        permissions: "r"
  ```
* **Host Translation**: The portable YAML declaration remains identical. At runtime, the local Ostrom agent translates it into the most secure isolation primitive available on the host OS: Linux device cgroups/namespaces, FreeBSD Jails (using `devfs.rules`), or an AWS Firecracker microVM.

---

## 9. Architectural Comparison: Ostrom vs. Apache Airflow

To clarify the positioning of Ostrom's execution model, we contrasted it with Apache Airflow:

| Metric | Apache Airflow | Ostrom |
| :--- | :--- | :--- |
| **Périmètre** | Orchestrates heavy big-data pipelines (ETL/AI). | Orchestrates systems deployments, Edge IoT, and configurations. |
| **DAG Definition** | Programmatic (written in Python code). | Declarative (declared in YAML envelopes). |
| **Data Transit** | External (triggers remote databases, no data flows through Airflow). | **Dataflow Container** (files, assets, and codebases flow *inside* the stream). |
| **Footprint** | Heavy (requires Python, Postgres, Redis/Celery). | Frugal (single Go binary, $O(1)$ memory, runs in WebAssembly). |
| **Security** | Accesses the entire host OS directly. | Sandboxed at the task-level via `ResourceInterface` and `ostrom-jail`. |

---

## 10. Graph Introspection & openCypher Support (Kùzu Integration)

To provide deep auditability and advanced dependency querying across complex enterprise streams, Ostrom supports representing and querying its DAG using the **openCypher** graph query language and **Kùzu** (an in-process embedded graph database):

### 10.1 Pragmatic CGO-Free Decoupled Design (Opt-In)
Kùzu is written in C++. Direct integration into Ostrom’s core (`pkg/ostrom`) would require **CGO**, which breaks static cross-compilation, slows compile times, adds binary footprint (15-20MB), and breaks WebAssembly (WASM) compatibility. 

To maintain strict frugality, the graph engine is decoupled as an **Opt-In** feature:
1. **Core DAG Solver (Default - Pure Go)**: The Ostrom engine uses a native Go implementation (lightweight structures and map-based adjacency lists) to resolve DAGs, check cycles, and validate links. This remains 100% CGO-free, lightweight, and WASM-compatible.
2. **Text-Based openCypher Exporter**: Ostrom can serialize the resolved DAG into a standard `.cypher` query script containing declarative SQL-like graph creations (e.g. `CREATE (t:Task)-[:USES]->(s:Sandbox)...`). This requires zero-dependencies and runs natively.
3. **Embedded Kùzu Engine (`.kuzu`)**: The compiled Kùzu database file is generated via:
   * An optional CLI exporter binary (`ostrom-kuzu` tool).
   * A dynamic gRPC service plugin (Mode 3).
   * A customized binary compiled JIT via `xostrom` (Mode 2) containing the CGO bindings.

### 10.2 Use-Cases for Graph Queries (Compliance Audits)
Exposing the execution DAG as a queryable graph database allows Security/Ops teams to run advanced compliance checks:
* **Security Boundary Auditing**: Querying for tasks that consume secrets but run in unsegmented sandboxes:
  ```cypher
  MATCH (t:Task)-[:CONSUMES]->(s:Secret {name: "db_password"}),
        (t)-[:RUNS_IN]->(sb:Sandbox {blockAllOther: false})
  RETURN t.name, sb.name
  ```
* **Impact Analysis**: Querying which downstream tasks depend on a modified asset:
  ```cypher
  MATCH (a:Asset {name: "auth-config.yaml"})-[:INFLUENCES*]->(t:Task)
  RETURN t.name
  ```

---

## 11. Mobile SDK Cross-Platform Bindings (`gomobile`)

Because the Ostrom core library (`pkg/ostrom` and `pkg/reactive`) is designed to be 100% Go-native and CGO-free, it can be compiled natively for mobile operating systems:

### 11.1 Native Framework Generation
* **Android**: Compiling to an Android Archive (`.aar`) containing JNI wrappers and Kotlin bindings using `gomobile bind -target=android`.
* **iOS**: Compiling to an Apple static Framework (`.framework`) containing Objective-C and Swift bindings using `gomobile bind -target=ios`.

### 11.2 Core Mobile Use-Cases
* **Offline Asset Sync**: Mobile apps download a single `.ostrom` package containing database updates, configurations, and media. The app JIT-parses the stream in RAM to update its local filesystem atomically, bypassing multiple HTTP requests.
* **Field Maintenance Diagnostics**: Technicians connect mobile devices to isolated edge equipment (via Wi-Fi/Bluetooth) to download diagnostic logs as `.ostrom` streams, parsing them JIT for localized graphing and reporting.

---

## 12. JIT Module Resolution & Self-Bootstrapping Deployments

Ostrom introduces a **Self-Bootstrapping, Single-File Deployment** paradigm:

### 12.1 Deploying via a Single File
A deployment package is distributed as a single `.ostrom` or `.oclqs` file containing all codebases, templates, configuration variables, and installation/validation scripts. The target host requires only the lightweight `ops-ostrom` agent.

### 12.2 JIT Plugin Auto-Resolution
During the **Pass 1 (Static Verification)** phase, Ostrom scans the stream for required custom resource types (e.g. `kind: S3Asset` or `kind: AnsibleTask`):
1. **Dependency Inventory**: Ostrom lists the required plugins and checks the host's local plugin directory (e.g. `/opt/ops-ostrom/plugins/`).
2. **JIT Auto-Installation**: If a plugin is missing, the agent automatically downloads it from a secure corporate registry, verifies its cryptographic signature, installs it JIT, and starts the DAG execution.
3. **Immutability Protection**: This prevents configuration drift and ensures that the server bootstraps itself to a clean, verified state with zero pre-installed runtime dependencies.

---
*Meeting recorded by `@Antigravity` in partnership with `@gpineda` (CPGE MPSI/MP, INSA CVL Promo 2020).*
