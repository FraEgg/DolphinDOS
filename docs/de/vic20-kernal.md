# VIC20DD26 – VIC-20-Kernal für DolphinDos26

Kernal für den VIC-20 (VC20) nach dem Vorbild des C64-Kernals DolphinDos26 (`../C64DD26`).
Grundlage ist der Commodore-Kernal 901486-06 (NTSC) bzw. 901486-07 (PAL) als kommentierter
ca65-Quelltext. Kassette und RS-232 sind entfernt; in dem frei gewordenen Platz stehen die
Übertragungsroutinen und die Bedienfunktionen.

| Datei | Fernsehnorm | Inhalt |
|---|---|---|
| `build/vic20dd26-pal.bin` → `Builds/VIC20DolphinDos26PAL.bin` | PAL | 8 KB, `$E000–$FFFF` |
| `build/vic20dd26-ntsc.bin` → `Builds/VIC20DolphinDos26NTSC.bin` | NTSC | 8 KB, `$E000–$FFFF` |

## Bauen

```
make            baut beide Kernals, listet die freien Bereiche
make builds     legt sie als VIC20DolphinDos26PAL.bin / VIC20DolphinDos26NTSC.bin in ../Builds ab
make clean
```

Der Quelltext in `src/vic20dd` wurde einmal aus dem Commodore-ROM und den Commodore-Quellen erzeugt
(`Referenzen/cbmsrc/KERNAL_VIC_04`) und wird seitdem von Hand gepflegt. Alle übernommenen Routinen
stehen an ihrer Originaladresse; freie Bereiche tragen ein Label `frei_xxxx` und sind mit `$AA` gefüllt.

## Übertragungsarten

Der Kernal wählt bei jedem Zugriff selbst das schnellste Verfahren, das das Gerät beherrscht:

| Gerät | Verfahren | Anzeige beim Laden |
|---|---|---|
| 1541/1571 mit DolphinDos26 (Multi-Speeder) und Parallelkabel | DolphinDOS parallel, Blockübertragung | `LOADING DD: $1201-$xxxx` |
| ebenso, mit `LOAD"name",8,2` oder Directory | parallel, byteweise | `LOADING DD:` |
| JiffyDOS-Laufwerk, SD2IEC | JiffyDOS, Schnellladen | `LOADING JD: $1201-$xxxx` |
| ebenso, mit `,8,2` oder Directory | JiffyDOS, byteweise | `LOADING JD:` |
| alle anderen | seriell wie beim Original | `LOADING` |

Die JiffyDOS-Routinen stammen aus dem JiffyDOS-Kernal des VIC-20 (PAL und NTSC haben eigene
Zeitschleifen). Die Parallelroutinen sind vom C64-Kernal DolphinDos26 auf die VIA 1 übertragen.

### Parallelkabel am VIC-20

Der Userport des VIC-20 hat die acht Datenleitungen an denselben Pins wie der C64, die Quittung zur
Floppy liegt aber an einem anderen Pin:

| Signal | C64 | VIC-20 |
|---|---|---|
| Daten D0–D7 | PB0–PB7, Pins C–L | PB0–PB7 der VIA 1, Pins C–L |
| Floppy → Rechner („Byte bereit“) | FLAG2, Pin B | CB1, Pin B |
| Rechner → Floppy (Quittung) | PC2, **Pin 8** | CB2, **Pin M** |

Ein C64-Parallelkabel passt deshalb nicht unverändert: die Ader von Pin 8 muss am VIC-20 an Pin M
liegen (Adapter oder eigenes Kabel). Pin 8 ist am VIC-20 der Kassettenschalter, Pin M am C64 eine
andere Leitung – ein Kabel für beide Rechner ist nicht möglich.

Unterschiede zum C64 im Ablauf:

- Der CIA des C64 erzeugt bei jedem Lesen oder Schreiben des Ports einen Puls an PC2. Die VIA tut das
  nur beim Schreiben. Bei der Blockübertragung steht CB2 deshalb auf „Pulsausgang“, und nach jedem
  gelesenen Byte folgt ein Schreibzugriff auf `$9110`.
- Der Merker für das Signal der Floppy (IFR Bit 4) wird nicht beim Abfragen gelöscht, sondern durch
  den Zugriff auf `$9110`.
- Bei der Kabelerkennung (nach jeder Sekundäradresse) schaltet der Kernal CB2 von Hand über das PCR.
  Die Floppy erwartet zwei Impulse im Abstand von etwa 20 bis 27 µs; die Schleife dauert 24 Takte
  (PAL 21,7 µs, NTSC 23,5 µs).
- Port B, seine Datenrichtung und das PCR werden vor dem Parallelbetrieb gesichert und danach
  wiederhergestellt.

## Bedienung

- **Einschaltbild** mit `DOLPHINDOS26  VIC-20`, `JIFFYDOS/SD2IEC` und `PAL` bzw. `NTSC`.
- **`@`-Befehle** (im Direktmodus): `@` Meldung des Laufwerks, `@befehl` Befehl an das Laufwerk mit
  anschließender Meldung, `@$` und `@$:muster` Directory ohne Verlust des Programms, `@8` bis `@19`
  Standardgerät wählen.
- **Standardgerät:** `LOAD`, `SAVE` und `VERIFY` ohne Gerätenummer benutzen das Standardgerät
  (Vorgabe 8, beim Original Kassette). `LOAD` ohne Namen lädt die erste Datei (`:*`).
- **CTRL+D** wechselt im Direktmodus zum nächsten vorhandenen Laufwerk (Geräte 8 bis 19) und zeigt
  dessen Nummer an, wie bei JiffyDOS. Bei der Suche wird an jedem geprüften Laufwerk der Befehlskanal
  geschlossen.
- **Funktionstasten:**

  | Taste | Wirkung |
  |---|---|
  | F1 | Bildschirm löschen, `LIST` |
  | F2 | `RUN:` |
  | F3 | Bildschirm löschen, `RUN` |
  | F4 | `VERIFY` |
  | F5 | `LOAD` (erste Datei) |
  | F6 | `SAVE"@:` |
  | F7 | Bildschirm löschen, Directory (`@$`) |
  | F8 | Meldung des Laufwerks (`@`) |
  | C= + F1 | `OPEN4,4:CMD4:LIST` |
  | C= + F3 | `@CD←` – SD2IEC: ein Verzeichnis zurück |
  | C= + F5 | auf einer Directory-Zeile: in dieses Verzeichnis oder Disk-Image wechseln |
  | C= + F7 | auf einer Directory-Zeile: `LOAD` vor den Namen schreiben (RETURN selbst drücken) |

- **`&`-Befehle** (im Direktmodus): `&` stellt die Standardbelegung der Funktionstasten wieder her,
  `&"name"` lädt eine eigene Belegung von Diskette (zwölf Texte, je mit `$00` beendet; der Zeiger
  steht danach auf der Ladeadresse der Datei), `&*` holt ein mit `NEW` gelöschtes Programm zurück.
- **`SYS`** nimmt auch `$hex` an (`SYS$A000`). `SYS 1` bis `SYS 15` springen nach `$1000` bis
  `$F000`, `SYS 0` führt einen BRK aus (BASIC-Warmstart).
- **`VERIFY`** hört beim ersten Unterschied auf.
- **Zusätzliche CTRL-Tasten:**

  | Taste | Wirkung |
  |---|---|
  | CTRL-A | Tastenwiederholung für alle Tasten ein/aus |
  | CTRL-B | Cursor an den Anfang der untersten Bildschirmzeile |
  | CTRL-G | Cursor 11 Spalten nach rechts (Tabulator) |
  | CTRL-K | Zeile ab dem Cursor bis zum Ende löschen |
  | CTRL-L | Zeile vom Anfang bis zum Cursor löschen |
  | CTRL-O / CTRL-P | Zeichen links vom Cursor in einen Zwischenspeicher löschen / wieder einfügen |
  | CTRL-V | Standardfarbe, Bildschirm und Ein-/Ausgabe zurücksetzen |
  | CTRL-X | Funktionstasten aus, Standardgerät zurück auf 8 (wieder ein mit `&`) |
  | CTRL-* | Bildschirm auf den Drucker (Gerät 4) ausgeben |
  | RUN/STOP | beendet Anführungszeichen- und Einfügemodus |

  CTRL-A, -B, -G, -K und -L wirken auch als Steuerzeichen in `PRINT` (`CHR$(1)`, `2`, `7`, `11`, `12`).
- **Scrollen anhalten:** Solange CTRL gedrückt ist, steht die Ausgabe. Die Taste C= friert sie ein,
  bis CTRL gedrückt wird. (Das verlangsamte Scrollen des Originals mit CTRL entfällt dafür.)
- **Reset-Tasten:** RUN/STOP beim Reset gedrückt = BASIC-Warmstart, das Programm bleibt erhalten.
  Leertaste beim Reset = Kaltstart ohne Start eines Moduls bei `$A000`.
- **Entfallen:** Kassette und RS-232 des Kernals. Gerät 1 und 2 melden `ILLEGAL DEVICE NUMBER`.
  Die Kassettenmotor-Steuerung im Interrupt ist ebenfalls entfernt.

Vom C64-Kernal **nicht** übernommen, weil der Platz nicht reicht (frei sind noch rund 100 Byte in
kleinen Stücken): `&zahl` (Umrechnung hexadezimal/dezimal), `SYS` ohne Argument, die Pause mit
CTRL-RETURN, die Reset-Tasten Q und ←+1 sowie die RESTORE-Tasten, die Farbbalken im Einschaltbild.

## Speicherzellen

| Adresse | Name | Bedeutung |
|---|---|---|
| `$0293` | `dd_flags` | Bit 7: Parallelbetrieb |
| `$0294` | `dd_defdev` | Standardgerät (0 = 8) |
| `$0295`–`$0297` | `dd_sav_prb`, `dd_sav_ddrb`, `dd_sav_pcr` | gesicherter Zustand der VIA 1 |
| `$0298` | `dd_tmp` | Zwischenspeicher |
| `$0299`–`$029A` | `fkey_ptr` | Zeiger auf die Texte der Funktionstasten (High-Byte 0 = aus) |
| `$029B` | `cpy_cnt` | Zahl der mit CTRL-O gemerkten Zeichen |
| `$033C`–`$03BB` | `cpy_buf` | gemerkte Zeichen (früher Kassettenpuffer) |

(früher Arbeitszellen des Kernal-RS-232)

## Tests

```
python3 tests/test_proto.py beide --nur=cbm,jd,dd,ddo
python3 tests/test_bedienung.py pal
python3 tests/test_zusaetze.py pal
```

- `test_proto.py`: je Aufbau (Commodore-Laufwerk, JiffyDOS, DolphinDos26 mit und ohne Parallelkabel)
  LOAD eines 20-KB-Programms mit Byte-Vergleich, SAVE, VERIFY, LOAD mit `,8,2`, Directory und
  Vergleich der gespeicherten Datei auf der Diskette; jeweils PAL und NTSC mit 32 KB Erweiterung.
- `test_bedienung.py`: Einschaltbild, `@`-Befehle, Gerätewahl, F-Tasten, CTRL+D.
- `test_zusaetze.py`: `&`-Befehle, eigene F-Tasten, CTRL-X, `SYS`, die CTRL-Tasten des Editors, `VERIFY`.

## Herkunft

- Kernal: Commodore 901486-06/-07.
- JiffyDOS-Übertragungsroutinen: JiffyDOS für den VIC-20 (CMD / RETRO Innovations). Wer den Kernal
  einsetzt, braucht eine JiffyDOS-Lizenz.
- Parallelroutinen und Bedienung: übertragen aus dem C64-Kernal DolphinDos26.
