# TWRP / OrangeFox device tree — Xiaomi 12S Ultra (thor / SM8475)

**仅在小米 12S Ultra（代号 thor，SoC SM8475 骁龙 8+ Gen 1）上测试通过。**

面向 **HyperOS 4 / Android 17 (API 37)** 的第三方 Recovery 设备适配。基底为
[minimal-manifest-twrp `android-12.1.0_r4`](https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp)
+ [OrangeFox sync](https://gitlab.com/OrangeFox/sync)，设备树源自
`dopaemon/TWRP-device_xiaomi_sm8475`（实为 OrangeFox R12.1 的 mayfly 树）改造而来。

仓库名称包含 SM8475，**不代表其他同 SoC 设备受支持**。没有测试其他机型或下文未列出的固件；
请勿把本机镜像刷到其他设备。

主要维护者：**fotlim** — <https://github.com/fotlim>

---

## 已验证范围

测试固件（Android 17 / HyperOS OS4.0，`ro.build.version.incremental=OS4.0.0.26.XPCNCNXM`）：

| 项目 | 结果 |
| --- | --- |
| Recovery 启动、主界面、触控、亮屏 | 真机通过 |
| metadata 加密分区 `/data` 解密 | **通过**（见下节「核心修复」） |
| 内部存储 `/sdcard` | 通过（解密后指向 `/data/media/0`） |
| MTP 双向传输 | 通过（需 `/data` 解密成功或走 `/data/media` 回落路径） |
| ADB / adb root / push / pull | 正常 |
| 分区备份与恢复（dd，含 super / modem / persist） | 通过，实测 1.0~1.3 GB/s |
| 刷写 img、格式化 data | 通过 |
| 文件管理（system/vendor/product/system_ext/odm 只读挂载） | 通过 |
| 电池 / 充电状态、亮度调节 | 正常 |
| 中文界面、主题、终端 | 通过 |
| 壁纸引擎（原图 / 压暗 / 渐变 / 模糊） | 可用，**实验性**（见「已知限制」） |
| 从 Recovery 重启系统 | 正常 |
| fastbootd、增量 OTA、多用户 | 未做真机测试 |

镜像为**独立 recovery**：header v4、无内嵌内核（内核使用设备树中预编译的 `prebuilt/thor/kernel`）、
分区上限 `104857600` 字节（100 MB）。

---

## 核心修复：Android 17 时代的 `/data` 解密

这是本仓库存在的最大理由。一行变量：

```make
# device/xiaomi/thor/BoardConfig.mk
DECRYPT_PLATFORM_VERSION := 17
```

`thor` 出厂即为 Android 17。Base 树的 `PLATFORM_VERSION` 默认写 14，于是：

1. `build/make/core/sysprop.mk` 把 `ro.*.build.version.release=14` 写进 ramdisk 的 `prop.default`（6 处）。
2. Keymaster TA 收到的 `TAG_OS_VERSION` 是 `140000`。
3. TA 与 keyblob 内记录的 `170000` 比对 → 判定「需要升级」→ 返回 `-62 KEY_REQUIRES_UPGRADE`。
4. `upgradeKey()` 尝试升级，但 TA 拒绝该方向 → `-38 INVALID_ARGUMENT`。
5. 表现为 `Unable to decrypt metadata encryption` → `FBE setup failed. Trying FDE...` → `/data` 挂不上。

把 `PLATFORM_VERSION` 改成 17 后，`prop.default` 里的 release 全部变为 17，TA 直接放行，`/data`
一次性解密并挂载为 `/dev/block/dm-7`。详细推导与「不重编的应急手改法」见
[docs/DECRYPT-A17.md](docs/DECRYPT-A17.md)。

> 这条结论并非来自「换 keystore2 / 换 vold / 补 VINTF / 改 keymaster_ver」—— 那些路我们都走完了，
> 全部无效。真正的开关是 `prop.default` 里的 release 版本号。

---

## 仓库结构

```
device/xiaomi/thor/
├── Android.bp / Android.mk / AndroidProducts.mk
├── BoardConfig.mk              # 平台、内核、分区、加密、屏保、AVB
├── device.mk                   # 产品继承、bootctrl、需要模块
├── system.prop                 # adb / gatekeeper / 电池属性
├── vendorsetup.sh              # OrangeFox 构建开关 + 维护者信息
├── twrp_thor.mk                # 产品定义（PRODUCT_NAME := twrp_thor）
├── init/                       # libinit_thor（init_mayfly.cpp）
├── bootctrl/                   # bootctrl.xiaomi_sm8475.recovery
├── gpt-utils/                  # 分区表工具
├── security/                   # miui_releasekey.x509.pem（公共验证证书）
├── prebuilt/thor/
│   ├── kernel                  # 5.10.177 预编译内核（小 45 MB）
│   └── vendor/lib/modules/1.1/ # 设备所需内核模块（ufs / drm / 触控 / 振动等）
├── recovery/root/
│   ├── first_stage_ramdisk/fstab.qcom
│   ├── system/etc/recovery.fstab
│   └── system/etc/init/        # keystore2.rc、health 等
└── customzip/                  # 附加工具包
patches/
└── 0001-bootable-recovery-thor-features.patch   # 框架层改动（见 docs/CHANGES.md）
```

仓库**不含**内核源码、厂商闭源二进制、触摸固件与 `prebuilt/` 里其他机型（diting / mayfly / unicorn）
的目录 —— 这些均不随本仓库分发，也不由本仓库授权。

---

## 构建

完整步骤、依赖、`.wslconfig` 三个必踩坑与编译脚本模板见 **[docs/BUILD.md](docs/BUILD.md)**。
最短路径：

```bash
repo init -u https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp.git -b android-12.1.0_r4
repo sync -c -j8 --force-sync --no-clone-bundle --no-tags
# 叠加 OrangeFox 层（见 https://gitlab.com/OrangeFox/sync 的 README）
git clone https://github.com/fotlim/TWRP-device_xiaomi_sm8475_thor.git device/xiaomi/thor
source build/envsetup.sh && lunch twrp_thor-eng && mka recoveryimage -j8
```

产物：`out/target/product/thor/OrangeFox-R12.0_1-Unofficial-thor.img`（100 MB）

刷入（thor 是 A/B 但**拥有独立 recovery 分区**，不是 recovery-as-boot）：

```bash
adb reboot bootloader
fastboot flash recovery OrangeFox-R12.0_1-Unofficial-thor.img
fastboot reboot recovery
```

> `fastboot fetch recovery` 不被支持，**原厂 recovery 无法备份**。建议先准备一份官方
> [LineageOS thor](https://download.lineageos.org/api/v2/devices/thor/builds) 的 `recovery.img` 作为还原包。

---

## 已知限制

- **壁纸功能是实验性的**。开启壁纸后，`/system` 与 `/data` 层面的滑动手势偶尔出现内容重叠（重影），
  静止数秒后自行消失；关闭壁纸则完全不出现。界面上已明确标注「实验性功能：启用壁纸可能导致渲染不稳定」。
  根因是 pixelflinger 纯软件渲染下，半透明底板无法覆盖局部旧帧；`gui.cpp` 中已对壁纸启用整页重绘，
  但仍有边界情况。**如遇渲染异常，请先把壁纸切回「关闭」。**
- 模糊模式是**预渲染盒式模糊**（一次性烘焙进缓存 surface），不是实时模糊。
- `libvintf@4.0` 读不懂 Android 17 的 `manifest.version="9.0"`，因此不要尝试在 recovery 内挂载
  system/product/system_ext 的 VINTF 目录 —— 会破坏 framework manifest 解析。
- 除 `.png` 外的图片格式（jpg/webp）在选择壁纸时不可用。
- 标准 zip 刷机：自带 `update-binary` 的包（Magisk 类）可装；纯 `updater-script` 的 OTA 包不可装。

---

## 致谢

感谢 **Team Win、Dees_Troy、bigbiff、TWRP、OrangeFox、AOSP、LineageOS、The Linux Foundation /
Qualcomm、dopaemon、酷安 @做梦书、7ac9d42、ymdzq 与 EROFS 作者**。
具体继承关系与固定版本见 [docs/UPSTREAM.md](docs/UPSTREAM.md)；致谢不代表上游为本项目背书。

原创工具与文档采用 MIT；继承代码与补丁继续遵守各自许可证，详见 [NOTICE](NOTICE) 与
[docs/LICENSING.md](docs/LICENSING.md)。

## 项目文档

- [构建指南](docs/BUILD.md)
- [★ Android 17 解密核心修复](docs/DECRYPT-A17.md)
- [踩坑清单（最有价值的部分）](docs/PITFALLS.md)
- [改动说明](docs/CHANGES.md)
- [上游致谢与固定版本](docs/UPSTREAM.md)
- [许可证范围](docs/LICENSING.md)
