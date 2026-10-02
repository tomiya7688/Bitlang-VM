# Bitlang VM

Bitlang VM is the Go implementation of the Bitlang virtual machine.

## Position in the Bitlang toolchain

The normal VM-oriented path is:

```text
Bitlang
    -> Bitlang Explicit
    -> Bitlang Low
    -> Bitlang VM Backend
       [optional internal TreeObject IR]
    -> Bitlang VM Assembly (Assam Core profile)
    -> Bitlang VM
                     -> x64 translator
                     -> ARM64 translator
                     -> RISC-V translator
```

Bitlang VM Assembly is not a separate high-level language. Its textual syntax is a strict low-level profile/subset of [Assam](https://github.com/tomiya7688/Assam), referred to as **Assam Core**.

The VM is expected to consume output generated from Bitlang Low. Human-authored Assam Core is also useful for tests, debugging, and low-level programs, but machine generation from Bitlang Low is a primary design constraint.

## Implementation language

The Bitlang VM implementation is written in **Go**.

## Design boundaries

- **Assam** owns the shared assembly syntax, Core profile, instruction definitions, and reference semantics.
- **Bitlang VM Backend** lowers validated Bitlang Low into Assam Core / Bitlang VM Assembly.
- **TreeObject**, when used, is an optional private/internal IR or serialization format inside the Bitlang VM Backend. It is not a required public pipeline language, VM input format, or architecture-translator input contract.
- A backend implementation may lower Low directly to Assam Core without materializing TreeObject.
- If TreeObject is materialized, it may contain resolved CFG, symbols, constants, target layout, helper calls, and other backend-ready data, but it must not require the VM or architecture translators to reconstruct Bitlang ownership/class/borrow semantics.
- **Bitlang VM** loads and executes the resulting low-level program.
- The VM must not reconstruct Bitlang source semantics that should already have been lowered by the backend.
- The VM must not duplicate a drifting private copy of the Assam grammar.
- Full Assam may provide higher-level conveniences, pseudo-instructions, generators, or tooling; features outside Assam Core are not accepted as VM input unless lowered to Core first.

## Architecture translation requirement

Bitlang VM Assembly must be directly translatable to at least these architecture families:

- x64 / x86-64
- ARM64 / AArch64
- RISC-V

Architecture translation is a required acceptance condition of the Bitlang VM design, not an optional future extension.

The architecture translators must be driven primarily by **JSON mapping tables** that associate Assam Core / Bitlang VM Assembly operations with target instruction sequences and operand rules.

A Core instruction does not need to map to exactly one native instruction. One Core operation may expand to multiple target instructions. However, the mapping must remain mechanical and explicit.

Target-specific code should be limited to genuinely target-specific encoding, register/calling-convention adaptation, relocation, and other unavoidable backend mechanics. Semantic meaning must not be hidden inside large hand-written per-architecture lowering branches when the operation can instead be expressed as a combination of simpler Core instructions.


## Future bytecode translation targets

After the required native architecture path is established, Bitlang VM Assembly should also support later-phase translation to:

- WebAssembly (Wasm)
- JVM bytecode

These are planned official targets, but they are **not** part of the initial x64 / ARM64 / RISC-V acceptance gate.

Because Wasm and JVM bytecode use stack-oriented execution models and impose structural constraints that differ from native ISAs, they do not have to be implemented as pure one-step JSON instruction substitution.

The expected path is:

```text
Bitlang VM Assembly / Assam Core
    -> target normalization / stack lowering
    -> Wasm or JVM bytecode
```

Simple opcode correspondence may reuse the Assam JSON mapping system, while CFG restructuring, stack scheduling, verifier/type adaptation, bytecode container emission, and runtime-memory adaptation may use dedicated target passes.

The target passes must consume only already-lowered Assam Core semantics. They must not reconstruct Bitlang Low ownership, borrow, class, closure, or other high-level language concepts.

Wasm/JVM support must not cause Assam Core to gain target-specific high-level instructions.

## Core simplicity requirement

Assam Core / Bitlang VM Assembly must consist of **very small, explicit, RISC-like operations**.

A proposed Core instruction is acceptable only when at least one of the following is true:

1. it can be represented mechanically for x64, ARM64, and RISC-V by the JSON mapping system; or
2. it is a target-independent VM/runtime primitive with a deliberately specified ABI boundary; or
3. it can be lowered into an explicit sequence of already accepted simpler Core instructions before architecture translation.

Complex convenience operations, language-level behavior, compound memory/control-flow operations, implicit allocation/cleanup, or instructions that require a target translator to rediscover high-level semantics do not belong in Core.

When an operation is difficult to map safely across the three required architectures, the preferred solution is to **decompose it into more primitive Core instructions**, not to make each architecture translator smarter.

Full Assam may retain pseudo-instructions or convenience instructions, but they must lower to Core before entering Bitlang VM or the architecture translators.

## Bitlang Low compatibility requirements

The VM target and backend must preserve Bitlang Low semantics, including where relevant:

- canonical arbitrary semantic bit widths, two's-complement signed representation, and deterministic div/mod behavior;
- checked/discard arithmetic and shift behavior;
- canonical Bool semantics without host-language truthiness;
- target-defined Address / Size / Offset widths;
- explicit runtime checks and trap/error paths;
- bounds-checked fixed/runtime-length array access where validity was not proven statically;
- runtime-length Array descriptors as guest pointer + guest Size;
- UTF-8 Str/Char semantics as guest bytes + guest Size rather than Go string layout;
- deterministic exact-layout records where used;
- deterministic evaluation/control-flow behavior;
- explicit ownership/release effects already lowered into executable operations;
- canonical alloc/try-alloc/free/trap runtime-helper semantics using guest allocation identity;
- validated Ptr/Ref guarantees without reintroducing invalid operations;
- a defined Bitlang VM target ABI, memory model, pointer width, alignment, and struct layout.

The VM target layout is a backend target in its own right. It must not silently inherit the host Go process layout or a C ABI.

## Acceptance criteria

The Bitlang VM instruction/profile design is not considered complete until:

- the same canonical Bitlang VM Assembly program can be validated and executed by the Go VM;
- architecture mapping JSON exists for x64, ARM64, and RISC-V;
- representative Core programs can be translated through those JSON mappings for all three required architecture families;
- Core instructions that cannot be mapped safely are decomposed or removed from Core;
- mapping/conformance tests prevent an instruction from being added to Core without required architecture coverage;
- target translation does not depend on reconstructing Bitlang Low or higher-level language semantics.


## Self-definition / self-hosted VM requirement

Bitlang VM must eventually be expressible by Bitlang VM Assembly itself.

The VM implementation may be authored directly in Assam Core / Bitlang VM Assembly, or it may be authored in Bitlang and lowered through the ordinary Bitlang -> Bitlang Low -> Bitlang VM Assembly pipeline. The requirement is about the resulting executable definition: the VM must not require hidden Go-only semantics that cannot be represented through the documented Core instruction set and runtime ABI.

The intended bootstrap model is:

```text
Go reference/bootstrap VM
    -> executes canonical VM implementation in Bitlang VM Assembly

Bitlang source implementation of VM (optional authoring form)
    -> Bitlang Explicit
    -> Bitlang Low
    -> Bitlang VM Assembly
    -> canonical self-hosted VM program
```

Once the canonical VM implementation exists as Core:

```text
canonical Bitlang VM Assembly implementation of Bitlang VM
    -> Go reference VM                (VM-on-VM)
    -> x64 translation                (native Bitlang VM)
    -> ARM64 translation              (native Bitlang VM)
    -> RISC-V translation             (native Bitlang VM)
    -> later Wasm translation         (Wasm-hosted Bitlang VM)
    -> later JVM bytecode translation (JVM-hosted Bitlang VM)
    -> developer mapping              (custom CPU / FPGA-hosted Bitlang VM)
```

This makes the Go implementation a bootstrap/reference implementation rather than the permanent semantic definition of the VM.

### Consequences

- The same canonical VM implementation can be transported to every target supported by Assam translation.
- The VM can execute another instance of itself, enabling VM-on-VM and recursive conformance tests.
- Native translators can compile the VM implementation itself without a target-specific VM rewrite.
- Custom CPU / FPGA developers can potentially obtain a Bitlang VM for their target by supplying the Assam mapping required for that target.
- The reference Go VM and the self-hosted VM can be differential-tested against the same conformance corpus.
- VM semantics remain independent from Go object layout, Go integer behavior, or other host implementation details.

### Non-cheating rule

Self-definition is not satisfied by moving essential VM semantics into opaque host callbacks.

A small, explicitly specified runtime/host ABI is allowed for unavoidable environment interaction such as process I/O, host memory reservation, clocks, or platform services. Core execution semantics, guest memory semantics, arithmetic/trap behavior, instruction dispatch, and other VM-defined behavior must remain implementable by the self-hosted program.

### Self-hosting acceptance

The self-hosted VM milestone is reached when:

- one canonical VM implementation can be produced as valid Bitlang VM Assembly;
- the Go reference VM can execute that implementation;
- that self-hosted VM can execute the shared Core conformance programs;
- its observable results match the Go reference VM;
- the same canonical VM implementation can pass through the required x64 / ARM64 / RISC-V translation pipeline without source-level VM rewrites.

## Current status

Initial architecture and Assam Core compatibility are being specified before the execution engine is expanded.
