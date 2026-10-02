### Build: kernel fails with `No rule to make target '...imx95-verdin-wifi-zinnia.dtb'`

The Toradex machine config (`meta-toradex-nxp/conf/machine/verdin-imx95.conf`)
lists `imx95-verdin-wifi-zinnia.dtb` and `imx95-verdin-nonwifi-zinnia.dtb` in
`KERNEL_DEVICETREE`, but the pinned walnascar kernel (6.12.x) does not ship the
`zinnia` device-tree sources. This is a symptom of walnascar being an
unreleased integration branch. The kernel itself builds fine; only the DTB
phase fails.

Remove the missing DTBs in `local.conf`:

```sh
cat >> ~/verdin-bsp/build/conf/local.conf <<'EOF'

# zinnia DTS is missing in the pinned walnascar kernel; drop it from the DTB list
KERNEL_DEVICETREE:remove = "freescale/imx95-verdin-wifi-zinnia.dtb freescale/imx95-verdin-nonwifi-zinnia.dtb"
EOF
```

Confirm it took effect (should print nothing):

```sh
cd ~/verdin-bsp/build
bitbake -e linux-toradex-imx | grep "^KERNEL_DEVICETREE=" | tr ' ' '\n' | grep -i zinnia
```

Then rebuild the kernel and the image:

```sh
bitbake -c cleansstate linux-toradex-imx
bitbake slint-dual-render-image
```

To check which Verdin device trees your kernel actually ships, list the
sources and compare against the config block (lines 27–45):

```sh
ls ~/verdin-bsp/build/tmp/work-shared/verdin-imx95/kernel-source/arch/arm64/boot/dts/freescale/ | grep imx95-verdin
```

### `uuu` requires 1.5.243 or newer