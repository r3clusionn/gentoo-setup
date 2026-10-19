# gentoo-setup

The Portage and system configuration of a minimal Gentoo laptop: 155 installed packages, about 145 MiB of memory in use after boot with X, dwm and a terminal running, and a login prompt 1.4 s after the kernel starts. It holds `make.conf` with GCC LTO and `-O3`, the USE flags, the `package.provided` and `profile/packages` entries that take packages out of the dependency graph (each with the reason it can go), the 33-package world set, and the small system files that replace whole packages: a clock sync without chrony or ntpd, a weekly SSD trim without cron, and Wi-Fi fixes for the RTL8922AE. It is made for one laptop (HP OmniBook 7 14-kh0xxx, Ryzen AI 9 HX 475, OpenRC, X11 with dwm, no desktop environment); the kernel, display, dwm and scripts each have their own repo (see How to install).

**Status:** the running configuration of the target laptop since October 2026, at 155 packages since 2026-10-18 (256 on 2026-10-11).

![Terminal output on a black background: "rfetch" 22 seconds after boot prints a white ASCII-art cat and next to it: rec@gentoo; OS: Gentoo Linux x86_64; Host: HP OmniBook 7 Laptop 14-kh0xxx; Kernel: Linux 7.3.0-rc7-min-b29; Uptime: 22 secs; Packages: 155 (emerge); Shell: bash 5.3.20; Terminal: st; WM: dwm (X11); Display (SDC4215): 2880x1800 @ 120 Hz in 14" [Built-in]; CPU: AMD Ryzen AI 9 HX 475 (8+16) @ 5.30 GHz; GPU: AMD Radeon 890M Graphics [Integrated]; Memory: 145.16 MiB / 54.63 GiB (0%); Swap: 0 B / 16.00 GiB (0%); Disk (/): 12.65 GiB / 936.41 GiB (1%) - xfs; Local IP (wlan0); Battery (Primary): 100% [Discharging, 3.8 W]; Power plan: balanced (auto); Locale: en_US.utf8; then two rows of colour blocks.](docs/images/rfetch.png)

## Features

- **`portage/make.conf`:** `-march=native -O3 -flto` for every package, with `-fno-semantic-interposition`, no stack protector, CET, stack clash protection or `_FORTIFY_SOURCE` (a deliberate trade-off for a single-user laptop), and LTO type-mismatch warnings as errors so a broken LTO build fails instead of miscompiling. `--as-needed`, packed relative relocations, lazy binding and no RELRO for faster program starts. Binary packages of everything built (`buildpkg`, zstd -19), so a package removed today can come back in seconds. Portage runs at idle priority, fetches with curl (no wget) and asks glibc's malloc for huge pages in build processes (about 4% faster builds, measured on glslang).
- **`portage/package.use/`:** `minimize` (one line per removed feature, dated and with its reason: no netifrc, no X in PipeWire, ffmpeg TLS through openssl so gnutls and its libraries go, no xattr or ACL support anywhere, ncurses minimal, openrc-init as PID 1 instead of sysvinit), `batch20` (GCC and glibc without the hardening defaults; packages that would pull in OpenMP), `lto-pgo` (PGO for Python, GCC, bash, xz and binutils), `zlib-ng` (zlib-ng in place of zlib, same `libz.so.1` ABI), dash as `/bin/sh`, iwd standalone (its own DHCP, no dhcpcd).
- **`portage/package.env` and `env/`:** `-O3` again for the packages whose ebuilds lower it and that run hot loops (ffmpeg, dav1d, mpv, libplacebo, libass, mesa, pixman, zstd, zlib-ng, PipeWire and the audio codecs); PipeWire without the Bluetooth codecs the headset can't use; the X server without XDMCP.
- **`portage/profile/package.provided`:** 68 packages Portage would install but nothing here uses at run time, each with a comment: empty `acct-group` packages for groups that stay in `/etc/group`, install-time helpers (locale-gen, desktop-file-utils), xrdb (replaced by [xresload](https://github.com/r3clusionn/xresload)), openresolv (replaced by `resolvconf` in [scripts](https://github.com/r3clusionn/scripts)), libXdmcp and libbsd (the X server is built without them), the virtuals whose providers are installed anyway, and wireless-regdb (built into the kernel).
- **`portage/profile/packages`:** what is taken out of `@system`: wget, net-tools, e2fsprogs, ssh, man pages, `which`, `less`, nano, kbd, iproute2 (the loopback interface comes up with `lo-up` from scripts), and the `app-alternatives` packages that only own symlinks.
- **`portage/repos.conf/gentoo.conf`:** the tree synced over shallow HTTPS git instead of rsync with signature checks (no gnupg chain).
- **`world`:** the 33 packages asked for; the other 122 are their dependencies.
- **`etc/local.d/ntp.start`:** sets the clock once per boot from the `Date` header of an HTTPS response, after Wi-Fi brings up a default route (1 s accuracy, no chrony, ntpd or openntpd).
- **`etc/local.d/fstrim.start`:** a TRIM of `/` and `/efi` at boot when the last one is more than 7 days old, at idle I/O priority. There is no cron.
- **`etc/local.d/nvme-hmb.start` and `etc/elogind/system-sleep/nvme-hmb`:** the DRAM-less SSD's host memory buffer off (64 MiB of RAM it would otherwise borrow), at boot and again after resume.
- **Wi-Fi (RTL8922AE):** `etc/modprobe.d/rtw89.conf` turns off the driver's power saving (50–150 ms LAN latency and a flaky link after resume with it on); `etc/elogind/system-sleep/rtw89` unloads the driver before suspend and loads it again after, so the card starts with fresh firmware state; `etc/iwd/main.conf` keeps the interface named `wlan0` across driver reloads, roams at −75 dBm, and writes DNS through `resolvconf`.
- **`etc/sysctl.d/90-tuning.conf`:** memory reserves sized for 64 GB instead of scaled up blindly, no proactive compaction (kcompactd sleeps instead of waking every 500 ms), no watermark boosting, longer writeback intervals.
- **Boot speed:** `etc/conf.d/hwclock` skips reading the RTC (the kernel already set the clock from it), and nothing in the boot runlevel waits for `udev-settle`. `etc/conf.d/agetty.tty1` logs in by itself only on a boot with `lowram=1` on the command line (the low-RAM screenshot in scripts).
- **`etc/X11/xorg.conf.d/`:** amdgpu with TearFree and variable refresh (the panel does 30–120 Hz), libinput tapping and natural scrolling, US and Canadian French layouts on Caps Lock.

The leftovers for gentoo-sources and linux-firmware are left out: the kernel is built from mainline sources with its firmware built in, so neither is installed.

## How to install

These are one machine's files, not an installer: read each one and copy what fits. Requires Gentoo with OpenRC and Portage.

```sh
git clone https://github.com/r3clusionn/gentoo-setup
cd gentoo-setup
doas cp -r portage/. /etc/portage/        # make.conf, package.use, env, profile, repos.conf
doas cp -r etc/. /etc/                     # local.d, sysctl.d, modprobe.d, iwd, X11, conf.d, sleep hooks
doas emerge -1 --noreplace $(cat world)    # the world set
doas emerge -uDN @world && doas emerge --depclean
```

Set `-march`, `MAKEOPTS`, `CPU_FLAGS_X86` (`portage/package.use/00cpu-flags`, from `cpuid2cpuflags`) and `VIDEO_CARDS` for your machine first, and pick mirrors near you. The rest of the system:

- **Kernel:** [kernel-config](https://github.com/r3clusionn/kernel-config).
- **dwm, st, dmenu and the X session:** [dwm-config](https://github.com/r3clusionn/dwm-config).
- **Power plans, audio and Bluetooth on demand, `xclip`, `lo-up`, `resolvconf`, `update`, the udev rules and `power.start`:** [scripts](https://github.com/r3clusionn/scripts).
- **10-bit X and accurate colours:** [omnibook7-display](https://github.com/r3clusionn/omnibook7-display); speakers: [omnibook7-speakers](https://github.com/r3clusionn/omnibook7-speakers).
- **Boot entries without efibootmgr:** [efi-entry](https://github.com/r3clusionn/efi-entry).

Runlevels as used: `boot` has bootmisc, elogind, fsck, hostname, hwclock, lo (instead of loopback), localmount, modules, mtab, procfs, root, seedrng, swap, sysctl and systemd-tmpfiles-setup; `default` has the two gettys (agetty.tty1, agetty.tty2), dbus, iwd and local. consolefont, keymaps, termencoding, netmount and udev-settle are not in any runlevel.

## How to use

```sh
doas update                     # from scripts: sync, update @world, depclean, news
grep -c . world                 # 33 asked for
ls -d /var/db/pkg/*/* | wc -l   # 155 installed
```

Before removing a package, check it in both directions: `emerge -pv --depclean cat/pkg` for what still needs it (and the ebuilds' `PDEPEND` and `IDEPEND`, which are easy to miss), and `ldd` on programs installed outside Portage (Thorium, Obsidian, AppImages), which Portage doesn't know about. Keep its binary package (`quickpkg`) so it can come back.

## How it works

- **Taking packages out of the graph.** `package.provided` tells Portage a package is installed when it isn't, so nothing pulls it in; it is used only where the dependency is unneeded at run time here. `profile/packages` with `-*cat/pkg` removes a package from the profile's `@system`. `INSTALL_MASK` keeps documentation, man pages, translations, the udev hardware database's vendor lists and GCC's plugin headers off the disk.
- **Hardening off.** GCC's `-default-*` USE flags and `-pie -ssp -cet` change the compiler's defaults, `NOHARDEN_FLAGS` covers what the defaults don't, and glibc and binutils are built without CET. The kernel side is in kernel-config.
- **The clock.** `ntp.start` waits for a default route in `/proc/net/route` (iproute2 is gone), then tries three HTTPS hosts and sets the clock and the RTC from the first `Date` header. `curl -k` because a clock far off would fail certificate validation; the header is only used for the time.
- **Suspend hooks.** elogind runs every executable in `/etc/elogind/system-sleep/` with `pre` before suspend and `post` after resume.

## Measurements

On the target laptop, with the kernel from kernel-config:

| | 2026-10-11 | 2026-10-17 | now |
|---|---|---|---|
| Installed packages | 256 | 183 | 155 |
| Memory in use after boot (rfetch, X + dwm + st) | | 185.8 MiB | 145.2 MiB |

- Login prompt 1.38 s after the kernel starts (was 4.09 s before `consolefont`'s wait for `udev-settle` and hwclock's wait for the RTC tick were removed).
- Disk: 12.65 GiB used on `/` (XFS), with Thorium, the media apps and the Node.js and Go toolchains.
- Malloc huge pages in build processes: glslang built in 77.1 s instead of 80.3 s.

## Verification

- Every file here is a copy of the one on the laptop, except `GENTOO_MIRRORS` in `make.conf`, which is set to Gentoo's default mirror.
- On the laptop: PID 1 is `openrc-init`, `/bin/sh` is dash, `vm.compaction_proactiveness` and `vm.watermark_boost_factor` read 0, the SSD's `hmb` reads 0, `rtw89_core`'s `disable_ps_mode` reads Y, and `/var/lib/misc/fstrim.last` carries the date of the last trim.
- `emerge -puDN @world` and `emerge -p --depclean` have nothing to do on the laptop with these files (2026-10-19).

## Limits

- One laptop's choices: no hardening, no ssh, no man pages, no text editor in the base system, no cron. Several of them are trade-offs that suit a single-user machine and not a server.
- `package.provided` entries hide dependencies: if a package that really needs one of them is installed later, it will fail at run time instead of at install time. Check the comments before installing new software.
- The clock sync has 1 s resolution and runs once per boot.

## License

MIT (see `LICENSE`).
