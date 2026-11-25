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
| `k20p_hyperos_a17_zundamon` | Xiaomi Redmi K20 Pro (raphael) | 4.14 Zundamon-Evo | 5 hunks of `susfs_patch_to_4.14.patch` fail: `fs/proc/cmdline.c` (vendor dynamic-partition cmdline), `fs/proc/task_mmu.c` (5.x-style `smaps_walk_ops` / 5-arg `walk_page_range` / missing `susfs_def.h` include), `kernel/sys.c` (existing bpfloader uname spoof) |

## Why the follow-up patch is required

`patch-susfs` runs the generic patch with `patch -p1 < ... || true`, so failed
hunks do not abort the step. The `.rej` files are removed afterwards and the
build continues, which surfaces the problem much later as a compile error.

For `k20p_hyperos_a17_zundamon` the observable failure was:

```
fs/proc/task_mmu.c:572:7: error: implicit declaration of function 'SUSFS_IS_INODE_OPEN_REDIRECT'
fs/proc/task_mmu.c:592:7: error: implicit declaration of function 'SUSFS_IS_INODE_SUS_MAP'
```

Hunk #1 of `fs/proc/task_mmu.c` inserts `#include <linux/susfs_def.h>` and it
failed, while hunks #2/#3 inserted the *callers* of the macros. The follow-up
patch restores the include plus the two remaining hunks.

## Testing

```bash
# in a clean kernel tree
patch -p1 < Patches/Patch/susfs_patch_to_4.14.patch
find . -name '*.rej' -delete
patch -p1 < <device_dir>/susfs_fixed.patch
```
