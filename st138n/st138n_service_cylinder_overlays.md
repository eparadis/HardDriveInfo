# ST-138N addressed service-cylinder artifacts

## Capture and decode

The source for this firmware dump is a capture of cylinder 0, heads 0
through 3 of a ST-138N chassis mechanism, read through an
ST-157R control PCB. Each track was acquired as five 250 MSa/s slices at
timebase offsets 0, 3.333, 6.667, 10, and 13.333 ms.

Each track produced:

- 46 candidate observations;
- 46 valid observations and no rejected observations;
- 45 exported sectors, numbered 0 through 44;
- an 11,520-byte complete decoded image.

This is direct confirmation that the `FF` selector in these records identifies
the selected physical head. It also provides complete, independently addressed
service corpora for all four recorded surfaces.

## Agreement with the earlier ST-138N investigation

Prior investigations captured by "catching the read" at startup agrees with
the data captured and analyized here.

- all 43 previously recovered selector-0 sectors match C0/H0;
- all 30 previously recovered selector-1 sectors match C0/H1;
- all eight previously recovered selector-3 sectors (17-20, 25-26, and
  28-29) match C0/H3;
- C0/H2 matches the earlier shared tail at sectors 33-35, 37-39, and 41-44,
  as well as the old selector-3 observations at sectors 28 and 29.

This establishes that this capture is the same ST-138N service corpus, not
merely another drive using a related format. This confirms the viability of
the "catching the read" methodology, which requires native interface head
positioning control instead of a control board replacement.

## Reconstructed binary components

The first 32 sectors from each head are preserved as 8 KiB binary components:

| Service location | Artifact | SHA-256 |
| --- | --- | --- |
| C0/H0, selector `00` | `st138n-c0h0-selector00.bin` | `366a9f0f5475ca7c080ef47f25e4dd9a06f77d08e0b6b5dd1412c2417cab63fb` |
| C0/H1, selector `01` | `st138n-c0h1-selector01.bin` | `366a9f0f5475ca7c080ef47f25e4dd9a06f77d08e0b6b5dd1412c2417cab63fb` |
| C0/H2, selector `02` | `st138n-c0h2-selector02.bin` | `2a3442907499f551d278ec954c7e6972fc6953f305fd1678848af0b6c85e7cf5` |
| C0/H3, selector `03` | `st138n-c0h3-selector03.bin` | `793b5f29d70d55f176495ab198a2b31220c043a29d6787cbe1515d4f75ea4f6c` |

H0 and H1 are both retained despite having identical contents so their
separate surface provenance is not lost.

## Storage layout and disassembly model

Selectors 00 and 01 are the already disassembled main program. Selectors 02
and 03 expose components that were missing or incomplete in the earlier
capture. Their sector layout matches the loading model inferred from the
ST-157N:

| File range | Selector 02 | Selector 03 |
| --- | --- | --- |
| `0000-0bff` (sectors 0-11) | dense 8052 code and data, followed by zero padding | identification/configuration storage, followed by fill |
| `0c00-0fff` (sectors 12-15) | `6c` fill | `6c` fill |
| `1000-1bff` (sectors 16-27) | dense 8052 code and data, followed by zero padding | dense 8052 code and data, followed by zero padding |
| `1c00-1fff` (sectors 28-31) | `6c` fill | `6c` fill |

The executable components use absolute addresses in external code space
`4000-5fff` and call both other disk-loaded regions and internal mask-ROM
services below `2000h`. The mapped listings therefore use the same explicit
layered analysis as the ST-157N investigation:

- `st138n-c0h2-selector02.composite.mapped.disasm51.asm` replaces
  `4000-4bff` and `5000-5bff` in the selector-00 baseline;
- `st138n-c0h3-selector03.composite.mapped.disasm51.asm` retains the
  selector-00 lower region and replaces `5000-5bff`;
- selector-03 sectors 0-15 remain storage data and are not presented as
  executable instructions.

The listings were generated with `generate_layered_8052_listing.py`,
`disasm51` 1.0.1, and `seagate_8052_symbols.inc`. Calls below `2000h` remain
explicit unresolved `maskrom_` references. `disasm51` also warned about
unidentified SFR values `C0h`, `DBh`, and `FFh` while recursively traversing
selector 02. These are not assigned speculative hardware names; some may be
data reached by recursive traversal. As with the existing listings, these are
analysis aids rather than rebuildable source.

## Selector-02 comparison

Selector 02 is a drive-specific build of the same kind of component recovered
from the ST-157N, not a copy of the main program:

- its lower module begins with coherent 8052 code at `4000h`;
- its upper module contains `SEAGATE ST138N` at file offset `11b1h` and
  `(C) COPY RIGHT SEAGATE TECHNOLOGY 1985,1986` at `1b86h`;
- sectors 12-15 and 28-31 are byte-identical `6c` fill in the ST-138N and
  ST-157N selector-02 artifacts;
- no code-bearing 256-byte sector is identical between the two drives.

Despite the lack of identical code sectors, order-preserving comparison shows
substantial source-lineage correspondence. In the lower active module it finds
2,804 bytes in 54 exact blocks of at least eight bytes, with a longest block of
512 bytes. In the upper module it finds 1,250 bytes in 35 such blocks, with a
longest block of 338 bytes.

The ST-138N selector-02 listing reaches 27 distinct internal mask-ROM
destinations. Twenty-three also appear in the ST-157N selector-02 composite.
The four currently unique ST-138N targets are `0871h`, `0ea1h`, `0ea5h`, and
`1f57h`; the difference reflects a related but drive-specific disk module
using the shared `80075-509` ROM interface.

## Selector-03 comparison

The ST-138N selector-03 lower storage area is much simpler than the ST-157N
factory-audit corpus. It begins with ASCII `000487810`, contains zero-filled
fields, has 20 spaces at `0500h`, and then uses `6c` fill from `0514h` through
`0fffh`. It does not contain the ST-157N's model-specific pretest/audit strings.

The executable upper component is extremely close to the ST-157N selector-03
component:

- 25 of the 32 storage sectors are byte-identical: 2-4, 6-7, 9-19, 21-22,
  24-25, and 27-31;
- nine of the 12 code-bearing upper sectors are entirely identical;
- only seven bytes differ in the upper module, at file offsets `1406h`,
  `1442h`, `1799h`, `179eh`, `17d6h`, `1a13h`, and `1ad7h`;
- the upper modules share 3,061 bytes in seven order-preserving exact blocks
  of at least eight bytes, with a longest block of 1,030 bytes.

The complete artifacts have 7,752 equal byte positions. Most lower-region
differences are identification and audit/configuration storage rather than
code. The ST-138N selector-03 composite reaches 43 internal mask-ROM
destinations; all 42 found in the ST-157N composite recur, plus `1993h`.

This makes selector 03 a shared controller-family test module with small
drive-specific constants or state assignments, paired with model-specific
storage data. Selector 02 is more extensively rebuilt for the individual
drive, although its structure and many code blocks remain recognizably shared.

## Result

Comparisons with the ST-157N show that both drives use the same four-surface
service-cylinder organization and the same layered 8052 firmware architecture.
They do not make all four surfaces interchangeable: selector 02 is
drive-specific, and selector 03 combines a nearly shared executable module with
model-specific identification/configuration storage.

The complete architectural comparison with the ST-125N and ST-157N component
sets is in `st1xxn_service_overlay_comparison.md`.
The command-level and factory-diagnostic behavior is analyzed in
`st1xxn_overlay_behavior_and_diagnostics.md`.
