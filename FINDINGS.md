# What binding FatFs found in sysl

Found against sysl 0.1.0-alpha.12; none is worked around. All three are gaps
[`sysl-lang/littlefs`](https://github.com/sysl-lang/littlefs) found first (its FINDINGS 1–3), met again
here. Binding FatFs found no new one: the `@export`ed disk layer, a table of `Option[*BlockDevice]`
trait objects as folded module storage, `*u32` and `*u16` writes through FatFs's `void *` ioctl
buffer, and `c const` layouts and offsets all worked as the reference describes.

## 1. A package's C cannot be compiled against a dependency's headers (gap)

On a bare-metal target `ff.c` needs `<string.h>` and four of its functions, which `sysl-lang/cstd`
carries, but nothing lets the package's C be compiled with cstd's `include/`:

```
sysl build-lib . --target thumb-freestanding
sysl: error: sh/sysl/fatfs/c/ff.c did not compile (clang, exit 1):
sh/sysl/fatfs/c/ff.c:22:10: fatal error: 'string.h' file not found
```

(`c.sysl`'s `c const` block does compile there: `ff.h` needs only `<stdint.h>`, which clang supplies
freestanding.)

**Status: accepted for alpha.14** (sysl branch `vendored-c`).

## 2. A dependency cannot be taken on some targets only (gap)

cstd must be linked where there is no C library and must not be where there is one.

**Status: accepted for alpha.14** (sysl branch `target-deps`).

## 3. Nothing stops a struct being copied after C has taken its address (gap)

`f_mount` stores the `FATFS`'s address and every `FIL` and `DIR` points at its `FATFS`, yet
`var copy = vol` after `vol.mount()` compiles. The tests move a `Volume` only before it is mounted.

**Status: accepted for alpha.14 as `@pinned`** (sysl branch `pinned`). `Volume`, `File` and `Dir` take
it then. `volume()`, `file()` and `dir()` keep their shape, since a fresh pinned value is built in the
caller's slot.
