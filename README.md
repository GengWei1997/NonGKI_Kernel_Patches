# NonGKI_Kernel_Patches

Device-specific SUSFS patch fixes for [NonGKI_Kernel_Build_2nd](https://github.com/JackA1ltman/NonGKI_Kernel_Build_2nd).

The generic patches in `Patches/Patch/susfs_patch_to_<version>.patch` assume a
vanilla kernel source tree. Kernels with heavy backports (or vendor
modifications) usually need a small follow-up patch, because some hunks of the
generic patch cannot be applied. These follow-up patches are stored here, one
directory per device, and are applied through the `SUSFS_FIXED` mechanism.

## Layout

```
<device_dir>/susfs_fixed.patch
```

Workflow configuration:

```yaml
SUSFS_FIXED        : true
PATCHES_SOURCE     : "GengWei1997/NonGKI_Kernel_Patches"
PATCHES_BRANCH     : "main"
SUSFS_FOLDER_FIXED : "<device_dir>"
SUSFS_PATCH_FIXED  : "susfs_fixed"
```

## Directories

| Directory | Device | Kernel | Notes |
|---|---|---|---|
| `k20p_hyperos_a17_zundamon` | Xiaomi Redmi K20 Pro (raphael) | 4.14 Zundamon-Evo | 7 hunks of `susfs_patch_to_4.14.patch` fail across 5 files: `fs/namespace.c`, `fs/super.c` (context changed by the new mount API backport), `fs/proc/cmdline.c` (vendor dynamic-partition cmdline), `fs/proc/task_mmu.c` (4.14 `walk_page_range` signature / missing `susfs_def.h` include), `kernel/sys.c` (existing bpfloader uname spoof) |

## Why the follow-up patch is required

`patch-susfs` runs the generic patch with `patch -p1 < ... || true`, so failed
hunks do not abort the step. The `.rej` files are removed afterwards and the
build continues, which surfaces the problem much later as a compile error.

Two kinds of failure show up here.

Missing includes. Hunk #1 of `fs/proc/task_mmu.c` inserts
`#include <linux/susfs_def.h>` and it fails, while later hunks insert the
*callers* of the macros:

```
fs/proc/task_mmu.c:572:7: error: implicit declaration of function 'SUSFS_IS_INODE_OPEN_REDIRECT'
fs/proc/task_mmu.c:592:7: error: implicit declaration of function 'SUSFS_IS_INODE_SUS_MAP'
```

The same happens in `fs/namespace.c` and `fs/super.c`, where the mount API
backport added `linux/fs_context.h`, `linux/fs_parser.h` and
`linux/anon_inodes.h` right above the hunk's context. That one is fatal,
because it is not a macro but a symbol: `get_anon_bdev()` ends up calling

```
fs/super.c:1063: error: use of undeclared identifier 'susfs_is_sdcard_android_data_not_decrypted'
fs/super.c:1064: error: implicit declaration of function 'susfs_is_current_ksu_domain'
fs/super.c:1065: error: use of undeclared identifier 'DEFAULT_KSU_MNT_MINOR_DEV'
```

and `fs/namespace.c` then tests `CL_COPY_MNT_NS` in six places without it
being defined anywhere.

Hunk context drift. `show_smap()` and `pagemap_read()` in this tree use the
4.14 `walk_page_range()` signature and a vendor-modified pagemap walk, so the
upstream-shaped hunks match different surrounding code.

The follow-up patch restores both the includes and the declarations, and
re-targets the `task_mmu.c` hunks at this tree's code.

## Testing

```bash
# in a clean kernel tree
patch -p1 < Patches/Patch/susfs_patch_to_4.14.patch
find . -name '*.rej' -delete
patch -p1 < <device_dir>/susfs_fixed.patch
```

The result was then run through the rest of the pipeline (SUSFS inline hook,
BakaSU static removal, droidspaces coccinelle fixes) and built with clang
r596125 for arm64: no errors, `Image.gz-dtb` produced.

Note that the patch must be paired with the matching kernel branch. The
`zundamon-evo-mountapi` kernel branch is the one this patch is written
against; it carries no SUSFS code itself, the generic patch and this file
supply all of it.
