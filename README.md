# Ghidra Processor Module — Motorola/Freescale MC68HC16 (CPU16)

A [Ghidra](https://ghidra-sre.org/) processor (SLEIGH) module for the **Motorola/Freescale
MC68HC16** family (CPU16 core). It provides disassembly, p-code, and decompiler support,
including the CPU16's 20-bit banked addressing, MAC/DSP instructions, and the full
PSHM/PULM register set. The module was originally built to reverse-engineer the **Chrysler
JTEC+ PCM/TCM** firmware (MC68HC16Z2/Z3), and that profile is preserved — but the default is
now a clean, chip-generic profile you can use on **any** MC68HC16Z binary.

The instruction set is generic CPU16; the difference between the profiles is entirely in the
bundled processor/compiler spec (memory map, symbols, entry points, bank defaults, calling
convention). They layer from bare core to firmware preset:

| Language id | Profile | Use for |
|---|---|---|
| `68HC16:BE:24:cpu16` | **Bare CPU16 core** — exception-vector names + register model only, no peripheral map | a non-Z2/Z3 MC68HC16, or a clean slate |
| `68HC16:BE:24:default` | **Generic MC68HC16Z2/Z3** — the core PLUS this chip's on-chip peripheral map, neutral bank defaults, no seeded entry points | any HC16Z binary |
| `68HC16:BE:24:JTEC` | **Chrysler JTEC+ PCM/TCM preset** — the Z2/Z3 map PLUS firmware entry points, memory blocks, `ZK=0x0F` default, and the IZ-preserving calling convention | JTEC+ PCM/TCM firmware |

All three are big-endian, 24-bit (1 MB address space), and share the same compiled `68HC16.sla`.

- **Target parts:** MC68HC16Z2 / MC68HC16Z3 (the `.pspec` ships the Z2/Z3 peripheral map)
- **Status:** in active use on real firmware; see *Profiles & limitations* below.

---

## Installation

This module installs as a **drop-in processor**, no build step required (a precompiled
`68HC16.sla` is included). Take the `…-dropin.zip` from
[Releases](https://github.com/HausRummel/Ghidra-Processor-MC68HC16Z/releases), or use a
repository checkout directly:

1. Copy the `MC68HC16Z3/` folder into your Ghidra installation under
   `<GhidraInstallDir>/Ghidra/Processors/`.
2. Restart Ghidra.
3. Import your binary. When prompted for the language, choose **68HC16** and pick the variant:
   `68HC16:BE:24:default` for a generic HC16Z binary, `68HC16:BE:24:JTEC` for Chrysler JTEC+
   firmware, or `68HC16:BE:24:cpu16` for a non-Z2/Z3 part or a clean slate. Set the base address
   appropriately for your dump, and proceed with disassembly.

Releases also carry a `…-ghidra-<version>.zip` for *File → Install Extensions*. Ghidra checks an
extension's version against its own and warns on a mismatch, so pick the asset matching your
install; the drop-in zip skips that check.

If you edit `68HC16.slaspec`, Ghidra recompiles `68HC16.sla` automatically on next load (or run
`<GhidraInstallDir>/support/sleigh 68HC16.slaspec` yourself).

**Existing JTEC projects:** a program already imported under `68HC16:BE:24:default` keeps the
spec it was created with — its analysis is unaffected. To adopt the JTEC preset explicitly, use
*Set Language* and choose `68HC16:BE:24:JTEC`.

---

## CPU16 model at a glance

- **20-bit banked addressing.** The CPU16 forms 20-bit (1 MB) addresses from a 16-bit value plus
  a 4-bit bank/extension register (`EK`, `XK`, `YK`, `ZK`, `SK`, `PK`). Addresses are computed to
  the full 20 bits so Ghidra builds correct cross-references across banks.
- **Static bank resolution via context.** `EXT16` operands read the `ctxEK` context variable (set
  in the `.pspec`) at disassembly time, so extended operands resolve to real addresses. This
  assumes EK is effectively constant in the target firmware. Z-indexed modes use the `ZK` register
  (since v1.1.0-fix.1): give the decompiler its value with *Set Register Values* at function
  entries, or a `tracked_set` in the `.pspec` as the JTEC preset does; `ctxZK` is kept as a context
  field but no longer used by the semantics.
- **Pipeline-relative branches.** Branch targets are computed as `inst_start + 6 + offset` to match
  the CPU16's 3-word instruction prefetch.
- **Registers** are modeled with their natural sub-register relationships (A/B as halves of D, E:D
  for 32-bit values, composite `SSP` = `SK:SP` for the 20-bit stack and `ZKIZ` = `ZK:IZ` for the
  Z index register, so a TSZ frame pointer reads as the stack) and the CCR flags as
  bit-ranges (`N`, `Z`, `V`, `C`, `H`, `S`, `MV`, `EV`, plus the MAC `SM` and interrupt-priority
  field).

---

## Profiles & limitations

The **instruction set, registers, and addressing modes are generic CPU16** and apply to any
MC68HC16 (shared `.slaspec`/`.sla`). The three profiles differ only in the environment they set up,
layering from bare core outward:

| Concern | `cpu16` (bare core) | `default` (Z2/Z3) | `JTEC` preset | Files |
|---|---|---|---|---|
| Exception-vector names | ✅ CPU16 table | ✅ + Z2/Z3 `VEC_PICR` | ✅ same | `.pspec` `default_symbols` |
| Peripheral (MMIO) symbol map | ✖ none | ✅ Z2/Z3 map (from the manual) | ✅ same map | `.pspec` `default_symbols` |
| I/O register memory blocks | ✖ none | ✅ on-chip blocks | ✅ + `EXT_PERIPH`, `INT_SRAM` | `.pspec` `default_memory_blocks` |
| Bank context defaults | `ctxEK=0`, `ctxZK=0` | `ctxEK=0`, `ctxZK=0` (reset) | `ctxEK=0`, `ctxZK=15` (JTEC startup) | `.pspec` `<context_data>` |
| Seeded entry points | none | none | `RESET_ENTRY`/`STARTUP`/`IRQ_*` | `.pspec` `default_symbols` |
| Default calling convention | `__asmcall` (IZ clobbered) | `__asmcall` (IZ clobbered) | `__pcm_default` (IZ preserved) | `.cspec` `<default_proto>` |

So both `cpu16` and `default` **carry no firmware assumptions** — no spurious entry points, no
external memory blocks, neutral bank defaults. Use `cpu16` for a part whose peripheral layout is
*not* the Z2/Z3 (or when you want to define the map yourself); use `default` for any HC16Z binary.
Either way, add your own entry points, memory map, and (if your firmware fixes a data bank at
startup) bank context via *Set Register Values*.

**Peripheral map caveat.** In the `default`/`JTEC` profiles the MMIO symbol map and register blocks
are chip-generic to the MC68HC16Z2/Z3 (SIM/QSM/GPT/ADC/MRM/SRAM), taken from the MC68HC16ZUM
manual — verify against your exact part. The `cpu16` profile omits them entirely.

**Calling convention.** MC68HC16 firmware of this era is typically **hand-written assembly** with no
formal ABI, so both conventions are *empirical* best-effort models — reasonable starting points for
the decompiler, not guarantees. The generic `__asmcall` passes the first arg in `D`/`E:D`, returns
in `D`/`E:D`, and preserves `IX` plus the sticky bank registers; it treats `IZ` as clobbered. The
JTEC `__pcm_default` is identical except it keeps `IZ` callee-saved (that firmware dedicates
`IZ=0x8000` as a global data pointer). Both specs also offer `__pcm_default`, `__table_1d`, and
`__table_2d` as selectable per-function conventions — reassign via right-click → *Edit Function
Signature* where a routine clearly deviates.

**Modeling notes**: the control instructions `BGND`, `WAI`, and `LPSTOP` lift to distinct opaque
pseudo-ops (`background_debug()`, `wait_for_interrupt()`, `low_power_stop()`) — accurate for
disassembly and control flow, though fine-grained side effects (e.g. WAI's context stacking) are
intentionally not modeled. Immediate/offset operands are pinned to hexadecimal display.

---

## What's included

```
MC68HC16Z3/
  data/languages/      SLEIGH spec: .slaspec, .sinc, .ldefs, compiled .sla
                       68HC16_cpu16.pspec                    (bare CPU16 core)
                       68HC16.pspec / 68HC16.cspec           (generic Z2/Z3)
                       68HC16_JTEC.pspec / 68HC16_JTEC.cspec (JTEC+ preset)
  data/GhidraScripts/  helper scripts (see below)
  data/Manual/         CPU16RM.pdf and MC68HC16ZUM.pdf reference manuals
tests/                 disassembly regression harness (see tests/README.md)
```

**Helper scripts** (`MC68HC16Z3/data/GhidraScripts/`, run via Ghidra's Script Manager):

- **`DisassembleBanks.java`** — disassembles code regions up to the `0xFF` padding boundary, to
  bootstrap analysis of a banked dump. Prompts for the bank ranges and FF-run threshold (defaults
  to the common `0x10000`/`0x20000`/`0x30000` layout).
- **`ApplyRegisterBitEquates.java`** + **`MC68HC16Z3_regs.h`** — apply symbolic bit-mask equates to
  MMIO register accesses (e.g. `SIMCR`, `SYNCR`) for readable peripheral code. Chip-generic to the
  Z2/Z3, so usable on any HC16Z binary.

---

## Changes in v1.1.0-fix.1 and .2

Semantic (p-code) fixes made while decompiling firmware for a CPU16 part, each
checked against the CPU16 Reference Manual (CPU16RM/AD, in `MC68HC16Z3/data/Manual/`). Only the
PSHM/PULM register-list text changes the disassembly (masks with the reserved bit 7 now read
`...,RSVD7`); everything else is p-code, so re-analyze programs (and re-import them, since the
register layout of IX/IY/IZ and XK/YK/ZK changed) to pick it up.

| Fix | Was | Now | CPU16RM |
|---|---|---|---|
| PSHM/PULM order | both register lists reversed | PSHM stacks D first (highest address), PULM pulls CCR first; the generator `68HC16_PSHM_PULM.sinc.py` writes the committed `.sinc` again and has `--check` | PSHM p. 6-173, PULM p. 6-177 |
| JMP/JSR `IND20` | added XK/YK/ZK to the base | `$0:index + 20-bit offset`, extension field not used | JMP p. 6-116, JSR p. 6-117, 4.3.3 |
| Stack pointer | pushes pre-decremented, so every push/pull was a word off | SK:SP points at the next free word: store then decrement, increment then load; PSHA/PSHB/PULA/PULB move SP by one byte | PSHA to PULM, pp. 6-171 to 6-177 |
| JSR/BSR/LBSR/RTS/RTI/SWI | RTS returned to itself; RTI ignored the PC offset | RTS pulls PK and PC and returns to PK:PC - 2; RTI to PK:PC - 6; SWI stacks its frame and clears PK. JSR/BSR/LBSR still only move SP by the 4-byte frame: storing PC and CCR (return + 2, and CCR with PK) is what the CPU does, but the stored PC lands on the word the caller's SP points at after the return, and the decompiler then keeps the stores as caller stack variables, so an emulator must push that frame itself | JSR p. 6-117, BSR p. 6-70, LBSR p. 6-134, RTS p. 6-194, RTI p. 6-193, SWI p. 6-212 |
| Calling conventions | `extrapop=0` (the caller's SP drifted by 4 at each call), stack arguments at SP+4 | `extrapop=4`, stack arguments at entry SP+6, return address declared at stack offset 2 | (follows from the frame above) |
| `,Z` addressing | `(ctxZK << 16) \| (IZ + offset)` from a context value | `ZK:IZ + offset`, a 20-bit sum using the new composite register `ZKIZ`; TSZ/TZS move the whole 20-bit value, so frame-pointer locals decompile as stack variables | 4.3.3, 4.3.5, Table 3-1 |
| ADCx/SBCx flags | C and V computed on `M + C`, which wraps when M is all ones and C is set | flags of the whole `A + M + C` / `A - M - C`; ADCA/ADCB H includes the carry | ADCA p. 6-9, Table 6-35 |
| ROR, RORA/B/D/E, RORW | V from the old sign bit | V = N xor C after the rotate | Table 6-35 |
| `,X`/`,Y` addressing (.2) | `(XK << 16) \| (IX + offset)`, within the bank | `XK:IX + offset` as a 20-bit sum with composite registers `XKIX`/`YKIY`; TSX/TSY/TXS/TYS and TXY..TZY move the 20-bit value as one | 4.3.3, 4.3.5, Table 3-1, TSX p. 6-241, TXS p. 6-245, TXY p. 6-247 |
| AIX/AIY/AIZ, ADX..ADZ, AEX..AEZ, ABX..ABZ, MOVB/MOVW post-increment (.2) | added to the 16-bit register only | add to the 20-bit XK:IX (YK:IY, ZK:IZ), carrying into the extension field | 3.3.2.5, AIX p. 6-25, ADX p. 6-19, AEX p. 6-21, ABX p. 6-5, Table 3-1 |
| PSHM K (.2) | packed the byte-wide extension registers unmasked | each field masked to 4 bits, so PULM K restores exactly what PSHM K saved | PSHM p. 6-173, 3.3.2.4 |

The JTEC preset gets a `tracked_set` of ZK = 0xF so its `,Z` accesses still resolve.

Known gaps, not changed here: JSR/BSR/LBSR don't store their frame (storing it makes the
decompiler keep the stores and show the caller's stack pointer after each call as a pointer into
its frame; changing stackshift, returnaddress or localrange, or locking the callee's prototype,
didn't help); PSHMAC/PULMAC frame order is unverified; EDIV/EDIVS/IDIV/FDIV branch inside the
instruction on a zero divisor.

---

## Tests

A lightweight disassembly regression harness lives in [`tests/`](tests/). It checks that known
instruction byte sequences disassemble to the expected text, using your local Ghidra install. See
[`tests/README.md`](tests/README.md) for how to run it and add vectors.

---

## Related projects

- [**M68HC16_Assembler**](https://github.com/HausRummel/M68HC16_Assembler): a Rust
  reimplementation of the Motorola MASM 4.6 assembler for the M68HC16 (CPU16), with a CLI
  and GUI, producing `.OBJ`, `.S19`, `.LST` and raw `.bin` output validated against the
  original DOS toolchain. The two projects share a history. Disassemble here, reassemble
  there when you want to patch or rebuild what you have recovered.

---

## License

Licensed under the **Apache License, Version 2.0** — see [`LICENSE`](LICENSE).

The bundled PDFs under `MC68HC16Z3/data/Manual/` are Motorola/Freescale reference manuals included
for convenience and remain the property of their respective copyright holders.
