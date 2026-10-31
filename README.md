# neotool3

A command-line tool for building Neo Geo (MVS/AES) ROMs: it converts BMP graphics
into sprite (C-ROM) and fix layer (S-ROM) tiles, writes maps and palettes as
assembler source, packs all graphics of a game into finished ROM files and
creates a MAME software list with checksums.

neotool3 was written for the Neo Geo game **NEUTRALIZER II** and is used for
its complete build. Every format it writes has been checked byte by byte
against what MAME expects.

- Author: Willi Baecker
- Platform: Windows (console program, written in PureBasic)
- Version: see `neotool3 -ver`
- Download: [Releases](https://github.com/wbaecker/neotool3/releases)

---

## Contents

1. [Quick start](#quick-start)
2. [General rules](#general-rules)
3. [Graphics requirements](#graphics-requirements)
4. [Command reference](#command-reference)
5. [The pack file](#the-pack-file)
6. [Generated files](#generated-files)
7. [Using neotool3 with make](#using-neotool3-with-make)
8. [Running the result in MAME](#running-the-result-in-mame)
9. [Error codes](#error-codes)
10. [Antivirus warnings](#antivirus-warnings)
11. [Support](#support)
12. [License](#license)

---

## Quick start

```
neotool3 -help                 list of all commands
neotool3 -help -pack           help for one command
neotool3 -pack mygame.pack     build C1/C2, S1, maps, palettes and includes
neotool3 -f prom.bin p1.p1     byte-swap the program ROM for MAME
```

A typical game build needs only three calls:

```
neotool3 -pack mygame.pack                        graphics -> C1, C2, S1 + includes
neotool3 -p    prom.bin prom_pad.bin 512 $FF       pad the program to 512 KB
neotool3 -f    prom_pad.bin mygame-p1.p1           byte-swap for MAME
```

---

## General rules

- **Syntax:** `neotool3 <command> <parameters> [-quiet]`. Commands start with `-`.
  Missing or extra parameters are an error (5) and print the usage line.
- **`-quiet`** is allowed with every command. It hides the info lines only;
  errors, warnings and detail lines are always shown.
- **Messages** are one line each:
  - `error N: text` – the command failed, nothing usable was written
  - `warning: text` – the command succeeded, but check the result
  - `> text` – info line
- **Exit code** = error number (0 = success, always 1..255 on failure), so
  `make` and batch files stop on the first error.
- **Input and output must differ.** Writing a file onto itself is error 6.
- **On error, no half-written files remain.** Outputs started by a failed command
  are deleted; `-pack` deletes *all* of its outputs, including the dependency file.
- **Reproducible output:** text files contain no date or time. The same input
  always gives byte-identical output, so a build without changes produces no
  differences in version control.
- **Text files** use CRLF line endings.

---

## Graphics requirements

All graphics commands (`-fix`, `-spr`, `-img`, `-pal`, `-palcheck`, `-pack`) read BMP files:

- **8-bit indexed** (256-colour palette in the file)
- **only colour indices 0..15** may be used. A pixel with a higher index is
  error 21; the message lists the wrong pixels (x, y counted from the top).
- **colour 0 is transparent** on the Neo Geo
- width and height: multiples of **16** for sprites (`-spr`, `-img`),
  multiples of **8** for the fix layer (`-fix`)
- **Animations go in one row:** all phases side by side, the image exactly one
  phase high. Then a `-img` map is automatically ordered phase by phase.

Neo Geo colours have 5 bits per channel; neotool3 converts the 8-bit RGB values
of the BMP palette (including the "dark bit").

---

## Command reference

| Command | Parameters | Purpose |
|---|---|---|
| `-help` | `[-command]` | list of commands, or help for one command |
| `-ver` | | print the version |
| `-f` | `<in> <out>` | swap the two bytes of each word (P-ROM for MAME) |
| `-p` | `<in> <out> <KB> <padbyte>` | pad a file to a size in KB |
| `-b` | `<in> <out1> <out2>` | split byte by byte |
| `-w` | `<in> <out1> <out2>` | split word by word |
| `-h` | `<in> <out1> <out2>` | split into two halves |
| `-m` | `<in1> <in2> <out>` | append `<in2>` to `<in1>` |
| `-s` | `<in> <out>` | swap first and second half |
| `-pal` | `<bmp> <out> <palindex> <label>` | palette as assembler source |
| `-palcheck` | `<bmp1> <bmp2>` | compare colours 0..15 of two BMPs |
| `-fix` | `<bmp> <out>` | fix layer tiles 8x8 (S-ROM format) |
| `-spr` | `<bmp> <out>` | sprite tiles 16x16 (C-ROM format), every tile kept |
| `-img` | `<bmp> <out> <map.s> <label> [<WxH> [<phases>]]` | optimized sprite tiles + map |
| `-dtl` | `<in> <out> <tileid> <count>` | delete tiles from a tile file |
| `-pack` | `<packfile>` | build all graphics of a game from one description |
| `-xml` | `<out.xml> <name> <p1> <s1> <m1> <v1> <c1> <c2>` | MAME software list with CRC32/SHA1 |

### -f – byte swap

```
neotool3 -f <infile> <outfile>
```
Swaps the two bytes of every word. The 68000 program ROM must be stored this
way for MAME. With an odd file size the last byte is copied unchanged and a
warning is printed (exit code 0).

### -p – pad

```
neotool3 -p <infile> <outfile> <padsize> <padbyte>
```
Pads `<infile>` to `<padsize>` KB (128 = 131072 bytes). `<padbyte>` is 0..255,
written as decimal (`255`), hex (`$FF`) or binary (`%11111111`); a bare `ff` is
an error. If the input is already larger, the error message shows both sizes.

### -b, -w, -h, -m, -s – split, join, swap

```
neotool3 -b <infile> <outfile1> <outfile2>    even bytes -> out1, odd bytes -> out2
neotool3 -w <infile> <outfile1> <outfile2>    words 0,2,4.. -> out1, words 1,3,5.. -> out2
neotool3 -h <infile> <outfile1> <outfile2>    first half -> out1, second half -> out2
neotool3 -m <infile1> <infile2> <outfile>     out = in1 followed by in2
neotool3 -s <infile> <outfile>                halves swapped
```
- `-b`: with an odd size `<outfile1>` gets one byte more (warning).
- `-w`: the size must be a multiple of 4. Typical use: split `-spr` output into C1 and C2.
- `-h`, `-s`: the size must be even.

### -pal – palette

```
neotool3 -pal <infile.bmp> <outfile.s> <palindex> <labelname>
```
Writes 16 Neo Geo colours as `dc.w` lines under `<labelname>:`.
`<palindex>` selects a block of 16 colours of the BMP palette
(0 = colours 0..15, 1 = 16..31, ...); `all` writes all 16 blocks.

```
explosion_palette:
	dc.w	$8000, $7EEE, $0500, $4A10, $4C40, $4C72, $6EB2, $0445
	dc.w	$0272, $22B3, $0236, $132C, $148E, $0222, $0778, $7BBC
```

### -palcheck – compare palettes

```
neotool3 -palcheck <infile1.bmp> <infile2.bmp>
```
Compares colours 0..15 of both BMPs. Exit code 0 if they are equal.
Useful for graphics that share one hardware palette (`-pack` does this check
automatically).

### -fix – fix layer tiles

```
neotool3 -fix <infile.bmp> <outfile>
```
Converts the image into 8x8 fix layer tiles (S-ROM format), row by row.

### -spr – sprite tiles

```
neotool3 -spr <infile.bmp> <outfile>
```
Converts the image into 16x16 sprite tiles, row by row, every tile kept
(no optimization). The output holds C1 and C2 interleaved word by word;
split it with `-w`.

### -img – optimized tiles and map

```
neotool3 -img <infile.bmp> <outfile> <mapfile.s> <labelname> [<WxH> [<phases>]]
```
Like `-spr`, but identical tiles are stored only once. The map (`<mapfile.s>`)
lists, column by column, which tile belongs to each 16x16 cell; the label of
the map is `<labelname>` (required).

- `<WxH>`: size of one animation phase in pixels, e.g. `32x32`. The map is then
  written phase by phase, and phase constants are added (see
  [Map files](#map-files)).
- `<phases>`: use only the first n phases (e.g. when the last phase repeats
  the first one).

### -dtl – delete tiles

```
neotool3 -dtl <infile> <outfile> <tileid> <numtiles>
```
Deletes `<numtiles>` tiles starting at `<tileid>` (first tile = 0) from a tile
file created by `-spr` or `-img`. The tiles behind the deleted range move
forward; their numbers decrease by `<numtiles>`. The range must lie inside the file.

### -pack – build all graphics

```
neotool3 -pack <packfile>
```
Reads one text file that describes all graphics of the game and writes
C1, C2, S1, all maps and palettes and the include files in one run.
See [The pack file](#the-pack-file).

### -xml – MAME software list

```
neotool3 -xml <outfile.xml> <name> <p1> <s1> <m1> <v1> <c1> <c2>
```
Writes a MAME software list for the `neogeo` system with size, CRC32 and SHA1
of the six ROM files. `<name>` is the short name in MAME: lowercase letters
a-z, digits and `_`, at most 16 characters. All ROM files must exist.

---

## The pack file

The pack file is the heart of a game build. One entry per line, lines starting
with `#` are comments. Graphics are stored in the ROM **in the order of the file**,
without gaps.

### Settings

| Line | Meaning |
|---|---|
| `start <tile>` | first C-ROM tile |
| `size <KB>` | size of each C-ROM (C1 and C2) in KB |
| `fstart <tile>` | first S-ROM tile |
| `fsize <KB>` | size of the S-ROM in KB |
| `crom1 <file>` / `crom2 <file>` | C1 / C2 output |
| `srom1 <file>` | S1 output |
| `tile <file>` | include file with `TILE_<NAME>` and `TILE_<NAME>_NUM` |
| `fixinc <file>` | include file with `FIX_<NAME>` and `FIX_<NAME>_NUM` |
| `mapdir <folder>` | folder for map and palette source files |
| `dep <file>` | dependency file for make |
| `quiet` | same as `-quiet` |

### Palettes

```
<label>  <file.bmp>  pal
```
Writes `<bmpname>_pal.s` into `mapdir` with the label `<label>` (colours 0..15).

### Graphics

```
<NAME>  <file.bmp>  spr  <palette>
<NAME>  <file.bmp>  img  <palette>  [<WxH> [<phases>]]
<NAME>  <file.bmp>  fix  <palette>
```
- `spr`: sprite tiles, every tile kept
- `img`: optimized tiles + `<bmpname>_map.s` in `mapdir`
- `fix`: fix layer tiles (S-ROM)
- `<palette>` is the label of a `pal` line. Colours 0..15 of the graphic must
  match that palette; `-pack` stops at the first mismatch. This catches a
  graphic that was saved with a slightly different palette before it shows up
  in the wrong colours on screen.

Each BMP may appear only once. If C-ROM or S-ROM overflow, it is an error.

### Example

```
# mygame.pack
quiet
start 256
size 1024
fstart 768
fsize 128
crom1   build/rom/mygame-c1.c1
crom2   build/rom/mygame-c2.c2
srom1   build/rom/mygame-s1.s1
tile    build/obj/crom_tiles.inc
fixinc  build/obj/fix_tiles.inc
mapdir  build/obj/
dep     build/obj/pack.d

# palettes
ship_palette     gfx/ship.bmp     pal
enemy_palette    gfx/mutant.bmp   pal
font_palette     gfx/font8.bmp    pal

# fix layer
FONT8    gfx/font8.bmp     fix  font_palette

# sprites
SHIP     gfx/ship.bmp      spr  ship_palette
MUTANT   gfx/mutant.bmp    spr  enemy_palette
WALKER   gfx/walker.bmp    img  enemy_palette  32x32
BOSS     gfx/boss.bmp      img  ship_palette   64x64  15
```

---

## Generated files

### Map files

`-img` and `img` lines in the pack file write a map like this
(`testimage.bmp`, 384x32 pixels = 3 phases of 128x32, `img ... 128x32`):

```
; ----------------------------------------------------------------------------------------------
; Converted via neotool3.exe written 2025/26 by Willi Baecker / Factor 5 - retro division cologne
; Filename: data/testimage.bmp
; ----------------------------------------------------------------------------------------------

TESTIMAGE_TILESX              EQU    24
TESTIMAGE_TILESY              EQU     2
TESTIMAGE_TILESNUM            EQU    44
TESTIMAGE_PHASES              EQU     3
TESTIMAGE_PHASEX              EQU     8
TESTIMAGE_PHASEY              EQU     2
TESTIMAGE_PHASESIZE           EQU    16

; ----------------------------------------------------------------------------------------------

testimage_map:
; Phase: 0
		dc.b	  0,  22
		dc.b	  1,  23
		...
		dc.b	  7,  29
; Phase: 1
		dc.b	  0,  22
		dc.b	  8,  30
		...
; Phase: 2
		...
		dc.b	 21,  43

; ----------------------------------------------------------------------------------------------
; End of auto generated source code map file.
; ----------------------------------------------------------------------------------------------
```

| Constant | Meaning |
|---|---|
| `_TILESX`, `_TILESY` | size of the image in tiles |
| `_TILESNUM` | number of **different** tiles stored in the ROM |
| `_PHASES` | number of phases (only with `<WxH>`) |
| `_PHASEX`, `_PHASEY` | size of one phase in tiles |
| `_PHASESIZE` | map entries per phase (`_PHASEX` * `_PHASEY`) |

Each `dc.b` line is one column of 16x16 cells, top to bottom. With a phase size
the map is written phase by phase (marked with `; Phase: n`), so phase n starts
at entry n * `_PHASESIZE`. The values are tile numbers relative to the first
tile of the graphic (`TILE_<NAME>` in the pack build). In the example the first
column is the same in all three phases (tiles 0 and 22), so only 44 of the
48 cells need their own tile.
With more than 256 different tiles the map uses `dc.w`.

### crom_tiles.inc / fix_tiles.inc

```
TILE_MUTANT equ 256
TILE_MUTANT_NUM equ 6
TILE_SHIP equ 262
TILE_SHIP_NUM equ 10
```
For `img` graphics `_NUM` is the number after optimization. Your code never
needs hand-written tile numbers.

### Dependency file

The `dep` line writes `PACK_INPUTS` (all BMPs) and `PACK_OUTPUTS` (all written
files) in make syntax.

---

## Using neotool3 with make

```make
-include build/obj/pack.d

$(PACK_OUTPUTS): mygame.pack $(PACK_INPUTS)
	neotool3 -pack mygame.pack
```

The first build has no `pack.d` yet; make then runs `-pack` anyway because the
include files are missing. After that, changing any BMP or the pack file
rebuilds the graphics, and nothing else does. Because neotool3 stops with a
non-zero exit code on every error, make stops too.

---

## Running the result in MAME

```
neotool3 -xml build/hash/neogeo.xml mygame p1.p1 s1.s1 m1.m1 v1.v1 c1.c1 c2.c2
mame neogeo -cart1 mygame -rompath build -hashpath build/hash
```
The ROM files must be in `build/neogeo/mygame/`. With the checksums in the
software list MAME starts the game without a warning.

---

## Error codes

The numbers never change between versions, so scripts can rely on them.

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | input file not found / cannot be opened |
| 2 | unknown command or no command given |
| 3 | illegal file size (e.g. input too small) |
| 4 | cannot create output file |
| 5 | missing, unexpected or invalid parameter |
| 6 | input and output are the same file |
| 7 | file size is not even |
| 8 | file size is not a multiple of 4 |
| 9 | pad size too small (input is already larger) |
| 10 | illegal BMP header |
| 11 | BMP could not be processed |
| 12 | BMP is compressed (only uncompressed BMPs are supported) |
| 13 | illegal image width |
| 14 | illegal image height |
| 15 | illegal colour depth (BMP must be 8-bit indexed) |
| 16 | illegal tile number / range (-dtl) |
| 17 | reserved (formerly -stm, no longer used) |
| 18 | reserved |
| 19 | reserved |
| 20 | illegal palette |
| 21 | colour index above 15 (the wrong pixels are listed) |
| 22 | palette check failed (colours 0..15 differ) |
| 23 | illegal label name |
| 24 | directory not found |
| 25 | unknown line in the pack file |
| 26 | C-ROM overflow |
| 27 | S-ROM overflow |
| 28 | pack file: image width does not fit the phase size |
| 29 | pack file: image height does not fit the phase size |
| 30 | pack file: illegal number of phases |
| 31 | pack file: palette error (e.g. unknown palette label) |
| 32 | cannot read data from file |
| 99 | internal error in the command table |

`neotool3 -help -<command>` lists the checks of each command.

---

## Antivirus warnings

neotool3 is an unsigned program. A few antivirus products with
machine-learning detection may report it as suspicious (for example
"Generic ML PUA"). These are false positives: the detection is based on the
fact that the program is new and unsigned, not on anything found in it.
neotool3 does not access the network and only writes the files you name.

Every release lists the SHA-256 checksum of `neotool3.exe`. Compare it with
your download:

```
certutil -hashfile neotool3.exe SHA256
```

---

## Support

neotool3 is free and stays free – there is nothing to pay and no feature is
held back.

If it saved you time and you would like to say thank you, you can buy me a
coffee: **[paypal.me/wbaecker1941](https://paypal.me/wbaecker1941)**

Just as welcome: a mention in your game's credits, a bug report, or a
screenshot of what you built with it.

---

## License

neotool3 is **freeware**. Copyright (c) 2025-2026 Willi Baecker. All rights reserved.

- You may use neotool3 free of charge for any purpose, including commercial
  projects. The ROMs and files you create with it belong to you.
- You may pass on the unchanged program together with this README.
- You may not sell neotool3, modify it, decompile it or distribute it as part
  of another tool without permission.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED. IN NO EVENT SHALL THE AUTHOR BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY ARISING FROM THE USE OF THE SOFTWARE.
