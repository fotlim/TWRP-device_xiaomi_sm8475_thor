# 改动说明

相对上游 Base 树（`minimal-manifest-twrp/platform_manifest_twrp_aosp` @ `android-12.1.0_r4`
+ OrangeFox 层）的差异。分三部分：设备适配、功能修复、构建与工具。

---

## 一、设备适配

### 1.1 `DECRYPT_PLATFORM_VERSION := 17` —— Android 17 解密

`BoardConfig.mk` 新增。这是整个项目最关键的一行，详细推导见 [DECRYPT-A17.md](DECRYPT-A17.md)。

上游树默认 `PLATFORM_VERSION := 14`，导致 `prop.default` 里的
`ro.*.build.version.release=14`，Keymaster TA 比对 keyblob 里的 `170000` 后返回
`-62 KEY_REQUIRES_UPGRADE`，`upgradeKey()` 又被拒 `-38 INVALID_ARGUMENT`，`/data` 永远解不开。

改成 17 后 `/data` 一次性解密为 `/dev/block/dm-7`。

### 1.2 设备树从 mayfly 迁移到 thor

原树（`dopaemon/TWRP-device_xiaomi_sm8475`）实为 OrangeFox R12.1 的 **mayfly（小米 12S）** 树，
同时挂着 `twrp_thor/mayfly/diting/unicorn` 四个产品。适配 thor 需要：

| 项 | 改动 |
| --- | --- |
| `product` | `PRODUCT_RELEASE_NAME := thor`（决定 `prebuilt/thor/` 与 `prebuilt/thor/kernel` 的路径） |
| `DEVICE_PATH` | `device/xiaomi/thor` |
| `kernel` | `prebuilt/thor/kernel`（5.10.177，45 MB；四个机型共用同一份） |
| `TARGET_BOARD_PLATFORM` | `xiaomi_sm8475` |
| `TARGET_CPU_VARIANT_RUNTIME` | `kryo300` |
| `PRODUCT_PLATFORM` | `taro` |
| 屏幕偏移 | `TW_Y_OFFSET := 106` / `TW_H_OFFSET := -106`（1440×3200 面板，主题按 1080×1920 缩放） |
| `TARGET_OTA_ASSERT_DEVICE` | `$(PRODUCT_RELEASE_NAME)` |

### 1.3 去掉 A/B 的 recovery-as-boot 路径

设备树里原有的

```make
ifneq ($(TW_RECOVERY_AS_BOOT),)
    BOARD_USES_RECOVERY_AS_BOOT := $(TW_RECOVERY_AS_BOOT)
endif
```

在 `FOX_AB_DEVICE` / `OF_AB_DEVICE_WITH_RECOVERY_PARTITION` 的组合下会被顶着变成 true，
从而走 recovery-as-boot 路径，表现为 **`ninja: no work to do.` 且不产出任何 img**。

thor 是 A/B，但**有独立 `recovery_a` 分区**。因此：

```make
BOARD_EXCLUDE_KERNEL_FROM_RECOVERY_IMAGE := true   # 内核走 prebuilt
BOARD_RECOVERYIMAGE_PARTITION_SIZE := 104857600
```

### 1.4 `recovery.fstab` 保留全部加密参数

`recovery/root/system/etc/recovery.fstab` 的 `/data` 行必须带
`fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized+wrappedkey_v0,keydirectory=/metadata/vold/metadata_encryption,metadata_encryption=aes-256-xts:wrappedkey_v0`，
否则 `Decrypt_Data()` 根本不会被调用。

`/metadata` 行必须带 `wrappedkey,first_stage_mount`。

### 1.5 `first_stage_ramdisk/fstab.qcom`

从真机 `adb pull /first_stage_ramdisk/fstab.qcom` 取得，比 `recovery.fstab` 多
`compress_mode=user,compress_cache,age_extent_cache,atgc`（f2fs 压缩相关）。

### 1.6 `TW_LOAD_VENDOR_MODULES`

只加载 `adsp_loader_dlkm.ko` 与 `fts_touch_spi.ko`。全量加载 360 个模块会让
`kernel_module_loader` 刷屏且拖慢启动。

---

## 二、功能修复

### 2.1 壁纸引擎（新文件）

`bootable/recovery/gui/wallpaper.cpp` / `wallpaper.hpp`，约 400 行，**全新文件**。

- 四种模式：`WP_MODE_OFF 0` / `ORIGINAL 1` / `DIM 2` / `GRADIENT 3` / `BLUR 4`
- `build_surface()`：`res_create_surface` 读原图 → `res_scale_surface` cover 缩放 → 居中裁剪
- `veil_surface()`：把压暗/渐变/模糊的遮罩 **一次性烘焙进缓存 surface**
  （原本每帧画整屏 `gr_fill` + 64 条渐变横条，滑动手感明显变差）
- `wallpaper_render()` 每帧只剩一次 `gr_blit(gSurface, sx, sy, cw, ch, 0, 0)`
- 通过 `/persist/.foxs` 与 `/sdcard/Fox/.foxs` 持久化 `wallpaper_mode` / `wallpaper_path`
- CLI：`/system/bin/twrp set wallpaper_mode 1` / `twrp get wallpaper_path`

**已知限制**：开启壁纸后半透明底板无法覆盖局部旧帧，滑动手势偶尔出现内容重叠（重影），
静止数秒自行消失。已在界面标注为实验性功能。

### 2.2 壁纸选择界面 + 实验性警告

`gui/theme/portrait_hdpi/pages/settings.xml`：新增 `<page name="ext_wallpaper">`
（五档模式 + 选择图片 + 清除），并在列表框下方加红色警告文本 `{@wallpaper_exp_warn}`。

`gui/theme/common/languages/en.xml` / `extra-languages/languages/zh_CN.xml`：
新增 `wallpaper_hdr` / `wallpaper_cptn` / `wallpaper_off` / `wallpaper_orig` / `wallpaper_dim` /
`wallpaper_grad` / `wallpaper_blur` / `wallpaper_pick` / `wallpaper_pick_hdr` / `wallpaper_clear` /
`wallpaper_exp_warn`。

### 2.3 半透明底板（`wallpaper_apply_translucency()`）

`gui/pages.cpp` 的 `PageSet::Load()` **末尾**（在 include 递归加载 `themes/styles/*.xml`
之后）调用，把 `background` / `power_bg` 压到 alpha `0x66`、`cards_bg` 压到 `0xB4`，
让壁纸从卡片下面透出来。

> **时序是关键**：主题变量在 `themes/styles/*.xml` 里定义，而它们在 `PageSet::Load()` 的
> include 阶段才被加载。放在 `LoadVariables` 末尾会被覆盖回去（我们最初就是这个坑，
> 日志里反复出现 `applied` / `restored` 抖动但视觉无任何变化）。

`gui/pages.cpp` 的 `Page::Render()` 同时钳制渲染时的 alpha —— 因为 `mBackground` 是
构造时从 XML 快照的常量，改 DataManager 影响不到它。

### 2.4 滑动重影缓解

`gui/gui.cpp` 主循环两处条件由

```cpp
if (ret > 1) PageManager::Render();
```

改为

```cpp
if (ret > 1 || (ret > 0 && wallpaper_is_active())) PageManager::Render();
```

原因：`ret == 1` 表示控件自己做了局部渲染，此时不重绘整页；半透明底板盖不住局部旧内容。
壁纸激活时强制整页重绘。

### 2.5 锁屏纳入壁纸范围

`gui/pages.cpp` 的 `Page::Render()`：当 `mName == "lock" && wallpaper_is_active()` 时
**先重画一次壁纸**（`wallpaper_render()`）再画半透明底板 —— 否则锁屏 overlay 会透出底下的
页面内容（因为 `PageSet::Render()` 是先渲染当前页、再渲染 overlays）。

### 2.6 MTP 内部存储路径修复

`partition.cpp` 的 `TWPartition::Setup_Data_Media()`（`@@ -1308`）：在 `/data` 挂载后主动
`Make_Dir(Mount_Point + "/media/0")` 并设 0770 权限。

原因：`/data` 刚格式化（或解密失败）时 `/data/media/0` 不存在，`Setup_Data_Media()`
会把 `Storage_Path` 回落成 `/data/media`，再 bind mount 到 `/sdcard`。此时电脑通过 MTP
上传的文件落在 `/data/media`，而界面上「内部存储」看的是 `/data/media/0` —— **传到一半发现
文件不显示**。补上 `/media/0` 后两边路径一致。

### 2.7 关于页构建人信息

`gui/theme/portrait_hdpi/pages/settings.xml` 的 `<page name="about">` 卡片内新增三行：
GitHub / 邮箱 / QQ。新增布局变量 `cardtxt_9/10/11`（`resources/vars.xml`），
新增 `font_xs`（`themes/font.xml`，Roboto-Regular 32），卡片 SVG 从
`viewBox="0 0 1080 336"` 加高到 `1080 384` 让五行文字与「更多」按钮进框内。

`orangefox.mk` 新增三个 cflag 注入点（`OF_MAINTAINER_URL/EMAIL/QQ`），
`variables.h` 新增三个变量名，`data.cpp` 在 `mConst.SetValue(OF_MAINTAINER_STR, ...)`
之后注入它们。`vendorsetup.sh` 里设置实际值。

---

## 三、构建与工具

### 3.1 `vendorsetup.sh`

```sh
export OF_MAINTAINER="fotlim"
export OF_BUILD_MAINTAINER_URL="https://github.com/fotlim"
export OF_BUILD_MAINTAINER_EMAIL="w87911743@126.com"
export OF_BUILD_MAINTAINER_QQ="3565486206"
export OF_DEFAULT_KEYMASTER_VERSION=4.x
export FOX_BUILD_DEVICE="Xiaomi 12S Ultra"
export FOX_MAINTAINER_PATCH_VERSION=1
export OF_USE_MAGISKBOOT=1
export OF_UNMOUNT_SYSTEM=1
export OF_RUN_POST_FORMAT_PROCESS=1
export OF_FBE_METADATA_MOUNT_IGNORE=1
```

> `FOX_VERSION` 在 `orangefox.mk:40` 已废弃（`$(error 'FOX_VERSION' is obsolete)`），
> 不要再用，改用 `FOX_MAINTAINER_PATCH_VERSION`。

### 3.2 移除编译期联网下载

`device/xiaomi/thor/customzip/magisk_prebuilt.sh` 原本在**编译期**从
`raw.githubusercontent.com` 下载 `Magisk.apk` 并校验。内网/受限网络下必然失败
（`FAILED: recovery/root/magisk_prebuilt-timestamp`）。已注释掉 `customzip` 的模块注册，
需要时用 TWRP 界面刷 Magisk zip。

### 3.3 `prebuilt/Android.mk`

`miui_prebuilt` 用 `cp -rf $(LOCAL_PATH)/$(PRODUCT_RELEASE_NAME)/vendor $(TARGET_RECOVERY_ROOT_OUT)/`，
只拷当前产品的目录。因此 `prebuilt/` 下放其他机型的目录不会污染 thor 的镜像。

---

## 四、与上游的兼容性

- 编译环境：Ubuntu 22.04 / OpenJDK 17 / Python 3.10 / lz4 / patchelf
- 框架层补丁以 `patches/0001-bootable-recovery-thor-features.patch` 形式提供，
  可 `git apply` 到 `android-12.1.0_r4` 的 `bootable/recovery`
- 设备树本身不依赖其他机型目录（`diting` / `mayfly` / `unicorn` 已移除）
