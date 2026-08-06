# ST-125N addressed service-cylinder artifacts

## Capture identity

Selector 02 says `SEAGATE ST125N`, and selector 03 embeds chassis serial `0CB365982`

## Extracted components

| Location | Artifact | SHA-256 |
| --- | --- | --- |
| C0/H0, selector `00` | `st125n-c0h0-selector00.bin` | `ca3070bcfa78fa32f9c6bfa824932fc90aa475d22ae7aef6c77bda545682260d` |
| C0/H1, selector `01` | `st125n-c0h1-selector01.bin` | `ca3070bcfa78fa32f9c6bfa824932fc90aa475d22ae7aef6c77bda545682260d` |
| C0/H2, selector `02` | `st125n-c0h2-selector02.bin` | `eed9babcac0f4e394ffbea26666c9f3fdbeb5cda80c6ac08c2bfc4e21ed57c16` |
| C0/H3, selector `03` | `st125n-c0h3-selector03.bin` | `501de1df87c2608d88102ad0e52da76dd844014d4a650c05b82d399290fda493` |

## Storage and disassembly model

Selectors 00/01 are the already mapped main program. Selectors 02/03 follow
the same layered layout as the ST-138N and ST-157N:

| File range | Selector 02 | Selector 03 |
| --- | --- | --- |
| `0000-0bff` | dense 8052 code/data, then zero padding | serial/audit storage and fill |
| `0c00-0fff` | `6c` fill | `6c` fill |
| `1000-1bff` | dense 8052 code/data, then zero padding | dense 8052 code/data, then zero padding |
| `1c00-1fff` | `6c` fill | `6c` fill |

`st125n-c0h2-selector02.composite.mapped.disasm51.asm` replaces baseline
`4000-4bff` and `5000-5bff`.
`st125n-c0h3-selector03.composite.mapped.disasm51.asm` retains baseline
`4000-4fff`, replaces `5000-5bff`, and leaves lower audit storage as data.

The listings use `disasm51` 1.0.1, `seagate_8052_symbols.inc`, and unresolved
`maskrom_` labels below `2000h`. Selector-02 traversal warned about unknown
SFR values `F5h` and `9Eh`; they remain unnamed because traversal may have
entered data.

## Selector-02 analysis

The lower module begins at `4000h`; the upper begins at `5000h`. Text includes
`SEAGATE ST125N` at file offset `109bh` and the 1985/1986 Seagate copyright at
`1ba0h`. The listing accesses the external buffer, AIC-301-compatible read and
write pointers, and AIC-010-compatible GPIO.

It reaches 28 distinct `80166-501` mask-ROM destinations. Nine occur in the
main listing; 19 additional targets show that this component exercises a
substantially different resident service set.

Only fill sectors 12-15 and 28-31 are identical across all three drives.
Active-code correspondence is:

| Comparison | Lower exact bytes in blocks ≥8 | Lower longest | Upper exact bytes in blocks ≥8 | Upper longest |
| --- | ---: | ---: | ---: | ---: |
| ST-125N / ST-138N | 1,031 | 96 | 320 | 96 |
| ST-125N / ST-157N | 1,119 | 96 | 790 | 77 |

The code is related but not interchangeable; every code-bearing sector differs.

## Selector-03 analysis

Lower storage begins with `0CB365982`. Its audit strings include:

| Offset | Text |
| ---: | --- |
| `0800h` | `IDT INTEGRATED OEM AUDIT TEST PROCESS, ST157N FAMILY P/N TC70047-001 REV1.43` |
| `0844h` | `SCSI TEST MODULES (IDT_4) P/N 71158-001 REV2.13` |
| `086fh` | `SCSI EXTENDED MOS TEST MODULES P/N 71393-001 REV1.58` |

The `ST157N FAMILY` wording on a component tied to this ST-125N by firmware
hash and chassis serial identifies a shared factory/test platform, not the
mechanism model.

The upper module accesses the buffer plus AIC-301-compatible host-data,
host-interface, DMA, and reset registers and AIC-010-compatible GPIO. Its
composite reaches 31 internal-ROM destinations; 30 occur in the main listing,
with `1cceh` newly exposed.

Against either later drive's selector-03 upper module it has 884 exact bytes
in 53 blocks of at least eight bytes, with a longest block of 74 bytes. It
shares storage sectors 6-7, 9-15, and 28-31 with each, principally common
audit-layout fill rather than executable sectors.

## Result

H0/H1 are redundant copies of the main overlay. H2 is a distinct two-part
service module. H3 combines drive/factory audit data with an upper controller
test module. See `st1xxn_service_overlay_comparison.md` for the three-drive
architecture.

See `st1xxn_overlay_behavior_and_diagnostics.md` for the identified Inquiry
response, Mode Sense/Select page dispatcher, controller-test behavior, and
possible factory engagement paths.
