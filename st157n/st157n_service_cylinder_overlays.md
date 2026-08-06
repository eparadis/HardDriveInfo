# ST-157N selector-02 and selector-03 service-cylinder artifacts

## Reconstructed artifacts

An ST-157R control board on the ST-157N mechanism made cylinder 0 and each
physical head directly selectable. The two newly recovered 8 KiB storage
artifacts are preserved beside the disassembly listings:

| Service location | Artifact | SHA-256 |
| --- | --- | --- |
| C0/H2, selector `02` | `st157n-c0h2-selector02.bin` | `3dcfa56dc8832898d864ece11d6c011c4c65c256fefc94a7faec0c795cb4707e` |
| C0/H3, selector `03` | `st157n-c0h3-selector03.bin` | `5551a66b36e4b6a7c1409e3206788419e8366d997d3e5eb4d529d171ba18c971` |

Each artifact is the ordered concatenation of its CRC-valid 256-byte payloads
for sector identifiers 0 through 31.

See the notes for the ST-138N for further information.

## Storage layout and loading model

The 8 KiB files are disk-storage artifacts, not two simple replacements for
the selector-00/01 program. Their sector boundaries reveal several regions:

| File range | Selector 02 | Selector 03 |
| --- | --- | --- |
| `0000-0bff` (sectors 0-11) | dense 8052 code and data | audit/configuration data and identification strings through sector 8, then `6c` fill |
| `0c00-0fff` (sectors 12-15) | `6c` fill | `6c` fill |
| `1000-1bff` (sectors 16-27) | dense 8052 code and data | dense 8052 code and data |
| `1c00-1fff` (sectors 28-31) | `6c` fill | `6c` fill |

The code contains absolute destinations in the established external overlay
range `4000-5fff`. Selector-02's lower module begins as valid instructions at
`4000h`; its upper module begins at `5000h`. The upper modules also call
destinations in the selector-00/01 program's lower half. Treating the literal
`6c` storage fill as executable memory would make those destinations invalid.
The most consistent interpretation is layered loading: firmware loads selected
modules over the resident selector-00/01 image and leaves unselected regions
intact. Proving the exact loader sequence still requires observing or tracing
the internal-ROM loader.

The mapped listings therefore use an explicit analysis composite:

- `st157n-c0h2-selector02.composite.mapped.disasm51.asm` replaces baseline
  ranges `4000-4bff` and `5000-5bff` with selector-02 sectors 0-11 and 16-27;
- `st157n-c0h3-selector03.composite.mapped.disasm51.asm` retains the baseline
  `4000` region and replaces `5000-5bff` with selector-03 sectors 16-27;
- selector-03 sectors 0-15 remain documented as storage data and are not
  falsely presented as 8052 instructions.

## Selector-02 observations

Selector-02 begins immediately with coherent 8052 code. Its first routine
initializes internal state, uses external RAM around `5b60-5b6e`, and branches
among overlay and internal mask-ROM services. Later paths access the shared
AIC buffer window, the AIC-301-compatible read/write pointers, and AIC-010
GPIO. The upper module contains the inquiry-style string `SEAGATE ST157N` at
storage offset `109bh` and `(C) COPY RIGHT SEAGATE TECHNOLOGY 1985,1986` at
offset `1bc1h`.

The selector-02 composite reaches 37 distinct internal-ROM destinations in
the current recursive pass. Only 14 appear in the selector-00/01 listing; 23
additional destinations include `0313h`, `033eh`, `0480h`, `0c6ch`, `1885h`,
`19eeh`, and `1faeh`. Because the physical MCU is unchanged, this is not a
different mask-ROM ABI. It shows that this disk module invokes a much broader
and different subset of the same `80075-509` resident services.

Byte correspondence with the known main overlays is weak. Within the active
selector-02 modules, the longest exact block found against the ST-157N main
program is only 66 bytes in the lower module and 20 bytes in the upper module.
The selector is therefore not a small patch or relink of the already recovered
main overlay.

## Selector-03 observations

Selector-03's lower storage region identifies its purpose directly:

| Offset | Text |
| ---: | --- |
| `0000h` | `075750410` |
| `0510h` | `GCHX1` |
| `0800h` | `ST157N/GCROM107PRETEST/AUDITPROCESSP/N71395-001REV1.10` |
| `0838h` | `ST157NFAMILYOEMIDTPRETEST/AUDITSEQUENCEP/N71322-001REV1.00` |
| `0874h` | `ST157NDRIVECODESROM107(RAM134,EMOS134,MOS131)P/N71562-001Rev1.10` |
| `08b6h` | `SCSITESTMODULES(IDT_4)P/N71158-001REV1.40` |

This is strong evidence that selector-03 holds factory pretest/audit material,
including component part numbers and revisions, rather than another normal
runtime firmware image.

The executable module stored in sectors 16-27 begins at composite address
`5000h`. It initializes external state and contains paths that access AIC-301
host-interface, host-data, DMA, read/write/stop-pointer registers and
the AIC-010 sequencer status/start, clock/stack, ECC syndrome, GPIO, and buffer
window. This behavior agrees with a manufacturing test module capable of
driving and checking both controller ASICs.

The selector-03 composite reaches 42 internal-ROM destinations. Thirty-seven
overlap the selector-00/01 pass; five additional observed targets are `0000h`,
`0202h`, `0678h`, `069bh`, and `1e7eh`. Its upper code module retains much more
family resemblance than selector-02: order-preserving comparison finds 757
bytes in exact blocks of at least eight bytes against the ST-157N main program,
868 bytes against the ST-138N overlay, and 404 bytes against the later ST-125N
overlay. The stronger ST-138N/ST-157N relationship is consistent with their
shared `80075-509` internal-ROM generation.

## Cross-selector and older-firmware comparison

The two new storage artifacts have eight identical sectors: 12-15 and 28-31.
All eight are `6c` fill, so this is layout identity rather than shared code.
Their active upper modules have only 30 bytes in exact blocks of at least eight
bytes, with a longest block of 20 bytes.

Whole-file same-position comparisons are similarly dominated by fill and
therefore should not be used as code-similarity scores:

| Comparison | Equal byte positions | Identical 256-byte sectors |
| --- | ---: | --- |
| selector-02 vs ST-157N main | 116 / 8,192 | none |
| selector-03 vs ST-157N main | 147 / 8,192 | none |
| selector-02 vs ST-138N | 90 / 8,192 | none |
| selector-03 vs ST-138N | 151 / 8,192 | none |
| selector-02 vs ST-125N | 792 / 8,192 | 30, 31 (`6c` fill) |
| selector-03 vs ST-125N | 833 / 8,192 | 30, 31 (`6c` fill) |

The important comparison is functional: selector-02 supplies substantially
different disk-resident code using the same controller interfaces and resident
ROM, while selector-03 combines explicit factory audit metadata with an ASIC
test overlay. This directly supports the earlier hypothesis that limited RAM
caused the controller to load different disk modules for different operations.

The complete architectural comparison with the ST-125N and ST-138N component
sets is in `st1xxn_service_overlay_comparison.md`.
The command-level and factory-diagnostic behavior is analyzed in
`st1xxn_overlay_behavior_and_diagnostics.md`.
