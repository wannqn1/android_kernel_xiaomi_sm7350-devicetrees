# Linux 6.18 device trees for renoir

The Xiaomi Mi 11 Lite 5G (`renoir`, SM7350 Shima) uses
`renoir/qcom/shima.dts` for its base DTB and
`renoir/qcom/renoir-sm7350-overlay.dts` for its board overlay.

| Repository | Contents |
| --- | --- |
| [Kernel](https://github.com/wannqn1/android_kernel_xiaomi_sm7350/tree/renoir-6.18) | Linux 6.18 source and build scripts |
| [External modules](https://github.com/wannqn1/android_kernel_xiaomi_sm7350-modules/tree/renoir-6.18) | Qualcomm vendor modules |
| [Kernel prebuilts](https://github.com/wannqn1/android_device_xiaomi_renoir-kernel-6.18/tree/renoir-6.18) | Kernel image, DTB, DTBO and modules for Android builds |
| [Renoir Android device tree](https://github.com/wannqn1/android_device_xiaomi_renoir/tree/renoir-6.18) | LineageOS 24.0 product and repository manifest |
| [Xiaomi SM8350 common tree](https://github.com/wannqn1/android_device_xiaomi_sm8350-common/tree/renoir-6.18) | Shared LineageOS configuration |

Follow the [renoir kernel build instructions](https://github.com/wannqn1/android_kernel_xiaomi_sm7350/blob/renoir-6.18/renoir/README.md) to build the DTB and DTBO with the matching kernel and modules.

## Redwood upstream reference

The devicetrees for the **Redmi Note 12 Pro 5G / POCO X5 Pro 5G** (`redwood`,
SM7325 "yupik") on Linux 6.18. This repo also tracks the status of the 6.18 port.

- `qcom/yupik.dts` builds the SoC base dtb (`yupik.dtb`).
- `qcom/redwood-overlay.dtso` builds redwood's board overlay (`dtbo.img`).

The `redwood-*.dtsi` files describe the board hardware: audio, camera, touch,
fingerprint, haptics, IR, NFC, Wi-Fi, Bluetooth, IPA, video, FastRPC, crypto,
thermal and the two M20 panel variants (dynamic refresh at 120, 110, 100, 90 and
60 Hz).

### Building for redwood

The dtb and dtbo are preprocessed with Clang and compiled with the kernel's
`dtc`, so build the kernel first. Then run:

```sh
export R=~/r618    # holds redwood-6.18-kernel, out/ and this repository
bash $R/redwood-6.18-modules/bringup/build-dt.sh
python3 mkdtboimg.py create dtbo.img $R/out-dt/redwood-overlay.dtbo
```

The output is `$R/out-dt/yupik.dtb` and `redwood-overlay.dtbo`.
`mkdtboimg.py` is AOSP's, from `system/libufdt/utils/src`.

### Redwood hardware status

Last updated 2026-11-15, on 6.18.39 with PixelOS 17 as the daily system and
KernelSU root.

Legend: ✅ works · 🟡 partly working or not proven end to end · ⛔ broken or kept off · ⬜ not started

#### Kernel and core

| Item | State | Notes |
|---|---|---|
| Boot | ✅ | GKI built-ins, 311 modules, all load |
| UFS storage | ✅ | Needs `IOMMU_IO_PGTABLE_FAST`. Without fastmap, apps read stale zeros (SIGILL in odex, dex2oat crashes) |
| arm-smmu | ✅ | `pgsize_bitmap` set on the domain, reserved regions restored (453892b). The TLB is flushed on every unmap (`iommu.strict=1`) until the deferred path is fixed |
| Rust | ✅ | ashmem (Rust-only on 6.18) and the Rust Binder (`binder.impl=rust`) |
| SELinux netlink | ✅ | The Android device trees carry the xperm rules. `redwood.nlmsg_legacy` stays as a fallback for older vendor policy |
| Deep sleep | ✅ | s2idle reaches AOSS, CX and DDR sleep (`qcom_stats`) |
| cpuidle | ✅ | TEO with the PSCI cluster domains |
| `qcom_lpm` governor | ⛔ | Panics, so it isn't loaded. TEO replaces it |
| Scheduler | ✅ | WALT removed: EAS, schedutil, fast PELT |
| KernelSU | ✅ | The LKM loads (the GKI KMI is kept) |
| Watchdog | ⬜ | `qcom_wdt_core` and `qcom_soc_wdt` left out |
| CoreSight | ⛔ | `coresight-funnel` resets the SoC, so it's kept out |
| `tz_log` | ⛔ | Probing it resets the phone (same on 6.12), so it isn't bundled |

#### Peripherals

| Item | State | Notes |
|---|---|---|
| Display | ✅ | M20 panel, 1080x2400, up to 120 Hz |
| GPU | ✅ | Adreno 642L, OpenGL ES and Vulkan (the UI renders with Vulkan) |
| Touch | ✅ | Goodix Berlin-D (mainline driver) with double tap to wake |
| Remote processors | ✅ | ADSP, CDSP, MPSS and WPSS all run |
| Modem, mobile data | ✅ | Both SIMs. datarmnet needs `CONFIG_IPA3`, and IPA uses a level-triggered IRQ so the SoC can sleep |
| Wi-Fi | ✅ | WCN6750, qcacld-3.0 |
| Bluetooth | ✅ | |
| Audio | ✅ | Media, calls on the earpiece and the speaker, Dolby |
| Speaker protection | 🟡 | Both AW88261 amps play. The VI feedback path fails to start (AFE port 0x5, -131) |
| Camera | ✅ | Photo, video and 120 fps slow motion |
| Camera, 240 fps | 🟡 | In progress |
| Video codecs | ✅ | msm-vidc sessions for the vendor Codec2 components |
| Sensors | ✅ | 50 sensors |
| Fingerprint | ✅ | |
| Haptics | ✅ | AW8697 |
| IR blaster | ✅ | `ir-spi` |
| Charging | ✅ | |
| Crypto, qseecom, smcinvoke | ✅ | Keymaster and Gatekeeper work |
| NFC | 🟡 | `nxp-nci` loads. Untested: the test unit is an India model without an NFC chip |

#### Android

| Item | State | Notes |
|---|---|---|
| PixelOS 17 | ✅ | Daily system, built from the prebuilt and `device_xiaomi_redwood` (`seventeen`) |
| AOSPA | ✅ | Built from `device_xiaomi_redwood_clo` (`6.18`) |

### Bugs fixed along the way

| Symptom | Cause | Fix |
|---|---|---|
| Oops while arm-smmu registers | `of_iommu_get_resv_regions` used a NULL translation | Kernel 453892b |
| WPSS fatal error after the mode request | WLAN IOVAs outside the firmware's window | Reserved regions back (453892b) |
| SIGILL in odex, dex2oat crashes | UFS DMA without Qualcomm's fastmap | `IOMMU_IO_PGTABLE_FAST=y` |
| zygote restart loop | No `/dev/ashmem` (Rust-only on 6.18) | `CONFIG_RUST`, ashmem, Rust Binder |
| CFI panic | `qcom-pdc` probe type | Probe wrapper |
| Oopses at `0xff000000` | The splash console kept drawing into freed splash memory | Stop and unmap when `msm_drm` loads |
| Silent calls (voice PCM start -EPIPE) | The no-buffer vendor hook is gone in ACK 6.18 | Hook restored, registered in `lahaina.c` |
| Sound cut out every 6 s | ACK moved the ASoC no-BE check to open, so the sound trigger HAL's Listen PCM failed and the audio HAL restarted | Reverted (6fd45bed80d8) |
| Camera ICP init -110, black preview | HFI queues in cached memory | Uncached system heap in `cam_mem_mgr` |
| Camera refcount crash | `cam_smmu` put a dma-buf twice | `get_dma_buf` |
| No `rmnet_data` links (-ENOMEM) | datarmnet built against the `ipa.h` stubs | `CONFIG_IPA3` for datarmnet |
| SoC never reached deep sleep | The CPU cluster domain never synced, and IPA kept the SoC awake | `qcom,lpm-cluster-dev`, `fw_devlink.sync_state=timeout`, level-triggered IPA IRQ |
| USB dropped once on plug-in | UCSI connector changes handled in the RX worker | Kernel 82821c258222 |
| Reset during module load | `coresight-funnel`, `tz_log_dlkm` | Kept out |
| Panic | `qcom_lpm` governor | Kept out, TEO instead |

### Next

1. 240 fps slow motion
2. Speaker VI feedback
3. A proper arm-smmu deferred TLB invalidation, instead of a strict flush
4. CoreSight, the watchdog and the `qcom_lpm` panic

### Testing safely

**Gatekeeper lives in the TEE, and both slots share it.** On an empty /data,
gatekeeperd finds no `/data/misc/gatekeeper/.coldboot` and wipes every
enrolled user once setup completes or a credential is set. The other slot
then rejects the right PIN. Every test boot with a non-daily /data must
create `.coldboot` or set `ro.gsid.image_running=1`, and must not finish
setup or set a lock.

The 6.12 notes this port started from are in [README-6.12.md](README-6.12.md)
and [STATUS.md](STATUS.md).

Ported and maintained by Suvojeet Sengupta.
