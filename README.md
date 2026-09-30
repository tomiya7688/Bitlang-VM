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

## Current status

Initial architecture and Assam Core compatibility are being specified before the execution engine is expanded.
