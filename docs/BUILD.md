# 构建指南 — thor / OrangeFox R12.0（Android 12.1 基底）

本文是给「想自己重编这个镜像」的人看的。所有命令都在 WSL2 的 Ubuntu 22.04 里执行。
先读一遍「三个前提」再动手，否则你会浪费掉一整个下午。

---

## 零、三个前提

1. **必须 WSL2 + ext4**。不要在 `/mnt/c`、`/mnt/d`、`/mnt/f` 里直接编译 —— AOSP 需要 Linux 的
   大小写敏感、权限位、inode、符号链接语义，在 DrvFs 上必然出现各种诡异失败与极慢构建。
   做法是把 WSL 的 vhdx 放到性能盘（SSD），在 ext4 内部编译。
2. **内存 16 GB 起，swap 要够**。soong 全树分析单进程 RSS 可到 9 GB，`MemAvailable` 会掉到几十 MB。
   建议 `memory=11GB` + `swap=48GB`，并把 swap 文件也放到 SSD 上。
3. **磁盘 300 GB 起**。`android-12.1.0_r4` 全量 sync 约 80 GB，加 `out/` 约 30 GB，
   再加 ccache 视配置而定。thor 设备树本身 387 MB（其中 `prebuilt/` 占 382 MB）。

---

## 一、`.wslconfig`（三个必踩坑）

文件：`C:\Users\<你的用户名>\.wslconfig`

```ini
[wsl2]
memory=11GB
processors=6
swap=48GB
swapFile=F:\\\\wsl\\\\swap.vhdx

[experimental]
sparseVhd=true
```

| 坑 | 现象 | 正解 |
| --- | --- | --- |
| 文件编码 | WSL 启动异常 / 配置不生效 | **必须无 BOM 的 UTF-8**。PowerShell 的 `Set-Content -Encoding utf8` 会写 BOM，用 `[System.IO.File]::WriteAllText($p,$s,(New-Object System.Text.UTF8Encoding($false)))` |
| `swapFile` 反斜杠 | `wslconfig 中的转义字符"5"无效` | 路径必须写**四个**反斜杠 `F:\\\\wsl\\\\swap.vhdx` |
| `sparseVhd` 段落 | `wsl2.sparseVhd: 中的键"6"未知` | `sparseVhd` 只能放 `[experimental]`，不能放 `[wsl2]` |

改完 `.wslconfig` 要 `wsl --shutdown` 再启动才生效。

---

## 二、把 Ubuntu 装到非系统盘

```powershell
wsl --import Ubuntu-22.04 F:\wsl\Ubuntu-22.04 F:\wsl\ubuntu\ubuntu-jammy-wsl-amd64-ubuntu22.04lts.rootfs.tar.gz --version 2
```

rootfs 下载地址（**noble 的 rootfs tar 在 cloud-images 上已经不存在了，用 jammy**）：

```
https://cloud-images.ubuntu.com/wsl/jammy/current/ubuntu-jammy-wsl-amd64-ubuntu22.04lts.rootfs.tar.gz
```

---

## 三、依赖

```bash
apt-get update
apt-get install -y openjdk-17-jdk-headless git-core gnupg flex bison build-essential \
  zip unzip curl wget python3 python-is-python3 rsync bc ccache lz4 brotli \
  libssl-dev libelf-dev libncurses5 libncurses-dev x11proto-core-dev \
  libx11-dev libgl1-mesa-dev libxml2-utils xsltproc fontconfig patchelf

# AOSP 的 dumpvars 用 #!/usr/bin/env python —— Ubuntu 22.04 只有 python3
ln -sf /usr/bin/python3 /usr/bin/python
```

> **必须做 `python` 符号链接**。否则 `lunch` 会报
> `dumpvars failed with: exit status 1` / `/usr/bin/env: 'python': No such file or directory`，
> 而且报错位置离真正原因很远，极难查。

---

## 四、拉源码

```bash
mkdir -p /root/OrangeFox_12.1 && cd /root/OrangeFox_12.1

repo init -u https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp.git \
          -b android-12.1.0_r4
repo sync -c -j8 --force-sync --no-clone-bundle --no-tags

# OrangeFox 层
git clone https://gitlab.com/OrangeFox/sync.git /tmp/ofsync
# 按 /tmp/ofsync/README.md 执行 orangefox_sync.sh，或手工合并 vendor/orangefox
```

> `repo sync` 常见 `cts` 仓库 TLS 失败：
> `curl 56 GnuTLS recv error (-9): Error decoding the received TLS packet`。
> **cts 用不到，`Failing repos (network): cts` 可以忽略**，其余仓库同步完成即可继续。

---

## 五、放设备树

```bash
git clone https://github.com/fotlim/TWRP-device_xiaomi_sm8475_thor.git /tmp/thor-dt
mkdir -p device/xiaomi/thor
cp -a /tmp/thor-dt/device/xiaomi/thor/. device/xiaomi/thor/
```

如果你是用别的来源的树，注意 `twrp_thor.mk` 里必须**同时**有：

```make
PRODUCT_NAME := twrp_thor
PRODUCT_DEVICE := thor
PRODUCT_RELEASE_NAME := thor
DEVICE_PATH := device/xiaomi/thor
TARGET_OTA_ASSERT_DEVICE := $(PRODUCT_RELEASE_NAME)
```

`DEVICE_PATH` 不对 → `BoardConfig.mk` 里的 `$(DEVICE_PATH)/prebuilt/thor/kernel` 找不到内核，
构建会在打包阶段才失败。

---

## 六、关键构建开关（`BoardConfig.mk` / `vendorsetup.sh`）

| 开关 | 值 | 说明 |
| --- | --- | --- |
| `DECRYPT_PLATFORM_VERSION` | `17` | **最重要**。Android 17 解密（见 [DECRYPT-A17.md](DECRYPT-A17.md)） |
| `BOARD_USES_RECOVERY_AS_BOOT` | **不要设 true** | thor 有独立 recovery 分区，走 recovery-as-boot 会编出空包 |
| `TW_RECOVERY_AS_BOOT` | 不要设 | 同上，且 `vendorsetup.sh` 里若反向开启会出现「`ninja: no work to do` 但无产物」 |
| `BOARD_EXCLUDE_KERNEL_FROM_RECOVERY_IMAGE` | `true` | 内核走 `prebuilt/thor/kernel`，镜像内不带 kernel |
| `BOARD_RECOVERYIMAGE_PARTITION_SIZE` | `104857600` | 必须等于分区实际大小，否则 AVB footer 位置错 |
| `TW_INCLUDE_CRYPTO` / `TW_INCLUDE_CRYPTO_FBE` | `true` | 开启 FBE 解密链路 |
| `TW_INCLUDE_FBE_METADATA_DECRYPT` | `true` | metadata 加密分区解密 |
| `BOARD_USES_METADATA_PARTITION` | `true` | 有独立 `/metadata` 分区 |
| `TW_USE_FSCRYPT_POLICY` | `2` | FBE v2 |
| `TW_NO_SCREEN_BLANK` | `true` | 阻止 `gr_fb_blank()` |
| `TW_NO_SCREEN_TIMEOUT` | **按需** | 设 true 彻底关掉 60 秒自动灭屏；不设则到点 dim→blank→lock overlay |
| `TW_LOAD_VENDOR_MODULES` | `"adsp_loader_dlkm.ko fts_touch_spi.ko"` | 只加载这两个，避免模块全是垃圾日志 |
| `OF_DEFAULT_KEYMASTER_VERSION` | `4.x` | 告诉 TWRP 走 keymaster HIDL 兼容层 |

> **`BOARD_USES_RECOVERY_AS_BOOT` 是本次最大的隐性坑**：设备树里原本有
> `ifneq ($(TW_RECOVERY_AS_BOOT),) BOARD_USES_RECOVERY_AS_BOOT := $(TW_RECOVERY_AS_BOOT) endif`，
> 一旦 `vendorsetup.sh` 里 `FOX_AB_DEVICE`/`OF_AB_DEVICE_WITH_RECOVERY_PARTITION` 组合把
> `TW_RECOVERY_AS_BOOT` 顶着变成 true，就会走 recovery-as-boot 路径 —— **构建成功但没有任何 img 产出**，
> ninja 只会告诉你 `ninja: no work to do.`。

---

## 七、`recovery.fstab` 的两条铁律

`device/xiaomi/thor/recovery/root/system/etc/recovery.fstab`：

```fstab
# /data —— 必须保留全部加密参数，否则 Decrypt_Data() 不会被调用
/dev/block/bootdevice/by-name/userdata  /data  f2fs  noatime,nosuid,nodev,discard,reserve_root=32768,resgid=1065,fsync_mode=nobarrier,inlinecrypt,gc_merge  latemount,wait,formattable,fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized+wrappedkey_v0,keydirectory=/metadata/vold/metadata_encryption,metadata_encryption=aes-256-xts:wrappedkey_v0,quota,reservedsize=128M,sysfs_path=/sys/devices/platform/soc/1d84000.ufshc,checkpoint=fs

# /metadata —— wrappedkey 必留
/dev/block/bootdevice/by-name/metadata  /metadata  ext4  noatime,nosuid,nodev,discard  wait,check,formattable,wrappedkey,first_stage_mount
```

1. **删掉 `fileencryption=` / `keydirectory=` / `metadata_encryption=` 会让 `/data` 彻底不解密**
   （`Decrypt_Data()` 根本不会被调用，设备直接 mount 失败）。
2. 想「绕开加密、强制挂载」可以临时删这三个参数，但**必须改源码树并重编** ——
   直接 `adb shell` 改 `/system/etc/recovery.fstab` 是改内存盘，重启即失效（这个坑我们踩过一整天）。

`recovery/root/first_stage_ramdisk/fstab.qcom` 比 `recovery.fstab` 多
`compress_mode=user,compress_cache,age_extent_cache,atgc`，从真机
`adb pull /first_stage_ramdisk/fstab.qcom` 拿。

---

## 八、编译

**脚本必须放在 Linux 原生路径（`/root/`），不要在 `/mnt/f` 下执行** —— `/mnt/f` 上的脚本可能带
CRLF/BOM，且 `source build/envsetup.sh` 在里面会静默失效，结果是 `mka: command not found`。

```bash
cat > /root/build.sh <<'EOF'
#!/bin/bash
cd /root/OrangeFox_12.1 || exit 1
export LC_ALL=C
unset USE_CCACHE CCACHE_DIR CCACHE_EXEC CC_WRAPPER   # 避免 ccache 目录权限问题
{
  date
  source build/envsetup.sh
  lunch twrp_thor-eng
  mka recoveryimage -j8
  echo "EXITCODE=$?"
  date
} > /root/build.log 2>&1
EOF

# ⚠ 必须 setsid nohup，否则 WSL 会话结束会发 SIGHUP 把编译杀掉
setsid nohup bash /root/build.sh > /root/build.nohup 2>&1 < /dev/null &
```

看进度：

```bash
grep -c EXITCODE /root/build.log     # 有输出就是跑完了
tail -c 500 /root/build.log
pgrep -c ninja
```

耗时参考（Ryzen 7 3700X / 8 核 / 16 GB / SATA SSD）：

| 阶段 | 耗时 |
| --- | --- |
| soong 自举（首次，一次性） | ~48 min |
| soong_build 全树分析（改 `.mk`/`.bp` 后必重跑） | ~40 min |
| ckati 解析 | ~6 min |
| 纯 C++ 增量（只改 `.cpp`） | 5~8 min |
| 首次完整编译 | 35~60 min |

产物：

```
out/target/product/thor/OrangeFox-R12.0_1-Unofficial-thor.img   104857600 B
out/target/product/thor/ramdisk-recovery.img                     ~59 MB
```

---

## 九、刷入

```bash
adb reboot bootloader
fastboot flash recovery F:\wsl\OrangeFox-R12.0_1-Unofficial-thor.img
fastboot reboot recovery
```

- thor 是 A/B，但 **recovery 是独立分区（`recovery_a`）**，只要一条 `fastboot flash recovery`。
- `fastboot fetch recovery` **不被支持**（`Device does not support fetch command`），
  原厂 recovery 无法备份。请先备好官方 LineageOS thor 的 `recovery.img` 当还原包。
- PowerShell 里 `fastboot.exe` 的 `stderr` 会被包成 ErrorRecord 抛红字；
  只要输出里有 `OKAY` 且 `Finished. Total time` 正常，`exit code: 1` 就是 PowerShell 的错觉。

---

## 十、环境侧遗留问题

| 症状 | 原因 | 处理 |
| --- | --- | --- |
| `Wsl/Service/WSAETIMEDOUT` | 内存吃紧（`vmmemWSL` > 7 GB 且宿主可用 < 2 GB） | 等 10~20 秒重试；把 `repo sync` 用 `nice -n 19` 降权 |
| `du -sh` 卡死 | 240 万文件的树上 | 别跑，或 `--max-depth=1` |
| `mka: command not found` | 脚本在 `/mnt/f` | 脚本放 `/root/`，用 `bash /root/x.sh` |
| 主题资源改了但镜像里还是旧的 | ninja 的 `ramdisk_files-timestamp` 是 phony 输入，没变化就不重打包 | `rm -f out/target/product/thor/obj/PACKAGING/recovery_intermediates/ramdisk_files-timestamp out/target/product/thor/ramdisk-recovery.cpio out/target/product/thor/ramdisk-recovery.img` 再编 |
| `ccache: Failed to create directory /home/builder/.ccache/tmp: Permission denied` | `CCACHE_DIR` 由 `/etc/profile.d` 注入，属主是 root | `chown -R root:root /home/builder/.ccache` 或在脚本里 `unset USE_CCACHE CCACHE_DIR CCACHE_EXEC CC_WRAPPER` |
