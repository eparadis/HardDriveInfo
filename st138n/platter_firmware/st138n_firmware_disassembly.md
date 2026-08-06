# ST-138N selector-0 firmware: 8052 mapped disassembly and ST-125N comparison

## Artifacts

- Input: `captures/st138n-startup/artifacts/st138n-stage-a-program.bin`
- Size: 8,192 bytes (32 256-byte sectors)
- SHA-256:
  `366a9f0f5475ca7c080ef47f25e4dd9a06f77d08e0b6b5dd1412c2417cab63fb`
- Corrected mapped listing: `st138n-stage-a-program.mapped.disasm51.asm`
- Historical offset-zero listing: `st138n-stage-a-program.disasm51.asm`
- Corrected comparison listing: `st125n-program.mapped.disasm51.asm`
- Addressed service-cylinder components and new selector-02/03 listings:
  `st138n_service_cylinder_overlays.md`
- Disassembler: `disasm51` 1.0.1

The corrected listing was generated from an analysis image with `0x4000`
bytes of padding before the recovered program. The padding is omitted from the
saved listing, and targets below `0x2000` are retained as unresolved
`maskrom_` references:

```sh
uv tool run disasm51 \
  --include docs/investigations/seagate_8052_symbols.inc \
  --entry 0x4000 \
  --entry 0x4003 \
  --entry 0x4013 \
  --entry 0x40eb \
  --entry 0x4111 \
  --entry 0x415f \
  /tmp/st138n-stage-a-mapped-at-4000.bin \
  > docs/investigations/st138n-stage-a-program.mapped.disasm51.asm
```

The historical listing's offset-zero vector names and resident-ROM conclusions
are superseded. Recursive traversal into tables still makes the mapped listing
an analysis aid rather than rebuildable source.

## Processor identification

The ST-138N controller is a Siemens `SAB 8052A-N`, additionally marked
`80075-509` and dated `89-35` (1989 week 35). It is a factory-mask-ROM 8052
with 8 KiB of internal program ROM, 256 bytes of internal RAM, and Timer 2.
The ST-125N uses the pin- and architecture-compatible Signetics
`SCN8052HCCA44`, marked `80166-501`; its `9215` marking is probably a 1992
week-15 date code. The parallel `80xxx-5xx` markings strongly suggest Seagate
programmed/custom-part numbers. The later observation of the same `80075-509`
marking on a Signetics `SCN8052HCCA44` in an ST-157N, together with an almost
identical low-ROM call map, makes this substantially stronger than a naming
coincidence, although the convention is still not confirmed by Seagate
documentation. See `docs/st157n_startup_decode.md`.

## High-level conclusion

The ST-138N image is unquestionably from the same 8052 firmware lineage as
the ST-125N image. It uses the same mask-ROM/service-overlay addressing model,
external-memory state blocks, memory-mapped controller range, command-flow
shape, and many byte-identical lookup tables. It is not merely a relink of the
same binary: internal RAM allocation, stack placement, overlay routine sizes,
mask-ROM call destinations, and substantial command/drive-specific code
all differ.

The most plausible interpretation is a related firmware build adapted to a
different drive/controller configuration, paired with a different internal
8052 mask-ROM revision.

## Direct comparison

| Property | ST-125N | ST-138N selector 0 |
|---|---:|---:|
| Overlay slot at file `0x0000` | `LJMP 0x417e` | `LJMP 0x415f` |
| Overlay slot at file `0x0003` | `LJMP 0x40eb` | `LJMP 0x40eb` |
| Overlay slot at file `0x0013` | `LJMP 0x4110` | `LJMP 0x4111` |
| Overlay entry | `0x417e` (file `0x017e`) | `0x415f` (file `0x015f`) |
| On-chip mask ROM | probable Seagate `80166-501` | probable Seagate `80075-509` |
| MCU | Signetics `SCN8052HCCA44` | Siemens `SAB 8052A-N` |
| Initial stack pointer | `0xbf` | `0x6a` |
| Uniform trailing fill | `0x6c`, `0x1d56..0x1fff` | `0x00`, `0x1c61..0x1fff` |
| Program SHA-256 | `ca3070bc...8260d` | `366a9f0f...b63fb` |

The repeated `+0x4000` relationship is now explained directly: both recovered
images occupy external code addresses `0x4000-0x5fff`, while each 8052's
factory-programmed internal ROM occupies `0x0000-0x1fff`. The first bytes of
the disk image form an overlay entry table; they are not the hardware vector
table used when the MCU comes out of reset.

## Shared initialization and hardware model

The main entries at ST-125N `0x417e` and ST-138N `0x415f` have the same broad
sequence:

1. initialize the stack and save an internal state byte;
2. clear 89 external-memory bytes beginning at `0x5b02`;
3. call internal mask-ROM and overlay initialization routines;
4. initialize parameter bytes used by two closely related local routines;
5. read state at `0x5b67` and `0x5b6c`;
6. enable interrupt-driven command processing.

The formerly unidentified `0x8050-0x807f` external-data region matches the
Adaptec AIC-300/301 and AIC-010 register maps exactly. The ST-138N pass exposes
AIC-301 host-data, host-interface, DMA, reset, read-pointer, write-pointer, and
stop-pointer registers at `0x8050-0x805f`. It also exposes the AIC-010 buffer
window, ECC control/syndrome, polynomial, status/start, operation, GPIO, and
clock/stack registers at `0x8070-0x807f`. See
`docs/investigations/adaptec_aic_register_map.md`.

The PCB has an Adaptec `AIC-301L ZA837 615700` in the buffer-controller
position. The AIC-010-compatible position is instead occupied by a Cirrus
Logic `11740-501-D 821 B 816-61005-0`. The exact register and bit-level
agreement establishes programming compatibility with the documented AIC-010
interface, but does not by itself distinguish a licensed second source from a
Cirrus or Seagate-specific compatible derivative.

The operations agree with the data sheets rather than merely sharing address
numbers: the overlay performs SCSI arbitration through AIC-301 register `52h`,
controls and polls DMA through register `53h`, programs stop pointers, starts
and polls the AIC-010 sequencer through register `79h`, pops its sequencer
stack through register `7Fh`, and uses the documented ECC registers to compute
and apply corrections through the buffer window.

The ST-138N and ST-157N listings expose the same logical register subset and
closely corresponding transfer/ECC routines. The ST-125N's smaller visible set
is now best explained by its different internal-ROM boundary or incomplete
indirect-path discovery, not a contradictory hardware register map.

## Command-processing correspondence

The command gate beginning at ST-125N `0x42ff` (file `0x02ff`) corresponds
closely to the ST-138N routine at `0x42ea` (file `0x02ea`). The current
command byte is held in internal RAM
`0x44` on ST-125N and `0x36` on ST-138N. Both versions perform the same ordered
command-byte comparisons against:

- `0x0a`, `0x2a`, `0x07`, and `0x04`;
- `0x03`;
- `0x16` and `0x17`;
- `0x11`.

The product manual names `0x11` READ USAGE COUNTER. After `0x04` has been
recognized, a separate comparison tests the next saved CDB byte for `0x18`;
that is a FORMAT UNIT option/parameter, not a command opcode. These tests occur
in nearly identical control-flow shapes and independently confirm that the
recovered ST-138N artifact is controller firmware, not arbitrary service data.
The surrounding
internal-RAM variables have been systematically reassigned—for example the
ST-125N state bytes around `0x5b6c` are loaded into `0x2e,0x50..0x53`, while
the ST-138N loads them into `0x2b,0x42..0x45`.

## Exact binary correspondence

Only sectors 12 and 13 are byte-identical at the same offsets. They contain
large lookup/dispatch tables, including monotonic numeric tables and arrays of
16-bit offsets. This is shared algorithmic data rather than proof that the
surrounding routines are identical.

Across the complete images:

- 1,586 of 8,192 byte positions are equal;
- an order-preserving binary comparison finds 121 exact blocks of at least
  eight bytes, totaling 2,744 bytes;
- the longest exact block is 678 bytes at `0x0bb6` in both images;
- another 186-byte block begins at `0x0acd` in both images.

Those longest blocks lie primarily in shared table regions. Shorter matching
blocks throughout executable regions frequently move by stable local deltas,
for example `-21`, `-58`, `-12`, or `-32` bytes. That pattern is consistent
with routines being edited or resized while later local routines and tables
retain their relative order.

## Internal mask-ROM calls

Targets in `0x4000-0x5fff`, including shared targets `0x40eb` and `0x4f00`,
are calls within the disk-loaded overlay. Targets below `0x2000` enter the
factory mask ROM inside the MCU. Many of those low targets differ between the
two builds even where the nearby overlay routines correspond closely. This is
consistent with different internal ROM layouts under Seagate part numbers
`80166-501` and `80075-509`. Neither image can therefore be understood fully
in isolation: the two on-chip ROM images are needed to name the firmware
services invoked by the overlays.

The mapped pass finds 30 distinct low-ROM targets from the ST-125N overlay and
43 from the ST-138N overlay, with no exact target address shared between those
discovered sets. The paired call sequences often retain the same shape while
every destination moves—for example the early initialization sequence changes
`0x02d4, 0x0214, 0x09cd, 0x1911` to
`0x02cb, 0x021e, 0x09b2, 0x19d5`. This is stronger evidence for two different
internal mask-ROM link maps than the earlier, reversed interpretation of the
`0x40xx` calls. Counts remain provisional because recursive discovery does not
cover every indirect path and can enter table data.

The 8052-aware pass found no explicit access to the standard Timer 2 SFRs in
either recovered overlay. Timer 2 setup may reside in the corresponding
on-chip ROM and cannot be assessed without those images.

## Next analysis steps

1. Locate documentation or establish pin-level correspondence for the Cirrus
   `11740-501-D`, then compare it with the ST-125N's Seagate-style
   `11740-511`.
2. Recover the internal mask ROMs and construct version-specific call maps for
   targets below `0x2000`.
3. Mark the common table region explicitly so recursive disassembly does not
   treat it as code.
4. Normalize internal-RAM variables and mask-ROM calls, then perform a
   function-level semantic diff rather than a raw textual listing diff.
5. Determine the internal-ROM loader sequence that selects and combines the
   now-complete selector-02 and selector-03 disk components documented in
   `st138n_service_cylinder_overlays.md`.

## Processor references

- [Signetics SCN8032/8052 product specification](https://www.tvsat.com.pl/pdf/8/8032_52_ph.pdf)
- [Siemens SAB 8052A-N data-sheet index](https://www.alldatasheet.com/datasheet-pdf/pdf/129162/SIEMENS/SAB8052A-N.html)
