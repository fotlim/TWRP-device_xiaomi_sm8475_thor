# Android 17 的 `/data` 解密：一行变量的事

本文记录 thor 上「recovery 能启动，但 `/data` 死活挂不上」这个问题的完整定位过程与最终修法。
这是我们绕了整整两天、走了无数条错误路线之后才找到的答案。

---

## 一、症状

Recovery 完全可用：触控、ADB、备份、刷写都正常。只有 `/data` 挂不上，
`/sdcard` 是空的（因为它是 `/data/media/0` 的绑定）。

`/tmp/recovery.log`：

```
I:setting Key_Directory to: /metadata/vold/metadata_encryption
I:Metadata contents 'aes-256-xts:wrappedkey_v0'
I:Retrieving key from keymaster
```

`logcat`：

```
E KeymasterUtils:       rsp_header->status: -62          # KEY_REQUIRES_UPGRADE
E KeyMasterHalDevice:   begin_operation → ret: -62
E KeymasterUtils:       rsp_header->status: -38          # INVALID_ARGUMENT
E KeyMasterHalDevice:   upgrade_key → ret: -38
E keystore2: 0: In upgrade_keyblob_if_required_with: Upgrade failed.
E keystore2: 1: Error::Km(ErrorCode(-38))
E recovery: keystore2 Keystore createOperation returned service specific error: -38
I:Unable to decrypt metadata encryption
I:FBE setup failed. Trying FDE...
I:Can't probe device /dev/block/sda36
I:Unable to mount '/data' (Invalid argument)
```

关键点：**metadata 分区里的 key 文件都在**（`encrypted_key` 278 B、`keymaster_key_blob` 210 B、
`secdiscardable` 16384 B、`version` 1 B），分区头读出来的也不是 f2fs magic（说明确实是
metadata-encrypted 的 wrappedkey，正常现象）。也就是说 **硬件、分区、key 文件都是好的，
是 TA 拒绝合作**。

---

## 二、根因链（源码级，逐条可查）

### 2.1 版本号从哪来

`build/make/core/sysprop.mk:62-63`：

```make
echo "ro.$(1).build.version.release=$(PLATFORM_VERSION_LAST_STABLE)"
echo "ro.$(1).build.version.release_or_codename=$(PLATFORM_VERSION)"
```

这段会被展开成 **6 组**（system / vendor / odm / product / system_ext / bootimage），
即 ramdisk 的 `prop.default` 里有 12 行 `release=` 与 `release_or_codename=`。

### 2.2 Keymaster 读的是哪一条

`system/keymaster/android_keymaster/keymaster_configuration.cpp:35`：

```cpp
constexpr char kPlatformVersionProp[] = "ro.build.version.release";

uint32_t GetOsVersion() {
    std::string version = android::base::GetProperty(kPlatformVersionProp, "0");
    ...
    // "14" -> 140000, "17" -> 170000
}

keymaster_error_t ConfigureDevice(...) {
    ...
    params.push_back(KeyParameter(TAG_OS_VERSION, GetOsVersion()));
}
```

`ConfigureDevice()` 把这个 `TAG_OS_VERSION` 送进 Keymaster TA。

### 2.3 TA 的判断

TA 拿 `140000` 与自己 keyblob 里记录的 `170000` 比对：

- `140000 < 170000` → 「这个 keyblob 是用更新的系统版本封装的，我这边版本旧了」→
  返回 `-62 KEY_REQUIRES_UPGRADE`
- 上层 `upgradeKey()` 尝试升级 → TA 拒绝该方向（不让降级/改写）→ `-38 INVALID_ARGUMENT`
- **死循环，永远解不开**

### 2.4 为什么默认是 14

Base 设备树（`dopaemon/TWRP-device_xiaomi_sm8475`）里本来就有这个开关，但从没赋值：

```make
ifdef DECRYPT_PLATFORM_VERSION
PLATFORM_VERSION := $(DECRYPT_PLATFORM_VERSION)
else
PLATFORM_VERSION := 14        # ← 默认走这里
endif
PLATFORM_VERSION_LAST_STABLE := $(PLATFORM_VERSION)
```

因为那是 Android 12.1 基底、面向 Android 14 的树。

---

## 三、修复（一行）

在 `device/xiaomi/thor/BoardConfig.mk` 的 `ifdef DECRYPT_PLATFORM_VERSION` **之前**插入：

```make
# thor ships Android 17 (API 37). The Keymaster TA compares TAG_OS_VERSION
# against the version used to wrap the keyblob; reporting 14 makes it think the
# blob needs an "upgrade" and upgradeKey() fails with -38 INVALID_ARGUMENT.
DECRYPT_PLATFORM_VERSION := 17
```

重编后 `prop.default` 里 6 组 release 自动变 17，`/data` 一次性挂载成功：

```
I:Successfully decrypted metadata encrypted data partition with new block device: '/dev/block/dm-7'
I:File Based Encryption is present
User 0 Decrypted Successfully
I:New storage path after decryption: /data/media/0
I:Processing '/metadata'
```

---

## 四、应急：不重编也能改

**前提**：手上已有本仓库或任何能启动的 12.1 基底镜像，只是版本号不对。

1. 解 `ramdisk-recovery.img`（它是**裸 lz4 流**，magic `02 21 4c 18`）：

   ```bash
   lz4 -d ramdisk-recovery.img ramdisk.cpio
   mkdir root && cd root && cpio -idm < ../ramdisk.cpio
   ```

2. 改根目录 `prop.default`，把所有 `release=14` → `release=17`
   （**6 处 `release=` + 6 处 `release_or_codename=` = 12 行**）

3. 重算校验清单（跳过目录项）：

   ```bash
   find . -type f | sort > ramdisk-files.txt
   find . -type f -exec sha256sum {} \; | sort > ramdisk-files.sha256sum
   ```

4. 重打包：

   ```bash
   find . | cpio -o -H newc | lz4 -l -9 > ../ramdisk-recovery.img
   mkbootimg --header_version 4 --os_version 14.0.0 --os_patch_level 2099-12 \
     --ramdisk ../ramdisk-recovery.img -o ../recovery-raw.img

   python3 external/avb/avbtool.py add_hash_footer \
     --partition_size 104857600 --partition_name recovery \
     --hash_algorithm sha256 --salt $(openssl rand -hex 32) \
     --algorithm SHA256_RSA4096 --key external/avb/test/data/testkey_rsa4096.pem \
     --rollback_index 1649116800 --rollback_index_location 1 \
     --prop com.android.build.boot.os_version:17
   ```

> **注意**：设备上 `getprop ro.build.version.release` 仍会显示 `12` 或 `14`，那是 boot image
> header 里的 `ro.bootimage.*`，只读且**不影响解密**。真正起作用的是
> `ro.system.build.version.release` 等五组。

> **不要用** `out/host/linux-x86/bin/avbtool`（那是二进制壳，`python3` 跑会报
> `bad magic number in 'avbtool': b'\xa7\r\r\n'`）。用 `external/avb/avbtool.py`。

---

## 五、走错的那些路（留给后来人省时间）

按时间顺序，我们试过并且**全部无效**的方案：

| 尝试 | 结果 | 为什么没用 |
| --- | --- | --- |
| 改 `OF_DEFAULT_KEYMASTER_VERSION=4.1` → `4.x` | 无效 | 只影响 TWRP 侧选哪个 HIDL 服务名，不改 TA 行为 |
| 换 REF 包（酷安 @做梦书）的 `keystore2` | 无效 | 两者二进制只差 16 字节（偏移 801..816，ELF 头附近，非代码段），逻辑一样 |
| 补 `keymint` AIDL 服务进 `/vendor` | 能注册成功，仍解不开 | 服务注册通了，但 TA 依然因为版本号拒绝 |
| 重写 VINTF fragment 声明全部 4 个 fqname | `servicemanager` 找到服务了，仍解不开 | 同上 |
| 把 `/vendor/etc/vintf/manifest/*.xml` 从 `version="9.0"` 降到 `1.0` | 12.1 的 `libvintf@4.0` 读不懂 17 的版本号，降级后能解析 | 但框架清单 `NULL VINTF MANIFEST!: framework` 问题依旧，且与本问题无关 |
| 换 `system/vold` | 完全无法编译 | vold 与 recovery 是同一份 `bootable/recovery` 里调用的，改不动 |
| 补 300+ 个 `lib64` 依赖（递归闭包扫描） | 只让 linker 不抱怨，不解密 | 缺库不是根因 |
| 改 `recovery.fstab` 删掉三组加密参数强行挂 | `/data` 挂得上但**数据全被当成明文** | 治标不治本，且用户数据语义全乱 |

**唯一有效的就是改 `prop.default` 里的 release 版本号。**

---

## 六、这条经验的通用性

任何「新系统 + 老 recovery 基底」的组合都可能踩同一个坑：

- **设备出厂 Android 版本 > 你用来编译的树的基准版本** → TA 一定认为 keyblob「太新」
- 症状永远是 `-62 KEY_REQUIRES_UPGRADE` → `-38 INVALID_ARGUMENT`
- 修法永远是让 `PLATFORM_VERSION_LAST_STABLE` 等于**设备的实际 Android 版本**，不是你在编的版本

判断设备实际版本：

```bash
adb shell getprop ro.build.version.release   # 例如 17
adb shell getprop ro.build.version.sdk       # 例如 37
```
