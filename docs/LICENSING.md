# 许可证范围

本仓库是**逐文件授权**的，不是单一许可证。这个结构照抄了 AOSP / LineageOS 的惯例：
每类文件沿用其上游的许可证，本仓库新增的部分用 MIT。

如果你只想遵守一条规则：**保留所有文件头的版权与许可证声明，并在分发时附带本文件与 `NOTICE`。**

---

## 一、按类别

| 范围 | 许可证 | 说明 |
| --- | --- | --- |
| `docs/*.md`、`README.md` | **MIT** | 本项目原创文档 |
| `patches/0001-bootable-recovery-thor-features.patch` 中标记为原创的部分（`gui/wallpaper.cpp`、`gui/wallpaper.hpp`） | **MIT** | 本项目新写的文件 |
| 同一 patch 中对 AOSP 文件的修改（`gui/pages.cpp`、`gui/gui.cpp`、`orangefox.mk`、`variables.h`、`data.cpp`、主题 XML） | 沿用原文件许可证（**Apache-2.0**，TeamWin / AOSP 头） | 修改不改变原许可证 |
| `device/xiaomi/thor/*.mk`、`*.bp`、`*.sh`、`device.mk`、`system.prop` | 沿用上游（**Apache-2.0**） | 源自 Android 设备树惯例 |
| `device/xiaomi/thor/gpt-utils/*` | **BSD-3-Clause (Linux Foundation)** | 上游为 `gptfdisk` / Qualcomm `gpt-utils`，保留原头 |
| `device/xiaomi/thor/recovery/root/init.recovery.usb.rc` | **BSD-3-Clause (Linux Foundation)** | 源自 AOSP / Linux Foundation 的 USB init 脚本 |
| `device/xiaomi/thor/security/miui_releasekey.x509.pem` | 权利人：小米 | **公钥证书**，不含私钥；随 MIUI OTA 公开分发 |
| `device/xiaomi/thor/prebuilt/**` | 权利人：小米 / 高通 | **不随本仓库分发**（见下） |
| `device/xiaomi/thor/recovery/root/vendor/**` | 权利人：小米 / 高通 | 仅保留 recovery 启动所需的小体积 HAL/守护进程（`qseecomd`、`keymaster@4.1`、`gatekeeper@1.0`、若干 `lib*.so`），以便设备树可直接构建；不提供任何授权 |
| `themes/font.xml` 引用的字体（Roboto、Google Sans、wqy-microhei） | Apache-2.0 / OFL-1.1 | 由 TWRP / OrangeFox 主题层提供 |

## 二、明确不包含的内容

本仓库**不包含**下列内容，也不会以任何形式提供：

- 厂商闭源二进制（`prebuilt/thor/vendor/**`、`prebuilt/thor/kernel`）
- 触摸控制器固件（`.ftb` / `.csv`）
- 小米 OTA 私钥或任何签名密钥
- `prebuilt/` 下其他机型（`diting` / `mayfly` / `unicorn`）的目录

拿到本仓库后若要让设备树可用，需要**自行从自己的设备或官方固件中提取**上述文件
（`tools/extract-files.py` 类的脚本可参考 LineageOS 的 `extract_utils`）。

## 三、构建产物

由本仓库编出的镜像包含上述闭源二进制，因此：

- 镜像**不是** MIT 授权
- 分发镜像时你分发的是「AOSP/TWRP（Apache-2.0）+ OrangeFox（其各自许可）+ 小米/高通闭源组件」的聚合体
- 建议在发布页注明来源与不提供任何担保

本项目发布镜像仅供**在你自己拥有的设备上做恢复用途**。

## 四、贡献

向本仓库提交的原创代码默认以 MIT 授权（与 `docs/` 一致）；
若修改的是 AOSP 文件，则该修改继续沿用该文件的许可证。

## 五、免责

本软件按「现状」提供，不附带任何明示或暗示的担保，包括但不限于对适销性、特定用途适用性
及不侵权的担保。在任何情况下，作者或版权持有人均不对任何索赔、损害或其他责任负责，
无论其源于合同、侵权或其他方式，亦无论其是否与本软件或本软件的使用或其他交易有关。

**刷机有风险。** 解锁 Bootloader、刷写 recovery、格式化分区都可能导致数据全部丢失或设备变砖。
请自行备份，并确认你理解每一步在做什么。
