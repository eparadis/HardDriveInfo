# ST-157A service-cylinder firmware

## Hardware and capture identity

Captures were made from an ST-157A
mechanism connected to an ST-157R control PCB. This permits direct selection
of physical cylinders and heads without relying on the ST-157A controller's
startup sequence.

The experiement was repeated with a
different ST-157A mechanism, chassis serial `58786951`, on the same kind of
ST-157R control PCB. The earlier mechanism has chassis serial `87759717`.
The second unit is therefore an independent test of both the physical layout
and the degree to which the platter firmware is model-wide or unit-specific.

The first capture set used lower rate 50MSa/s sampling. The second capture
used higher-rate 250MSa/s sampling, which required six independent offset
samples. A ~1.42 ms region overlap between samples allowed correspondences
between the samples at the sector decoding level.

The second unit was captured again at 250 MS/s, but with only 5 captures
for a 0.86 ms overlap region between captures. This resulted in a less robust
set of correspondences, but complete sector sets were recovered for H2, H3,
and H4. H0, H1, and H5 were partial, but their recovered sectors are
sufficient to check the redundant program and fill copies independently.

The ST-157A was not the only capacity in this ATA family. Seagate's
contemporary family documentation names the ST-125A, ST-138A, and ST-157A,
and a later official installation guide also lists all three. This makes the
ATA family more directly parallel to the SCSI ST-1xxN family than an
ST-157A-only product line would have been. Access to an ST-125A or ST-138A
would allow platter firmware captures to be made and confirm this hypothesis.

## Record format

The exact layouts are:

```text
A1 FF <physical head> <sector ID> 00 <CRC32>
A1 F8 <256-byte payload> <CRC32>
```

The header selector equals the selected physical head on H0-H5. The data rate,
IBM RLL code map, sync pattern, `0x41044185` MSB-first CRC polynomial, zero
initial value, and CRC coverage are all identical to the ST-125N service
profile. No ST-157A-specific sector-framing variation was found.

The 50 MSa/s decode found service records only on C0. C1-C5 produced none
under this profile, so the evidence identifies C0 as the reserved/service
cylinder and does not indicate a multi-cylinder service area.

## Surface organization

The high-rate results from both mechanisms and the independent 50 MSa/s
observations establish:

| Surface | Contents |
| --- | --- |
| C0/H0 | firmware copy; 37 high-rate sectors recovered, all equal to H1/H2 |
| C0/H1 | complete firmware copy, IDs 0-45 |
| C0/H2 | complete firmware copy, IDs 0-45; byte-identical to H1 |
| C0/H3 | chassis serial and physical-defect list in IDs 0-1, zeros in 2-4, then `0x6c` fill |
| C0/H4 | complete IDs 0-45, all `0x6c` fill |
| C0/H5 | complete IDs 0-45, all `0x6c` fill |

H1 and H2 have identical 11,776-byte decoded images with SHA-256
`944ff1b1509182b8919e54b7255a5d18a2c3baaf628aa1b04abcd117df28934c`.
Every recovered H0 sector also agrees. The lower-rate capture independently
agrees for all recovered sectors on H0, H1, and H2.

H4 and H5 have identical complete decoded images with SHA-256
`efa3e80175b8a9ad37eb4417d7256289b20b5f4b762b0860b2c9318a07d5106a`.
Their valid headers and uniform payloads show that these are deliberately
formatted fill surfaces. They are not analogous to the non-repeatable,
unrecognized H4/H5 activity seen on the addressed ST-157N.

The second unit has exactly the same selector roles:

- H2 is a complete firmware copy; all recovered H0 and H1 sectors agree with
  it, with no conflicts across 36 H0 sectors and 43 H1 sectors;
- H3 has unit data in sectors 0-4 and `0x6c` fill thereafter;
- H4 is a complete, byte-identical fill track with the same
  `efa3e801...d5106a` SHA-256;
- all 29 recovered H5 sectors are also identical `0x6c` fill.

The second unit's complete H2 image is 11,776 bytes with SHA-256
`ac42d40e11af9520e03e66862a48c6b54681db56608dde22182bf13d14a81590`.
It differs from the earlier `944ff1b1...8934c` image because it contains an
older firmware revision and different unit data, not because the track layout
changed.

At sector granularity, the complete H2 comparison is:

| Sectors | Between-unit result |
| --- | --- |
| 0-27 | all differ; these contain the revision-dependent program |
| 28 | identical zeros |
| 29 | two differing bytes at the end |
| 30 | 105 differing bytes in serial/remapping data |
| 31 | identical `0x6c` fill |
| 32 | three differing ATA IDENTIFY firmware bytes |
| 33 | identical zeros |
| 34 | two differing prefix bytes; description text is identical |
| 35-45 | identical `0x6c` fill |

On H3, only sectors 0 and 1 differ. H4 is completely identical, and every
recovered H5 sector is identical. No unexplained content or layout difference
remains outside the firmware revision and expected unit-specific data.

This differs materially from the ST-125N/ST-138N/ST-157N organization. The
SCSI drives store redundant main overlays on H0/H1 and distinct service/test
modules on H2/H3. The ST-157A instead stores three redundant program copies
on H0/H1/H2 and uses H3 for compact drive-specific data. Shared physical
format does not imply shared selector roles.

## Extracted program and data

`st157a-program.bin` concatenates sectors 0-34 from two independently
selected complete surfaces on serial `87759717`:

- size: 8,960 bytes;
- SHA-256:
  `3b8fde73916d1501a596d78ba90cde814c52850670cede09673e4a43b61e457d`;
- sources: high-rate C0/H1 and C0/H2 sector manifests.

`st157a-58786951-program.bin` is the corresponding artifact from the second
mechanism:

- size: 8,960 bytes;
- SHA-256:
  `fa062ae10f3907e7470282905ce30c0238117e3ffda81dbcca80e0fc6d72f22e`;
- source: complete high-rate C0/H2, checked without conflict against every
  recovered H0 and H1 sector;
- firmware banner: `ST157A RAM Rev. 3.3`;
- ATA IDENTIFY firmware field: `3.2/012`.

The first artifact's banner is `ST157A RAM Rev. 4.2`, and its ATA IDENTIFY
firmware field is `4.2/064`. The banner/IDENTIFY subrevision mismatch in the
Rev. 3.3 artifact is present in CRC-verified platter data; it is not a string
decoding error.

Both artifacts use this layout:

| Sector range | File range | Contents |
| --- | --- | --- |
| 0-27 | `0000-1bff` | 8052 external code, tables, and zero padding |
| 28 | `1c00-1cff` | zeros |
| 29 | `1d00-1dff` | zeros followed by a two-byte revision-dependent value |
| 30 | `1e00-1eff` | drive-specific serial/remapping data and `0xff` padding |
| 31 | `1f00-1fff` | `0x6c` fill |
| 32 | `2000-20ff` | 256-byte ATA IDENTIFY data template |
| 33 | `2100-21ff` | zeros |
| 34 | `2200-22ff` | two revision-dependent bytes, identify-block description, and zeros |
| 35-45 | not included | `0x6c` fill |

The first bytes identify the two payloads directly:

```text
GA# < ST157A RAM Rev. 4.2, Copyright (C) 1989, Seagate Technology >
GA# < ST157A RAM Rev. 3.3, Copyright (C) 1989, Seagate Technology >
```

The leading `GA#` text follows the first entry jump and is incidental to code
layout. The revision strings themselves are literal firmware identification.

The ATA IDENTIFY template uses the ATA word-byte order and decodes as:

| Field | Value |
| --- | --- |
| Native cylinders | 560 |
| Native heads | 6 |
| Native sectors per track | 26 |
| Firmware revision | `4.2/064 ` |
| Model | `Seagate Technology ST157A` |
| Serial field | blank in the template |

The following sector labels itself
`<<<ST157A Identify Drive Command Data Block Rev 1.0>>>`. The geometry
corresponds to 87,360 addressable 512-byte sectors.

The second template retains the same geometry, model, and blank serial field;
only three bytes differ, changing the firmware field to `3.2/012`. Both
sector-30 records contain the chassis serial in ATA word-byte order:
`78577971` decodes to `87759717`, and `85879615` decodes to `58786951`.
The rest of sector 30 contains a much longer unit-specific table on the second
mechanism, consistent with its larger physical-defect list described below.

`st157a-c0h3-service-data.bin` preserves H3 sectors 0-4:

- size: 1,280 bytes;
- SHA-256:
  `848cec23007d2d9d69e928929fd1c42f3da28b2d33f9ec50fd1bd4939bc5575c`;
- sector 0 begins with ASCII `87759717` and is otherwise zero;
- sector 1 contains a counted 12-entry physical-defect list;
- sectors 2-4 are all zero.

`st157a-58786951-c0h3-service-data.bin` is the corresponding second-unit
component:

- size: 1,280 bytes;
- SHA-256:
  `356cf4d8303b54d52e362095df1ff22deb2e7a0d0841c25046ed2279d784cda5`;
- sector 0 contains ASCII `58786951` and zeros;
- sector 1 contains a 48-entry physical-defect list;
- sectors 2-4 are byte-identical zeros.

Including sectors 5-45 of common `0x6c` fill, the complete H3 track hashes are
`cc94eb6355b0d5f76b88681176e36ea49ca93478f587d18270d70a01e5ba82b2`
for `87759717` and
`1823e785f28aa5a10530d8ff172ada827fbdf9f8943e142bce0ac40e4a852446`
for `58786951`. Only sectors 0 and 1 differ.

Sector 1 now has a clear structure:

```text
BE ED <count> (<32-bit big-endian cylinder> <head>){count}
00 00 <3-byte trailer> <zero padding>
```

This interpretation is supported independently by both units. The count byte
is exactly `0x0c`/12 or `0x30`/48; every entry's cylinder is within the
ST-157A's 0-559 physical-cylinder range; every final byte is a valid head
number 0-5; and entries are sorted first by head and then by cylinder. The
`BE ED` marker also matches the controller family's previously identified
defect-list marker. The two reserved bytes after the entries are zero on both
units. The following three-byte trailers are `ff6018` and `fff056`;
their checksum or validation rule has not yet been established.

The complete decoded lists are:

| Unit | Head | Listed cylinders |
| --- | ---: | --- |
| `87759717` | 0 | 429 |
| `87759717` | 1 | 144, 165, 423, 465 |
| `87759717` | 2 | 66-70 |
| `87759717` | 4 | 220, 537 |
| `58786951` | 0 | 30, 35-41, 97, 194-197, 252, 275-277, 347-349, 415-417, 547-549 |
| `58786951` | 1 | 125, 169-171, 176, 181-185 |
| `58786951` | 2 | 3, 89 |
| `58786951` | 5 | 72, 349-352, 434, 529-532 |

This corrects the earlier conservative description of sector 1 as an
uninterpreted parameter region. It is unit-specific factory defect data.
The captures do not show whether the list was produced by a single factory
test, accumulated across later diagnostics, or rewritten during low-level
formatting.

All four extracted binaries have neighboring metadata files containing source
manifests, sector hashes, selector/head identity, sizes, and artifact hashes.

## 8052 mapping and ATA behavior

`st157a-program.mapped.disasm51.asm` maps file offset zero to external code
address `4000h`. The first six bytes are three two-byte entry jumps:

| Slot | Destination |
| --- | --- |
| `4000h` | `4322h` |
| `4002h` | `4047h` |
| `4004h` | `4223h` |

The first is the main initialization entry, and `4223h` saves processor state
and handles controller interrupt status. Calls below `2000h` enter unresolved
on-chip mask-ROM services. The recursive listing currently exposes 33 such
destinations, principally in compact groups around `0036h`, `0800h`, and
`1002h`.

The program reads an ATA command byte from external register `8011h` and uses
a 16-entry high-nibble jump table at `441fh`. The decoded paths account for:

| Opcode or family | Behavior indicated by the code |
| --- | --- |
| `1xh` | recalibrate family |
| `2xh` | read-sector family |
| `3xh` | write-sector family |
| `4xh` | verify family |
| `5xh` | format-track family |
| `7xh` | seek family |
| `90h` | execute drive diagnostics |
| `91h` | initialize drive parameters |
| `E4h` | read buffer |
| `E8h` | write buffer |
| `ECh` | identify drive |
| `C0h`, `F0h` | gated vendor-specific paths requiring task-file signatures |

The `ECh` handler reaches the routine associated with the captured IDENTIFY
template. Unsupported high-nibble groups converge on a common error/status
path. The C0/F0 signature tests are visible, but their higher-level purpose
is not yet named.

There is no direct access to the 8052 UART registers or flags (`SCON`, `SBUF`,
`RI`, or `TI`) in the mapped ST-157A listing. Timer 1 is used as a timing
counter; that use is not evidence of serial I/O.

The listing also accesses the same external register ranges used by the
ST-157R controller family, including the `807dh/807eh` GPIO pair represented
by the existing AIC-010 symbols. This is code-level evidence of controller
hardware compatibility in addition to the successful physical substitution.

`st157a-58786951-program.mapped.disasm51.asm` applies the same mapping and
entry analysis to Rev. 3.3. The comparison establishes that Rev. 3.3 and
Rev. 4.2 are revisions of the same ATA firmware branch, not unrelated
programs:

- entry slots `4002h -> 4047h` and `4004h -> 4223h` are unchanged; the main
  entry moves by one byte, from `4321h` in Rev. 3.3 to `4322h` in Rev. 4.2;
- the 16-way high-nibble command table moves from `440eh` to `441fh` but has
  the same implemented/default slot topology;
- both listings explicitly dispatch opcodes `90h`, `91h`, `E4h`, `E8h`, and
  `ECh`, retain the same `1xh`, `2xh`, `3xh`, `4xh`, `5xh`, and `7xh`
  families, and retain the signature-gated C0/F0 paths;
- both use the same ATA task-file and controller register ranges and neither
  accesses the 8052 UART;
- normalized instruction sequences are strongly conserved despite routine
  relocation, while Rev. 4.2 extends code about 271 bytes farther before the
  long zero-filled tail (file offsets `1bb1h` versus `1aa2h`);
- the Rev. 4.2 recursive listing reaches seven mask-ROM service destinations
  not used by Rev. 3.3: `0052h`, `1010h`, `1014h`, `1016h`, `1018h`,
  `1021h`, and `1024h`.

One concrete dispatcher change is visible before the common command table:
Rev. 4.2 special-cases command `C0h` and saves it at external address
`5f01h`; Rev. 3.3 proceeds directly into common dispatch. The higher-level
meaning of that saved byte remains unresolved. More broadly, the later
revision adds and relocates substantial code while preserving the same ATA
command architecture. Across the complete 8,960-byte artifacts, 2,661 bytes
are equal at the same offset and 6,299 differ; insertion/deletion-aware
matching and the disassemblies show that much of the apparent byte churn is
routine relocation rather than wholesale replacement.

## Comparison with the SCSI overlays

The ST-157A program is not an ST-1xxN SCSI overlay with ATA strings patched
in. The following same-offset comparison uses the Rev. 4.2 artifact and the
first 8,192 bytes:

| Comparison | Equal bytes at the same offset | Identical sectors |
| --- | ---: | --- |
| ST-157A / ST-125N | 543 | sector 31 only (`0x6c` fill) |
| ST-157A / ST-138N | 622 | none |
| ST-157A / ST-157N | 606 | none |

The longest ST-138N/ST-157N matches are zero-filled regions. Short exact
controller-register instruction sequences remain, consistent with shared
hardware conventions and an 8052 implementation, but there is no
code-bearing sector identity. The ATA command dispatcher, task-file register
handling, IDENTIFY data, three-copy redundancy, and different mask-ROM call
map make this a distinct firmware branch.

## Conclusion

The central hypothesis is now confirmed on two independent mechanisms. Both
ST-157As use the same C0 reserved cylinder, 46-sector service-record format,
H0-H2 program redundancy, H3 serial/defect data, and H4/H5 formatted fill.
The on-platter payload is indisputably secondary ST-157A ATA firmware, not an
accidental decode of user data. The differing Rev. 3.3 and Rev. 4.2 payloads
also show that this area was a real firmware distribution and revision
mechanism rather than a fixed factory pattern.

That the blink-code failure indicator at startup is due to a failure to load
the firmware overlay described here is therefore plausible, although the captures
alone do not reveal the mask-ROM loader's retry/fallback policy or the precise
blink-code implementation.

The important qualification is architectural: the ATA and SCSI product
families share the physical service format and controller conventions but
assign the surfaces differently and load different host-interface firmware.
Within the ATA family, the surface roles remain stable while firmware,
serial/remapping data, and physical-defect lists vary by revision and unit.

## Documentation sources

- Seagate ST-125A/ST-138A/ST-157A Product Manual 36045-006 Rev. F,
  transcribed in the model pages at
  <https://stason.org/TULARC/pc/hard-drives-hdd/seagate/ST125A-1-21MB-3-5-HH-IDE-AT.html>
- Seagate ST9144 Family Installation Guide Rev. B, whose compatibility list
  explicitly includes ST125A, ST138A, and ST157A:
  <https://www.seagate.com/staticfiles/support/disc/iguides/ata/9144ig.pdf>
