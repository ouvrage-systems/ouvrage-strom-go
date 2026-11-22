# Meeting Notes: 2026-07-25 — Ostrom: Declarative Stream Architecture, Pratt Integration, and Embedded Testing

## Metadata
* **Date**: July 25, 2026
* **Participants**: `@gpineda` (Lead Architect & Systems Engineer), `@Antigravity` (AI Coding Assistant)
* **Status**: Decided & Documented
* **Location / Repository**: `github.com/ouvrage-systems/ouvrage-strom-go`

---

## 1. Context & Architectural Boundary

During this session, we formalized the boundaries between **Ostrom** (`ouvrage-strom-go`) and **Ocalque** (`ouvrage-calque-go`), and established the specifications for the Ostrom stateful execution runtime.

### The Boundary:
* **Ostrom** acts as the generic infrastructure and transport runtime. It manages the physical container format (`.ostrom`), the pipeline execution lifecycle, the in-memory Virtual File System (VFS), and concurrency scheduling.
* **Ocalque** maps its compiler/rendering passes (Extract, Parse, Compile, Harvest, Link, Render) onto Ostrom's temporal phases, registering custom Kinds (`Calque`, `CalqueInterface`, `SymbolTable`) and domain-specific transforms (such as AST-level node replacements).

---

## 2. Pratt Parser & Expression Evaluator Integration

To support conditional execution guards within declarative pipeline transforms (e.g., `when: "env.TARGET == 'prod' && status.ready"`), Ostrom requires expression evaluation capabilities.
* **Decision**: Rather than spawning a third repository (`okernel`) immediately, the custom Pratt parser engine and expression evaluator will reside inside Ostrom under `pkg/lib/syntax/pratt/`.
* **Imports**: Ocalque will import this package directly: `github.com/ouvrage-systems/ouvrage-strom-go/pkg/lib/syntax/pratt`. This keeps the initial release architecture limited to two repositories while maintaining clean, unidirectional dependencies.

---

## 3. High-Performance Caching, VFS Lazy Loading & DAG GC

To handle large projects and maintain strict memory boundaries, the Ostrom runtime implements a two-pass, lazy-loaded memory model:

### 3.1 Two-Pass Execution
* **Pass 1: Static Indexing & Linking**: Scans only the envelope headers of the stream to index byte offsets and sizes. It resolves the `dependsOn` declarations to build the execution Directed Acyclic Graph (DAG) and validates the integrity of bindings.
* **Pass 2: JIT Execution**: Runs the handlers according to the resolved DAG.

### 3.2 Concurrent Goroutine Processing
* Once the DAG and index map are resolved, Ostrom parallelizes the processing of independent DAG branches.
* Worker Goroutines load resource payloads concurrently using thread-safe, non-blocking **`os.File.ReadAt`** calls directly on the `.ostrom` file.

### 3.3 Lazy Loading & Access Profiles
Resource specs declare their access profile to optimize resource usage:
* **`file` (VFS Cache)**: Payload is decompressed and loaded in the in-memory VFS only when requested by a handler.
* **`stream` (Passthrough)**: Payload is streamed directly to disk or a destination pipe using `io.Copy` with a small buffer, bypassing RAM caching entirely.

### 3.4 DAG-Based Garbage Collection
When a worker finishes executing a resource phase, Ostrom updates the DAG. If an asset in the in-memory VFS is no longer required by any remaining queue items, it is immediately deleted from the VFS map to free RAM.

---

## 4. Self-Contained & Self-Testing Release Bundles

Ostrom introduces a unified delivery format for code, configs, and testing:

```text
========================================================================
                 ANATOMY OF A SELF-TESTING RELEASE BUNDLE
========================================================================
  - kind: S3Bucket           (Infrastructure provision specs)
  - kind: ESIngestPipeline   (Database/Search pipeline configs)
  - kind: AppDeployment      (Application container configs)
  - kind: IntegrationTest    (Embedded integration test script/binary)
========================================================================
```

* **Codebase Embedding**: Complete workspaces (Kotlin code, Gradle scripts, Makefiles) are packed as `kind: Asset` resources inside the `.oclqs` (Ocalque-specialized `.ostrom` container).
* **CLI Pack & Unpack**: The Ostrom CLI supports `pack` (compressing files into `kind: Asset`) and `unpack/dump` (fully reconstructing original directory trees JIT on disk).
* **Self-Testing Deployments**: The `IntegrationTest` resource executes JIT on the target server. It queries the Keystore (e.g. to read generated bucket names or database credentials), runs integration tests against the live deployment, and reports back. If tests fail, Ostrom triggers an automatic rollback of the state.

---

## 5. The 3-Mode Integration Matrix

To support various integration paths across development, operations, and polyglot teams, Ostrom defines three concrete execution modes:

| Mode | Architecture | Point of Entry | Key Advantage |
| :--- | :--- | :--- | :--- |
| **1. Embedded Library** | Ocalque imports Ostrom core directly via standard Go module dependency. | `ocalque` | **Native DevEx**: A single, clean binary dedicated to templating, hiding Ostrom complexity from the end-user. |
| **2. Compile-Time Plugins (`xostrom`)** | Ostrom is the main program; Ocalque is compiled directly into it as a static plugin JIT. | `ostrom` (Specialized) | **Orchestrated Runtime**: The general-purpose `ostrom` CLI natively parses and renders `apiVersion: ocalque.ouvrage.io/v1` resources. |
| **3. gRPC Dynamic Plugins** | Ostrom communicates with external plugins over UNIX sockets via gRPC. | `ostrom` (Generic) | **Polyglot Extensibilité**: Allows teams to write custom handlers in Python, Rust, or Bash without rebuilding the engine. |

---

## 6. Concrete Enterprise Scenario: RPM Specfile Orchestration

To bridge the gap between traditional OS package managers (such as RPM/Yum/DNF) and dynamic configuration systems, an operations team can leverage **Mode 2** (`xostrom`) for secure offline deployments:

### 6.1 Preparation
1. The Ops team compiles a customized deployment binary `ops-ostrom` using `xostrom`:
   ```bash
   xostrom build \
     --with github.com/ouvrage-systems/ouvrage-calque-go/pkg/plugin \
     --with github.com/my-org/ops-custom-plugin
   ```
2. The binary is pre-installed on target servers at `/opt/ops-ostrom/bin/ops-ostrom`.

### 6.2 Packaging & Deployment Workflow
1. The entire project workspace (Kotlin source files, Makefiles, configurations, and templates) is packed into a single `app-release.ostrom` file.
2. This `app-release.ostrom` file is packaged inside a standard system **RPM** package (placed at `/etc/my-app/app-release.ostrom` on install).
3. In the RPM **specfile** (post-install trigger section), the Ops team writes the execution hooks:
   ```spec
   %post
   # JIT unpack, render configs based on local server context, and execute the build/tests
   /opt/ops-ostrom/bin/ops-ostrom render /etc/my-app/app-release.ostrom --context=/etc/sysconfig/server-env
   /opt/ops-ostrom/bin/ops-ostrom unpack /etc/my-app/app-release.ostrom -o /var/lib/my-app/
   make -C /var/lib/my-app/ build
   ```
* **Result**: The operating system's native package manager manages the package audit trail and signature, while the custom `ops-ostrom` binary handles the JIT unpacking, contextual rendering, and local compilation with absolute determinism and zero runtime dependencies.

---
*Meeting recorded by `@Antigravity` in partnership with `@gpineda` (CPGE MPSI/MP, INSA CVL Promo 2020).*
