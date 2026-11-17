# VIC20DD26 – VIC-20 kernal for DolphinDos26

Kernal for the VIC-20 (VC20) modelled on the C64 kernal DolphinDos26 (`../C64DD26`).
It is based on the Commodore kernal 901486-06 (NTSC) and 901486-07 (PAL) as commented
ca65 source code. Cassette and RS-232 are removed; the space gained holds the
transfer routines and the operating functions.

| File | TV standard | Contents |
|---|---|---|
| `build/vic20dd26-pal.bin` → `Builds/VIC20DolphinDos26PAL.bin` | PAL | 8 KB, `$E000–$FFFF` |
| `build/vic20dd26-ntsc.bin` → `Builds/VIC20DolphinDos26NTSC.bin` | NTSC | 8 KB, `$E000–$FFFF` |

## Building

```
make            builds both kernals, lists the free areas
make builds     puts them into ../Builds as VIC20DolphinDos26PAL.bin / VIC20DolphinDos26NTSC.bin
make clean
```

The source code in `src/vic20dd` was generated once from the Commodore ROM and the Commodore sources
(`Referenzen/cbmsrc/KERNAL_VIC_04`) and has been maintained by hand since. All routines taken over
are at their original address; free areas carry a label `frei_xxxx` and are filled with `$AA`.

## Transfer methods

On every access the kernal itself chooses the fastest method the device supports:

| Device | Method | Display when loading |
|---|---|---|
| 1541/1571 with DolphinDos26 (Multi-Speeder) and parallel cable | DolphinDOS parallel, block transfer | `LOADING DD: $1201-$xxxx` |
| the same, with `LOAD"name",8,2` or directory | parallel, byte by byte | `LOADING DD:` |
| JiffyDOS drive, SD2IEC | JiffyDOS, fast loading | `LOADING JD: $1201-$xxxx` |
| the same, with `,8,2` or directory | JiffyDOS, byte by byte | `LOADING JD:` |
| all others | serial as in the original | `LOADING` |

The JiffyDOS routines come from the JiffyDOS kernal of the VIC-20 (PAL and NTSC have their own
timing loops). The parallel routines are ported from the C64 kernal DolphinDos26 to VIA 1.

### Parallel cable on the VIC-20

The user port of the VIC-20 has the eight data lines on the same pins as the C64, but the
acknowledge line to the drive is on a different pin:

| Signal | C64 | VIC-20 |
|---|---|---|
| Data D0–D7 | PB0–PB7, pins C–L | PB0–PB7 of VIA 1, pins C–L |
| Drive → computer (“byte ready”) | FLAG2, pin B | CB1, pin B |
| Computer → drive (acknowledge) | PC2, **pin 8** | CB2, **pin M** |

A C64 parallel cable therefore does not fit unchanged: the wire from pin 8 must go to pin M on the
VIC-20 (adapter or a cable of its own). Pin 8 is the cassette switch on the VIC-20, pin M is a
different line on the C64 – one cable for both computers is not possible.

Differences from the C64 in the sequence:

- The CIA of the C64 generates a pulse on PC2 with every read or write of the port. The VIA does so
  only on a write. During block transfer CB2 is therefore set to “pulse output”, and every byte
  read is followed by a write access to `$9110`.
- The flag for the signal of the drive (IFR bit 4) is not cleared by testing it, but by the access
  to `$9110`.
- During cable detection (after every secondary address) the kernal switches CB2 by hand through
  the PCR. The drive expects two pulses about 20 to 27 µs apart; the loop takes 24 cycles
  (PAL 21.7 µs, NTSC 23.5 µs).
- Port B, its data direction and the PCR are saved before parallel operation and restored
  afterwards.

## Operation

- **Power-on screen** with `DOLPHINDOS26  VIC-20`, `JIFFYDOS/SD2IEC` and `PAL` or `NTSC`.
- **`@` commands** (in direct mode): `@` message of the drive, `@command` command to the drive
  followed by its message, `@$` and `@$:pattern` directory without losing the program, `@8` to `@19`
  select the default device.
- **Default device:** `LOAD`, `SAVE` and `VERIFY` without a device number use the default device
  (preset 8, cassette in the original). `LOAD` without a name loads the first file (`:*`).
- **CTRL+D** changes in direct mode to the next drive that is present (devices 8 to 19) and shows
  its number, as with JiffyDOS. During the search the command channel is closed on every drive
  that is checked.
- **Function keys:**

  | Key | Effect |
  |---|---|
  | F1 | clear screen, `LIST` |
  | F2 | `RUN:` |
  | F3 | clear screen, `RUN` |
  | F4 | `VERIFY` |
  | F5 | `LOAD` (first file) |
  | F6 | `SAVE"@:` |
  | F7 | clear screen, directory (`@$`) |
  | F8 | message of the drive (`@`) |
  | C= + F1 | `OPEN4,4:CMD4:LIST` |
  | C= + F3 | `@CD←` – SD2IEC: one directory back |
  | C= + F5 | on a directory line: change into this directory or disk image |
  | C= + F7 | on a directory line: write `LOAD` in front of the name (press RETURN yourself) |

- **`&` commands** (in direct mode): `&` restores the default assignment of the function keys,
  `&"name"` loads your own assignment from disk (twelve texts, each ended with `$00`; the pointer
  then points to the load address of the file), `&*` brings back a program deleted with `NEW`.
- **`SYS`** also accepts `$hex` (`SYS$A000`). `SYS 1` to `SYS 15` jump to `$1000` to
  `$F000`, `SYS 0` executes a BRK (BASIC warm start).
- **`VERIFY`** stops at the first difference.
- **Additional CTRL keys:**

  | Key | Effect |
  |---|---|
  | CTRL-A | key repeat for all keys on/off |
  | CTRL-B | cursor to the start of the bottom screen line |
  | CTRL-G | cursor 11 columns to the right (tabulator) |
  | CTRL-K | delete the line from the cursor to the end |
  | CTRL-L | delete the line from the start to the cursor |
  | CTRL-O / CTRL-P | delete the character left of the cursor into a buffer / insert it again |
  | CTRL-V | reset default colour, screen and input/output |
  | CTRL-X | function keys off, default device back to 8 (on again with `&`) |
  | CTRL-* | print the screen on the printer (device 4) |
  | RUN/STOP | ends quote mode and insert mode |

  CTRL-A, -B, -G, -K and -L also work as control characters in `PRINT` (`CHR$(1)`, `2`, `7`, `11`, `12`).
- **Halting the scrolling:** As long as CTRL is held down, the output stands still. The C= key
  freezes it until CTRL is pressed. (The slowed-down scrolling of the original with CTRL is dropped
  in exchange.)
- **Reset keys:** RUN/STOP held during reset = BASIC warm start, the program is kept.
  Space bar during reset = cold start without starting a cartridge at `$A000`.
- **Removed:** cassette and RS-232 of the kernal. Devices 1 and 2 report `ILLEGAL DEVICE NUMBER`.
  The cassette motor control in the interrupt is removed as well.

**Not** taken over from the C64 kernal, because there is not enough space (about 100 bytes are
still free, in small pieces): `&number` (hexadecimal/decimal conversion), `SYS` without an argument,
the pause with CTRL-RETURN, the reset keys Q and ←+1 and the RESTORE keys, the colour bars in the
power-on screen.

## Memory cells

| Address | Name | Meaning |
|---|---|---|
| `$0293` | `dd_flags` | bit 7: parallel operation |
| `$0294` | `dd_defdev` | default device (0 = 8) |
| `$0295`–`$0297` | `dd_sav_prb`, `dd_sav_ddrb`, `dd_sav_pcr` | saved state of VIA 1 |
| `$0298` | `dd_tmp` | temporary storage |
| `$0299`–`$029A` | `fkey_ptr` | pointer to the texts of the function keys (high byte 0 = off) |
| `$029B` | `cpy_cnt` | number of characters remembered with CTRL-O |
| `$033C`–`$03BB` | `cpy_buf` | remembered characters (formerly the cassette buffer) |

(formerly working cells of the kernal RS-232)

## Tests

```
python3 tests/test_proto.py beide --nur=cbm,jd,dd,ddo
python3 tests/test_bedienung.py pal
python3 tests/test_zusaetze.py pal
```

- `test_proto.py`: for each setup (Commodore drive, JiffyDOS, DolphinDos26 with and without parallel
  cable) LOAD of a 20 KB program with byte comparison, SAVE, VERIFY, LOAD with `,8,2`, directory and
  comparison of the saved file on the disk; each for PAL and NTSC with 32 KB expansion.
- `test_bedienung.py`: power-on screen, `@` commands, device selection, function keys, CTRL+D.
- `test_zusaetze.py`: `&` commands, own function keys, CTRL-X, `SYS`, the CTRL keys of the editor, `VERIFY`.

## Origin

- Kernal: Commodore 901486-06/-07.
- JiffyDOS transfer routines: JiffyDOS for the VIC-20 (CMD / RETRO Innovations). Anyone who uses the
  kernal needs a JiffyDOS licence.
- Parallel routines and operation: ported from the C64 kernal DolphinDos26.
