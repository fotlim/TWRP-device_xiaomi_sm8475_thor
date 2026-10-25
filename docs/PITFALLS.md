# 踩坑清单

**这一篇是整个仓库里最值钱的部分。** 如果你只读一节，读这一节。
每条坑都写了「现象 → 根因 → 解法」，因为我们当初就是靠这些线索一条条爬出来的。

---

## 一、构建系统

| 坑 | 现象 | 根因 / 解法 |
| --- | --- | --- |
| **`lunch` 两段式失效** | `Invalid lunch combo: twrp_thor-eng` / `No release config found for TARGET_RELEASE: trunk_staging` | 用了 TWRP `android-14.1`。它的 `build/make` 是带 release-config 的新版，只认 `ap2a`。**换 12.1 基底** |
| **缺 `python` 符号链接** | `dumpvars failed with: exit status 1` / `/usr/bin/env: 'python': No such file or directory` | Ubuntu 22.04 只有 `python3`。`ln -sf /usr/bin/python3 /usr/bin/python` |
| `gsi_keys` 缺失 | ckati 报缺 `keys/gsi_keys/*.pem` | 用 `external/avb/test/data/testkey_rsa4096.pem` 顶替，或补 `build/make/target/product/security/` |
| `BOARD_BUILD_SYSTEM_ROOT_IMAGE` obsolete | `error: BOARD_BUILD_SYSTEM_ROOT_IMAGE is obsolete` | 设备树里删掉这一行 |
| `kernel.mk` 语法错 | `vendor/twrp/build/tasks/kernel.mk:594: commands commence before first target` | TWRP 14.1 模板 bug，`NEEDS_KERNEL` 那段缩进错位 |
| **soong 卡住不输出** | `[99% 1161/1162] cp out/.../soong_build` 之后长时间无输出 | **正常**。下一步是全树 ckati 解析（6 min），没有进度条 |
| **`mka: command not found`** | 脚本跑起来立刻退出 | 脚本放在 `/mnt/f` 下。**必须放 `/root/`**，用 `bash /root/x.sh`（`/mnt/f` 有 CRLF/BOM，`source` 会静默失效） |
| **后台任务被 SIGHUP 杀** | 编译跑一半进程没了，日志只有开头 | 用 `setsid nohup bash /root/x.sh > log 2>&1 < /dev/null &`，缺 `setsid` 就会随 WSL 会话结束被杀 |
| **`&&` 长链静默中断** | 脚本不生成日志文件，或只跑了一半 | `cd x && export Y && source envsetup && lunch && mka` 里若 `source` 返回非零，整条链断掉且无提示。**改成 `{ ... } > log 2>&1` 块** |
| **ccache 权限** | `ccache: error: Failed to create directory /home/builder/.ccache/tmp: Permission denied` | `/etc/profile.d/99-android-build.sh` 注入 `CCACHE_DIR`，属主是 root。`chown -R root:root /home/builder/.ccache` 或脚本里 `unset USE_CCACHE CCACHE_DIR CCACHE_EXEC CC_WRAPPER` |
| **主题/资源改了但镜像里是旧的** | 改了 `twres/` 下的 XML，重编后镜像里还是旧文件 | `out/build-*.ninja` 里 `ramdisk-recovery.cpio` 的输入只有一个 phony 的 `ramdisk_files-timestamp`，该文件时间戳不变就不会重打包。**手动删掉 `ramdisk_files-timestamp` + `ramdisk-recovery.cpio` + `ramdisk-recovery.img` 再编** |
| `du -sh` 卡死 | 在 240 万文件的 `avium` 树上 | 别跑，或 `--max-depth=1` |
| `Wsl/Service/WSAETIMEDOUT` | 命令随机超时 | `vmmemWSL` > 7 GB 且宿主可用 < 2 GB 时几乎必现。等 10~20 秒重试，`repo sync` 用 `nice -n 19` |

---

## 二、设备与刷机

| 坑 | 现象 | 根因 / 解法 |
| --- | --- | --- |
| **`/dev` 是空的** | recovery 起来了但 `/dev/block`、`/dev/binder`、`/dev/cpuset` 全不存在 | `init.rc` 里 `mount tmpfs tmpfs /dev` 没执行。查 `first_stage_ramdisk/` 和 `init.recovery.qcom.rc` 的 `on fs` 段 |
| **recovery 每 5 秒重启** | `init: Service 'recovery' (pid N) exited with status 1` 刷屏 | ①`CANNOT LINK EXECUTABLE` 缺库 → `dmesg \| grep CANNOT`；②`createProcessGroup` 失败（缺 `/dev/cg2_bpf`）→ 给 service 加 `disabled` 或补 cgroup 节点 |
| **绝对不要 bind mount 覆盖可执行文件** | `init: cannot execv(...): Permission denied` + `status 127` 每 5 秒循环 | bind mount **丢 seclabel**，init 拒绝执行，而且报错完全不提 seclabel。用 `cp x.new && mv -f x.new x` |
| `cp` 报 Text file busy | `cp: /system/bin/keystore2: Text file busy` | 进程在跑。`cp x.new && mv -f x.new x` |
| **手动 kill recovery 后黑屏、adb/fastboot 全失** | 设备完全失联 | 长按电源 + 音量减 10 秒强制重启。**别在无人看管时做会失联的操作** |
| `fastboot fetch` 不支持 | `Unable to get max-fetch-size. Device does not support fetch command.` | thor 不支持分区备份，原厂 recovery 无法导出。备好官方 LineageOS `recovery.img` 当还原包 |
| PowerShell 把 fastboot 的 stderr 当错误 | 输出红字 `exit code: 1`，但实际刷成功了 | 看有没有 `OKAY` 和 `Finished. Total time`。有就是成功 |
| adb 路径必须 Windows 形式 | `adb push /mnt/f/x` 报 `cannot stat` | 写 `F:\wsl\x` |
| **自动化触摸测试不可信** | `sendevent` 注入后 `recovery.log` 不增长，但真人操作正常 | 注入路径与真实触摸不等价。**别用日志 delta 判断触摸好坏**，让真人测 |

---

## 三、屏幕 / 触摸 / 渲染

| 坑 | 现象 | 根因 / 解法 |
| --- | --- | --- |
| **点一下要等 5 秒** | `top` 显示 recovery 某个线程 100% CPU | `bootable/recovery/twrp.cpp` 的 `battery_monitor` 线程在 `GetBatteryInfo()` 里死循环（recovery 环境没有 health HAL，`waitForService` 无限轮询）。注释掉 `static std::thread battery_monitor(monitorBatteryInBackground);` |
| **屏保循环（dim→灭→亮）** | 屏幕每 3 秒变暗再灭再亮，滑动解锁卡死 | `TW_NO_SCREEN_BLANK := true` **只跳过 `gr_fb_blank()`**，不阻止 `blanktimer.cpp` 的 dim(亮度5)→blank(亮度0)→`ChangeOverlay("lock")`。要加 `TW_NO_SCREEN_TIMEOUT := true` 才会把三个函数体用 `#ifndef` 掏空 |
| **壁纸不显示 / 卡片挡死** | 壁纸解码成功（日志有 `built 1440x3200 surface`）但屏幕上完全看不到 | 两层原因：①`Page::Render()` 每页先整屏 `gr_fill(mBackground)`；②`themes/styles/*.xml` 在 `PageSet::Load()` 的 **include 阶段**才加载，在 `LoadVariables` 末尾改颜色一定会被覆盖回去。**正确时机是 `PageSet::Load()` 末尾（include 之后）**，且 `Page::Render()` 要在渲染时钳制 alpha（`mBackground` 是构造时的快照常量，改 DataManager 影响不到它） |
| **滑动出现重影** | 开启壁纸后滑动页面出现内容重叠，静止数秒后消失 | `gui.cpp` 主循环：`if (ret > 1) PageManager::Render(); if (ret > 0) flip();` —— `ret == 1` 表示控件自己做了局部渲染，不重绘整页。半透明底板盖不住局部旧内容。**壁纸激活时把 `ret > 0` 也当整页重绘**（仍有边界情况，所以壁纸标为实验性） |
| 每帧几十次 `gr_fill` 导致卡顿 | 滑动手感明显变差 | DIM / GRADIENT 模式原本每帧画整屏 fill + 64 条横条。**把遮罩烘焙进 `build_surface()` 的缓存 surface**（`veil_surface()`），每帧只剩一次 `gr_blit` |
| 无 `screencap` | 想截图取证但没工具 | recovery 里只有 `/dev/dri/card0`，无 `/dev/fb0`。TWRP 自带截图：**长按 `音量下 + 电源`** 200 ms，存到 `Fox_Home/screenshots/` |

---

## 四、MTP

| 症状 | 根因 | 解法 |
| --- | --- | --- |
| MTP 完全不启动 | `orangefox.crash_counter` 非 0（反复 `pkill recovery` 调试把计数器推上去了）。`twrp.cpp:296` 要求 `!crash_counter` 才启动 MTP | `setprop orangefox.crash_counter -1` + `setprop mtp.crash_check 0`，重启 recovery |
| MTP 能连但看不到文件 | `/data` 未解密时 `Setup_Data_Media()` 回落 `Storage_Path=/data/media`，而界面看的是 `/data/media/0` | 在 `partition.cpp` 的 `Setup_Data_Media()` 里主动 `Make_Dir(Mount_Point + "/media/0")`；或把解密修好（正解） |
| `sys.usb.ffs.mtp.ready=1` 但无传输 | `E:[MTP] cannot open bulk in ep` / `request read returned -1, errno: 108` | 检查 `init.recovery.usb.rc` 的 ffs 链（`sys.usb.config=mtp,adb` 时 ffs.mtp→f1、ffs.adb→f2、UDC `a600000.dwc3`） |
| Windows 侧 `WPD` 设备在但 `NameSpace()` 返回 null | Windows 的 WPD 遍历层级限制 | 用 `Shell.Application` 的 `NameSpace(17)` 从「此电脑」逐层进到 `Internal Storage`，别直接用 SID 路径 |

---

## 五、解密与 VINTF（历史教训，本仓库已修好）

| 坑 | 现象 | 结论 |
| --- | --- | --- |
| 误判「keymint AIDL 是必需」 | `keystore2: Cannot connect to Keymint` → `BAD_TYPE` → `NAME_NOT_FOUND` | `keystore2` 找不到 keymint 会走 `android.security.compat` 的 keymaster@4.x 兼容层，**能解密的包（含酷安 REF）都是走兼容层的** |
| 补 keymint 服务进 ramdisk | 服务 4 个 fqname 全部注册成功（`Found ... in device VINTF manifest`），仍解不开 | 服务通了也没用，因为根因是版本号（见 [DECRYPT-A17.md](DECRYPT-A17.md)） |
| VINTF fragment 放错目录 | `Cannot add a device manifest to a framework manifest` | device manifest 只放 `/vendor/etc/vintf/manifest/`；`/system/etc/vintf/manifest/` 是 **framework** manifest 目录 |
| servicemanager 不重读 VINTF | 改了 xml 但 `Could not find ... in the VINTF manifest` | servicemanager 启动时缓存 VINTF。`kill -9 $(pidof servicemanager)` 让它重启 |
| 单文件 bind mount 不生效 | `mount --bind x.xml /vendor/etc/vintf/manifest/x.xml` 在 `/proc/mounts` 里成功，但仍读不到 | 要 **bind 整个目录** |
| `libvintf@4.0` 读不懂 Android 17 | `VINTF parse error: Illformed file: ... Unrecognized manifest.version 9.0 (libvintf@4.0)` | 12.1 基底的 libvintf 太老。**不要在 recovery 里挂 system/product/system_ext 的 vintf 目录** |
| `keymaster_ver` 改了不起作用 | 改成 `4.x` 依然 `-62` | 该 prop 只被 TWRP 自己的 `Process_Keymaster_Version()` 消费，`system/vold` 里没有任何消费者。**它不影响 TA** |
| FDE 符号全缺 | `undefined symbol: cryptfs_check_footer` | Android 12.1+ 的 `system/vold` 删了 FDE API。thor 是纯 FBE，FDE 分支是死代码，加空实现垫片即可 |

---

## 六、调试方法论（比具体补丁更值钱）

### 6.1 判断「是包的问题还是设备的问题」

**刷一个已知能启动的第三方包做对照** —— 这是最快的分诊手段：

- 官方包能起、自己的不能 → 问题在你的 ramdisk 内容
- 官方包也不能起 → 问题在内核 / 分区 / 设备状态

我们当初就是刷官方 LineageOS recovery 起来后，才彻底排除「Android 17 内核不兼容」。

### 6.2 二分法排除体积假说

往能启动的 ramdisk 里塞 51 MB `/dev/urandom`（不可压缩）重打包，如果照样能起，
说明**体积不是问题**。这一招省掉了无数猜测。

### 6.3 用 linker 日志抓缺库

```bash
adb shell dmesg | grep "CANNOT LINK"
```

出现的库名就是缺的。用递归闭包扫描补：

```python
# 对 ramdisk 里所有 ELF 跑 readelf -d 取 NEEDED，
# 在 system/lib64、vendor/lib64、system/lib64twrp 三个目录里 BFS 解析
```

> 但我们最终发现：**补齐所有库并不能解决「卡第一屏」** —— 那个是底座问题。
> 缺库假说能修 `CANNOT LINK`，不该被当成所有启动失败的万能解释。

### 6.4 不刷机验证法（极其重要）

有 adb 的时候，**不需要刷机就能测新二进制**：

```bash
adb push F:\wsl\recovery_new.bin /system/bin/recovery_new
adb shell chmod 755 /system/bin/recovery_new
adb shell "nohup env LD_LIBRARY_PATH=/system/lib64twrp:/system/lib64 /system/bin/recovery_new > /tmp/r.out 2>&1 &"
```

- 用独立的库目录，**不要污染 `/system/lib64`**
- 不满意就 `kill -9` 重来，机器不会变砖
- **一旦动 `/system/lib64` 就极危险**，会牵连整个系统（我们曾因此差点需要拆机）

### 6.5 逐线程 CPU 增量定位空转

```bash
# 3 秒间隔采样 /proc/PID/task/*/stat 的第 14、15 字段（utime/stime）
```

`wchan=0` + `syscall=running` + ticks 持续增长 = 纯用户态空转。
`wchan=do_sys_poll` = 在等输入，正常。

### 6.6 PowerShell × WSL 的坑

- `wsl.exe bash -c "..."` 里的 `$(...)`、`>`、引号会被 **PowerShell 本地展开**
  → 一律写 `.sh` 文件再 `bash /mnt/f/wsl/x.sh`
- 写 `.sh` 必须无 BOM：
  ```powershell
  [System.IO.File]::WriteAllText($p, $s.Replace("`r`n","`n"), (New-Object System.Text.UTF8Encoding($false)))
  ```
- 设备侧脚本同理：写 `.sh` → `adb push` → `adb shell 'sh /data/local/tmp/x.sh'`
