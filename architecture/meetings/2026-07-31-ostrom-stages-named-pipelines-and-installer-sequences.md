# Meeting Notes: 2026-07-31 — Ostrom: Stage Execution, Named Pipelines, and Installer Sequence Abstractions

## Metadata
* **Date**: July 31, 2026
* **Participants**: `@gpineda` (Lead Architect & Systems Engineer), `@Antigravity` (AI Coding Assistant)
* **Status**: Decided & Documented
* **Location / Repository**: `github.com/ouvrage-systems/ouvrage-strom-go`

---

## 1. Context & Core Architectural Separation

In this session, we clarified the orchestration model of Ostrom, defining the boundaries between global scheduling (Ostrom) and local business state transitions (the domain Handler).

* **Macro-Scheduling (Ostrom Stages)**: Ostrom orchestrates the global sequence of a stream run ($S_i \to S_{i+1}$) using a fixed sequence of standard execution hooks (Stages), similar to Systemd targets or GitLab CI stages.
* **Micro-Orchestration (Handler Phases)**: The domain-specific handler (e.g., Ocalque) manages its own internal lifecycle phases (like `parse`, `compile`, `render` for `kind: Calque`). Ostrom remains completely agnostic to these domain terms.

---

## 2. Named Pipelines with Control Flow & Dynamic Context

To avoid code duplication and keep spec declarations DRY:
* **The Structure**: The user declares named pipelines containing lists of transforms within the resource spec (`spec.pipelines.<name>`).
* **Control Flow Blocks**: Pipelines support `if/then/else` and `for` control blocks evaluated JIT by Ostrom's Pratt evaluator.
* **Dynamic Context Injection (`extend_ctx`)**: The handler manages its internal state transitions. When it triggers a pipeline (e.g. `ctx.ExecutePipeline(res, "onCompile")`), it injects dynamic runtime variables JIT (e.g., `ctx.Set("phase", "ingest")`).
* **Execution**: Ostrom evaluates the pipeline instructions, executing only the transforms that satisfy the `when` condition guards against the extended context.

---

## 3. WiX/MSI Installation Sequence Analogy & Rollback Checkpoints

We established a conceptual analogy between Ostrom’s stage execution and historical installer sequences (such as Windows Installer MSI/WiX Standard and Custom Actions):
* **Standard Actions**: Ostrom defines standard execution checkpoints (`Ingest`, `Validate`, `Link`, `Mutate`, `Execute`, `Flush`).
* **Custom Actions**: Resource handlers register themselves to execute relative to these standard checkpoints. A handler can dynamically:
  * Skip its execution JIT if context conditions are not met.
  * Request VFS assets or publish events.
* **Atomic Rollbacks**: Similar to MSI transactional installers, this structured sequence checklist allows Ostrom to orchestrate safe, deterministic, in-memory rollbacks if a task or an `IntegrationTest` fails at a specific execution checkpoint.

---

## 4. The Shared Algorithmic Core (`okernel`)

To avoid duplicating core compiler and systems logic between Ostrom and Ocalque, we confirmed the design of a shared repository: **`okernel`** (or `ouvrage-core-go`):
* **Contents**: Zero-dependency, first-principles implementations of fundamental computer science blocks (Pratt parser engine, expression evaluator, basic AST structures, VFS interface, Event Bus).
* **CLI Utility**: Exposes educational CLI tools allowing engineers to learn and visualize compiler and parser designs.
* **Integration**: Imported by both Ostrom and Ocalque, maintaining a clean, CGO-free, and WASM-compatible dependency tree.

---
*Meeting recorded by `@Antigravity` in partnership with `@gpineda` (CPGE MPSI/MP, INSA CVL Promo 2020).*
