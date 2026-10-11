# Sekonic SPL-400 "InScribe" ROM — analysis notes

ROM: `Inscribe_PROM.BIN`, 32 KB, from a D27256 EPROM labelled **"©InScribe, Inc. R12.3"** (U3).
Compared against the stock Sekonic SPL-410 firmware in `../Sekonic SPL-410/` (U3 + U4, 2 × 16 KB,
`$0000–$3FFF` / `$4000–$7FFF`).

All addresses are SPL-400 unless marked. Disassembly: `z80dasm -a -t -g 0`.

## Summary

- The firmware is a stock Sekonic **HP 7475A-compatible HP-GL** interpreter (Z80, not scrambled).
  Command table at `$2022`: exactly the 56 HP 7475A commands — nothing added.
- **No handwriting font** in the ROM. The only glyph data is the standard Sekonic stroke font
  (19 HP character sets, extracted to `font/` by `extract_font.py`). The handwriting must have come
  from InScribe's host software as ordinary HP-GL vectors (or `UC` user-defined characters).
- **No dongle check.** Nothing probes the Centronics port, its data lines are never driven, and
  nothing accesses memory outside the ROM and RAM.
- **The plotter is locked to InScribe's software**: every byte received on the Centronics port is
  XOR-decoded before the HP-GL parser sees it (below). Plain HP-GL sent to this plotter is garbage.
- **Motion planning is unchanged**: stepper ramp table and step-generation code are identical to
  the SPL-410 (only relocated addresses differ).
- InScribe's changes look like **binary patches** to Sekonic's firmware: NOP/JP over original
  instructions that are still present in the SPL-410.

## Hardware (from the board photos and code)

| Part | Function |
|---|---|
| U3 D27256 "InScribe R12.3" | firmware, `$0000–$7FFF` |
| U2 HM6116 | 2 KB RAM, `$C000–$C7FF` |
| U4, U5 (unpopulated) | **RAM expansion**: at boot `$0088` reads port `32` bit 3 and sets the stack to `$DFFF` (expanded) or `$C7FF` (2 KB only). Not font sockets. |
| 8251-type USART, ports `00`/`01` | RS-232 (serial parts appear to have been removed from this board) |
| 8255 #1, ports `10–13`, mode `9A` | Centronics: port A (`10`) = data in, port C (`12`) = handshake/status |
| 8255 #2, ports `30–33`, mode `89` | stepper phases (`30`/`31`), pen solenoid (port `31` bits 6–7), inputs (`32`) |
| port `20` | DIP switches |

## The input lock

Centronics receive, interrupt path `$00D9` → `$0B70` → `$0B77`:

```
0B77  in   a,(10h)      ; byte from the Centronics data port
0B79  or   a
0B7A  jp   m,0B7Fh      ; bit 7 set -> skip the first XOR
0B7D  xor  0E9h
0B7F  xor  0A4h
0B81  ret
```

| Received byte | Decoded value |
|---|---|
| `80–FF` (bit 7 set) | byte XOR `A4` |
| `00–7F` (bit 7 clear) | byte XOR `4D` (= `E9` XOR `A4`) |

The SPL-410 at the same point (`$0171`) stores the byte unchanged: `in a,(10h) / ld (C1F5h),a`.

**To encode for this plotter**, each plain character `c` (`00–7F`) can be sent either way;
both decode to `c`:

- 8-bit form: `c XOR A4` (always `80–FF`)
- 7-bit form: `c XOR 4D` (always `00–7F`)

Example: `IN;` → `ED EA 9F` (8-bit form) or `04 03 76` (7-bit form).

```python
def inscribe_encode(hpgl: bytes) -> bytes:
    return bytes(b ^ 0xA4 for b in hpgl)        # 8-bit form

open('out.bin', 'wb').write(inscribe_encode(open('plot.hpgl', 'rb').read()))
```

Not yet tested on hardware. Which form InScribe's software used is unknown.

### Possible unlock adapter

A passive cable cannot undo the lock — rewiring only permutes data lines, and the decoder needs
some lines **inverted**. A small active adapter can. HP-GL is 7-bit ASCII (host bit 7 always 0),
so a fixed inversion of a few data lines is enough:

| Adapter | Inverts | Centronics pins | Plotter receives | Decoder path | Result |
|---|---|---|---|---|---|
| A — 3 inverters | D7, D5, D2 | 9, 7, 4 | `c XOR A4` (bit 7 → 1) | XOR `A4` | `c` |
| B — 4 inverters | D6, D3, D2, D0 | 8, 5, 4, 2 | `c XOR 4D` (bit 7 stays 0) | XOR `4D` | `c` |

All other data lines and the handshake lines pass straight through. One 74LS04 (hex inverter),
74LS86 (XOR) or 74LS240 (inverting buffer) would do it, small enough for a connector hood. It needs
+5 V — commonly Centronics pin 18 on the device side, or from the host; not known for this plotter.

Either adapter only passes `00–7F` correctly: a byte with bit 7 set from the host would arrive
XORed with `E9`. Irrelevant for HP-GL, but 8-bit label characters would come out wrong.

**Checking a found cable/adapter:**

- look for a chip in the connector hood or a small inline box
- continuity-test data pins 2–9 (D0–D7) end to end: straight lines beep; the inverted set should
  be {9, 7, 4} (A) or {8, 5, 4, 2} (B)
- check whether STROBE / BUSY / ACK and the remaining pins are wired one-to-one — a non-standard
  handshake pinout would fit the patch that stopped pen state being written to port `12` bit 3

The alternative is that InScribe's host software applied the XOR itself and the cable was
ordinary or merely rewired. Either works with this firmware. Without the original adapter, the
plotter can be used by encoding in software (above) or by building adapter A (three gates).

**Interface selection:** flag `$C052` is cleared at boot (`$025F`) → Centronics is the normal
input. Serial is only selected inside one routine (`$3D4B` sets it, `$3E21` clears it), probably
for host responses. Serial bytes (`$00CE`) are stored **without** the XOR.

## Differences from the SPL-410

Real code changes, with relocation (shifted addresses) ignored:

| SPL-400 address | SPL-400 (InScribe) | SPL-410 (stock) | Probable origin |
|---|---|---|---|
| `$0171`, `$0B70–$0B81` | Centronics byte → XOR decoder (above) | stored as received | InScribe patch |
| `$62D9`, `$6318` (pen up / pen down) | `NOP NOP` | `out (12h),a` — pen state copied to port `12` bit 3, a Centronics-side status line | InScribe patch |
| `$0703` | `NOP ×3`; status mask `and C0h` | `ld a,(C0F6h)`; `and CFh` | InScribe patch |
| `$0E30` | `NOP ×5` | `ld a,1 / call 46BFh` | InScribe patch (purpose unknown) |
| `$0E45` | `jp 0E78h` over the check | `cp 52h` ('R') `/ jp nz / call 0569h` | InScribe patch (purpose unknown) |
| `$1415–$14DB` (`CI`) | bad chord angle → falls back to 5°; radius conversion chosen by flag `$C05A` (`$27D3` or `$2826`) | bad chord angle → abandons the command; one conversion (`$27CD`) | unclear |
| `$42EC`, `$436A` (P1/P2 scale factors) | small ratios (±64) ×64 for precision, flag `$C110` | different test, ×256, flag `$C111` | Sekonic revision (recompiled, not patched) |
| `$4DCD–$4F65` (arc helper, `AA`/`AR`) | temporary in RAM `$C1C1` | same logic using register BC | Sekonic revision |
| `$6A` glyph `j` | **no dot**: `45 FF C3 B4 94 83 60` | dot added: `47 C7 45 FF C3 B4 94 83 60` | Sekonic fix |

Unchanged (apart from relocation):

- stepper acceleration ramp (1550 bytes, `$72FF–$790C`; SPL-410 `$7317`) and the table before it
- step-generation and stepping loop (`$6000–$653F`)
- pen-lift timing (two `$629E` delays with `CB`)
- HP-GL command set, character-set tables, all other glyphs (1739 of 1756 set/code entries
  identical; the 17 differences are all the `j` above)
- I/O port usage, except the two pen-state writes to port `12`

Pen routines for reference (SPL-400):

- **pen up** `$62C0`: port `31` |= `80`, port `12` &= `F7` (write NOPed), delay ×2
- **pen down** `$62E4`: port `31` &= `3F`, delay ×2, port `31` |= `40`, port `12` |= `08` (write NOPed)

## Font

- Format and extraction: see the docstring in `extract_font.py` (also handles the SPL-410 pair).
- Character-set table `$29C4`, lookup routine `$307E`, glyph interpreter `$2F28`, glyph data
  `$6978–$7110`.
- Glyph bytes: bit 7 pen down, bits 6–4 x, bits 3–0 y; `60` end; `FF` negative y; `FE lo hi`
  include sub-glyph; `FD`/`FB` back up one/two cells (non-spacing); `FC` x + 6.

## Open questions

- What the removed calls at `$0E30` (`$46BF`) and `$0E45` (`'R'` check → `$0569`) did.
- Whether InScribe's host interface used port `12` bit 3 for something else (would explain the
  pen-state patch).
- Which encoding form InScribe's software sent; untested on hardware.
- Whether an adapter existed (A or B above) or the host software did the XOR; whether the
  plotter supplies +5 V on Centronics pin 18.
- Where the handwriting font lives: InScribe's host software (disks / PC image) or a capture of
  the Centronics data stream would show it.
