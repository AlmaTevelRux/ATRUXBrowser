# Project Briefing & Architectural Guidelines (v2)

This document serves as a comprehensive briefing guide and state-retrieval context for any subsequent AI assistant helping you build your custom web browser and interface solutions. It defines the foundational roadmap, design patterns, and engineering strategies established for our custom web browser project and desktop environment configurations.

---

## 1. Browser Architecture Strategy

### Engine Selection: Pure Interpreters (`QuickJS` / `Boa`)
We have bypassed heavyweight JIT engines (like V8 or SpiderMonkey) in favor of a lean, secure, and resource-efficient runtime footprint.

*   **Selected Candidates:** 
    *   **QuickJS:** A mature, ultra-lightweight C engine with near-instant instantiation (<300μs) and exceptional memory efficiency (<1MB RAM).
    *   **Boa:** A modern, memory-safe ECMAScript engine written entirely in Rust, moving toward future JIT capabilities.
*   **The Non-JIT Paradigm:** The architecture strictly relies on **pure interpretation** (compiling JS to bytecode and executing via a virtual machine loop) rather than JIT compilation to native machine code.
    *   *Benefits:* Complete immunization against JIT-based memory-corruption security vulnerabilities, zero warm-up latencies, and minimal memory overhead.
    *   *Tradeoffs:* Heavy computational operations (physics loops, massive arrays, canvas rendering) and dense SPA frameworks will run slower compared to JIT-driven environments.

---

## 2. Dual-Engine Architecture & Abstract Engine Wrappers

To maximize compatibility, security, and developer control, the browser implements a **Dual-Engine Strategy**, housing both QuickJS and Boa simultaneously. This allows runtime selection of the engine based on execution contexts, compatibility requirements, or tab-level isolation boundaries.

### The Abstract Engine Interface
To keep the browser UI shell clean and decoupled from low-level engine nuances, a unified **Abstract Engine Wrapper** sits between the shell and the execution layer. The browser core interacts exclusively with this abstract interface.


```

```
   [ Browser Shell UI / Navigation ]
                   │
                   ▼
      [ Abstract Engine Wrapper ]
       (Unified eval, bind, call)
                   │
    ┌──────────────┴──────────────┐
    ▼                             ▼

```

[ QuickJS ]                    [ Boa ]
(C / WASM Binding)          (Native Rust)

```

#### Reference Rust Interface Blueprint
```rust
// A unified abstract wrapper implementation template
pub trait BrowserJSEngine {
    fn create_context(&mut self) -> Result<(), EngineError>;
    fn eval_string(&mut self, code: &str) -> Result<JSValue, EngineError>;
    fn register_native_fn(&mut self, name: &str, callback: NativeCallback) -> Result<(), EngineError>;
    fn collect_garbage(&mut self);
}

```

### Language Integration Blueprints

#### Scenario A: The Browser Shell is in Rust

* **Boa Integration:** Directly embedded via the native `boa_engine` crate.
* **QuickJS Integration:** Embedded via a safe wrapper layer (such as the `rquickjs` crate) handling the underlying C compilation out of the box.
* **Wrapper Logic:** Driven by an `enum EngineType { QuickJS, Boa }` dynamically routing calls through the trait interface.

#### Scenario B: The Browser Shell is in C/C++

* **QuickJS Integration:** Compiled directly as modular C sources (`quickjs.c`, `quickjs-libc.c`).
* **Boa Integration:** Compiled as a static library target (`.a` or `.lib`) using Rust's `cdylib` configurations, exposing C-compatible bindings (`extern "C"`) to be linked by the main host toolchain.

### Core Architectural Use Cases

1. **Fault-Tolerant Execution Fallback:** If a script triggers a processing error or hits an un-implemented syntax node in the developing `Boa` interpreter, the wrapper catches the exception and transparently hot-swaps execution to the spec-complete `QuickJS` instance.
2. **Isolated Sandboxing:** Risky or untrusted background workloads route directly through `Boa` to benefit from Rust's safety guarantees, while everyday UI rendering relies on the battle-tested runtime speed of `QuickJS`.
3. **Comparative Profiling:** DevTools can house an instant toggle to hot-swap engine environments for real-time profiling of memory usage, garbage collection latency, and bytecode parsing performance.

---

## 3. Browser Interface & UX Logic (Vivaldi Case Studies)

When building user workflows or analyzing UX paradigms, we leverage advanced interface mechanics modeled after Vivaldi’s layout control engine:

### Tab Tiling & Screen Splitting

* **Mechanics:** Native support for multi-tab rendering within a single window viewport without extensions. Layouts must adapt to horizontal, vertical, or grid matrices.
* **State Management:** Viewports must allow granular resizing via draggable boundary lines, preserving layout states inside session structures.

### Component Customization Boundaries

* **Systemic Components:** Core browser panels (e.g., Bookmarks, History, Downloads) feature hardcoded system schemas. They support visual asset overrides (SVG icons) but prevent system-level renaming.
* **Dynamic Components (Web Panels):** Custom web wrappers injected into the UI side-bar. These components possess editable headers, allowing explicit overrides of titles and dimensions directly via user interaction or global configuration screens.

---

## 4. Network & Request Manipulation (Gmail Mobile Override)

When dealing with application layouts or sandboxed environments that aggressively enforce desktop redirects, use structural overrides:

* **URL Path Injection:** Target direct endpoint wrappers that bypass front-end responsive routers (e.g., Google's `https://mail.google.com/mail/mu/mp/`).
* **User-Agent Spoofing:** When handling layout routing, manipulate the client headers to mimic touch-optimized ecosystems (iOS Safari / Android Chrome) to force low-overhead mobile layouts natively.

---

## 5. JS Engine Reference Matrix (Performance & Contexts)

When optimizing server-side utilities, tools, or building comparison models, refer to this performance baseline:

* **Raw Core Speed:** Apple's **JavaScriptCore (JSC)** provides the fastest raw startup times and low memory footprints among JIT engines.
* **Tooling & Serverless:** **Bun** (built on JSC and Zig) dominates script execution, cold-starts, and local package management speed.
* **Throughput:** **Deno** (built on V8 and Rust) provides peak HTTP single-core JSON request throughput.

---

## Instructions for Next AI Assistant

> When assisting with this project, prioritize **memory safety, code scannability, and structural minimalism**. Avoid suggesting heavy frameworks or unnecessary layers unless explicitly requested. Keep code architectures modular so that switching the JS engine binding layer between C (`QuickJS`) and Rust (`Boa`) remains frictionless through the wrapper layer.
