# 小米 12S Ultra (thor / SM8475) 第三方 Recovery 从零构建与修复备忘

> 记录人：大肥鱼（AI 助手）  
> 时间跨度：2026-10-22 ~ 2026-10-25  
> 目标设备：Xiaomi 12S Ultra，代号 **thor**，SoC **SM8475 / 骁龙 8+ Gen 1**  
> 出厂系统：HyperOS，**Android 17 (API 37)**，已解锁 Bootloader  
> 最终成果：**OrangeFox R12.0（Android 12.1 基底）自制版**，可正常启动、可解密 Android 17 的 `/data`、MTP 双向可用、支持壁纸与自定义关于页

本文写给两类人：
1. 以后的我（或同类 AI）再遇到"给新系统内核的设备编老 recovery"时的速查表；
2. 想给 thor 或类似高通骁龙机型自己做 recovery 的后来人。

后半部分直接是**踩坑清单**，赶时间可以跳到 [第六节](#六踩坑清单最关键的部分)。

---

## 一、先说结论（TL;DR）

| 问题 | 结论 |
|---|---|
| TWRP 官方支持 thor 吗 | **不支持**。`dl.twrp.me/thor/` 404，TWRP 官方无此设备 |
| OrangeFox 官方支持 thor 吗 | **不支持** |
| 有现成的树吗 | 有一个 `dopaemon/TWRP-device_xiaomi_sm8475`，但它其实是 **OrangeFox R12.1 的 mayfly（12S）树**，需要改成 thor |
| 能用 TWRP android-14.1 吗 | **不建议**。14.1 的 `build/make` 用新版 release-config，`lunch` 两段式全废，且 vold 删了 FDE API，patch 成本极高。**改用 12.1 基底 + OrangeFox 脚手架** |
| 最重要的一招 | **`DECRYPT_PLATFORM_VERSION := 17`** —— 一个变量解决 Android 17 `/data` 解密（见第三节） |
| 编译耗时 | 首次 soong 自举 ~48 min；全树分析 40 min；增量编译 7~40 min |
| 硬件够用吗 | 16 GB 内存 + 8 核够，但**很吃 swap**，且 WSL 要放 SSD（不要放机械盘或网络盘） |

---

## 二、环境搭建（Windows + WSL2）

### 2.1 为什么必须 WSL2 而不是 `/mnt/f` 直接编

AOSP 编译需要 Linux 文件系统语义（大小写、权限、符号链接、inode）。**在 `/mnt/f` 下直接编译必然失败**（表现为 mka/ninja 各种诡异错误、构建极慢）。正确做法是把 WSL 的 vhdx 放在性能盘上，在 ext4 里编译。

### 2.2 `.wslconfig` 关键坑（三个都能让你怀疑人生）

文件：`C:\Users\<user>\.wslconfig`

```ini
[wsl2]
memory=11GB
swap=48GB
swapFile=F:\\\\wsl\\\\swap.vhdx
processors=6

[experimental]
sparseVhd=true
```

- **坑 1：必须是「无 BOM 的 UTF-8」**。用 PowerShell `Set-Content -Encoding utf8` 会带 BOM，WSL 解析异常。  
  正确写法：
  ```powershell
  [System.IO.File]::WriteAllText($p, $s, (New-Object System.Text.UTF8Encoding($false)))
  ```
- **坑 2：`swapFile` 必须写四个反斜杠** `F:\\\\wsl\\\\swap.vhdx`。写两个会报 `wslconfig 中的转义字符"5"无效`。
- **坑 3：`sparseVhd` 只能放 `[experimental]`**。放 `[wsl2]` 会报 `wsl2.sparseVhd: 中的键"6"未知`。

### 2.3 页面文件放 F 盘

```powershell
New-CimInstance -ClassName Win32_PageFileSetting -Property @{
  Name='F:\pagefile.sys'; InitialSize=[uint32]8192; MaximumSize=[uint32]8192
}
```
`InitialSize` **必须强转 `[uint32]`**，否则报「属性 InitialSize 的类型不匹配」。

### 2.4 Ubuntu 导入到非系统盘

```powershell
wsl --import Ubuntu-22.04 F:\wsl\Ubuntu-22.04 F:\wsl\ubuntu\ubuntu-jammy-wsl-amd64-ubuntu22.04lts.rootfs.tar.gz --version 2
```
rootfs 下载地址（noble 的 rootfs tar 在 cloud-images 上已不存在，用 jammy）：
```
https://cloud-images.ubuntu.com/wsl/jammy/current/ubuntu-jammy-wsl-amd64-ubuntu22.04lts.rootfs.tar.gz
```

### 2.5 必装依赖

```bash
apt-get install -y openjdk-17-jdk-headless git-core gnupg flex bison build-essential \
  zip unzip curl wget python3 python-is-python3 rsync bc ccache lz4 brotli \
  libssl-dev libelf-dev libncurses5 libncurses-dev x11proto-core-dev \
  libx11-dev libgl1-mesa-dev libxml2-utils xsltproc fontconfig patchelf
ln -sf /usr/bin/python3 /usr/bin/python   # ← 没有 python 符号链接，dumpvars 会挂
```

> **坑**：Ubuntu 22.04 只有 `python3`，没有 `python`。AOSP 的 `dumpvars` 会用 `#!/usr/bin/env python` 直接报 `No such file or directory`。

---

## 三、★ 核心突破：Android 17 的 `/data` 解密

### 3.1 症状

recovery 能启动，但 `/data` 挂不上，logcat：

```
E KeymasterUtils:     rsp_header->status: -62          # KEY_REQUIRES_UPGRADE
E KeyMasterHalDevice: begin_operation → ret: -62
E KeymasterUtils:     rsp_header->status: -38          # INVALID_ARGUMENT
E KeyMasterHalDevice: upgrade_key → ret: -38
E keystore2: 0: In upgrade_keyblob_if_required_with: Upgrade failed.
E keystore2: 1: Error::Km(ErrorCode(-38))
E recovery: keystore2 Keystore createOperation returned service specific error: -38
I:Unable to decrypt metadata encryption
I:FBE setup failed. Trying FDE...
```

### 3.2 根因链（源码级）

1. `device/xiaomi/thor/BoardConfig.mk`（原始树）已经有这个开关，但从没赋值：
   ```make
   ifdef DECRYPT_PLATFORM_VERSION
   PLATFORM_VERSION := $(DECRYPT_PLATFORM_VERSION)
   else
   PLATFORM_VERSION := 14        # ← 默认走这里
   endif
   PLATFORM_VERSION_LAST_STABLE := $(PLATFORM_VERSION)
   ```
2. `build/make/core/sysprop.mk:62-63` 把 `PLATFORM_VERSION_LAST_STABLE` 写进 ramdisk 的 `prop.default`，共 **6 处**（system/build/vendor/odm/product/system_ext）：
   ```make
   echo "ro.$(1).build.version.release=$(PLATFORM_VERSION_LAST_STABLE)"
   echo "ro.$(1).build.version.release_or_codename=$(PLATFORM_VERSION)"
   ```
3. `system/keymaster/android_keymaster/keymaster_configuration.cpp:35`
   ```cpp
   constexpr char kPlatformVersionProp[] = "ro.build.version.release";
   ```
   `GetOsVersion()` 把它转成数字：`"14"` → 140000，`"17"` → 170000。
4. `ConfigureDevice()` 把这个 `TAG_OS_VERSION` 送进 Keymaster TA。
5. TA 拿它跟 keyblob 里存的版本比对：**140000 < 170000**，判定「这个 blob 太旧，需要升级」→ 返回 `-62 KEY_REQUIRES_UPGRADE`。
6. `upgradeKey()` 想升级，但 TA 拒绝「降级」→ `-38 INVALID_ARGUMENT`。**死循环。**

> 这条链解释了为什么改 `keymaster_ver=4.x`、换 keystore2、换 vold、补库……全都没用——**版本号根本不对，TA 从一开始就拒绝合作**。

### 3.3 修复（一行）

在 `device/xiaomi/thor/BoardConfig.mk` 的 `ifdef DECRYPT_PLATFORM_VERSION` **之前**插入：

```make
# thor ships Android 17 (API 37). The Keymaster TA compares TAG_OS_VERSION
# against the version used to wrap the keyblob; reporting 14 makes it think the
# blob needs an "upgrade" and upgradeKey() fails with -38 INVALID_ARGUMENT.
DECRYPT_PLATFORM_VERSION := 17
```

重编后 `prop.default` 里六处 release 自动变 17，`/data` 一次性挂载成功：

```
I:Successfully decrypted metadata encrypted data partition with new block device: '/dev/block/dm-7'
I:File Based Encryption is present
User 0 Decrypted Successfully
I:New storage path after decryption: /data/media/0
```

### 3.4 如果不想重编，手工改也行（应急）

1. 解 `ramdisk-recovery.img`（是裸 lz4 流，magic `02 21 4c 18`）：
   ```bash
   lz4 -d ramdisk-recovery.img ramdisk.cpio
   mkdir root && cd root && cpio -idm < ../ramdisk.cpio
   ```
2. 改根目录 `prop.default` 里所有 `release=14` → `release=17`（**6 处 release + 6 处 release_or_codename = 12 行**）
3. 重算 `ramdisk-files.sha256sum`（跳过目录项）和 `ramdisk-files.txt`
4. 重打包：
   ```bash
   find . | cpio -o -H newc > ../new.cpio
   lz4 -l -9 -f ../new.cpio ../ramdisk-recovery.img
   mkbootimg --header_version 4 --os_version 14.0.0 --os_patch_level 2099-12 \
     --ramdisk ../ramdisk-recovery.img -o ../recovery-raw.img
   avbtool add_hash_footer --partition_size 104857600 --partition_name recovery \
     --hash_algorithm sha256 --salt <32字节hex> \
     --algorithm SHA256_RSA4096 --key external/avb/test/data/testkey_rsa4096.pem \
     --rollback_index 1649116800 --rollback_index_location 0 \
     --prop com.android.build.boot.os_version:17
   ```

> **注意**：设备上 `getprop ro.build.version.release` 仍会显示 `12`，那是 boot image header 里的 `ro.bootimage.*`，只读且不影响解密。真正起作用的是 `ro.system.build.version.release` 等五组。

---

## 四、踩坑清单（最关键的部分）

### 4.1 构建系统类

| 坑 | 现象 | 解法 |
|---|---|---|
| `lunch` 两段式失效 | `Invalid lunch combo: twrp_thor-eng` / `No release config found for TARGET_RELEASE: trunk_staging` | 用 **android-12.1** 基底，别用 14.1 |
| 缺 python | `dumpvars failed with: exit status 1` / `env: 'python': No such file` | `ln -sf /usr/bin/python3 /usr/bin/python` |
| `gsi_keys` 缺失 | `ckati` 报缺 `keys/gsi_keys/*.pem` | 用 `external/avb/test/data/testkey_rsa4096.pem` 顶替，或补 `build/make/target/product/security/` |
| `BOARD_BUILD_SYSTEM_ROOT_IMAGE` obsolete | `error: BOARD_BUILD_SYSTEM_ROOT_IMAGE is obsolete` | 设备树里删掉这一行 |
| `kernel.mk` 语法错 | `vendor/twrp/build/tasks/kernel.mk:594: commands commence before first target` | 上游 TWRP 14.1 模板 bug，把 `NEEDS_KERNEL` 那段改成正确的 make 语法 |
| soong 卡住不输出 | `[99% 1161/1162] cp out/.../soong_build` 卡很久 | **正常**，下一步是全树 ckati 分析（6 min），没有进度输出 |
| `mka: command not found` | 脚本放 `/mnt/f` 执行时 | **脚本必须放 `/root/`**，用 `bash /root/xxx.sh`，否则有 CRLF/BOM 污染 |
| 后台任务被 SIGHUP 杀 | 编译/下载跑一半没了 | `setsid nohup bash /root/xxx.sh > /root/x.log 2>&1 < /dev/null &` |
| 编译中途 WSL 超时 | `Wsl/Service/WSAETIMEDOUT` | 内存吃紧（`vmmemWSL` > 7 GB + 宿主机可用 < 2 GB 时几乎必现）。等 10~20 秒重试；把 `repo sync` 用 `renice 19` 降权 |
| `du -sh` 卡死 | 在 240 万文件树上 | 别跑，或者只 `du -sh --max-depth=1` |

### 4.2 设备 / 刷机类

| 坑 | 现象 | 解法 |
|---|---|---|
| **`/dev` 是空的** | recovery 起来了但 `/dev/block` 不存在、`/dev/binder` 没有、`/dev/cpuset` 没有 | 说明 init.rc 里的 `mount tmpfs tmpfs /dev` 没跑。查 `first_stage_ramdisk/` 与 `init.recovery.qcom.rc` 的 `on fs` 段 |
| **recovery 每 5 秒重启** | `init: Service 'recovery' exited with status 1` 刷屏 | 通常是缺库（`CANNOT LINK EXECUTABLE`）或 `createProcessGroup` 失败（缺 `/dev/cg2_bpf`）。先 `dmesg | grep CANNOT` |
| `mount --bind` 丢 seclabel | 覆盖 `/system/bin/keystore2` 后 `init: cannot execv(...): Permission denied` + `status 127` 每 5 秒循环 | **绝对不要用 bind mount 覆盖可执行文件**。改用 `cp 到 xxx.new` 再 `mv -f`（规避 ETXTBSY） |
| `cp` 报 Text file busy | `cp: /system/bin/keystore2: Text file busy` | 进程在跑。用 `cp xxx.new && mv -f xxx.new xxx` |
| 手动 kill recovery 后黑屏 | adb/fastboot 全失 | 长按电源 + 音量减 10 秒强制重启。**别在无人看管时做会失联的操作** |
| 自动化触摸测试不可信 | `sendevent` 注入了但 `recovery.log` 不增长 | 注入路径与真实触摸不等价，**别用日志 delta 判断触摸好坏**，让真人测 |
| 无 `screencap` | recovery 里看不到界面 | 只有 `/dev/dri/card0`（DRM），无 `/dev/fb*`。UI 效果必须真人看 |
| adb 路径必须 Windows 形式 | `adb push /mnt/f/xxx` 报 `cannot stat` | 写 `F:\wsl\xxx`（Windows 形式） |

### 4.3 编译期源码补丁（14.1 专属，12.1 一般不需要）

这些是**在 TWRP android-14.1 上**必须打的补丁，供参考（我们最终放弃了 14.1）：

- `bootable/recovery/libtar/libtar.h` 需要 `fscrypt_policy.h`，但 vold 14.1 已经删了相关 API → 新建 `fscrypt_tar_compat.h` 垫片
- `system/vold` 删了 `cryptfs_check_footer` / `set_partition_data` / `cryptfs_get_password_property` 等 FDE 符号 → 新建 `fde_compat.h` 提供空实现（thor 纯 FBE，FDE 是死代码）
- `system/vold` 的 `android.security.apc` 等模块不再有 `-ndk_platform` 变体 → 改成 `-ndk`
- `init.recovery.mksh.rc` 在 `TW_USE_TOOLBOX := true` 时**没被装进 ramdisk**（上游 bug）→ 手工拷进 `TARGET_RECOVERY_ROOT_OUT`

### 4.4 屏保 / 触摸延迟（两个不同的病）

| 病 | 症状 | 根因 | 解法 |
|---|---|---|---|
| CPU 空转 | 点一下等 5 秒，`top` 显示 recovery 线程 100% | `battery_monitor` 线程在 `GetBatteryInfo()` 里死循环 | 注释掉 `bootable/recovery/twrp.cpp` 里的 `static std::thread battery_monitor(...)` |
| 屏保循环 | 屏幕每 3 秒 dim→灭→亮，滑锁卡死 | `TW_NO_SCREEN_BLANK := true` 只跳过 `gr_fb_blank()`，**不阻止 `blanktimer.cpp` 的 dim/blank/lock overlay** | 加 `TW_NO_SCREEN_TIMEOUT := true` |

### 4.5 MTP

| 症状 | 根因 | 解法 |
|---|---|---|
| MTP 不启动 | `orangefox.crash_counter` 非 0（反复 kill recovery 把计数器推上去了） | `setprop orangefox.crash_counter -1` + `setprop mtp.crash_check 0`，重启 recovery |
| MTP 传的文件看不到 | `/data` 没解密时 `Setup_Data_Media()` 回落 `Storage_Path=/data/media`，而界面看的是 `/data/media/0` | 在 `partition.cpp` 的 `Setup_Data_Media()` 里主动 `Make_Dir(Mount_Point + "/media/0")`；或者干脆把 `/data` 解密修好（见第三节） |
| `sys.usb.ffs.mtp.ready=1` 但无传输 | `E:[MTP] cannot open bulk in ep` | 通常伴随 ffs 配置问题，检查 `init.recovery.usb.rc` 的 ffs 链 |

---

## 五、OrangeFox 12.1 构建全流程（可直接照搬）

### 5.1 拉源码

```bash
mkdir -p /root/OrangeFox_12.1 && cd /root/OrangeFox_12.1
repo init -u https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp.git -b android-12.1.0_r4
repo sync -c -j8 --force-sync --no-clone-bundle --no-tags
git clone https://gitlab.com/OrangeFox/sync.git /tmp/ofsync
# 然后按 /tmp/ofsync/README.md 的说明把 OrangeFox 的 vendor 层叠上去
```

> `repo sync` 常见 `cts` 仓库 TLS 失败（`curl 56 GnuTLS recv error -9`），**cts 用不到，忽略即可**。

### 5.2 放设备树

```bash
cp -a /path/to/TWRP-device_xiaomi_sm8475/. device/xiaomi/thor/
```

### 5.3 关键设备树改动（thor 相对 mayfly 树）

必改项清单：

1. `twrp_thor.mk`
   - `DEVICE_PATH := device/xiaomi/thor`
   - `PRODUCT_DEVICE := thor` / `PRODUCT_NAME := twrp_thor`
   - `TARGET_OTA_ASSERT_DEVICE := thor`
2. `BoardConfig.mk`
   - `TARGET_BOARD_PLATFORM := xiaomi_sm8475`
   - `TARGET_PREBUILT_KERNEL := $(DEVICE_PATH)/prebuilt/thor/kernel`
   - `BOARD_RECOVERYIMAGE_PARTITION_SIZE := 104857600`
   - `BOARD_USES_RECOVERY_AS_BOOT := false`（**thor 是独立 recovery 分区，不是 A/B 的 recovery-as-boot**）
   - `TW_INCLUDE_CRYPTO := true`、`TW_INCLUDE_CRYPTO_FBE := true`
   - `TW_NO_SCREEN_BLANK := true`、**`TW_NO_SCREEN_TIMEOUT := true`**
   - **`DECRYPT_PLATFORM_VERSION := 17`** ← 最关键
3. `recovery/root/system/etc/recovery.fstab`
   - userdata 行**必须保留** `fileencryption=` / `keydirectory=` / `metadata_encryption=`（删了就不会解密）
   - metadata 行保留 `wrappedkey`
4. `recovery/root/first_stage_ramdisk/fstab.qcom`
   - 从真机 `adb pull /first_stage_ramdisk/fstab.qcom` 拿，比 recovery.fstab 多 `compress_mode=user,compress_cache,age_extent_cache,atgc`

> **坑**：`BOARD_USES_RECOVERY_AS_BOOT` 如果被 `vendorsetup.sh` 反向设置，会出现 `ninja: no work to do.` 但**没有产物**。

### 5.4 编译

```bash
cd /root/OrangeFox_12.1
cat > /root/build.sh <<'EOF'
#!/bin/bash
cd /root/OrangeFox_12.1
export LC_ALL=C
source build/envsetup.sh > /dev/null 2>&1
lunch twrp_thor-eng > /dev/null 2>&1
{ date; mka recoveryimage -j8; echo "EXITCODE=$?"; date; } > /root/build.log 2>&1
EOF
setsid nohup bash /root/build.sh > /root/build.nohup 2>&1 < /dev/null &
```

产物：`out/target/product/thor/OrangeFox-R12.0_1-Unofficial-thor.img`（104857600 B）

### 5.5 刷入

```bash
adb reboot bootloader
fastboot flash recovery F:\wsl\OrangeFox-ofXX.img
fastboot reboot recovery
```

> thor 是 **A 槽单分区 recovery**，不用管 slot。`fastboot fetch recovery` 不支持，**原厂 recovery 无法备份**——直接用官方 LineageOS recovery.img 当还原包。

---

## 六、调试方法论（比具体补丁更值钱）

### 6.1 二分法定位「recovery 起不来」

1. **先确认底座能起**：刷一个能启动的第三方包（酷安/官方 LineageOS recovery），确认设备本身没问题
2. **换 ramdisk 内容**：把能启动的 ramdisk 拷出来，逐批替换成自己的
3. **用 PADTEST 排除体积因素**：往能启动的 ramdisk 里塞 51 MB 随机数据重打包，如果还能起，说明**体积不是问题**
4. **用 linker 日志抓缺库**：
   ```bash
   dmesg | grep "CANNOT LINK"
   ```
   出现的库名就是缺的，用递归闭包扫描补：
   ```python
   # 对 ramdisk 里所有 ELF 跑 readelf -d 取 NEEDED，在 system/lib64、vendor/lib64 递归解析
   ```

### 6.2 判断「是设备问题还是包问题」

**刷官方包对照**是最快的分诊手段：
- 官方包能起、自己的不能 → 问题在自己的 ramdisk 内容
- 官方包也不能起 → 问题在内核/分区/设备状态

### 6.3 不刷机验证法（极其重要）

当还能进 recovery 且有 adb 时，**不需要刷机就能测新二进制**：

```bash
adb push F:\wsl\recovery_new.bin /system/bin/recovery_new
adb shell chmod 755 /system/bin/recovery_new
adb shell "nohup env LD_LIBRARY_PATH=/system/lib64twrp:/system/lib64 /system/bin/recovery_new > /tmp/r.out 2>&1 &"
```

- 用 `LD_LIBRARY_PATH` 指向单独的库目录，**不要污染 `/system/lib64`**
- 改完不满意 `kill -9` 重来，机器不会变砖
- **一旦要动 `/system/lib64` 就极危险**，会牵连整个系统（我们曾因此差点拆机）

### 6.4 后台任务管理

| 场景 | 正确做法 |
|---|---|
| 长时间编译 | `setsid nohup bash /root/xxx.sh > log 2>&1 < /dev/null &` |
| 脚本位置 | **必须 `/root/`**，不能 `/mnt/f/` |
| 下载不中断 | `nice -n 19 repo sync ...`，避免和编译抢 CPU |
| 检查进度 | `pgrep -af ninja`、`tail -3 /root/build.log`、`grep -c "build completed"` |

### 6.5 PowerShell 与 WSL 交互的坑

- `wsl.exe bash -c "..."` 里的 `$(...)`、`>`、引号会被 **PowerShell 本地展开** → 一律写 `.sh` 文件再 `bash /mnt/f/wsl/xxx.sh`
- 写 `.sh` 文件必须无 BOM：
  ```powershell
  [System.IO.File]::WriteAllText($p, $s.Replace("`r`n","`n"), (New-Object System.Text.UTF8Encoding($false)))
  ```
- `| head` 在 PowerShell 不可用
- WSL 命令超时先看 `Get-Process vmmemWSL` 和可用内存

---

## 七、源码补丁索引（本次实际改动）

### 7.1 设备树（`device/xiaomi/thor/`）

| 文件 | 改动 | 目的 |
|---|---|---|
| `BoardConfig.mk` | `DECRYPT_PLATFORM_VERSION := 17` | **A17 解密核心修复** |
| `BoardConfig.mk` | `TW_NO_SCREEN_TIMEOUT := true` | 修屏保循环 |
| `recovery/root/system/etc/recovery.fstab` | userdata 行保留全部加密参数 | 让 `Decrypt_Data()` 被调用 |
| `recovery/root/first_stage_ramdisk/fstab.qcom` | 新增（从真机 pull） | 提供 `GetFstabPath()` 找到的 fstab |
| `vendorsetup.sh` | `OF_MAINTAINER="fotlim"` + 三个 `OF_BUILD_MAINTAINER_*` | 关于页构建人信息 |

### 7.2 框架层（`bootable/recovery/`）

| 文件 | 改动 | 目的 |
|---|---|---|
| `twrp.cpp` | 注释 `static std::thread battery_monitor(...)` | 修 100% CPU 空转 |
| `partition.cpp` | `Setup_Data_Media()` 里主动建 `/data/media/0` | 修 MTP 存储路径 |
| `orangefox.mk` | 新增 `OF_BUILD_MAINTAINER_URL/EMAIL/QQ` 的 cflag 注入 | 构建人信息 |
| `variables.h` | 新增三个 `*_STR` 宏 | 同上 |
| `data.cpp` | `mConst.SetValue(...)` 注入三个变量 | 同上 |
| `gui/theme/.../pages/settings.xml` | 关于页加三行 `<text>`；新增 `ext_wallpaper`/`wallpaper_pick` 页 | 构建人信息 + 壁纸 |
| `gui/theme/.../resources/vars.xml` | 新增 `cardtxt_9/10/11`、四个 `wallpaper_*` 变量 | 布局 |
| `gui/wallpaper.cpp` + `wallpaper.hpp` | **新增文件**，约 280 行 | 四模式壁纸引擎 |
| `gui/pages.cpp` | `Page::Render()` 有壁纸时不填底色；`PageManager::Render()` 先画壁纸 | 壁纸全局化 |

### 7.3 壁纸引擎要点

- `PageManager::Render()` 里统一画一次壁纸 → 全局底层
- `wallpaper_is_active()` 让各页在壁纸激活时**不填不透明底色**
- `ConvertStrToColor()` **支持 8 位 `#RRGGBBAA`**，`gr_fill()`/`gr_color()` 支持 alpha 混合 → 半透明控件可行
- 模糊是**软件盒式模糊、预渲染缓存**，不是实时（实时会卡死：pixelflinger 是纯软件渲染）

---

## 八、命令速查卡

```bash
# ---- 解包 recovery.img ----
python3 -c "
import struct,sys
d=open('recovery.img','rb').read()
m,ks,rs,osv,hs,ov,onv=struct.unpack('<8sIIIIII', d[:32])
print('kernel',ks,'ramdisk',rs,'hdr',hs)
off=((hs+4095)//4096*4096)+((ks+4095)//4096*4096)
open('rd.lz4','wb').write(d[off:off+rs])
"
lz4 -d rd.lz4 rd.cpio
mkdir root && cd root && cpio -idm < ../rd.cpio

# ---- 打包 ----
find . | cpio -o -H newc > ../new.cpio
lz4 -l -9 -f ../new.cpio ../rd.lz4
mkbootimg --header_version 4 --os_version 14.0.0 --os_patch_level 2099-12 --ramdisk ../rd.lz4 -o ../raw.img

# ---- AVB 签名 ----
python3 external/avb/avbtool.py add_hash_footer \
  --partition_size 104857600 --partition_name recovery \
  --hash_algorithm sha256 --salt $(openssl rand -hex 32) \
  --algorithm SHA256_RSA4096 --key external/avb/test/data/testkey_rsa4096.pem \
  --rollback_index 1649116800 --rollback_index_location 0

# ---- 验证签名 ----
python3 external/avb/avbtool.py verify_image --key external/avb/test/data/testkey_rsa4096.pem recovery.img

# ---- 查看镜像信息 ----
python3 external/avb/avbtool.py info_image --image recovery.img

# ---- 设备侧常用 ----
adb shell getprop | grep -i "decrypt\|fstab\|keymaster"
adb shell "cat /tmp/recovery.log | tail -50"
adb logcat -d | grep -i "keymaster\|keystore2\|fstab"
adb shell "cat /proc/cmdline"       # 看 bootconfig
adb shell "ls /dev/block/by-name/"  # 分区表
```

---

## 九、给后来人的六条忠告

1. **先验证底座，再动自己的东西**。手上有能启动的第三方包，比读一百行源码都快。
2. **`prop.default` 是解密的总开关**。`PLATFORM_VERSION` 必须等于设备实际系统版本，不是你在编的 Android 版本。
3. **绝对不要 bind mount 覆盖可执行文件**。丢了 SELinux 上下文，init 直接拒绝执行，而且报错信息完全不提示"seclabel"。
4. **不刷机验证法能救命**。用 `LD_LIBRARY_PATH` + 独立库目录跑新二进制，比刷机测快十倍，也不会变砖。
5. **`/mnt/f` 是毒药**。所有脚本、编译目录、后台任务都必须待在 Linux 原生路径（`/root`）。
6. **看着像"加密坏了"的问题，八成不是加密模块的问题**。我们绕了整整两天——换 keystore2、换 vold、换 vendor、补库、改 VINTF——最后发现只是 `Release 版本号写成了 14`。

---

## 十、参考资料

- LineageOS thor 官方支持：`https://download.lineageos.org/api/v2/devices/thor/builds`（有官方 24.0 版本，可作为能启动的对照底座）
- TWRP 源码：`https://github.com/TeamWin/android_bootable_recovery`
- OrangeFox 同步脚手架：`https://gitlab.com/OrangeFox/sync.git`
- 设备树（需改造）：`https://github.com/dopaemon/TWRP-device_xiaomi_sm8475`（branch android-14，实为 OrangeFox R12.1 mayfly 树）
- AVB 工具：AOSP `external/avb/avbtool.py`（**不要用 `out/host/.../avbtool`，那个是二进制壳**）
- 酷安帖「发个修改A14的twrp 去支持A15解密 方法」——虽未直接给出版本号机制，但启发我们去看 `prop.default`

---

## 附：本次时间线

| 时间 | 事件 |
|---|---|
| 10-22 | 环境评估、WSL 搭建 |
| 10-23 | TWRP 14.1 建树、lunch 打通、首次编出 recovery.img |
| 10-23 晚 | 刷入卡第一屏，开启漫长排查 |
| 10-24 | 卡第一屏二分（体积/签名/init.rc/libclosure 全部排除）、HYB4 混合包能启动、定位 `battery_monitor` 空转、转 OrangeFox 12.1 |
| 10-24 晚 | OrangeFox 首次完全成功（能启动、MTP 通、备份快）；然后开始攻解密 |
| 10-25 上午 | 排除 keystore2 / vold / vendor / VINTF；发现 REF 也走 `upgrade_key` 但成功 |
| 10-25 11:25 | **`DECRYPT_PLATFORM_VERSION := 17` 一击命中**，build_of13 全功能通过 |
| 10-25 11:30+ | 关于页、壁纸、玻璃化美化 |
