# Plan 2 低检测方案（GhostLock + 自维护 KernelSU LKM）

适用：Galaxy Tab S8+（SM-X800），Android 16，`5.10.236-android12-9-2810866-abX800ZCSBEZH3`，锁 BL。

## 流程

1. GhostLock（CVE-2026-43499）拿内核读写，不改分区、不解 BL。
2. root 脚本里 `ksud late-load --kmi android12-5.10 --allow-shell` 加载 `kernelsu.ko`。
3. 加载成功后立即：
   - `ksud feature set selinux_hide 1`
   - `ksud feature set kernel_umount 1`
   - `ksud feature set sulog 0`
   - `echo 1 > /sys/fs/selinux/enforce`
   - `rm -f` 临时推入的 ksud
4. 重启后内核状态消失，需要重跑。

## 检测面（本方案已覆盖 / 仍暴露）

| 检测点 | 谁能看到 | 现状 |
|---|---|---|
| Play Integrity STRONG | App / 服务端 | 锁 BL + green，大概率过；服务端裁定不可控 |
| SELinux permissive | App（间接）/ shell | 已写回 enforcing，`selinux_hide` 隐藏修改 |
| `/proc/modules` 里的 kernelsu | shell（AOSP 16 下 App 无读权限） | 仍暴露；改模块名收益低且会破坏 GhostLock 的 grep 判断 |
| `/sys/module/kernelsu` | shell | 仍暴露 |
| 管理器包名 | App | **仍暴露，优先级最高**；需重打包 APK 改 applicationId 并让 ko 的 KSU_PACKAGE_NAME / 签名哈希匹配 |
| `/data/adb/ksu`、模块目录 | shell / 取证 | 仍暴露（App 读不到 /data/adb） |
| ksud / su 路径 | App 可 stat/access | 部分隐藏；`EACCES` 与 `ENOENT` 差异仍是信号 |
| 内核 taint / dmesg / kallsyms | shell / root | 仍暴露（外部模块 taint） |
| 挂载 statfs/inode 侧信道 | App | `kernel_umount` 只挡一部分；SUSFS 才彻底（需内核补丁，LKM 做不了） |
| supercall / syscall 行为 | native 检测 | KSU 自带对抗有限 |
| 运行时窗口 | App 采样 | uid 0 进程 + permissive 瞬间，已压到最小 |
| 三星 KDP/RKP | hypervisor | 可能拦内核 text 写入或 panic；需真机验证 |
| CFI/LTO | 内核 | 自编 ko 必须匹配，否则加载失败/崩溃 |

## 构建

- 源码：`akm-04/Samsung_Kernel_sm8450_common_gts8x` 分支 `A16_common-5.10.236`。
- 配置：固件提取的 `kernel.config`，改 `LOCALVERSION` 为 `-android12-9-2810866-abX800ZCSBEZH3`，关 `TRIM_UNUSED_KSYMS`。
- 工具链：clang-r416183b，`CROSS_COMPILE=aarch64-linux-gnu- CLANG_TRIPLE=aarch64-linux-gnu- LLVM=1 LLVM_IAS=1`。
- 先全量编内核拿 `Module.symvers`，再 `M=KernelSU/kernel CONFIG_KSU=m modules`。
- CI：`Transwarpcom/smx800-ksu-build`（私有）。

### 隐藏版构建的坑

- SDK 平台包名是 `platforms;android-37.0`（带 minor），不是 `platforms;android-37`；`build-tools;37.0.0` 正常。
- ksud 的路径由 `concatcp!(ADB_DIR, "ksu/")` 这类宏拼出，必须单独替换宏参数，否则 ko 改了 `.sysd`、ksud 仍写 `/data/adb/ksu`。
- ksud 的 bindgen 必须给 NDK sysroot：`BINDGEN_EXTRA_CLANG_ARGS_aarch64_linux_android="--sysroot=.../sysroot -I.../sysroot/usr/include/aarch64-linux-android"`，否则会去主机 `/usr/include/linux/ioctl.h` 找 `asm/ioctl.h`。
- 管理器 APK 的 `lib/arm64-v8a/libksud.so` 要手动把 Rust 编出的 `ksud` 放进去；Gradle 不会构建它。

## 待办

- [x] CI 产出 kernelsu.ko：`https://github.com/Transwarpcom/smx800-ksu-build/releases/tag/v1-5.10.236`
  - sha256 `f9ccffcba2b782ce15623f446aa5f8ff64330e50b33415e826d16a349065de48`
  - vermagic `5.10.236-android12-9-2810866-abX800ZCSBEZH3 SMP preempt mod_unload modversions aarch64`
  - 231 个未定义符号全部存在于设备 kallsyms，`ksud insmod` 可解析
- [x] 管理器 APK 改包名 + 重编 ko 的 `KSU_PACKAGE_NAME`/签名哈希。
  - 包名 `com.sysd.core`，应用名 `System Service`。
  - 管理器 APK：`hidden-manager.apk`，6,385,747 字节，sha256 `e294584ae5cf4f097b52b4530e2b3fda93bc1ab955fde64a4db7f0c8dd11bf14`。
  - 签名证书：DER 1,304 字节（0x518），sha256 `3b2c03824b35ca718277bb2d6a7ebe8e3072c6d0b20fd266265fb177bea6df78`；ko 里编入同一组 `KSU_EXPECTED_SIZE/HASH`。
  - ko：6,330,968 字节，sha256 `3f62a4f3d4078e1df3095764ae82f2eb144609d735f1e2cfd4d625f2058a2877`，vermagic 不变。
- [x] `/data/adb` 路径 fork：KSU 工作目录改为 `/data/adb/.sysd`，ksud 入口 `/data/adb/.sysd/ksud`，模块目录 `/data/adb/.sysd/modules`；kernel、ksud、manager 三处同步。
- [ ] 真机 late-load 验证（KDP/RKP/CFI）。

## 设备侧

```sh
adb push kernelsu.ko /data/local/tmp/kernelsu.ko
# GhostLock 的 root 脚本会优先：
#   ksud insmod /data/local/tmp/kernelsu.ko allow_shell=1
# 失败则回落到 ksud late-load（内置 ko）。
```

ko 是 thin LTO + CFI 构建，设备内核是 full LTO；CFI 类型哈希理论上兼容，但必须真机验证。三星 KDP/RKP 可能拦内核 text 写入或 panic。
