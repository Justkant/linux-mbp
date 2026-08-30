# linux-mbp

Reproducible Arch Linux kernel packaging for the MacBookPro11,5. The repository
tracks Arch's official `linux` package, applies a small reviewed patch series,
and builds a separately named package with GitHub Actions.

The current package identity remains `linux-mbp-suspend` so an update preserves
the existing rEFInd and mkinitcpio recovery setup. The repository's broader
`linux-mbp` name allows additional MacBook-specific fixes to be reviewed here
without implying that every patch is limited to suspend.

## Patch series

1. `amdgpu-skip-suspend-reset-when-switcheroo-off.patch`
   skips the deep-suspend ASIC reset only when DRM switcheroo has already put
   the discrete AMD GPU in `DRM_SWITCH_POWER_OFF`.
2. `mtrr-expedite-cache-init-on-resume.patch`
   backports Chris Paulson-Ellis's pending x86/MTRR fix for very slow secondary
   CPU bring-up after S3 on a mid-2015 MacBook Pro. The upstream submission is
   <https://www.spinics.net/lists/kernel/msg6037880.html>.

The MTRR patch is not yet merged upstream. A successful build proves only that
it applies and compiles; its resume warning and measured CPU-hotplug timings are
the runtime acceptance criteria.

## CI build

Every push, pull request, tag, and manual dispatch runs
`.github/workflows/build.yml`. The workflow:

- reclaims unused SDK space on GitHub's disposable Ubuntu runner;
- starts a clean official `archlinux:base-devel` container;
- imports the PGP keys shipped by Arch's package repository;
- verifies the signed kernel and Arch patchset plus every checksum;
- runs an unprivileged `makepkg` build inside the container;
- restores a bounded 5 GiB `ccache` and reports its hit rate;
- uploads the package and `SHA256SUMS` for seven days.

GitHub's standard runner starts with limited free disk. The workflow requires
25 GiB after its explicit cleanup and fails before compilation if that cannot be
provided. All destructive cleanup commands target only the ephemeral runner.

The compiler cache uses makepkg's native `ccache` build environment. Its key
includes the kernel configuration, `PKGBUILD`, and local patch series, while a
broader fallback permits safe content-addressed reuse across nearby revisions.
The complete mutable kernel source/build directory is never cached. A first run
populates the cache; later runs report direct/preprocessed hits and misses in
the build log.

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

This package installs alongside Arch's stock `linux` package. It must not
conflict with or replace `linux`. Keep the stock kernel and AMD rEFInd path as
recovery choices.

Before installing, verify the package metadata and available `/boot` space:

```bash
pacman -Qip ./linux-mbp-suspend-*.pkg.tar.zst
df -B1 /boot
```

On the target MacBook, `/boot` is normally read-only and has little spare
space. Do not enable a fallback image. Install only after confirming at least
80,000,000 bytes are free:

```bash
sudo mount -o remount,rw /boot
sudo pacman -U ./linux-mbp-suspend-*.pkg.tar.zst
pacman -Q linux linux-mbp-suspend
ls -lh /boot/vmlinuz-linux /boot/initramfs-linux.img \
  /boot/vmlinuz-linux-mbp-suspend /boot/initramfs-linux-mbp-suspend.img
cat /etc/mkinitcpio.d/linux-mbp-suspend.preset
sync
sudo mount -o remount,ro /boot
```

Do not reboot if package installation or initramfs generation fails. Recover
while `/boot` is still writable, then retain the stock kernel as the first boot
choice.

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
3. Keep the custom package name and append both local patches after Arch's
   patchset in `source`.
4. Run `makepkg --verifysource` and verify both patches apply with zero fuzz.
5. Push a branch and let CI compile it before merging or installing anything.

An upstream kernel update is intentionally not installed automatically. A
patch conflict should stop CI and require review rather than silently changing
the target code.

`scripts/check-arch-update` compares `ARCH_PACKAGING_COMMIT` with the official
packaging repository. A scheduled read-only check runs weekly and opens one
GitHub issue when review is needed; it never rewrites packaging or publishes a
kernel automatically.
