# SYNCHRONIX EMBEDDED v2.0.0 — Self-Hosted Bare-Metal Native Programming Language

**Synchronix Embedded v2.0.0** is the **bare-metal / microcontroller (MCU) parity edition** of Synchronix v2, implemented in **100% pure `.synchronix` source files** (zero C, C++, Rust, Python, Go, PowerShell, or Makefile source files) with full 1-to-1 architectural parity across all 5 compiler, runtime, and ecosystem domains.

Designed for resource-constrained embedded systems (ARM Cortex-M4/M7, bare-metal ARM64, freestanding x86_64) with **zero dynamic heap churn**, **compile-time `noalloc` & ISR safety enforcement**, **volatile MMIO / hardware barrier / interrupt intrinsics**, **$\Phi$ (Phi) static formal verification**, **deterministic static-slab ARC + bounded-pause Tracing GC**, and **host-side HAL simulation** for rapid development and CI testing.

## Strict 1-to-1 Embedded Architecture Mapping (V2 Parity $\rightarrow$ Embedded v2)

| Domain | Subsystem | Embedded Capabilities | Pure Synchronix Source |
|---|---|---|---|
| **Portable & HAL** | ABI, MCU Memory Map & Hardware Constants | Cortex-M4 Thumb-2 / ARM64 / x86_64-bare ABIs, UART0/GPIOA/SysTick/NVIC MMIO defs | `include/portable_asm.synchronix`, `include/sx_defs.synchronix` |
| **Build System** | Self-Hosted Build & 3-Arch Cross-Verifier | Compiles all 22 units + cross-verifies Cortex-M4, ARM64-bare & x86_64-bare + `.a` RT archives | `build.synchronix` |
| **Phase 1: Frontend** | Lexer & Context Utilities | Hex literals (`0x4002_0000`), `_` separators, embedded hardware keywords & SRAM/Flash budget ctx | `src/frontend/lexer.synchronix`, `src/frontend/utils.synchronix` |
| **Phase 1: Frontend** | Parser (AST) | `irq`, `noalloc`, `bare` fn qualifiers, `Reg32` type, `mmio_read`, `mmio_write`, `irq_disable`/`enable`, `wfi`, `dmb` | `src/frontend/parser.synchronix` |
| **Phase 1: Frontend** | Semantic Analyzer & Embedded Verifier | Full V2 types/generics/Phi/Lambda + **`noalloc`/`irq` allocation ban**, **ISR `await` ban**, **Phi MMIO alignment & null-addr prover** | `src/frontend/sema.synchronix` |
| **Phase 2: Middle-End** | IR Generator $\rightarrow$ Embedded SUMIR | Volatile hardware SUMIR opcodes (`IR_MMIO_READ/WRITE`, `IR_IRQ_*`, `IR_WFI`, `IR_DMB`) | `src/middle/ir_gen.synchronix` |
| **Phase 2: Middle-End** | Memory Safety & SRAM Budgeter | Ownership/borrow checker, ARC/GC insertion + **static SRAM budget estimator (`--sram-kb`)** | `src/middle/mem_safety.synchronix` |
| **Phase 2: Middle-End** | $\Phi$ & $\Lambda$ Optimizer | 0-Flash-byte elimination of statically proven Phi proofs + ARC peephole elision | `src/middle/phi_lambda_opt.synchronix` |
| **Phase 2: Middle-End** | Volatile-Safe Lowering | SSA-safe constant folding, ISR-aware inlining, volatile-preserving fixed-point DCE | `src/middle/lowering.synchronix` |
| **Phase 3: Backend** | Target Optimization | Linear-scan regalloc (`r4-r8` Cortex-M4, `x19-x23` ARM64, `rbx/r12-r15` x86_64) & volatile-safe scheduler | `src/backend/target_opt.synchronix` |
| **Phase 3: Backend** | 5-Target Bare-Metal Codegen | **ARM Cortex-M4 Thumb-2** (`.isr_vector` + `Reset_Handler`), **ARM64-bare**, **x86_64-bare**, **x86_64 host sim**, **Freestanding LLVM IR** | `src/backend/codegen.synchronix` |
| **Phase 3: Backend** | Bare-Metal Assembler & ELF Linker | `clang` cross-assembly + `ld.lld --gc-sections` bare-metal `.elf` linking + `llvm-size` | `src/backend/emitter.synchronix` |
| **Foundation: Runtime** | Static-Pool Unified Memory | **64-slot static ARC slab**, **64-slot bounded-pause GC slab**, **4096 B bump arena** (zero runtime heap churn!) | `src/runtime/memory_rt.synchronix` |
| **Foundation: Runtime** | Deterministic RTOS Scheduler | Static 32-task priority-aware cooperative M:N fiber scheduler + tick counter | `src/runtime/async_rt.synchronix` |
| **Foundation: Runtime** | Freestanding $\Phi$ & Static Closures | Zero-OS Phi fault trap + 32-slot static closure pool | `src/runtime/phi_lambda_rt.synchronix` |
| **Foundation: Stdlib** | Embedded Standard Library (`std`) | V2 math/generics + **Q16.16 fixed-point DSP**, **bitfields/popcount**, **CRC-8**, **static ring buffer FIFO**, **host HAL sim** | `src/runtime/stdlib.synchronix` |
| **Ecosystem Tooling** | Firmware Package Manager & `size` | Scaffolds `Synchronix.toml` + Cortex-M4 `memory.ld` + starter firmware; `size` Flash/RAM analyzer | `src/tooling/cargo_pkg.synchronix` |
| **Ecosystem Tooling** | Formatter & Embedded Safety Linter | Canonical `fmt` + `[PHI-001]`, `[EMB-001]`, `[EMB-002]` embedded lint rules | `src/tooling/fmt_lint.synchronix` |
| **Ecosystem Tooling** | Synchronix Embedded LSP | JSON-RPC 2.0 server with bare-metal keyword & intrinsic completions | `src/tooling/lsp.synchronix` |
| **Ecosystem Tooling** | Debugger & Embedded Profiler | DWARF `-g` debugger + 3-phase profiler with MMIO/IRQ & static SRAM budget report | `src/tooling/debug_prof.synchronix` |
| **CLI Entry** | Unified `synchronix_emb` CLI Driver | Full V2 CLI + `--target cortex-m|arm64|x86_64-bare|x86_64|llvm`, `--bare`, `--noalloc`, `--flash-kb`, `--sram-kb`, `size` | `src/main.synchronix` |

## Quick Start & Bare-Metal Firmware Compilation

```sh
# 1. Self-host rebuild + 3-architecture cross-verification (Cortex-M4, ARM64-bare, x86_64-bare)
./build/sx_emb_build.exe

# 2. Run built-in embedded static-pool, RTOS & DSP self-test
./build/synchronix_emb.exe --self-test

# 3. Compile & link standalone bare-metal firmware ELF for ARM Cortex-M4 (Thumb-2)
./build/synchronix_emb.exe compile examples/bare_blink.synchronix --target cortex-m --bare --noalloc -o build/bare_blink_m4

# 4. Compile & link standalone bare-metal firmware ELF for ARM64 & x86_64-bare
./build/synchronix_emb.exe compile examples/bare_blink.synchronix --target arm64 --bare --noalloc -o build/bare_blink_a64
./build/synchronix_emb.exe compile examples/bare_blink.synchronix --target x86_64-bare --bare --noalloc -o build/bare_blink_x64

# 5. Simulate & profile embedded firmware natively on Windows host
./build/synchronix_emb.exe profile examples/embedded_firmware.synchronix
./build/synchronix_emb.exe run examples/showcase.synchronix

# 6. Verify static safety rejections (Ownership, Phi, noalloc ISR, unaligned MMIO)
./build/synchronix_emb.exe compile examples/move_fail.synchronix
./build/synchronix_emb.exe compile examples/phi_fail.synchronix
./build/synchronix_emb.exe compile examples/noalloc_fail.synchronix
./build/synchronix_emb.exe compile examples/mmio_fail.synchronix

# 7. Scaffold a new Cortex-M4 firmware project (with Synchronix.toml + memory.ld)
./build/synchronix_emb.exe new my_mcu_firmware
./build/synchronix_emb.exe size build/bare_blink_m4.elf
./build/synchronix_emb.exe lint examples/embedded_firmware.synchronix
```

## License

Synchronix Embedded is licensed under the [Apache License, Version 2.0](./LICENSE). End-user terms are provided in [EULA.md](./EULA.md).
