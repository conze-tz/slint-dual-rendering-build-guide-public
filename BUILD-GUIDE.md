# Building the Slint Dual-Rendering SDK Image for the Toradex Verdin iMX95

A step-by-step guide to building a Yocto image that combines NXP's SAFC
framework (Cortex-M7 safety UI) with the Slint Dual-Rendering SDK (Cortex-A55
Linux UI), composited onto a single display.

> **Tested host:** Ubuntu 24.04 (also works under WSLv2).
> **Target hardware:** Toradex Verdin iMX95 (B0 silicon) on a Dahlia carrier,
> Toradex DSI-to-HDMI adapter (LT8912B, 1080p60), HDMI monitor.

---

## Overview

The SDK itself contains **no NXP code**. You supply NXP's eCockpit delivery,
and the build combines it with the SDK on your machine. The result is a
provisioning bundle you flash with `uuu`.

---

## Prerequisites

- A Verdin iMX95 on **B0 silicon** (V1.0B, V1.0C, or V1.1x). Earlier A0 modules
  will not boot with eCockpit 1.1.3.
- A Dahlia carrier, the Toradex DSI-to-HDMI adapter on the DSI connector, and
  an HDMI monitor.
- A Linux host with **`uuu` 1.5.243 or newer**:
  ```sh
  uuu -V   # must be >= 1.5.243
  ```
- Your **NXP eCockpit Software Pack, Release 1.1.3** (for i.MX95).
- At least ~100 GB of free disk space for the Yocto build.

---

## Step 0 — Install Yocto host packages

```sh
sudo apt update
sudo apt install -y gawk wget git diffstat unzip texinfo gcc build-essential \
  chrpath socat cpio python3 python3-pip python3-pexpect xz-utils debianutils \
  iputils-ping python3-git python3-jinja2 python3-subunit zstd liblz4-tool \
  file locales libacl1 repo tmux
sudo locale-gen en_US.UTF-8
```

---

## Step 1 — Unpack the NXP delivery

From your `eCockpit_Release_1.1.3` folder you only need **two** archives; the
rest (AAOS release, Android app, `meta-imx-ecockpit-release`, the PDFs) is
**not used**:

| Archive | Provides |
| --- | --- |
| `safc-ecockpit-lvgl-cluster-application_evk_mx95_SAFC_ECOCKPIT_LVGL_CLUSTER_RELEASE_1.1.3.tar.gz` | SAFC framework, MCU SDK, shared config, build tools |
| `meta-safc_ecockpit-lvgl-cluster_1.1.3.tar.gz` | The `sa-vkms` virtual-KMS kernel driver |

The first archive is a bundle of four sub-archives plus NXP's
`unpack_release.sh`. Unpack everything into one directory:

```sh
mkdir -p ~/nxp-safc && cd ~/nxp-safc

# Adjust the path to your delivery (e.g. a mounted drive under /mnt/...)
tar xf "/path/to/eCockpit_Release_1.1.3/safc-ecockpit-lvgl-cluster-application_evk_mx95_SAFC_ECOCKPIT_LVGL_CLUSTER_RELEASE_1.1.3.tar.gz"
```

### If `unpack_release.sh` fails with "Cannot open: No such file or directory"

NXP's script looks for the four sub-archives **one level above** its own
location. Run it from a subdirectory so that `..` resolves to where the
sub-archives actually are:

```sh
cd ~/nxp-safc
ls *_SAFC_ECOCKPIT_LVGL_CLUSTER_RELEASE_1.1.3.tar.gz   # confirm they are here

mkdir -p staging && cd staging
bash ../unpack_release.sh
```

Then add the `sa-vkms` layer into the same staging directory:

```sh
tar xf "/path/to/eCockpit_Release_1.1.3/meta-safc_ecockpit-lvgl-cluster_1.1.3.tar.gz" \
  -C ~/nxp-safc/staging
```

### Verify the delivery is complete

```sh
ls -la ~/nxp-safc/staging
ls -d ~/nxp-safc/staging/meta-safc
find ~/nxp-safc/staging/safc-firmware -name "libsafc_lib.a"
```

You should see all five components and `libsafc_lib.a`:

```
~/nxp-safc/staging/
├── safc-firmware/        (contains lib/libsafc_lib.a)
├── mcu-sdk-safc/
├── safc-shared-config/
├── safc-tools/
└── meta-safc/
```

**Your `SAFC_RELEASE_DIR` is `~/nxp-safc/staging`.**

---

## Step 2 — Validate the delivery with `overlay.sh` (dry run)

Copy the SDK into the Linux filesystem first (avoid building on NTFS/`/mnt`
mounts), then let the SDK fingerprint your delivery:

```sh
cp -r /path/to/slint-dual-rendering-sdk ~/slint-dual-rendering-sdk
cd ~/slint-dual-rendering-sdk
tools/overlay.sh --safc-release ~/nxp-safc/staging --out ~/slint-work/ --force
```

A successful run prints:

```
Framework: eCockpit 1.1.3 (i.MX95 B0)
```

That confirms the correct version **and** B0 silicon. (If it reports an
untested release, you can add `--allow-unknown-release`, but with a clean 1.1.3
that should not be needed.)

---

## Step 3 — Set up the Toradex BSP (walnascar)

> **Note:** walnascar is an unreleased integration branch. Confirm the exact
> manifest URL/branch in the Toradex documentation.

```sh
mkdir -p ~/verdin-bsp && cd ~/verdin-bsp
repo init -u https://git.toradex.com/toradex-manifest.git -b walnascar \
  -m tdxref/default.xml
repo sync -j"$(nproc)"
```

---

## Step 4 — Initialize the build environment

```sh
cd ~/verdin-bsp
MACHINE=verdin-imx95 . export
```

This drops you into `~/verdin-bsp/build/` and creates `conf/local.conf` and
`conf/bblayers.conf`.

> `MACHINE=... . export` only sets `MACHINE` for the current shell. It must
> also be pinned in `local.conf` (Step 6), or bitbake fails with
> `Directory name ... contains unexpanded bitbake variable ${MACHINE}`.

---

## Step 5 — Clone and register the layers

The SDK layer lives in your SDK copy. Clone the two external layers:

```sh
cd ~/verdin-bsp/layers
git clone https://github.com/slint-ui/meta-slint
git clone https://github.com/rust-embedded/meta-rust-bin
git clone -b walnascar https://github.com/kraj/meta-clang
```

Verify each is walnascar-compatible:

```sh
grep LAYERSERIES_COMPAT layers/meta-slint/conf/layer.conf
grep LAYERSERIES_COMPAT layers/meta-rust-bin/conf/layer.conf
grep LAYERSERIES_COMPAT layers/meta-clang/conf/layer.conf
```

Each must list `walnascar`.

Add them to `~/verdin-bsp/build/conf/bblayers.conf`. Order matters:
`meta-clang` must come **before** the Slint layer, because the Linux demo
depends on `clang-cross-${TARGET_ARCH}`.

```
BBLAYERS += " \
  ${TOPDIR}/../layers/meta-rust-bin \
  ${TOPDIR}/../layers/meta-clang \
  ${TOPDIR}/../layers/meta-slint \
  /home/YOUR_USER/slint-dual-rendering-sdk/meta-slint-dual-rendering \
"
```

Check:

```sh
cd ~/verdin-bsp/build
bitbake-layers show-layers | grep -iE "clang|slint|rust"
```

---

## Step 6 — Configure `local.conf`

```sh
cd ~/verdin-bsp/build
cat >> conf/local.conf <<'EOF'

# --- Slint Dual-Rendering SDK ---
MACHINE = "verdin-imx95"
SAFC_RELEASE_DIR = "/home/YOUR_USER/nxp-safc/staging"
ACCEPT_FSL_EULA = "1"
EOF
```

Verify:

```sh
grep -nE "SAFC_RELEASE_DIR|^MACHINE|ACCEPT_FSL_EULA" conf/local.conf
```

---

## Step 7 — Parse-test the layer stack

Cheap sanity check (parses all recipes, builds nothing):

```sh
bitbake -p
```

A clean result looks like:

```
Parsing of NNNN .bb files complete (... 0 errors).
```

---

## Step 8 — Build the image

Run inside `tmux` so the build survives a closed terminal:

```sh
tmux new -s build
cd ~/verdin-bsp/build
bitbake slint-dual-render-image
```

tmux controls:
- Detach (build keeps running): `Ctrl-b` then `d`
- Reattach: `tmux attach -t build`
- Scroll back: `Ctrl-b` then `[`, arrow keys, `q` to quit

The first build takes several hours (toolchain, kernel with `sa-vkms`, SAFC
device tree, both Slint halves, M7 firmware, boot container, provisioning
bundle).

---

## Step 9 — Locate the provisioning bundle

```sh
ls -la ~/verdin-bsp/build/tmp/deploy/images/verdin-imx95/slint-dual-render-bundle-verdin-imx95/
```

Contents:

| File | Purpose |
| --- | --- |
| `safc-imx-boot.bin` | SM + M7 firmware + data + U-Boot |
| `ramboot.bin` | Stock container (`flash_a55`), RAM-booted as the flashing agent |
| `demo.wic.part0` / `demo.wic.part1` | Disk image split for `uuu`'s 512 MB transfer limit |
| `emmc-provision.lst` | uuu script; block counts computed from real artifact sizes |
| `boot0-only.lst` | M7 firmware only (fast firmware iteration) |
| `tezi-load.lst` | |
| `README.md` | Its stated md5 matches `part0 ‖ part1` |

---

## Step 10 — Flash with `uuu`

**Always flash from the generated bundle, never a hand-kept copy** — the `.lst`
block counts are computed from the real artifact sizes; a stale copy writes
past the end of the image.

```sh
cd ~/verdin-bsp/build/tmp/deploy/images/verdin-imx95/slint-dual-render-bundle-verdin-imx95/
sudo uuu emmc-provision.lst
```

Enter recovery mode first: hold the **RECOVERY** button through a **cold
power-on** (unplug and replug the supply), not a reset. Board USB IDs:
`1fc9:015d` (ROM), `1b67:4059` (U-Boot).

After flashing:
- **Power-cycle** the board (unplug/replug) — U-Boot may otherwise come up in
  the USB download gadget.
- **Remove any SD card** — `mmc1` takes priority over eMMC (`boot_targets` is
  `mmc1 mmc0 dhcp`).

---

## Step 11 — Verify a healthy boot

On the M7 console:

```
verdin serdes: bridge @0x48 programmed (123 cmds, err 0)
start slint
Starting virtual KMS task (Linux)
```

On the Linux side, within ~5 seconds:

```sh
dmesg | grep -i vkms
# handshake received
# Initialized sa-vkms 0.2.0 ... minor 0
```

The monitor shows the composited output (safety UI over the Linux frame,
1080p60).

---

## Troubleshooting

### Build: `Nothing PROVIDES 'clang-cross-aarch64'`

`meta-clang` is missing from the layer stack. See Step 5 — clone it and add it
to `bblayers.conf` **before** the Slint layer.

### Build: `contains unexpanded bitbake variable ${MACHINE}`

`MACHINE` is not set in `local.conf`. See Step 6.

### Build: `No stock boot container ... flash_a55`

Set `SLINT_STOCK_BOOT_TARGET` in `local.conf` to whichever stock, non-SAFC
container the BSP actually built (the recipe lists the available options in its
error message).

### Flash succeeded, M7 runs, but Linux never starts

U-Boot has a stale `bootcmd` left over from recovery-mode work (e.g.
`fastboot 0`). It lives in the last sectors of the `boot0` hardware partition
and survives reflashing. Catch the U-Boot prompt and repair only that variable:

```
Verdin iMX95 # env default -f bootcmd
Verdin iMX95 # setenv fdtfile imx95-verdin-wifi-dahlia-safc.dtb
Verdin iMX95 # saveenv
Verdin iMX95 # boot
```

Do **not** run `env default -a` — it discards the Toradex-specific variables
the board needs. A healthy boot shows `Machine model: Toradex Verdin iMX95 WB
on Dahlia (SAFC)` — the `(SAFC)` confirms the right device tree loaded.

### Screen stays dark, Linux logs `flip_done timed out`

The framework is parked in `INIT`. From the M7 console:

```
SAFC > fw state set run
```

### `uuu` requires 1.5.243 or newer

Older builds fail the i.MX95 SDPS-to-SDPV handoff with
`LIBUSB_ERROR_NO_DEVICE`. Run `uuu` with `sudo`.

---

## Notes

- Build in the Linux filesystem (`~`, ext4), not on `/mnt/c` or `/mnt/s` — NTFS
  mounts cause permission/symlink/case-sensitivity problems.
- Under WSLv2, the VHDX grows dynamically but does not shrink automatically;
  keep an eye on free space.
- This guide was derived from a working build session; the SDK's own docs
  (`docs/NXP-DELIVERY.md`, `docs/STATUS.md`, `docs/TROUBLESHOOTING.md`,
  `docs/WRITING-YOUR-UI.md`) remain the authoritative reference.
