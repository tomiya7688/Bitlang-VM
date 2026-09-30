# Bitlang VM

Bitlang VM is the Go implementation of the Bitlang virtual machine.

## Position in the Bitlang toolchain

The normal VM-oriented path is:

```text
Bitlang
    -> Bitlang Explicit
    -> Bitlang Low
    -> Bitlang VM Backend
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

- canonical arbitrary semantic bit widths and signedness;
- checked arithmetic and deterministic shift behavior;
- explicit runtime checks and trap/error paths;
- bounds-checked memory access where validity was not proven statically;
- deterministic evaluation/control-flow behavior;
- explicit ownership/release effects already lowered into executable operations;
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

## Current status

Initial architecture and Assam Core compatibility are being specified before the execution engine is expanded.
