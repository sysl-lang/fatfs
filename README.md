# FatFs for sysl

FAT12, FAT16 and FAT32 volumes with long UTF-8 names, bound to ChaN's
[FatFs](http://elm-chan.org/fsw/ff/) R0.16 (with its two official patches), vendored unchanged.

*An independent sysl package for FatFs. It is not affiliated with or endorsed by FatFs's authors or trademark holders.*

- **It is what a PC reads.** An SD card, a USB stick or a disk image written here opens on Windows,
  macOS and Linux, and the tests check that against three independent tools.
- **It needs no heap.** A volume's sector window and each open file's sector buffer are arrays inside
  the sysl values that own them; the long-name buffer is FatFs's own static one.
- **The disk layer is sysl.** The five functions FatFs calls for sectors and the one it calls for the
  time are `@export`ed from sysl, so a program supplies a `BlockDevice` and never writes C.

It is for people who prefer FatFs. [`sysl-lang/fat`](https://github.com/sysl-lang/fat) is FAT written
in sysl; for raw flash with no PC on the other end, [`sysl-lang/littlefs`](https://github.com/sysl-lang/littlefs).

```hocon
dependencies {
  fatfs { git = "github.com/sysl-lang/fatfs", version = "<the first tag>" }
}
```

The module is `sh.sysl.fatfs`.

## Using it

A device implements `BlockDevice`: its sector size and count, reading and writing whole sectors, and
sync (plus, if it wants, an erase-block size, trim, initialization and write protection). Then:

```sysl
import sh.sysl.fatfs.{CREATE_ALWAYS, READ, WRITE, WORK_MIN, file, format_options, volume}

var card = volume(0, &sd)
if card.mount().is_err() then
    var work: [WORK_MIN]u8 = [0; WORK_MIN]
    card.format(format_options(), work[..]).unwrap()
    card.mount().unwrap()

var f = file()
card.open(&f, "logs/today.txt", WRITE | CREATE_ALWAYS).unwrap()
f.write_all("volume=7".bytes).unwrap()
f.close().unwrap()
```

A `Volume` is a drive (0 to `VOLUMES - 1`, four of them) and the device behind it. `mount` and
`format` register the device on that drive for FatFs's disk calls and `unmount` lets it go; a second
volume on a drive that is held answers `Error.DriveTaken`. Paths are the volume's own, from its root
(`"logs/today.txt"` or `"/logs/today.txt"`); the binding puts the drive number in front and hands FatFs
a NUL-terminated copy, at most `MAX_PATH` (511) bytes.

**A mounted `Volume`, an open `File` and an open `Dir` must stay where they are**, because FatFs keeps
their addresses: `f_mount` registers the volume by address, and every open file and directory points
at its volume. On a board they are module storage; a copy taken after `mount` or `open` is one FatFs
knows nothing about. `Volume`, `File` and `Dir` are `@pinned`, so sysl refuses such a copy at compile time:
`var copy = vol` is an error, and `volume()`, `file()` and `dir()` build their value in the caller's slot.

An `Info`'s names are views of its own storage, so listing a directory allocates nothing; copy a name
to keep it past the next `Dir.read`. FatFs's `f_stat` answers the name as the path spelled it
(`stat("readme.txt")` of `README.txt` says `readme.txt`); a directory listing answers it as stored.

| | |
|---|---|
| volume | `volume`, `format` (with `format_options`: FAT or FAT32 or either, partition table or none, one or two FATs, alignment, root entries, cluster size), `mount`, `unmount`, `is_mounted`, `fat_type`, `sector_size`, `free_space`, `label`, `set_label` |
| paths | `mkdir`, `remove`, `rename`, `stat`, `exists` |
| files | `Volume.open` (with `READ`, `WRITE`, `CREATE_NEW`, `CREATE_ALWAYS`, `OPEN_ALWAYS`, `OPEN_APPEND`), `read`, `write`, `write_all`, `seek`, `tell`, `size`, `at_end`, `is_open`, `truncate`, `sync`, `close` |
| directories | `Volume.open_dir`, `read`, `rewind`, `close` |
| time | `Clock`, `set_clock`, `clear_clock`, `Timestamp`; with no clock, files are stamped 1980-01-01 00:00 |

Every `FRESULT` code is an `Error` of its own, named for what it means (`NoFile`, `NoPath`, `Denied`,
`Exists`, `Locked`, …). The binding adds four: `Full` (a `write_all` the volume had no room for —
FatFs reports a full volume as a short write, not an error), `PathTooLong`, `DriveTaken`, and
`Unknown(code)` for a code from a newer FatFs.

## What FatFs is compiled with

`sh/sysl/fatfs/c/ffconf.h` is FatFs's application configuration file by design. It starts from
upstream's template; every setting changed from it:

| setting | template | here | why |
|---|---|---|---|
| `FF_USE_MKFS` | 0 | 1 | `format` |
| `FF_USE_LABEL` | 0 | 1 | `label` and `set_label` |
| `FF_CODE_PAGE` | 932 | 437 | the US OEM code page, which DOS and Windows use for 8.3 names; single-byte, so `ffunicode.c` compiles one small table |
| `FF_USE_LFN` | 0 | 1 | long names, with FatFs's working buffer static (BSS): no heap and no 512 bytes of stack per call. Not thread-safe, which no reentrancy already implies |
| `FF_LFN_UNICODE` | 0 | 2 | the API is UTF-8, which is what a sysl `string` is |
| `FF_LFN_BUF` / `FF_SFN_BUF` | 255 / 12 | 765 / 34 | the room `FILINFO` needs for the longest UTF-8 names (255 UTF-16 units, 3 bytes each), per `config.html` and CVE-2026-6688; with less, a long name is silently listed as its 8.3 name |
| `FF_VOLUMES` | 1 | 4 | several devices at once (an SD card and a RAM disk) |
| `FF_MAX_SS` | 512 | 4096 | 4K-sector devices (`FF_MIN_SS` stays 512); FatFs then asks the device its sector size |
| `FF_USE_TRIM` | 0 | 1 | `BlockDevice.trim`, so flash can erase what a removed file freed (it defaults to doing nothing) |
| `FF_FS_LOCK` | 0 | 8 | FatFs refuses a second open that writes, and a remove or rename of an open entry (`Error.Locked`), instead of letting them corrupt the volume; up to eight open files and directories |

Kept as the template has them, deliberately: `FF_FS_RPATH` 0 (no current directory: it is global
state per drive, and a volume's paths are already from its root; `.` and `..` are not listed);
`FF_FS_EXFAT` 0 (exFAT is not what this package is checked against, the tools below do not check it,
and it widens `FSIZE_t` and `LBA_t` to 64 bits); `FF_LBA64` 0; `FF_MULTI_PARTITION` 0 (one volume per
device, found in sector 0 or the first partition); `FF_FS_REENTRANT` 0; `FF_FS_NORTC` 0 (the time
comes from `Clock`); `FF_FS_TINY` 0; `FF_FS_READONLY` 0; `FF_FS_MINIMIZE` 0.

The declarations check what they assume when the package compiles: `UINT`, `LBA_t` and `FSIZE_t` are
32 bits and `TCHAR` a byte (so a change to exFAT, 64-bit LBA or another encoding is refused), and the
layouts sysl declares for `FILINFO` and `MKFS_PARM` match the header's `sizeof` and `offsetof`.

## What is bound, and what is left out

Not bound: `f_chmod` and `f_utime` (`FF_USE_CHMOD`), `f_findfirst`/`f_findnext`, `f_expand`,
`f_forward`, fast seek, the string functions (`f_gets`, `f_putc`, `f_puts`, `f_printf`), `f_chdir`,
`f_chdrive` and `f_getcwd` (relative paths), `f_fdisk` and multiple partitions, `f_setcp`. Each is a
`ffconf.h` switch and an addition rather than a change, when something needs it.

`format` refuses a work area under `WORK_MIN` (`FF_MAX_SS`, 4096 bytes) itself: FatFs checks only for
one sector of the device's size, then clears 4096 bytes of the buffer when it writes a partition
table. AddressSanitizer found that overflow; `f_mkfs`'s manual does say "`FF_MAX_SS` at least".

FatFs's `f_setlabel` writes the label into the root directory, where Windows and mtools read it, and
leaves the boot sector's copy as formatted. fsck.fat reports the difference (a test pins it).

## Bare metal

The package needs no operating system and no allocator, and the drive table is laid into the image as
data. But `ff.c` includes `<string.h>` and calls `memset`, `memcpy`, `memcmp` and `strchr`, and today a
sysl package has no way to hand its C `sysl-lang/cstd`'s headers. Two sysl changes accepted for
alpha.14 close that (findings 1 and 2 below); this package does not take them up yet, so `sysl build-lib . --target thumb-freestanding`
stops at `ff.c:22:10: fatal error: 'string.h' file not found`. A target with a C library (the Pico
SDK's newlib, Zephyr, FreeRTOS) has the headers already.

## Tests

`sysl test .` runs 30 tests. Beyond every operation, over a RAM device and over one that fails,
refuses writes and counts what it is asked:

- **Every error that can be provoked**, each from the case that causes it: a missing file and a
  missing directory, a bad name (`*`, `?`, `|`, a colon, a NUL, 256 characters), a name that exists,
  a write or truncate on a file opened for reading and a read on one opened for writing, a closed or
  never-opened file, an open file locked against a writer, a remove and a rename, a ninth open file, a
  full volume (a short write, `Full`, `Denied` for a directory), a device whose reads or writes fail
  (`Disk`), one that cannot start (`NotReady`), one that is write-protected, a blank device
  (`NoFilesystem`), an unmounted volume and an unregistered drive (`NotEnabled`), a drive past the
  last and one already held, a work area too small, a device too small for FAT or FAT32, and a sector
  size outside 512 to 4096. The five codes no input reaches (`Internal`, `Timeout`,
  `InvalidParameter`, an unknown code) are checked through the mapping, which is checked code by code.
- FAT12, FAT16 and FAT32; 512- and 4096-byte sectors; with a partition table and without; four
  volumes at once; long names in Latin, Greek, Japanese and past the Basic Multilingual Plane; a 1.5 MB
  file in odd-sized pieces; sixty long names running a directory over several clusters; the format
  options read back from the boot sector; the data area aligned to the device's erase block; trims of
  the formatted volume and of a removed file's clusters; timestamps from a clock, at both ends of FAT's
  range, and the 1980 stamp without one.
- **fsck**: macOS's `fsck_msdos -n` and dosfstools' `fsck.fat -n -v` both find a FAT16 volume and a
  FAT32 volume clean after FatFs formatted them and wrote 78 files (empty to 300,000 bytes, long
  Unicode names, three directories deep), removed nine and moved one.
- **mtools reads what FatFs wrote**: `mcopy` copies back every one of those files byte for byte, and
  `mdir -b` lists every long name, with the removed ones gone and the moved one in its new place, on
  both volumes.
- **FatFs reads what mtools wrote**: into FAT16 and FAT32 volumes `mkfs.fat` formatted, mtools copies
  the same files; FatFs mounts each, finds the label, reads every file byte for byte and lists every
  name as written; then FatFs writes a file and removes one, fsck.fat still finds the volume clean,
  and mtools reads FatFs's file back.

The tools are `/sbin/fsck_msdos`, and `brew install dosfstools mtools`; a test skips when one is
missing. mtools keeps only the low sixteen bits of a character past U+FFFF, so the tool tests use
names in the Basic Multilingual Plane and a surrogate pair is tested in the package alone.

**Mutations**, each run and each red: the sector size reported as 512 to FatFs (the 4K-sector and
sector-size tests); a multi-sector write dropping its last sector (five tests, fsck among them); the
drive table read one drive off (28 tests); `FR_NO_FILE` and `FR_NO_PATH` swapped (four); timestamps
ignored (the clock test); the sector count reported one short (the trim test, and the boot sector's
total once that test was added for it); the erase block ignored (the alignment test); `CTRL_SYNC`
never reaching the device; write protection left out of the status; a failed initialization
reported ready; a path handed over without its drive number (four, the several-volumes test among
them).

**AddressSanitizer**: `SYSL_EXTRA_CFLAGS="-fsanitize=address -g" sysl test .` passes 30/30. The test
binary imports ten `__asan_report_*` hooks and `__asan_memcpy`, and FatFs's own C is instrumented: the
run is what caught `f_mkfs` clearing 4096 bytes of a 512-byte work area, reported inside
`create_partition`.

## Vendored

`sh/sysl/fatfs/c/` holds `ff.c`, `ff.h`, `diskio.h` and `ffunicode.c` from
[FatFs R0.16](http://elm-chan.org/fsw/ff/arc/ff16.zip), from the zip whose SHA-256 is
`99f7dc1f7e095356e4a9e3dbe29959090d8b948afe2bbc5441e52fdf4b85449e`, with ChaN's two official patches
to `ff.c` from [the patches page](http://elm-chan.org/fsw/ff/patches.html) applied and nothing else:

- patch 1 (`ff16p1.diff`, SHA-256 `7996ddc3135f3d534a8153f7d75d587030e5c8551b003c3b7ee823ffb7fc64bf`):
  the smallest volume is 64 sectors rather than 128;
- patch 2 (`ff16p2.diff`, SHA-256 `5cd39f1fc299f0f1dec9fa1ce544dc0bc048b2f6eb3b8161476ed20f9cf1e290`):
  fixes CVE-2026-6682 (a broken FAT size in a boot sector), CVE-2026-6683 and CVE-2026-6687 (exFAT,
  not compiled here).

The patches are LF and the sources CRLF, so they were applied with their lines converted to CRLF; the
files keep upstream's CRLF line endings. The replacement `ffunicode.c` on the same page shrinks the
double-byte code-page tables only and is not taken. `ffconf.h` is upstream's template with the
settings above. Vendoring means upstream fixes do not arrive on their own: watch the patches page and
re-vendor.

Licensing: FatFs has its own one-clause BSD-style license; the rest of this package is ISC. Both texts
are in `LICENSE`.

## What binding FatFs found in sysl

In full in `FINDINGS.md`. Found against sysl 0.1.0-alpha.12; none is worked around. Finding 3 is fixed in sysl 0.1.x and taken up here;
findings 1 and 2 are open (checked against sysl 0.1.1). All three are the gaps
[`sysl-lang/littlefs`](https://github.com/sysl-lang/littlefs) found first (its FINDINGS 1–3), met
again here. Binding FatFs found no new one: the `@export`ed disk layer, a table of
`Option[*BlockDevice]` trait objects as folded module storage, `*u32` and `*u16` writes through
FatFs's `void *` ioctl buffer, and `c const` layouts and offsets all worked as the reference describes.

1. **A package's C cannot be compiled against a dependency's headers.** On a bare-metal target `ff.c`
   needs `<string.h>` and four of its functions, which `sysl-lang/cstd` carries, but nothing lets the
   package's C be compiled with cstd's `include/`:

   ```
   sysl build-lib . --target thumb-freestanding
   sysl: error: /Users/ed/dev/sysl-lang/fatfs/sh/sysl/fatfs/c/ff.c did not compile (clang, exit 1):
   /Users/ed/dev/sysl-lang/fatfs/sh/sysl/fatfs/c/ff.c:22:10: fatal error: 'string.h' file not found
   ```

   (`c.sysl`'s `c const` block does compile there: `ff.h` needs only `<stdint.h>`, which clang
   supplies freestanding.) Accepted for alpha.14 (sysl branch `vendored-c`).
2. **A dependency cannot be taken on some targets only**: cstd must be linked where there is no C
   library and must not be where there is one. Accepted for alpha.14 (sysl branch `target-deps`).
3. **Nothing stops a struct being copied after C has taken its address.** `f_mount` stores the
   `FATFS`'s address and every `FIL` and `DIR` points at its `FATFS`, yet `var copy = vol` after
   `vol.mount()` compiled. Fixed in sysl 0.1.x; workaround removed: `Volume`, `File` and `Dir` are
   `@pinned`, and `volume()`, `file()` and `dir()` keep their shape, since a fresh pinned value is built
   in the caller's slot.
