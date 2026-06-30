# HP OmniBook Ultra Flip (14-fh0xxx)

This repo documents issues and solutions I've experienced for the HP OmniBook Ultra Flip (Lunar Lake).

Most things work out of the box for me on Arch Linux, including the speakers and fingerprint sensor.

## Auto Rotation

By default, auto-rotation is broken under linux. This is because the kernel loads the Intel Integrated Sensor Hub firmware from `/usr/lib/firmware/intel/ish/` at boot, but the kernel doesn't currently ship the correct HP-signed blob.

If you check `sudo dmesg | grep ish`, the kernel tries to load ISH firmware and fails:

```
intel_ish_ipc 0000:00:12.0: ISH loader: load firmware: intel/ish/ish_lnlm.bin
intel_ish_ipc 0000:00:12.0: ISH loader: cmd 2 failed 10
```

HP's own Windows driver package ships the correct HP-signed image - so as a workaround until HP submits the correct firmware upstream, you can extract it and
drop it where the kernel looks.

### Extracting the firmware

The firmware blob is included in this repo, but if you want to extract it yourself:

HP's own Windows driver package ships the correct HP-signed image, so you can extract it and drop it where the kernel looks.
Download the "Intel Integrated Sensor Solution Driver" (sp172652) from HP's support page for the OmniBook Ultra Flip 14-fh0xxx. Newer revisions should also work, just use the matching `ishC_SI_*.bin`.

The SoftPaq is a self-extracting installer. On Linux, unpack it without running it:

```bash
# p7zip provides 7z; on Arch: sudo pacman -S p7zip
7z x sp172652.exe -osp172652
```

The firmware you want is here:

```
sp172652/src/Driver/IshHeciExtensionTemplate/FWImage/0003/ishC_SI_20260309.bin
```

The `ishC_SI_*` filename includes a build date and may differ between SoftPaq revisions, just use whatever `ishC_SI_*.bin` is in that folder.

### Target Filename

The kernel looks for a file named from CRC32 hashes of your DMI fields, most-specific
first:

```
ish_lnlm_<sys_vendor>_<product_family>_<product_name>_<product_sku>.bin
```

For the OmniBook Ultra that resolves to:

```
ish_lnlm_12128606_e0c3b2ba_cf5da58b_146bbbed.bin
```

If you have a different model/SKU, you can compute your own hashes:

```bash
for f in sys_vendor product_family product_name product_sku; do
  v=$(cat /sys/class/dmi/id/$f)
  printf '%s = %08x\n' "$f" \
    "$(python3 -c "import zlib,sys; print(zlib.crc32(sys.argv[1].encode()))" "$v")"
done
```

Assemble them in the order `sys_vendor_product_family_product_name_product_sku`.

### Install Firmware

Either extract the firmware bin or download it from this repo. Then install it:

```bash
sudo cp ishC_SI_20260309.bin \
  /usr/lib/firmware/intel/ish/ish_lnlm_12128606_e0c3b2ba_cf5da58b_146bbbed.bin
```

Rebuild the initramfs so the firmware is available early at boot:

```bash
# Arch Linux
sudo mkinitcpio -P

# Debian / Ubuntu
sudo update-initramfs -u
```

Reboot, and auto-rotation should work.

### Credits

Approach mirrors the Zenbook S14 (UX5406SA) fix by
[dantmnf](https://github.com/dantmnf/zenbook-s14-ux5406sa-linux), which uses the
equivalent OEM-signed image from the ASUS driver package.

## HDR

Out of the box in KDE, HDR is not detected.
This can be fixed by setting the environment variable `KWIN_FORCE_ASSUME_HDR_SUPPORT=1` in `/etc/environment`.
See [this thread](https://discussion.fedoraproject.org/t/unable-to-activate-hdr/180708) for more details.
For whatever reason, enabling HDR increases the maximum brightness of the display, so I would reccomend doing this even if you don't have any HDR content.

In KDE, I would make sure to calibrate HDR brightness and set sRGB colour intensity in the Display Settings. I found a brightness of 590 cd/m^2 for both maximum and paper white luminance to be best, as well as setting sRGB intensity to 100%.
