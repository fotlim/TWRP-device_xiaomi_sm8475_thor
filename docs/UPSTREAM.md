# 上游致谢与固定版本

本仓库不是从零开始的。下面列出所有实质上游，以及本项目固定的版本，
以便任何人能复现同一份镜像。

---

## 一、基础构建框架

| 上游 | 用途 | 本项目固定的版本 / 引用 |
| --- | --- | --- |
| [Team Win Recovery Project](https://github.com/TeamWin/android_bootable_recovery) | Recovery 主体、GUI、分区管理 | 经 [minimal-manifest-twrp](https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp) tag `android-12.1.0_r4` 引入 |
| [platform_manifest_twrp_aosp](https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp) | manifest（`android-12.1.0_r4`） | tag `android-12.1.0_r4` |
| [OrangeFox Recovery](https://gitlab.com/OrangeFox) | 主题、`orangefox.mk`、`FFiles`、`customzip` 机制 | [sync 脚手架](https://gitlab.com/OrangeFox/sync) `master`（`03434a833701f73430cd085738ae0bf54f2231f1`） |
| [AOSP](https://android.googlesource.com/) | `build/make`、`system/vold`、`system/keymaster`、`system/security/keystore2`、`system/libvintf`、`external/avb` | 随 12.1 manifest |

## 二、设备适配来源

| 上游 | 用途 | 引用 |
| --- | --- | --- |
| `dopaemon/TWRP-device_xiaomi_sm8475` | 本设备树的起点（实为 OrangeFox R12.1 的 **mayfly / 小米 12S** 树，含 `twrp_thor/mayfly/diting/unicorn` 四个产品） | branch `android-14`，HEAD `73dbad51fe5940a524679ba190267ee89d1aca85`（2024-07-04） |
| `LineageOS/android_hardware_qcom_bootctrl` | `bootctrl.xiaomi_sm8475.recovery` | `lineage-21.0-caf` |
| `LineageOS/android_device_xiaomi_sm8450-common` | `gpt-utils`、`init` 骨架、`recovery.fstab` 判定 | 参考实现 |
| `7ac9d42/twrp_device_xiaomi_SM8475` | README / docs / 许可证布局的参考风格；同为 SM8475 平台的 TWRP 16.0 适配 | branch `main` |
| 小米 12S Ultra 官方 LineageOS 构建 | 作为「已知能启动」的对照基线、原厂 recovery 还原包 | [thor builds API](https://download.lineageos.org/api/v2/devices/thor/builds)，`24.0-20261017-NIGHTLY-thor` |

## 三、固件与闭源二进制（不随本仓库分发）

以下内容在**构建时**从设备或厂商固件取得，本仓库不包含、也不再分发它们：

- 预编译内核 `prebuilt/thor/kernel`（`Linux 5.10.177-android12-9-00001`，小米 / 高通版权）
- `prebuilt/thor/vendor/` 下的内核模块与 HAL 二进制（`.ko`、`qseecomd`、
  `android.hardware.gatekeeper@1.0-service-qti`、`android.hardware.keymaster@4.1-service-qti`、
  `vendor.xiaomi.hardware.vibratorfeature.service`、`libkeymasterdeviceutils.so` 等）
- 触摸固件 `st_fts_l1.ftb` / `st_fts_l1_thp.ftb` / `stm_fts_production_limits.csv`
- `security/miui_releasekey.x509.pem`（MIUI OTA 验证用的**公钥**证书，本身不含私钥）

这些文件的所有权与再分发条件归各自权利人，详见 [LICENSING.md](LICENSING.md)。

## 四、致谢

感谢 **Team Win、Dees_Troy、bigbiff、TWRP 社区、OrangeFox 团队、AOSP、LineageOS、
The Linux Foundation / Qualcomm（`gpt-utils` 与 `recovery-ufs-bsg` 的上游）、
dopaemon（设备树起点）、酷安 @做梦书（thor 的可启动参考包，用于对照分析）、
7ac9d42（README 与文档布局参考）、ymdzq 与 EROFS 的作者**。

**Attribution does not imply endorsement** —— 上述项目与个人未审阅、未测试、未背书本仓库。
本项目的问题与缺陷只与本仓库的维护者有关。

## 五、修复过程中的外部参考

- Android 12.1 `system/vold` 对 FBE / metadata 加密的实现（`MetadataCrypt.cpp`、
  `fscrypt_mount_metadata_encrypted` 的 7 参签名）
- `system/security/keystore2` 的 `globals.rs`（`keymint_service_name()`、
  `get_aidl_instances()`）与 `km_compat` 回退路径
- `system/libvintf` 的 AIDL 版本语义（`kFakeAidlMajorVersion = SIZE_MAX`，
  `<version>N</version>` 即 minorVersion）
- `bootable/recovery/kernel_module_loader.cpp` 的 `Write_Module_List`（写 `modules.load.twrp`）
