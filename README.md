# linux-mbp

Reproducible Arch Linux kernel packaging for the MacBookPro11,5. The repository
tracks Arch's official `linux` package, applies a small reviewed patch series,
and builds a separately named package with GitHub Actions.

The current package identity remains `linux-mbp-suspend` so an update preserves
the existing rEFInd and mkinitcpio recovery setup. The repository's broader
`linux-mbp` name allows additional MacBook-specific fixes to be reviewed here
without implying that every patch is limited to suspend.

## Patch series

The 7.2.11 update retains these five patches from the installed revision 4:

1. `mtrr-expedite-cache-init-on-resume.patch`: early MTRR restoration for slow
   secondary CPU bring-up after S3; original submission by Chris Paulson-Ellis:
   <https://www.spinics.net/lists/kernel/msg6037880.html>.
2. `amdgpu-uvd-check-validation-status-register.patch`: check firmware validation
   status in the status register rather than the key-selection register.
3. `amdgpu-mbp-s3-reset-experiment.patch`: exact-hardware, default-off reset
   diagnostic. Keep `amdgpu.mbp_skip_s3_reset` disabled; it did not fix AMD S3.
4. `amdgpu-gfx6-initialize-rings-before-cp-start.patch`: upstream GFX6 ring
   initialization ordering backport.
5. `amdgpu-gfx6-clear-rings-on-init.patch`: upstream GFX6 ring clearing backport.

The original switcheroo-OFF suspend guard is now part of upstream Linux, in
[eadb85cca111](https://github.com/torvalds/linux/commit/eadb85cca111152bd334b861fb53747ab851d13a),
and its duplicate local patch has been removed. The MTRR and remaining AMD
patches still require local application against the 7.2.11 stable source.

This branch updates the base kernel and official Arch configuration. It does
not establish that dedicated-AMD deep resume works. The last recorded revision-4
AMD deep-S3 test failed without enough persisted logs to identify the stage.
Build success and patch application must be followed by separately approved
hardware testing.

## CI build

Every push, pull request, tag, and manual dispatch runs
`.github/workflows/build.yml`. The workflow:

- reclaims unused SDK space on GitHub's disposable Ubuntu runner;
- starts a clean official `archlinux:base-devel` container;
- imports the PGP keys shipped by Arch's package repository;
- verifies the signed kernel and Arch patchset plus every checksum;
- runs an unprivileged `makepkg` build inside the container;
- uploads the package and `SHA256SUMS` for seven days.

GitHub's standard runner starts with limited free disk. The workflow requires
25 GiB after its explicit cleanup and fails before compilation if that cannot be
provided. All destructive cleanup commands target only the ephemeral runner.

Download an artifact from the successful workflow run and verify it before
installation:

```bash
sha256sum -c SHA256SUMS
pacman -Qip ./linux-mbp-suspend-*.pkg.tar.zst
```

## Local build

Install the additional dependencies and build without root privileges:

```bash
sudo pacman -S --needed bc pahole rust rust-bindgen rust-src
./build-test-kernel
```

The helper uses a temporary GnuPG home, verifies upstream signatures and all
checksums, refuses to start below 25 GiB free, and defaults to six parallel
jobs. Set `MBP_KERNEL_JOBS=4`, for example, to change the local parallelism.

## Installation safety

This package can install alongside Arch's stock `linux` package because it does
not conflict with or replace it. A validated machine may instead use this
package as its only installed kernel, with explicit Intel and AMD rEFInd entries
pointing to the same custom kernel and matched initramfs.

Before installing, preserve the currently installed custom-kernel package,
kernel, initramfs and boot configuration as a verified rollback set. This is an
upgrade of the existing custom package. Verify the package metadata and
available `/boot` space:

```bash
pacman -Qip ./linux-mbp-suspend-*.pkg.tar.zst
pacman -Q linux-mbp-suspend
df -B1 /boot
```

The target MacBook currently mounts `/boot` read-only. Do not enable a
fallback image. Install only after confirming at least 80,000,000 bytes are
free; restore the read-only mount after verifying the installation:

```bash
sudo mount -o remount,rw /boot
sudo pacman -U ./linux-mbp-suspend-*.pkg.tar.zst
pacman -Q linux-mbp-suspend
ls -lh /boot/vmlinuz-linux-mbp-suspend \
  /boot/initramfs-linux-mbp-suspend.img
cat /etc/mkinitcpio.d/linux-mbp-suspend.preset
sync
sudo mount -o remount,ro /boot
```

Do not reboot if package installation or initramfs generation fails. Repair or
reinstall the custom package while the running kernel and userspace remain
available.

## Runtime acceptance test

After booting the custom kernel, confirm its identity and the intended boot
arguments before touching the discrete GPU:

```bash
uname -r
cat /proc/cmdline
```

Select Intel in firmware, keep Hyprland routed to Intel, leave AMD
`power/control` set to `on`, and switch the AMD GPU off through the already
validated vgaswitcheroo path. The required state is:

```text
0:IGD:+:Pwr:0000:00:02.0
1:DIS: :Off:0000:01:00.0
2:DIS-Audio: :DynOff:0000:01:00.1
```

For the first lid-only deep-S3 cycle, record both the physical lid-open to
visible-content time and raw kernel-ring timestamps. Expected evidence for the
new patch is:

- `mtrr: your CPUs had unexpected MTRR settings on resume` when firmware
  changed the boot CPU's MTRRs;
- secondary CPU bring-up falling from several seconds toward milliseconds;
- no amdgpu PCI reset failure, suspend callback `-22`, or s2idle fallback.

## Tracking Arch updates

The package is pinned to the version, Arch patchset, configuration, checksums,
and signing keys in Arch's official packaging repository:
<https://gitlab.archlinux.org/archlinux/packaging/packages/linux>.

For each update:

1. Review the new official `PKGBUILD` and configuration changes.
2. Copy the official version, source checksums, configuration, and any signing
   key updates into this repository.
3. Keep the custom package name and append the retained local patches after
   Arch's patchset in `source`.
4. Run `makepkg --verifysource` and verify all retained patches apply with zero
   fuzz.
5. Push a branch and let CI compile it before merging or installing anything.

An upstream kernel update is intentionally not installed automatically. A
patch conflict should stop CI and require review rather than silently changing
the target code.

`scripts/check-arch-update` compares `ARCH_PACKAGING_COMMIT` with the official
packaging repository. A scheduled read-only check runs weekly and opens one
GitHub issue when review is needed; it never rewrites packaging or publishes a
kernel automatically.

## AMD resume candidates and observed results

The UVD firmware validation guard correction is retained. Revision
7.2.3.arch1-2 completed one successful dedicated-AMD s2idle cycle, but deep S3
failed the gfx ring test with -110. That comparison also changed the base kernel
from 7.1.11 to 7.2.3 and does not isolate the UVD patch as the sole cause.

Revision 7.2.3.arch1-3 added the default-off, exact-hardware diagnostic parameter
`amdgpu.mbp_skip_s3_reset`. A deep-S3 test confirmed the bypass marker, but the
GPU still failed the gfx ring test after low-level resume. The bypass is not a
working fix. It remains default-off and must stay off for the ring-fix test.

## Upstream GFX6 ring initialization candidate

Revision 7.2.3.arch1-4 keeps the same 7.2.3 base and existing patches, and adds:

1. [Initialize compute rings before CP start](https://github.com/torvalds/linux/commit/c9ddc90f91597ba06c9fcbbfd34f62d4b9b524d0).
   Initialize all three rings before enabling the shared command processor.
2. [Clean up rings during reset](https://github.com/torvalds/linux/commit/5e996008db32112bbf25aba39ea43cd162f4c139).
   Clear stale commands and initialize CPU and hardware ring pointers.

Both upstream patches affect the same initialization path that fails during
S3 resume. They were developed for reset support; they are not confirmed fixes
for MacBook S3. Both apply to the exact upstream 7.2.3 source with zero fuzz.
Compilation and complete source validation run on GitHub Actions.

Before installation, preserve the currently installed custom-kernel package and
boot images. This is an upgrade of `linux-mbp-suspend`, not a parallel kernel
package. Do not assume a stock-kernel recovery entry is installed on the target.
After an approved installation, verify a clean dedicated-AMD boot and confirm
`mbp_skip_s3_reset=N`. Only then perform a separately approved deep-S3 test with
RTC wake. Check actual S3 resume, gfx/SDMA ring health, UVD/VCE initialization,
and usable graphics. Build success alone does not establish hardware success.

## 7.2.11 update branch

`update/7.2.11` starts at installed-source commit
`b6c57b9bfe5848090e4c7e1bfe05846e41885652`. It tracks official Arch packaging
commit `0c178d550ec6392265d3a443333229823634bd08` for `7.2.11.arch1-1`,
retaining the single `linux-mbp-suspend` package and its existing build scope.
Patch application rejects fuzz and reversed/already-applied patches. The
expected release is `7.2.11-arch1-1-mbp-suspend`.

The historical revision-4 candidate description above is retained as context;
its subsequent deep-S3 test failed, with no persisted evidence identifying the
failure stage. No dedicated-AMD sleep fix is claimed by this update.
