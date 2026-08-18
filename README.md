# HyperOS port for the OnePlus 9 Pro

An experimental auto-porter for the **OnePlus 9 Pro** (`lemonadep`, OnePlus9Pro / LE212x family).
It combines a genuine OnePlus 9 Pro stock ROM with a HyperOS donor ROM:

- `vendor` and `odm` come from the **OnePlus 9 Pro stock ROM** so the target hardware stack stays OnePlus/OPlus.
- `system`, `system_ext` and `product` come from the **HyperOS donor**.
- `mi_ext` is merged when the donor contains it; donors without `mi_ext` are also accepted.

The output is an uncompressed zip containing `system.img`, `system_ext.img`, `product.img`, `vendor.img` and `odm.img`.

> This is a first-stage OnePlus 9 Pro porting base, not a claim that every HyperOS donor will boot unchanged. Android/vendor compatibility still matters. Keep a known-good recovery/fastboot path before flashing.

## Why this fork is different from the original OP13 porter

The original project contained OnePlus 13-specific values: ultrasonic-FOD coordinates, OP13 display geometry/configuration, a 600-dpi override, OnePlus 13 vendor display props and an ODM attestation block containing the OnePlus 13 market name. Those values are deliberately **not** reused on the OnePlus 9 Pro.

This branch instead keeps OnePlus 9 Pro stock `vendor`/`odm` as close to stock as possible and only applies generic HyperOS-side assembly changes. Hardware fixes can then be added under `RES/` after they are verified on the actual device.

The reference `coloros_port` project identifies the OnePlus 9 Pro as `OnePlus9Pro` and uses a super size of `11190403072` bytes for OnePlus 9 / 9 Pro. That value is documented in `devices/OnePlus9Pro/device.conf` for future super-image/flash-package work; the current porter still outputs individual dynamic-partition images.

## Requirements

Linux x86_64 is recommended.

```bash
./requirements.sh
```

## Local build

```bash
./port.sh \
  --stock /path/to/OnePlus9Pro-stock.zip \
  --hyperos /path/to/HyperOS-donor.zip
```

Both inputs may be a URL, OTA/fastboot zip, `payload.bin`, or a directory containing raw partition images.

Useful options:

```text
--name <basename>         output zip basename
--out <dir>               output directory (default: out)
--work <dir>              working directory (default: work)
--res <dir>               overlay directory (default: RES)
--density <dpi>           explicitly override HyperOS logical density
--skip-target-check       bypass OnePlus 9 Pro stock marker validation
--keep-work               keep the working tree between runs
```

`port.sh` is a thin wrapper around `port.py`, so both entry points use exactly the same implementation.

## Target validation

After unpacking the stock `vendor` and `odm`, the porter looks for known OnePlus 9 Pro identifiers (`OnePlus9Pro`, `lemonadep`, or LE212x model markers). If none are present, the build stops rather than silently producing an image for the wrong device.

Use `--skip-target-check` only when you have independently verified that the supplied stock package is for the OnePlus 9 Pro but its build props do not expose one of those identifiers.

## Porting flow

1. Extract OnePlus 9 Pro `vendor` and `odm` from the stock ROM.
2. Validate the target stock package.
3. Extract HyperOS `system`, `system_ext` and `product`; extract `mi_ext` when present.
4. Fold `mi_ext/product` into `product` and `mi_ext/system` into `system/system`.
5. Merge `mi_ext/etc/build.prop` while excluding `ro.vendor.build.ab_ota_partitions`.
6. Add the generic MIUI home/dexopt compatibility props.
7. Move `product/pangu/system` into `system/system` when present.
8. Remove `system_ext/priv-app/qcrilmsgtunnel` to avoid carrying the donor Qualcomm RIL tunnel into the OPlus hardware stack.
9. Preserve target `vendor`/`odm` instead of injecting OP13 display/FOD/attestation values.
10. Auto-detect `ro.sf.lcd_density` from the target stock vendor/odm when available; otherwise keep the donor density unless `--density` is supplied.
11. Apply optional files from `RES/<partition>/...`.
12. Regenerate EROFS `fs_config` / `file_contexts`, repack, and create the output zip.

## RES overlays

`RES/` is intentionally empty of OP13 hardware overrides. To test a verified OnePlus 9 Pro fix, mirror the partition path under `RES`, for example:

```text
RES/vendor/etc/...
RES/product/etc/...
RES/system_ext/...
```

The files are copied over the assembled tree before SELinux metadata is regenerated.

## What still needs real-device validation

The first boot should be used to collect evidence before adding hardware-specific patches. In particular, verify:

- fingerprint enrolment and FOD position/brightness,
- 60/120 Hz switching and LTPO/display modes,
- brightness curve and AOD,
- camera providers and MiuiCamera behavior,
- RIL/IMS, Wi-Fi/Bluetooth/NFC,
- audio/Dolby,
- sensors and auto-rotation,
- encryption/decryption and recovery compatibility.

If a build reaches boot animation or Android, the most useful next inputs are `adb logcat`, `dmesg`/kernel log when available, and the relevant `getprop` output. Those are preferable to copying device-specific OP13 constants blindly.

## References / credits

- Original `palazik/HyperOS_OP13` porter and its HyperOS assembly flow.
- `tqmane/coloros_port` for the OnePlus 9 Pro device handling reference and dynamic-partition geometry.
- The uploaded `hyperos-port-to-oneplus_test` project for the OnePlus 9 Pro super-size mapping and general OnePlus porting structure.
- MIO Kitchen / erofs image tools and `payload-dumper-go` used by the project.

## License

GPLv3. See `LICENSE`.
