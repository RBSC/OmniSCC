# OmniSCC — MSX MultiCart

Current firmware: **2.0.0**. See [release notes — changes since 1.0.0](RELEASE_NOTES.md).

> **Upgrading from 1.0.0:** back up every file on the cartridge before flashing.
> This release changes the flash layout and automatically formats a recognized
> old volume on its first boot. Restore your files afterwards. Install the new
> UF2 and menu ROM together; older menu signatures are no longer recognized.

OmniSCC is an RP2040-based MSX cartridge that loads ROM and floppy disk images
from the Pico's 16 MB flash. It emulates plain and mapped ROM cartridges, the
Konami SCC sound chip, and floppy drives backed by `.dsk` files.

**ROM-game saves:** KUC supports flash saves in a companion `.sav`
for ROMs explicitly named with `[KUC]`, on compatible 16 MiB SPI NOR hardware.
See [KUC setup, capacity and validation status](src/docs/kuc.md).
For an unknown flash profile, both menus accept `kuc_allow_unknown_flash=1`
(default `0`); the hardware requirements still apply. Other cartridge flash,
SRAM and EEPROM profiles remain unsupported. Writable `.dsk` games continue to
save into their disk image.

When powered from USB without the MSX, it appears as a removable drive for
copying images by drag-and-drop. On the MSX, the universal menu browses
files and folders and remembers the last booted image. It automatically uses
SCREEN 1 on MSX1 and SCREEN 5 on recognized MSX2+ BIOS versions with enough VRAM. An alternative menu
provides ten configurable favourites.

![OmniSCC](images/OmniSCC.jpg)

*OmniSCC — a 16 MB "purple" Pico clone soldered directly onto the carrier
board.*

---

## Contents of this repository

| Path | What it is |
|------|------------|
| [`firmware/`](firmware/) | Prebuilt firmware images. |
| [`tools/`](tools/) | Menu ROMs, and MSX-DOS utilities. |
| [`src/`](src/) | RP2040 firmware (C11 / C++17) + the Z80 side (`msx/preloader.asm`, `msx/loader.asm`, `msx/loader_fav.asm`). See [`src/README.md`](src/README.md). |
| [`board/OmniSCC/`](board/OmniSCC/) | KiCad project for the base board: schematic, PCB, 3D STEP, fabrication ZIP. |
| [`board/OmniSCC+/`](board/OmniSCC+/) | KiCad project for the level-shifted board. |
| [`enclosure/`](enclosure/) | Printable 3MF files for the base and OmniSCC+ cases. |
| [`images/`](images/) | Board photos used by this file. |

Two software git submodules are used, so clone with:

```bash
git clone --recursive https://github.com/RBSC/OmniSCC.git
# or, in an existing clone:
git submodule update --init --recursive
```

---

## Hardware variants

Both boards run the same firmware sources; they differ only in the board header
selected at build time (`-DPICO_BOARD=omniscc` or `omniscc_plus`).

### OmniSCC (base)

![OmniSCC solder detail](images/OmniSCC-solder.png)

*The Pico module is soldered flat against the carrier board along both pin
rows.*

The Pico drives the MSX bus directly at 3.3 V. Fewest parts, lowest cost. Three
3-pole solder jumpers pick what reaches the Pico on GP26–GP28:

| Jumper | Choice | Required setting for this firmware |
|--------|--------|------------------------------------|
| `IORQ` / `AUDIO` | GP26 | **AUDIO** — GP26 is the SCC PDM audio output. |
| `BUSDIR` / `M1` | GP27 | **BUSDIR** — the firmware never reads `/M1`. |
| `WAIT` / `INT` | GP28 | **WAIT** — the loader stretches every bus cycle with `/WAIT`. |

### OmniSCC+

![OmniSCC+](images/OmniSCC+.jpg)

*OmniSCC+ — same Pico module, plus SN74LVC8T245 bus transceivers for
proper 5 V level translation.*

Address, data and control lines run through level shifters, so the MSX sees
real 5 V logic. There are no option jumpers: GP26 is AUDIO, GP27 is the data
transceiver direction (`DATA_DIR`), GP28 is `/WAIT`. Build with
`-DPICO_BOARD=omniscc_plus`, which defines `WITH_SHIFTERS`.

---

Current [OmniSCC+ hardware revision 1.1](board/OmniSCC+/README.md) adds a
10 kΩ pull-up on `/RES` to address resets in USB mode when touching the connector.
Use the 1.1 fabrication archive for new boards.

## Quick start

1. **Flash the firmware.** Hold **BOOTSEL**, plug into USB, release; copy a
   `.uf2` to the `RPI-RP2` drive. Pre-built images live in
   [`firmware/`](firmware/) — `OmniSCC.uf2` for the base board,
   `OmniSCC+.uf2` for OmniSCC+. To build your own, see
   [`src/README.md`](src/README.md).
2. **Fill the drive.** Plug the cart into USB with the MSX powered off. A
   removable drive named `OmniSCC` appears (the first boot formats the flash and
   writes `WELCOME.TXT`). Copy [`tools/loader.rom`](tools/loader.rom)
   and your `.rom` files to its root. Keep only one `loader*.rom` in the root. To build the menu yourself, see
   [`src/README.md`](src/README.md).
3. **Nothing to configure.** The menu lists what is on the drive and writes
   `/loader.ini` itself when you boot something. Write it by hand only to set a
   countdown or a title before the first boot:

   ```ini
   delay=3000
   scc_on_exit=1
   scc_volume=100
   name=gradius.rom,Gradius
   ```

   For the ten-favourites menu, copy
   [`tools/loader_fav.rom`](tools/loader_fav.rom) to the drive
   **under its original name**. Copy `loader_fav.ini.example` to the drive as
   `loader_fav.ini`, then edit its `favN=` entries to match your files.
   Keep only one `loader*.rom` in the root; see
   [`src/README.md`](src/README.md#which-menu-rom-to-copy).
4. **Eject, plug into the MSX, power on.** The menu appears; it auto-boots
   whatever you booted last when the countdown runs out.

### Menu keys

| Key | Action |
|-----|--------|
| Cursor up / down | Move the highlight; the list scrolls at the edges |
| Cursor left / right | Page up / page down |
| `RETURN` / `SPACE` | Enter a directory, or boot a file |
| `F1` | Open Help, including the running firmware version |
| `BS` | Up one directory |
| `INS` | Assign the highlighted `.dsk` to drive B; pressing INS again keeps it assigned |
| `DEL` | Clear drive B, regardless of the selected entry |
| `ESC` | Leave the menu for BASIC (maps a bare SCC cartridge first if `scc_on_exit` is non-zero) |

Keyboard input or either joystick port cancels the auto-boot countdown.
Joystick directions navigate, A opens/boots, and B goes back or exits.

### SCC volume

The firmware and menus support a global SCC master level in
`loader.ini` (browser) or `loader_fav.ini` (favourites):

```ini
scc_volume=100
```

Use a decimal percentage from `0` to `100`. The default is `100`, which keeps
the previous output level; `50` halves the signal amplitude and `0` mutes SCC.
This is a linear amplitude control, so half the value need not sound half as
loud. The MSX's built-in PSG is unaffected. Channel volumes programmed by the
game keep their relative levels.

The setting applies to SCC ROM games, disk/Nextor SCC audio and the bare SCC
mode selected on exit. It survives SCC chip resets and cartridge changes.
Each menu reads its own INI at startup; restart the menu after editing it.
Invalid or out-of-range values leave the previous value (initially `100`)
unchanged. The browser preserves the setting when rewriting `loader.ini`;
the favourites menu preserves the other lines when saving volume and `default=`.
Install the matching UF2 and updated menu ROM together to use this option.

In either menu, press **`+` / `-`** to change the level in **10% steps**,
clamped to 0–100%. A centred `SCC Volume: 100%` popup appears only while
adjusting, then disappears after about two seconds. ESC dismisses the popup;
press ESC again to leave the menu. Other navigation keys dismiss it and work
normally. Any adjustment cancels auto-boot. The setting is saved when you boot
an entry or leave the menu (and whenever the browser otherwise saves its INI),
not on every keypress. In favourites, literal `-` now controls volume instead
of acting as the AZERTY shortcut for slot 6; the digit `6` still selects it.

### Floppy drive emulation

The cartridge serves `.dsk` files on its USB volume as floppy drives to the
MSX. Select an image in the browser menu to mount it as drive A; the MSX then
resets and boots through the emulated Disk ROM. Press `INS` on another `.dsk`
first to put it in drive B; `DEL` clears B regardless of the selection. The browser saves that choice as `disk2=` in
`loader.ini`. A diskette icon at the left of the row marks the image assigned to B:.
After INS/DEL, drive B status temporarily replaces the centered F1 Help /
Return Open hint. The next keyboard or joystick input restores the hint. Only folders and the assigned B: image have icons;
ordinary ROM and DSK files use the same plain rows in both modes.
You can also set `disk2=` by hand. Whether B: appears to the MSX
depends on the Disk ROM's own drive count; some dumps advertise only A:.

You must supply your own **16 KB MSX Disk ROM dump**; none is distributed with
this project. Copy it to the USB volume as `disk.rom`, or set `diskrom=<path>`
in `loader.ini` to use a different file. The firmware patches a copy of the
dump in RAM when mounting a disk and leaves the file untouched. A dump must
pass the firmware's validation, so an arbitrary 16 KB ROM will not work.
The following 16 KiB dumps have been tested. External downloads are available
from File-Hunter; verify the MD5 checksum before use:

- [yis805-128r2_disk.rom](https://download.file-hunter.com/System%20ROMs/machines/yamaha/yis805-128r2_disk.rom) — MD5: `814aad415b58cc3f78c7b571512b2006`
- [NATIONALDISK.rom](https://download.file-hunter.com/System%20ROMs/machines/yamaha/NATIONALDISK.rom) — MD5: `86269da485e852d9f581ac27f4ba32ff`

If disk games fail or glitch when another disk interface is installed, see
[freeing RAM with DEL during disk boot](#disk-games-fail-when-other-disk-interfaces-are-present).

Images are **writable**: MSX-DOS and Disk BASIC can save files directly into
the `.dsk` on the cartridge's flash. Set an image file's read-only attribute
on the USB volume to make the emulated disk write protected. `CALL FORMAT`
initializes a 360 KB or 720 KB image on either drive. SCC audio remains
available while a disk is mounted.

The firmware accepts any non-empty image whose size is a multiple of 512
bytes; it does not validate the disk's filesystem or geometry when mounting.
Standard 360 KB and 720 KB images work with the supported Disk ROMs. Other
sizes may mount, but Disk BASIC or MSX-DOS may fail when reading them, and
`CALL FORMAT` supports only the two standard sizes.

After booting a disk, you can change images without rebooting. For example,
`CALL OMOUNT("B:SIDE-B.DSK")` inserts an image into drive B and `CALL OEJECT("B:")`
removes it. `CALL ODISK` reports both drives; `CALL ODIR("GAMES")` lists disk
images on the cartridge's USB volume.

The same outer-volume also supports file management from BASIC:

```basic
CALL ODEL("OLD.DSK")
CALL OCOPY("SOURCE.DSK","BACKUP.DSK")
CALL ONEW("EMPTY.DSK")           ' 720 KiB formatted FAT12 image
CALL ONEW("SINGLE.DSK",360)      ' 360 KiB formatted FAT12 image
```

`ODEL` refuses an image mounted in drive A or B. `OCOPY` and `ONEW` never
overwrite an existing file, and copy transfers preserve every source sector.
New images are empty FAT12 images and are not mounted automatically. Paths are
literal `.dsk` paths on the cartridge volume; subdirectories and spaces are
supported.

The firmware unifies BASIC and DOS command names with the `O` prefix.
The old names are replaced. `tools/OmniSCC.dsk` includes all seven commands
and standard MSX-DOS 1.03 / COMMAND 1.11, requiring 64 KiB RAM.
See [disk contents and system-file provenance](tools/DOS-SYSTEM.md).

The matching MSX-DOS programs are `OMOUNT.COM`, `OEJECT.COM`, `ODISK.COM`,
`ODIR.COM`, `ODEL.COM`, `OCOPY.COM` and `ONEW.COM`; copy them into a DOS image
before use. See the [Disk BASIC command guide](src/README.md#disk-commands-from-basic)
and [MSX-DOS command guide](src/msx/dos/README.md) for syntax, protocol limits
and error handling.

---

Numbered DSK switching uses a held top-row digit `1–9` when the game next
accesses the disk BIOS. Press the game's own disk-check key while holding the
digit. Set `multidisk=0` in `loader.ini` or `loader_fav.ini` to disable it
(default `1`), for example if the game uses digits for its controls.

## Supported mappers

Detected automatically from the ROM contents: plain/linear, Konami (8 KB
pages), Konami SCC, ASCII8, ASCII16. Detection can be overridden by putting
`[Plain]`, `[8kB]`, `[Konami]`, `[KonamiSCC]`, `[SCCI]`, `[SCCPlus]`, `[ASCII8]`, `[ASCII16]` or `[KUC]` in the filename
before the extension. KUC supports up to 8 MiB and can also be selected by a known external database
hash.

Plain/Mirrored ROMs choose their start address from the ROM header, including
48 KiB images starting at `0000h`. `[Normal]` disables mirroring; `[0x0000]`,
`[0x4000]` and `[0x8000]` explicitly select a non-mirrored start address.
`[Mirrored4000]` fixes a mirrored ROM at `4000h`. Generic `[8kB]` is selected
by filename or database and switches all four 8 KiB windows, including `4000h`.
It is distinct from Konami and ASCII8; the instruction heuristic does not select it.

`[SCCI]` and `[SCCPlus]` emulate an expanded 128 KiB SCC-I RAM cartridge.
SCCI starts in compatible SCC mode; SCCPlus starts in enhanced mode for ROM
conversions using `B800h`. `[SCC+]` is an alias for `[SCCPlus]`. Both support
RAM-write modes and the `BFFEh`/`BFFFh` mode register. The ROM image initializes
cartridge RAM, with the unused space filled with `FFh`; changes are volatile.
These types are selected by tag or database, not by the instruction heuristic.

The SCC is emulated on core 1 and rendered as a PDM stream through PIO + DMA on
GPIO 26. A bare SCC cartridge with no ROM can also be mapped from the menu (just 
exit to basic), for software that expects an SCC to be present but supplies its 
own code.

ROMs up to 128 KiB are cached in RAM; larger ones are read directly from their
verified flash pages, without a permanent second copy. Mapper limits are Plain
64 KiB, Konami 256 KiB, Konami SCC 2 MiB, SCC-I/SCCPlus 128 KiB, ASCII8 2 MiB, ASCII16 4 MiB and explicit KUC 8 MiB.
The FAT storage area occupies 15 MiB, less flash-translation overhead.

## Nextor on the shared USB volume

Copy `tools/nextor.rom`, `tools/NEXTOR.SYS` and `tools/COMMAND2.COM` to the
cartridge root, then select `nextor.rom` in the menu. Nextor uses the existing
FAT12/FAT16 volume directly; no HDD image or repartitioning is required.
DOS2 needs at least 128 KiB of mapped MSX RAM. See the
[Nextor guide](src/docs/nextor.md), including `SET EXPERT ON` for COM programs.
The kernel and DOS files are distributed with [their licence](tools/NEXTOR-LICENSE.md).

## Storage recovery

If flash mapping is valid but FAT cannot mount, the cartridge starts USB MSC
for repair or formatting. Connect through USB, repair the existing filesystem
or format FAT12/FAT16, safely eject and restart. Files are not automatically
erased merely because FAT failed to mount.

A damaged or unrecognized flash map is recreated automatically with a fresh
FAT. **That reset loses the previous files.** If flash itself fails, USB still
enumerates but the storage remains not-ready. Install the matching firmware and menu ROM together.

## MSX1 and MSX2 menus

The 16 KiB universal `tools/loader.rom` includes a 32×24 SCREEN 1 interface with
the existing 8×8 font, rounded frame, folder and assigned-disk icons and inverse
selection. It retains the full browser functionality and 16 visible entries.
MSX1 or unknown BIOS versions select this interface; recognized MSX2+ BIOS
versions use SCREEN 5 only after a restoring probe confirms at least 32 KiB VRAM.
A V9938 by itself does not imply MSX2 BIOS or enough VRAM. No separate lite ROM
is required. The optional 16 KiB favourites menu uses the same automatic mode selection, font,
frame and inverse selection, with all ten slots, digit shortcuts, Help and auto-boot.

Both graphical menus show only the centered OmniSCC logo, without a menu subtitle,
while reading configuration
and preparing the menu. With at least 64 KiB VRAM, initialization uses a hidden
page and reveals the completed menu without a partial redraw. The splash adds
no fixed delay; MSX1 text mode and 32 KiB graphics retain the hidden-screen startup.

**Browser — MSX1 (SCREEN 1)**

![MSX1 browser](images/msx1-screen1.png)

**Browser — MSX2 (SCREEN 5)**

![MSX2 browser](images/msx2-screen5.png)

**Favourites — MSX1 (SCREEN 1)**

![MSX1 favourites menu](images/loader-fav-msx1-screen1.png)

**Favourites — MSX2 (SCREEN 5)**

![MSX2 favourites menu](images/loader-fav-msx2-screen5.png)

Menu mapper detection uses the exact eight bytes `OmniSCC\0` at ROM offset
`0x10`, independently of the filename. Explicit mapper tags retain priority.
The preloader still searches the root for the first `loader*.rom`; keep only
one such file. Verify release payloads with `sha256sum -c SHA256SUMS` from the
repository root.

---

## ROM mapper database

Copy `tools/romdb.bin` to the cartridge root with the matching menus. Both INI
files accept `romdb=/romdb.bin`; an empty value disables lookup. Filename tags
win, then known hashes, then the instruction heuristic. Matched tags are hidden
in both menus. See [database generation and custom hashes](src/docs/rom-database.md).

## Troubleshooting

### Older SCC+ ROM conversions hang or play only PSG

Select enhanced SCC+ mode with `[SCCPlus]` or `[SCC+]` in the filename; `[SCCI]`
starts in compatible SCC mode. Some older Konami Game Collection conversions
also contain loader bugs: bank switching with Z80 interrupts enabled, or reliance
on Disk ROM RAM-slot variables before those variables have been initialized.
Such images can work in one emulator configuration and fail on another machine
or boot differently after a reset.

Use a corrected conversion if the mapper is selected correctly but the game
still hangs or its music fails. The
[conversion loader fixes](https://www.msx.org/forum/msx-talk/general-discussion/konami-game-collection-roms-patches)
describe these issues. Changing SCC volume does not repair the game's loader.

### Disk games fail when other disk interfaces are present

Each initialized Disk ROM can reserve MSX RAM for its drives and disk buffers.
If the machine already has another Disk ROM and drives, adding OmniSCC's disk
interface can sharply reduce free RAM. A disk game may then fail to start,
hang or behave incorrectly even though the same image works with fewer disk
interfaces.

On computers with built-in disk drives, try holding **CTRL** immediately after
starting a DSK image in the OmniSCC menu. Keep it held through the restart and
initial disk boot. On Disk ROMs that support this option, CTRL disables the
second logical floppy drive and reduces disk-buffer memory usage, leaving more
RAM for the game. The effect depends on the machine and Disk ROM.

For the supported **Yamaha YIS805 Disk ROM** (`yis805-128r2_disk.rom`),
use the following boot procedure:

1. Select the `.dsk` image in the OmniSCC menu.
2. Press **Enter**, then immediately press and hold **CTRL**.
3. Keep **CTRL held until the game has finished loading**.

Hold CTRL during the restart and disk boot, not just while the menu is visible.
This is a Disk ROM boot option; its behavior can differ with another Disk ROM
or machine.

On some models, holding **DEL** during startup disables the built-in network
BIOS and can free additional RAM for disk games. This is a separate,
machine-specific boot option.

### ROM games fail after a warm reset

If a ROM game does not start from the OmniSCC menu after a reset, switch the
computer off for **at least 10 seconds**, then switch it on and try again.
A warm reset may leave RAM or peripheral state that affects the next launch;
a full power cycle gives the machine and cartridge a fresh startup.


---

## Enclosure

[`enclosure/`](enclosure/) contains ready-to-print `carts_v1.3mf` and `carts_v2.3mf`.
Feel free to remove non required parts in the slicer.
Use 2.5x15 screws for v1 and 2.9x6.5 for v2.

---

## Documentation

- [`src/README.md`](src/README.md) — building, flashing, `loader.ini`
  reference, mapper tags, troubleshooting.
- [`src/docs/omniscc_api.md`](src/docs/omniscc_api.md) — the Z80↔Pico
  command protocol, for writing your own MSX-side software.

---

## IMPORTANT!
The RBSC provides all the files and information for free, without any liability. 
The provided information, software or hardware must not be used for commercial purposes unless permitted by the RBSC.
Producing a small amount of bare boards for personal projects and selling the rest of the batch is allowed without the permission of RBSC.
When the sources of the tools are used to create alternative projects, please always mention the original source and the copyright!

---

## Contact information

The members of RBSC group Tnt23, Wierzbowsky, Pyhesty, Ptero, GreyWolf, SuperMax, VWarlock, ALSP and DJS3000 can be
contacted via the group's e-mail address:

info@rbsc.su

The group's coordinator could be reached via this e-mail address:

admin@rbsc.su

The group's website can be found here:

https://rbsc.su/
https://rbsc.su/ru

The RBSC's hardware repository can be found here:

https://github.com/rbsc

The RBSC's 3D model repository can be found here:

https://www.thingiverse.com/groups/rbsc/things

-= ! MSX FOREVER ! =-
