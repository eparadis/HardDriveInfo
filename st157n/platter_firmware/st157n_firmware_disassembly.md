# ST-157N service firmware: mapped 8052 disassembly and comparison

## Artifacts

- Input: `captures/st157n-startup/artifacts/st157n-program.bin`
- Size: 8,192 bytes (32 256-byte sectors)
- SHA-256:
  `8664358f4380df705f44b37082459665cb26545468429608c06a50b3eb35c835`
- Mapped listing: `st157n-program.mapped.disasm51.asm`
- Comparison listings:
  `st125n-program.mapped.disasm51.asm` and
  `st138n-stage-a-program.mapped.disasm51.asm`
- Symbol include: `seagate_8052_symbols.inc`
- Disassembler: `disasm51` 1.0.1

Addressed service-cylinder captures subsequently recovered selector-02 and
selector-03 storage artifacts from C0/H2 and C0/H3. Their layered disassemblies
and comparison with this main selector-00/01 overlay are documented in
`st157n_service_cylinder_overlays.md`.

As with the corrected ST-125N and ST-138N passes, the input was prefixed with
`0x4000` bytes of disposable `0xff` padding. This makes file offset zero appear
at its actual external-code address `0x4000`. The pass entered the image at
`0x4000`, `0x4003`, `0x4013`, `0x40eb`, `0x410d`, and `0x417d`. The saved
listing omits attempted traversal into the padding and names targets below
`0x2000` as unresolved `maskrom_XXXX` references.

The listing remains an analysis aid rather than rebuildable source. Recursive
discovery cannot see every indirect target and can still mistake table data
for code.

## Address and processor model

The board uses a Signetics `SCN8052HCCA44` marked `80075-509
229B3079005KB`. The recovered program follows the established Seagate layout:

- internal factory mask ROM: `0x0000-0x1fff`;
- disk-loaded external-code overlay: `0x4000-0x5fff`;
- overlay code address = file offset + `0x4000`.

The `9005` marking is consistent with a 1990 week-5 date code. The same
`80075-509` identifier appears on the Siemens `SAB 8052A-N` in the ST-138N,
despite the different MCU manufacturer.

The ST-157N PCB also carries an Adaptec `AIC-010FL` programmable mass-storage
controller and `AIC-301L` dual-port buffer controller. Their data-sheet
register maps resolve the overlay's external-data addresses exactly:

- `0x8050-0x805f` contains the AIC-301 host interface, DMA control, reset, and
  buffer-pointer registers;
- `0x8070-0x807f` contains the shared buffer window and AIC-010 ECC,
  sequencer, operation, GPIO, and clock/stack registers.

The enriched listing uses these names directly. Detailed register definitions,
behavioral correlations, and source links are in
`docs/investigations/adaptec_aic_register_map.md`.

## Direct comparison

| Property | ST-125N | ST-138N | ST-157N |
| --- | ---: | ---: | ---: |
| MCU | Signetics SCN8052 | Siemens SAB 8052A-N | Signetics SCN8052 |
| Seagate mask identifier | `80166-501` | `80075-509` | `80075-509` |
| Overlay slot at file `0x0000` | `LJMP 0x417e` | `LJMP 0x415f` | `LJMP 0x417d` |
| Overlay slot at file `0x0003` | `LJMP 0x40eb` | `LJMP 0x40eb` | `LJMP 0x40eb` |
| Overlay slot at file `0x0013` | `LJMP 0x4110` | `LJMP 0x4111` | `LJMP 0x410d` |
| Initial stack pointer | `0xbf` | `0x6a` | `0x6a` |
| Current command byte | internal RAM `0x44` | internal RAM `0x36` | internal RAM `0x36` |
| Command routine | `0x42ff` | `0x42ea` | `0x430e` |
| Distinct discovered mask-ROM targets | 30 | 43 | 42 |
| Trailing fill | `0x6c` from file `0x1d56` | `0x00` from file `0x1c61` | `0x00` from file `0x1c69` |

The command routine addresses are code addresses; their corresponding file
offsets are `0x02ff`, `0x02ea`, and `0x030e`.

## Internal mask-ROM interface

All 42 low-ROM destinations referenced by the discovered ST-157N paths occur
at exactly the same addresses in the ST-138N listing. The ST-138N pass has one
additional target, `0x1993`, which is called during its early initialization
path but is not reached by the ST-157N pass. None of the 42 ST-157N targets is
shared with the 30-target ST-125N set.

The early initialization sequences make the relationship concrete. Both the
ST-157N and ST-138N call services at `0x1fa1`, `0x05d7`, `0x05dc`, `0x1f69`,
and `0x01ff`, while the corresponding ST-125N sequence calls the differently
linked `80166-501` services at `0x1d62`, `0x05e8`, `0x05ed`, `0x1d2a`, and
`0x01f5`. The ST-157N and ST-138N therefore link against the same internal-ROM
ABI, while the ST-125N links against a different revision.

This is independent confirmation that `80075-509` identifies the programmed
mask-ROM build rather than a Siemens catalog part. The ST-157N's Signetics
device appears to be a second-source implementation of the same Seagate ROM
contents. Absolute proof of byte-identical internal ROMs still requires
reading the two MCUs; identical entry addresses demonstrate interface and
revision correspondence, not every hidden byte.

## Initialization and RAM allocation

The ST-157N main entry at `0x417d` is much closer to the ST-138N entry at
`0x415f` than to the ST-125N entry at `0x417e`:

- ST-157N and ST-138N initialize `SP` to `0x6a`; ST-125N uses `0xbf`.
- ST-157N and ST-138N save state in byte `0x4e` and use parameter bytes
  `0x5a`, `0x35`, `0x59`, and `0x33`.
- ST-125N uses the shifted allocation `0x5b`, `0x69`, `0x43`, `0x68`, and
  `0x41` for the corresponding roles.
- All three clear 89 external-data bytes beginning at `0x5b02`.

The ST-138N initialization calls mask-ROM target `0x1993`; the ST-157N instead
sets internal bit `0x22.4` and adds or rearranges overlay-local calls around
`0x5067` and `0x5160`. This explains why the ST-157N call set is a one-target
subset rather than proving that either program is a simple relink of the
other.

Both overlays pulse `AIC301_RESET_CTL`, arbitrate on the SCSI bus through
`AIC301_HOST_IF_CTL`, control transfers with `AIC301_DMA_CTL`, and operate the
AIC-010 sequencer through `AIC010_STATUS_START`. Their shared ECC routine
programs `AIC010_POLY_1_8` and `AIC010_POLY_25_31`, shifts and reads the ECC
syndrome, then combines the correction bytes with `AIC_BUFFER_DATA` while
temporarily adjusting the AIC-301 read/write pointers. These semantically
matching hardware operations strengthen the common-source conclusion beyond
raw instruction correspondence.

The ST-157N's physically marked Adaptec AIC-301L and AIC-010FL provide the
reference implementation for this comparison. The ST-138N retains an
AIC-301L but substitutes a Cirrus Logic `11740-501-D` in the AIC-010 position;
the ST-125N uses a Cirrus `11739-502` and a Seagate-style `11740-511` in the
corresponding two positions. Their matching register-level behavior therefore
shows that this firmware family targets a stable programming interface across
multiple physical implementations, rather than requiring Adaptec-branded
silicon in every drive.

## Command processing

The ST-157N command routine begins at `0x430e`, versus `0x42ea` in the ST-138N
and `0x42ff` in the ST-125N. ST-157N and ST-138N both read the current command
from internal byte `0x36`; ST-125N uses byte `0x44`.

All three retain the same recognizable SCSI command tests and ordering,
including `0x0a`, `0x2a`, `0x07`, `0x04`, `0x03`, `0x16`, `0x17`, and
`0x11`. The product manual identifies `0x11` as READ USAGE COUNTER. A nearby
`0x18` comparison is made against the next saved CDB byte only after FORMAT
UNIT (`0x04`) has been recognized; it is a format option/parameter, not a SCSI
opcode. The ST-157N control flow and mask-ROM calls closely follow the ST-138N
version, with overlay-local destinations shifted as routines grow or move.
The ST-125N implements the same higher-level state machine with a different
RAM allocation and mask-ROM call map.

## Binary and table correspondence

| Comparison | Equal byte positions | Identical 256-byte sectors |
| --- | ---: | --- |
| ST-157N vs ST-125N | 1,607 / 8,192 | 11, 12, 13 |
| ST-157N vs ST-138N | 3,024 / 8,192 | 12, 13, 29, 30, 31 |

The ST-157N and ST-125N share a 911-byte same-offset block beginning at file
offset `0x0acd`. The ST-157N and ST-138N share a 799-byte same-offset block
beginning at `0x0bb6`; the common three-way portion includes the large lookup
table region already identified in the earlier comparison. Exact sectors
29-31 between ST-157N and ST-138N are part of the zero-filled tail and should
not be interpreted as executable-code identity.

The combination of shared tables, the same command state machine, and two
distinct mask-ROM ABIs supports a firmware family built from common source:
ST-138N and ST-157N use the same `80075-509` resident services but different
disk overlays, while the later ST-125N uses the `80166-501` mask-ROM revision.

## Next steps

1. Mark table and indirect-dispatch regions explicitly to reduce false code
   paths in all three listings.
2. Normalize shifted overlay-local labels and RAM variables for a semantic
   function-by-function diff of the ST-138N and ST-157N.
3. Recover either `80075-509` MCU's internal ROM; one image should resolve the
   42 shared service calls in both overlays if the same-mask hypothesis holds.
4. Recover the ST-125N `80166-501` ROM separately and map its corresponding
   service entry points.
