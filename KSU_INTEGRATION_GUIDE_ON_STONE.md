# KernelSU Integration Guide for stone

This guide explains what was done to integrate KernelSU into the `stone`
kernel source for Redmi Note 12 5G (`sunstone`) and POCO X5 5G (`moonstone`).
It is written for beginners, so it explains both the commands and the reason
behind each change.

## 1. What this kernel tree is

The `stone` tree is a unified Android kernel source for:

- Redmi Note 12 5G, codename `sunstone`
- POCO X5 5G, codename `moonstone`

The kernel version was checked from the root `Makefile`:

```text
VERSION = 5
PATCHLEVEL = 4
SUBLEVEL = 302
```

So this is a Linux `5.4.302` kernel tree.

The matching defconfig found for this device family was:

```text
arch/arm64/configs/stone_defconfig
```

## 2. Important KernelSU version decision

The latest official KernelSU release page currently points to `v3.2.5`.
However, the official KernelSU non-GKI guide says KernelSU `v1.0` and newer no
longer officially support non-GKI devices, and that `v0.9.5` is the last
supported non-GKI release.

Because this `stone` kernel is a non-GKI vendor kernel and the integration uses
manual non-GKI hooks, the official `tiann/KernelSU` tag used here was:

```text
v0.9.5
```

This keeps the integration aligned with the official non-GKI instructions
instead of using a newer GKI-focused KernelSU release that no longer provides
the same manual hook API.

## 3. Branch used

A new branch was created for this work:

```bash
git checkout -b KSU
```

All KernelSU integration commits were made on this branch.

## 4. KernelSU submodule

KernelSU was added from the official upstream repository:

```text
https://github.com/tiann/KernelSU
```

The submodule path is:

```text
drivers/kernelsu
```

The `.gitmodules` file now contains:

```ini
[submodule "drivers/kernelsu"]
	path = drivers/kernelsu
	url = https://github.com/tiann/KernelSU
```

The checked-out KernelSU revision is:

```text
b766b98513b5a7eb33bc1c4a76b5702bf1288f07 drivers/kernelsu (v0.9.5)
```

## 5. Build-system integration

Two kernel build-system files were updated so the kernel can see and build
KernelSU when `CONFIG_KSU=y`.

### drivers/Kconfig

This line was added near the end of `drivers/Kconfig`:

```kconfig
source "drivers/kernelsu/Kconfig"
```

This makes the KernelSU config option visible to Kconfig.

### drivers/Makefile

This line was added to `drivers/Makefile`:

```makefile
obj-$(CONFIG_KSU) += kernelsu/
```

This tells the kernel build system to enter `drivers/kernelsu/` when
`CONFIG_KSU` is enabled.

## 6. Manual VFS hooks

KernelSU needs hooks in common filesystem/syscall paths so it can intercept
specific operations. For this non-GKI integration, hooks were added manually in
four files.

No file inside `drivers/kernelsu/` was edited directly.

### fs/exec.c

Purpose: handle `execveat`, which is used when a process executes a program.
KernelSU uses this path for `su` compatibility and ksud behavior.

Added declarations guarded by `CONFIG_KSU`:

```c
extern bool ksu_execveat_hook __read_mostly;
extern int ksu_handle_execveat(int *fd, struct filename **filename_ptr,
			       void *argv, void *envp, int *flags);
extern int ksu_handle_execveat_sucompat(int *fd,
					struct filename **filename_ptr,
					void *argv, void *envp, int *flags);
```

Added hook call inside `do_execveat_common()` before calling
`__do_execve_file()`:

```c
if (unlikely(ksu_execveat_hook))
	ksu_handle_execveat(&fd, &filename, &argv, &envp, &flags);
else
	ksu_handle_execveat_sucompat(&fd, &filename, &argv, &envp, &flags);
```

### fs/open.c

Purpose: handle `faccessat`, which KernelSU uses for `su` compatibility checks.

Added declaration:

```c
extern int ksu_handle_faccessat(int *dfd,
				const char __user **filename_user, int *mode,
				int *flags);
```

Added hook call at the start of `do_faccessat()`:

```c
ksu_handle_faccessat(&dfd, &filename, &mode, NULL);
```

### fs/read_write.c

Purpose: handle `vfs_read`, used by KernelSU's ksud communication path.

Added declarations:

```c
extern bool ksu_vfs_read_hook __read_mostly;
extern int ksu_handle_vfs_read(struct file **file_ptr,
			       char __user **buf_ptr, size_t *count_ptr,
			       loff_t **pos);
```

Added hook call at the start of `vfs_read()`:

```c
if (unlikely(ksu_vfs_read_hook))
	ksu_handle_vfs_read(&file, &buf, &count, &pos);
```

### fs/stat.c

Purpose: handle stat calls, which KernelSU uses for `su` compatibility.

Added declaration:

```c
extern int ksu_handle_stat(int *dfd, const char __user **filename_user,
			   int *flags);
```

Added hook call at the start of `vfs_statx()`:

```c
ksu_handle_stat(&dfd, &filename, &flags);
```

## 7. Defconfig change

The device defconfig already had:

```text
CONFIG_OVERLAY_FS=y
CONFIG_KPROBES=y
```

KernelSU `v0.9.5` requires OverlayFS in its Kconfig, and OverlayFS was already
enabled. `CONFIG_KSU=y` was added to:

```text
arch/arm64/configs/stone_defconfig
```

The added line is:

```text
CONFIG_KSU=y
```

It was placed near the existing filesystem options, next to `CONFIG_OVERLAY_FS`.

## 8. Extra checks performed

### Header check

There was no top-level file named:

```text
include/linux/kernelsu.h
```

So the VFS files use local `extern` declarations that match KernelSU `v0.9.5`
instead of including a missing header.

### Export symbol check

KernelSU `v0.9.5` lists these requested exports:

```text
register_kprobe
unregister_kprobe
```

This kernel already exports both:

```text
kernel/kprobes.c:EXPORT_SYMBOL_GPL(register_kprobe)
kernel/kprobes.c:EXPORT_SYMBOL_GPL(unregister_kprobe)
```

So no extra export patch was needed.

### Security directory check

The `security/` directory did not contain an existing KernelSU integration.
No security files were changed.

### path_umount check

`fs/namespace.c` does not provide `path_umount`. KernelSU `v0.9.5` detects this
and disables that optional module unmount helper at compile time. The core
KernelSU integration does not require backporting `path_umount`, so it was left
unchanged.

## 9. Commits made

These commits were created on the `KSU` branch:

```text
8f363749467c KSU: Add KernelSU v0.9.5 as submodule
d30a458c7a16 KSU: Source KernelSU Kconfig in drivers/Kconfig
d3d2229ac73e KSU: Add KernelSU obj entry in drivers/Makefile
44efe4ff7076 KSU: Add KernelSU execveat hook to fs/exec.c
d90dc4fcba32 KSU: Add KernelSU faccessat hook to fs/open.c
ef10e4354ebd KSU: Add KernelSU vfs_read hook to fs/read_write.c
0fcbf213ca2b KSU: Add KernelSU stat hook to fs/stat.c
49350c39814a KSU: Enable CONFIG_KSU in stone_defconfig
```

Each source change was committed separately so the history is easy to review.

## 10. Final verification commands

These commands were used for no-build verification:

```bash
git branch
git log --oneline -12
git diff --stat origin/16...HEAD
cat .gitmodules
git submodule status
grep -n "kernelsu\|KSU" drivers/Kconfig drivers/Makefile
grep -n "ksu_handle\|CONFIG_KSU" fs/exec.c fs/open.c fs/read_write.c fs/stat.c
find arch/arm64/configs -name "*stone*" -o -name "*sunstone*" -o -name "*moonstone*" | xargs grep -H "CONFIG_KSU"
git status --short
```

Expected important results:

```text
Current branch: KSU
Submodule: drivers/kernelsu (v0.9.5)
drivers/Kconfig: source "drivers/kernelsu/Kconfig"
drivers/Makefile: obj-$(CONFIG_KSU) += kernelsu/
arch/arm64/configs/stone_defconfig:CONFIG_KSU=y
```

The only uncommitted change left after integration was a pre-existing
`.gitignore` modification unrelated to KernelSU.

## 11. What was not done

No build command was run. In particular, none of these were executed:

```text
make
make menuconfig
make defconfig
```

No remote push was performed.

No files inside the KernelSU submodule were edited directly.

No KernelSU forks such as MKSU, APatch, or SuKi were used.

## 12. How to inspect the result later

To check the branch:

```bash
git branch
```

To check KernelSU's submodule version:

```bash
git submodule status
```

To check the main integration points:

```bash
grep -n "kernelsu\|KSU" drivers/Kconfig drivers/Makefile
grep -n "ksu_handle\|CONFIG_KSU" fs/exec.c fs/open.c fs/read_write.c fs/stat.c
grep -H "CONFIG_KSU" arch/arm64/configs/stone_defconfig
```

If these commands show the expected lines, the source-level KernelSU
integration is present.
