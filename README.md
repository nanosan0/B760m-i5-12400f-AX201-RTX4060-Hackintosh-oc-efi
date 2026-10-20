# Hackintosh EFI · ASUS TX GAMING B760M WIFI D4

> OpenCore **1.0.8** 引导配置 · Intel i5-12400F · NVIDIA 独显（NullMoth Metal 驱动）  
> 已通过官方 `ocvalidate` 1.0.8 校验：**No issues found.**

这是一份经过逐项审计的 OpenCore EFI。所有配置项都能在官方 `Configuration.tex` 中找到依据，没有来路不明的键，也没有靠「抄别人的 EFI」堆出来的参数。

---

## ⚠️ 先读这一段

**这个仓库里的 SMBIOS 三码是占位符，直接拿来用是启动不了的。**

```xml
<string>MLB-GOES-HERE</string>          
<string>SERIAL-GOES-HERE</string>       
<string>UUID-GOES-HERE</string>         
<data>AAAAAAAA</data>                    
```

请用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 为机型 **MacPro7,1** 生成你自己的三码，填进 `EFI/OC/config.plist` 的 `PlatformInfo → Generic`：

| 字段                   | 怎么填                               |
| -------------------- | --------------------------------- |
| `SystemProductName`  | 保持 `MacPro7,1`                    |
| `SystemSerialNumber` | GenSMBIOS 生成的 Serial              |
| `MLB`                | GenSMBIOS 生成的 MLB                 |
| `SystemUUID`         | GenSMBIOS 生成的 SmUUID              |
| `ROM`                | 你网卡的 MAC 地址（6 字节），GenSMBIOS 会自动生成 |



> 为什么不直接放作者的：公开发布真实三码会让别人也能用同一个身份登录 iMessage / FaceTime，最终导致该序列号被 Apple 拉黑，对谁都没好处。

---

## 硬件

| 部件   | 型号                                        | 状态                                       |
| ---- | ----------------------------------------- | ---------------------------------------- |
| 主板   | ASUS TX GAMING B760M WIFI D4              | ✅                                        |
| CPU  | Intel Core i5-12400F (Alder Lake, 6P/12T) | ✅ 需 `CPUFriend`                          |
| 核显   | 无（F 后缀无 iGPU）                             | —                                        |
| 独显   | NVIDIA GeForce（Turing 及更新架构）              | ⚠️ 第三方 Metal 驱动，装于 `/Library/Extensions` |
| 有线网卡 | Realtek RTL8125 2.5GbE                    | ✅ `RTL812xLucy`                          |
| 无线网卡 | Intel Wi-Fi 6 AX201                       | ✅ `AirportItlwm` + OCLP-Mod              |
| 蓝牙   | Intel 蓝牙（AX201 集成）                        | ✅ `IntelBluetoothFirmware`               |
| 声卡   | Realtek ALC897                            | ✅ `layout-id = 77`                       |
| 存储   | NVMe SSD                                  | ✅ `NVMeFix`                              |

---

## 软件版本

| 组件                                 | 版本                                                                                                 |
| ---------------------------------- | -------------------------------------------------------------------------------------------------- |
| OpenCore                           | 1.0.8（x64，PE Machine `0x8664`）                                                                     |
| 显卡驱动                               | [NullMoth nvidia-macos-driver](https://github.com/nullmoth/nvidia-macos-driver)（Metal 翻译层，Turing+） |
| 目标系统                               | macOS 26 Tahoe（本配置按 **Ventura 版 `AirportItlwm`** + OCLP-Mod 根补丁适配）                                 |
| Lilu                               | 1.7.3                                                                                              |
| VirtualSMC                         | 1.3.9（含 SMCProcessor / SMCSuperIO）                                                                 |
| WhateverGreen                      | 1.7.2                                                                                              |
| AppleALC                           | 1.9.9                                                                                              |
| RestrictEvents                     | 1.1.7                                                                                              |
| CPUFriend + DataProvider           | 1.3.1 / 1.0.1                                                                                      |
| NVMeFix                            | 1.1.4                                                                                              |
| AMFIPass                           | 1.4.1                                                                                              |
| AirportItlwm                       | 2.3.0（**Ventura 构建**）                                                                              |
| IntelBluetoothFirmware / BTPatcher | 2.5.0                                                                                              |
| BlueToolFixup                      | 2.6.9                                                                                              |
| USBToolBox + UTBMap                | 1.2.0 / 1.1                                                                                        |
| RTL812xLucy                        | 1.1.1                                                                                              |

---

## 目录结构

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── ACPI/                 6 个 SSDT
    │   ├── SSDT-EC.aml               嵌入式控制器（必须）
    │   ├── SSDT-PLUG-ALT.aml         CPU 电源管理（12 代必需）
    │   ├── SSDT-USBX.aml             USB 供电属性
    │   ├── SSDT-USB-Reset.aml        USB 复位
    │   ├── SSDT-RTCAWAC.aml          RTC 唤醒修复
    │   └── SSDT-SBUS.aml             SBUS 设备
    ├── Drivers/              4 个 UEFI 驱动
    │   ├── HfsPlus.efi               读 HFS+ 分区
    │   ├── OpenRuntime.efi           运行时服务（必须）
    │   ├── ResetNvramEntry.efi       开机菜单里的重置 NVRAM 项
    │   └── OpenCanopy.efi            图形化启动菜单
    ├── Kexts/                22 个启用的 kext
    ├── Resources/            GoldenGate 主题（26 个图标）
    ├── Tools/                OpenShell.efi
    └── config.plist
```

---

## 安装

1. **准备一个 ≥ 200 MB 的 FAT32 U 盘**（或直接用你的 ESP 分区）。
2. 把本仓库的 `EFI/` 整个复制到 U 盘根目录，形成 `U盘/EFI/BOOT/BOOTx64.efi` 这样的结构。
   > 注意是复制 `EFI` 这一层，不要变成 `EFI/EFI/BOOT`。
3. **填好你的三码**（见开头警告）。
4. BIOS 设置：
   - `Secure Boot` → **Disabled**
   - `Fast Boot` → **Disabled**
   - `VT-d` → Disabled（或保持开启 + 依赖 `DisableIoMapper`）
   - `Above 4G Decoding` → **Enabled**
   - `Re-Size BAR Support` → Enabled（与 `ResizeGpuBars` 配合）
   - `CFG-Lock` → Disabled（若找不到此项，配置里已开 `AppleXcpmCfgLock` 兜底）
   - `SATA Mode` → AHCI
5. 从 U 盘启动，进 macOS 安装器。

---

## 配置要点：为什么是这么写的

### 1. `SecureBootModel = Disabled` 与 `FixupAppleEfiImages`

这两个是一对，缺一不可。

官方 `Configuration.tex` 对 `FixupAppleEfiImages` 的原话：

> On a system with such a modern, stricter loader this quirk is required to load Mac OS X 10.4 to macOS 10.12, and **is required for all newer macOS when `SecureBootModel` is set to `Disabled`**.

macOS 的 `boot.efi` 本身带有 `W^X` 权限错误，严格的 PE 加载器会拒绝加载。本配置因为要配合 OCLP-Mod 根补丁，`SecureBootModel` 必须是 `Disabled`，所以 `FixupAppleEfiImages` **必须** 为 `true`。

> 很多从别处下载的 EFI 缺这个键（`ocvalidate` 会报 `Missing key`），因为它的 Failsafe 是 `false`，缺了就等同于关着 —— 平时能启动，遇到严格固件就卡住。

### 2. 显卡：NullMoth NVIDIA Driver

NVIDIA 没有为近年的 macOS 提供官方驱动（Web Driver 时代早已终结）。这台机器用的是  
**[nullmoth/nvidia-macos-driver](https://github.com/nullmoth/nvidia-macos-driver)** —— 一个把 Apple Metal 指令实时翻译成 Vulkan 的第三方 Metal 驱动。

```
Metal App → NVMTLDriver.bundle → 翻译器 (AIR → SPIR-V) → NVK (Mesa Vulkan) → NVRM / NVAccel / NVRMFB → GPU
```

| 组件                                   | 落盘位置                          | 作用                                           |
| ------------------------------------ | ----------------------------- | -------------------------------------------- |
| `NVMTLDriver.bundle`                 | `/Library/GPUBundles/`        | Metal 驱动插件                                   |
| `libnvmtl_translate.dylib`           | 插件内部                          | 着色器翻译（基于 metal2vulkan，LGPL-3.0）              |
| `nvk/`                               | `/Library/GPUBundles/nvmtl/`  | Vulkan 后端（Mesa + NAK 编译器）                    |
| `NVRM` `NVAccel` `NVRMFB` `NVRMAGDC` | `/Library/Extensions/`        | 内核扩展：驱动本体 / 加速器 / 显示 framebuffer / 显示策略 shim |
| GSP 固件                               | `/Users/Shared/nvfw/nvidia/…` | NVIDIA 原厂固件                                  |

**支持范围**：Turing 及更新架构 —— GTX 16 系、RTX 20 / 30 / 40 / 50、TITAN RTX、Quadro RTX、T 系列。  
作者主要在 RTX 5060 上测试，其他型号可能存在尚未发现的问题。

> **关键：这些 kext 绝对不要在 `EFI/OC/Kexts` 里注入。**  
> 它们从 Auxiliary Kernel Collection 加载，OpenCore 只负责设置 SIP 和 boot-args。  
> 本仓库的 `config.plist` 里因此**没有任何 NVIDIA 相关条目** —— 这是正确的，不是漏配。

#### 驱动对 OpenCore 的要求（本配置逐条核对）

| 要求                                                                                                                        | 本配置                                       | 结果 |
| ------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- | -- |
| boot-args 含 `nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0` | 全部存在                                      | ✅  |
| 不能有 `nv_disable=1`、`-wegnoegpu`（两者都会把 NVIDIA 卡藏起来）                                                                        | 均无                                        | ✅  |
| `csr-active-config` 包含 `0x0A43` 位                                                                                         | `0x0FFF`（超集，含该位）                          | ✅  |
| `Misc > Security > SecureBootModel` = `Disabled`                                                                          | Disabled                                  | ✅  |
| `Lilu.kext` 与 `AMFIPass.kext` 在 `Kernel > Add` 且启用                                                                        | 都在                                        | ✅  |
| `Kernel > Block` 屏蔽 `com.apple.iokit.IONDRVSupport`                                                                       | 已屏蔽                                       | ✅  |
| 不能有该卡的 WhateverGreen NVIDIA patch、不能有 `agdpmod=pikera`                                                                    | 均无                                        | ✅  |
| 设进 NVRAM 的键必须同步写进 `NVRAM > Delete`                                                                                        | `boot-args`、`csr-active-config` 都在        | ✅  |
| 机型须能被该 macOS 版本接受                                                                                                | `MacPro7,1` —— 1401 从 1.14 起对带独显的台式机就是选它，macOS 15 与 26 都接受（`iMacPro1,1` 会被 macOS 26 拒绝） | ✅  |
| BIOS：Above 4G Decoding ON、CSM OFF                                                                                         | 见上方「安装」章节第 4 步                            | —  |

**为什么 `SecureBootModel` 必须是 `Disabled`**：Apple Secure Boot 会拒绝 Apple 未签名的 kext，而 NVRM 系列正是未签名加载的。  
这同时解释了 §1 里 `FixupAppleEfiImages` 为什么在这里是**强制项** —— 两者是同一条因果链上的上下游。

#### 安装

推荐用作者配套的 **1401** 应用（[nullmoth/1401](https://github.com/nullmoth/1401)）：它会自动定位启动用的 OpenCore 分区、备份 `config.plist`、写入上面这些设置、安装驱动，并在启动菜单里加一条 **1401: Remove NVIDIA driver** 用于一键还原。

手动安装则要 `install.sh` + 重建 Auxiliary Kernel Collection，且必须自己改对 OpenCore 配置 —— 顺序错了就是黑屏。

> 首次以驱动启动时，会停在 `PCI configuration end` 最多约一分钟等 GPU 起来，这是正常现象，不要以为卡死了。

#### `gfxrst=4` 到底是干什么的

**不是显卡复位。** 官方说明是：装了 WhateverGreen 的情况下，WhateverGreen 的启动画面拷贝会在 NVIDIA 显示打开时读取已释放的内存，导致 panic；`gfxrst=4` 就是消掉这个冲突的。  
所以它跟显卡无关，跟 **WhateverGreen** 有关 —— 本机为 Intel 平台保留了 WEG，所以需要它。

#### 排障用 boot-args（平时不要加）

| 参数                  | 作用                            |
| ------------------- | ----------------------------- |
| `-nvrmnoflip`       | 关闭零拷贝扫描输出，桌面闪烁 / 移动光标时撕裂优先试这个 |
| `-nvrmnopm`         | 关闭系统睡眠支持，症状是「睡眠后唤醒显卡回不来」      |
| `-nvrmnogo`         | 驱动先不自启，等 `nvrmctl go` 手动放行    |
| `-nvrmnobootscreen` | 不截取启动画面                       |

### 3. `Kernel > Block` 的三条屏蔽

| 被屏蔽的 kext                          | 原因                                                                    |
| ---------------------------------- | --------------------------------------------------------------------- |
| `com.apple.iokit.IOSkywalkFamily`  | 用仓库里的改版替换，不屏蔽的话系统原版会抢先加载，Intel 网卡起不来                                  |
| `com.apple.driver.AppleEthernetRL` | 系统自带 RTL 驱动与本机的 `RTL812xLucy` 冲突                                      |
| `com.apple.iokit.IONDRVSupport`    | 不屏蔽的话，固件自带的 framebuffer 会从 `NVRMFB` 手里抢走 display index 0，NVIDIA 输出起不来 |

### 4. Wi-Fi 的三段式依赖

Intel 网卡在 macOS 14+ 上**不是**只放一个 `AirportItlwm` 就够：

```
IOSkywalkFamily.kext            ← 网卡协议栈基座（Apple 原版，需屏蔽系统同名 kext）
IO80211FamilyLegacy.kext        ← 老版 802.11 框架
└── AirPortBrcmNIC.kext         ← 它的插件，名字带 Brcm 但 Intel 也靠它挂载
AirportItlwm-Ventura.kext       ← 真正的 Intel 驱动（Ventura 构建）
```

> 这三个 Apple 原版文件不是「Broadcom 专用、装在 Intel 机上白装」。凡是看到这种说法，都是误判。
>
> 为什么用 **Ventura 构建** 的 AirportItlwm：Tahoe 上的社区方案就是 Ventura 构建 + OCLP-Mod 根补丁，Sequoia 构建在 Tahoe 上不工作。

### 5. boot-args 逐条

```
keepsyms=1 amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0
nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth -lilubetaall gfxrst=4
```

| 参数                                       | 作用                                                                |
| ---------------------------------------- | ----------------------------------------------------------------- |
| `keepsyms=1`                             | 内核崩溃时保留函数符号，否则 panic 日志全是裸地址。**不要删**                              |
| `amfi_get_out_of_my_way=0x1` `amfi=0x80` | 关闭 AMFI 校验，配合 `AMFIPass` + 根补丁                                    |
| `ipc_control_port_options=0`             | 修复部分 App 的 XPC 通信                                                 |
| `nvfb=1` `nvaccel=1` `nvfbheads=4`       | NullMoth 驱动的必需项：启用 framebuffer、启用 NVAccel 加速、4 个显示头               |
| `-nvkmsnosmooth`                         | 同上（驱动必需），关闭 nvkms 平滑缩放                                            |
| `-lilubetaall`                           | 允许 Lilu 及其插件在较新的系统版本上加载                                           |
| `gfxrst=4`                               | 装了 WhateverGreen 时必需：否则 WEG 的启动画面拷贝会在 NVIDIA 显示打开时读已释放内存，直接 panic |

### 6. 日志已关闭

`Misc > Debug > Target` 设为 **0**。官方定义：`0x01` 位是总开关，关掉后「all log is discarded」，ESP 分区根目录不会再生成 `opencore-*.txt`；`AppleDebug`、`ApplePanic`、`DisableWatchDog` 一并置 `false`。

需要抓日志排查时：

```xml
<key>Target</key><integer>67</integer>   
<key>AppleDebug</key><true/>
<key>ApplePanic</key><true/>
```

再在 boot-args 前面加回 `debug=0x100`。

### 7. 其他关键项

| 键                                 | 值                        | 说明                      |
| --------------------------------- | ------------------------ | ----------------------- |
| `Misc > Security > ScanPolicy`    | `0`                      | 不限制扫描范围，方便从任意分区启动       |
| `Misc > Security > Vault`         | `Optional`               | 不做文件保管库校验               |
| `csr-active-config`               | `0x0FFF`                 | SIP 基本全关，根补丁需要          |
| `DeviceProperties`                | `layout-id = 77`         | ALC897 的可用 layout       |
| `PlatformInfo > UpdateSMBIOSMode` | `Custom`                 | 配合 `SpoofVendor = true` |
| `UEFI > Output > Resolution`      | `Max`                    | 用显示器原生分辨率               |
| `Misc > Boot > PickerVariant`     | `Acidanthera\GoldenGate` | 图形化选单主题                 |

---

## 这台机器上做过什么修复

本 EFI 经过完整审计，修掉了以下问题（记录见 commit 历史）：

| 问题                                                         | 处理                |
| ---------------------------------------------------------- | ----------------- |
| `SSDT-USBX.aml` 在配置里 `Enabled=true` 但文件不存在                 | 补回文件，USB 供电属性恢复注入 |
| `FixupAppleEfiImages` 缺失（`SecureBootModel=Disabled` 下的必需项） | 补为 `true`         |
| 15 处 schema 缺键，`ocvalidate` 报 16 项                         | 全部补齐，校验清零         |
| `Misc > Debug` 开着文件日志（`Target=67`）                         | 关闭                |
| 一个未被引用的孤儿 kext（15.4 MB）                                    | 从仓库剔除             |

---

## 已知限制

- **显卡驱动是第三方逆向工程产物。** 驱动的 kext 装在 `/Library/Extensions` 并从 Auxiliary Kernel Collection 加载，macOS 每次大版本升级都可能需要等驱动跟进新版本。升级系统前务必先看 [releases](https://github.com/nullmoth/nvidia-macos-driver/releases) 和 `CHANGELOG`。
- **显卡只支持 Turing 及更新架构。** Pascal、Maxwell 等更老的卡不在支持范围内。
- **Wi-Fi 需要根补丁。** 同上，`AirportItlwm` 装进 `/Library/Extensions` 后依赖 OCLP-Mod 补丁链。
- **`i5-12400F` 无核显**，无法使用 Sidecar / 随航的硬件编码，也无法在无独显时启动 GUI。
- **不保证移植到其他主板可用。** ACPI 中的 `SSDT-RTCAWAC`、`SSDT-SBUS` 是针对这块板子生成的；换主板请重新生成。
- **macOS 大版本升级前请先备份 EFI**，并确认 kext 已跟进。

---

## 常见安装问题排查

> 第一件事永远是：**用最新版的 1401 和驱动**。下面很多问题在后续版本里已经修掉了，而用旧版 1401 做的 U 盘会一直带着旧问题，除非重做一次。

### A. 卡在选单 / 还进不了 macOS

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 停在 `PCI configuration` | 通常是 Resizable BAR 或 Above 4G Decoding 的设置 | BIOS 里打开 Above 4G Decoding、关掉 CSM。仍卡住就导出 U 盘日志 |
| `EB.MM.AKM`、`STOP 0x16`、`StartImage failed - Aborted`、`Couldn't allocate runtime area` | 固件内存映射没给 `boot.efi` 留出空间（Ryzen 7000/9000 上常见） | 用最新 1401 重做 U 盘 —— 它会套用正确的内存设置，一组失败会自动换下一组。重启旧 U 盘没用 |
| 停在 `EXITBS:START` | macOS 已从固件接管，但在打印任何东西之前就停了。Intel 12 代及更新、AMD 笔记本上多为 CPU 核心布局或内存映射问题 | 先用最新 1401；再导出 U 盘的 `opencore-*.txt`；若出现 panic，在啰嗦模式下把整屏拍下来 |
| `This version of Mac OS X is not supported on this platform!` 加一串 `Mac-…` 代码 | 配置里的机型不是这个 macOS 版本接受的机型。**macOS 26 会拒绝 `iMacPro1,1`**，macOS 15 拒绝一批更老的机型 | 用 `MacPro7,1` —— 1401 从 1.14 起对带独显的台式机就是选它，macOS 15 和 26 都接受。**本仓库正是 `MacPro7,1`** |
| 安装进度到约 19 分钟时整个冻住（Caps Lock 都不亮），Ryzen 7000/8000/9000 | 这些主板在安装器写固件设置时会挂住 | 重做 U 盘（1.14 已修）。自己维护 EFI 的话设 `Booter > Quirks > DisableVariableWrite = true` |
| `OCB: LoadImage failed - Unsupported`、`Failed to retrieve boot file from DMG` | U 盘上的恢复镜像损坏 | 换一个 U 盘重做（1.14 写完会回读校验） |
| `OS.dmg.root_hash … Err(0xE)` | 旧版 1401 做的 U 盘，或恢复镜像与目标 macOS 版本不匹配 | 用最新 1401 重做 |
| 选完 U 盘后偶尔卡住，重做一次又好了 | U 盘或接口在读那 900 MB 恢复镜像时太慢/不稳 | 换机箱后面的 USB 口，或换 U 盘 |

### B. 驱动装好之后的显示问题

| 现象 | 原因 | 处理 |
| --- | --- | --- |
| 桌面正常显示约 1 分钟后黑屏，只剩光标 | 启动约 1 分钟后驱动切到「从显卡显存直接扫描输出」（`NVAccel::autoGo`）。如果桌面其实画在集成显卡上、或显卡的 BAR 只有 256 MB 小窗口，这个画面就是空的 | 1.14 已修（这类机器上驱动会保持拷贝路径）。旧版加 `-nvrmnoflip` |
| 100–240 Hz 下黑屏或闪烁，或改分辨率/刷新率后出现 | 超过 DisplayPort 1.4 带宽的高刷模式需要 DSC 压缩，而 macOS 没被告知 | 1.14 会下发压缩设置。仍复现就在故障发生时按 **Send logs**，应用会记录显示路径 |
| macOS 26 (Tahoe) 下桌面全黑 | 包里 macOS 26 版的加速器（NVAccel）过期了 | 1.14 已修 |
| 显示器切到别的输入再切回来就没画面 | 驱动只点亮「没见过的新显示器」 | 1.14 已修。旧版把线拔了重插 |
| 无 Resizable BAR 的老 Intel 板上约 100 秒后黑屏，日志里是 `cannot hold … not placing` | 固件没有列出 64 位 PCI 地址范围，显卡的显存窗口无处安放 | 1.14 已修 |
| 显示器进入睡眠后，屏幕仍停留在最后一张画面上（不熄灭） | 驱动没有关掉显示输出 | 1.14 已修 |
| 睡眠后屏幕黑掉、机器必须硬复位（`Sleep Wake Failure`、`Darkwake Exit Failure`） | 驱动没有挂起/恢复 GPU，唤醒时把画面推到一块已经断电的卡上 | 新版已支持系统睡眠（见驱动仓库 `docs/SLEEP-WAKE.md`）。旧版先把系统睡眠关掉：`sudo pmset -a sleep 0`，也不要用苹果菜单里的「睡眠」 |
| 唤醒后桌面黑屏只剩光标，或停在睡前那张画面上 | WindowServer 没被告知 framebuffer 已经唤醒，或它的显存被替换掉了 | 新版已修。仍复现就发日志，里面有 `NVRM-xnu: wake #` 和 `NVRM-fb: system wake` |
| 睡眠中大约每分钟自己醒一次（`pmset -g log` 里 `DarkWake ... due to XDCI ... XHCI`） | 固件允许 USB 设备唤醒系统（GPE `0x6D`）。**这不是驱动的问题** | 用 `SSDT-GPRW` 在 `ACPI > Patch` 里把 `GPRW` 重命名为 `XPRW`，之后只有电源键能唤醒 |
| 窗口变黑块或停留在旧画面，点一下才刷新；预览里图片显示成马赛克 | 图像内容在 GPU 画完之前就被标记成「已是最新」 | 1.14 已修（社区补丁） |
| 浏览器报「Your browser does not support WebGL」 | 部分 WebGL 着色器翻译失败 | 1.14 已修 |
| 照片 App 人脸识别反复崩溃（`mediaanalysisd`） | 驱动里有一个 Metal 对象被释放了两次 | 1.14 已修 |
| 登录界面出现那一刻内核 panic（WindowServer、`IOFramebuffer::postOpen`、`memcpy` 缺页），且装了 WhateverGreen | WhateverGreen 在第一次 framebuffer 启动时，会从它开机时记下的地址拷贝启动画面；而 macOS 可能已经把那段映射释放了 | boot-args 加 `gfxrst=4` —— 它关掉这次拷贝，改由驱动自己画启动画面。**本仓库已包含** |
| 在 Turing（GTX 16 / RTX 20）上装完驱动后卡在 `AppleKeyStore` 循环 | 显卡固件需要更多时间，驱动才去点屏 | 1.13 起 Setup 会自动加 `nvrmsettle=10000`。手动装的自己加进 boot-args |

### C. 出问题了怎么退回去（最重要的一段）

驱动出问题时可能连画面都没有，**先记住这三条逃生路线**，按严重程度递进：

**1. 只想这一次不加载驱动**

开机在 OpenCore 选单里选 **`1401: Start once without NVIDIA driver`**，按 `Y`，再选 macOS 或 Recovery。下次开机会自动恢复驱动。

**2. 要彻底卸载驱动**

选单里选 **`1401: Remove NVIDIA driver`**，按 `Y`，再选 macOS。它会自己卸载并重启，同时把保存的分辨率和刷新率恢复成安全默认值 —— 这样即使之前设过集成显卡显示不了的模式，也仍能看到画面。

**3. 连选单和画面都看不见了**

进 macOS Recovery（驱动在 Recovery 里不运行），打开「终端」执行：

```bash
nvram 7C436110-AB2A-4BBB-A880-FE41995C9F82:nullmoth-remove=1
```

然后重启进 macOS。机器会自己把驱动卸掉再重启一次，**全程不需要画面**。效果与选单里的卸载项完全一致。

#### 选单里看不到那两个 1401 条目？

官方说明：安装器会把选单切成 **`Builtin`**、并且不再把条目藏在空格键后面。

本仓库的 `Misc > Boot` 当前是：

- `HideAuxiliary = false` ✅ 条目不需要按空格才出现
- `PickerMode = External`（OpenCanopy 图形选单）⚠️ 与官方默认的 `Builtin` 不同

**如果你的显示出了问题、而选单里又找不到那两个条目，会非常被动。** 届时把 `PickerMode` 临时改成 `Builtin` 就能看到并选择它们，处理完再改回 `External`。这是本仓库相对官方默认配置唯一需要留意的地方。

> 另注：早期版本会往 OpenCore 里加一个 `NullMothLogo.efi` 启动项，现已废弃；新版安装/更新/卸载驱动时会自动移除它的条目和文件。**若你的选单从来不出现**，而 EFI 里还留着这个条目，就把它禁用掉。本仓库已不含任何 `NullMothLogo.efi` 条目。

### D. 确认驱动到底有没有加载

重启后跑这两条：

```bash
kmutil showloaded --list-only | grep nullmoth
system_profiler SPDisplaysDataType | head -20
```

第一条应该输出 **4 行**，对应 `NVRM` / `NVAccel` / `NVRMFB` / `NVRMAGDC` 四个 kext。
第二条应该能看到你的 GeForce 显卡，并且 **Metal: supported**。

只满足一半（kext 加载了但 Metal 不支持）通常意味着 `NVAccel` 没有为该 macOS 版本正确编译或安装。

### E. 拿日志

上面的表覆盖不到时，不要自己猜：

- **Mac 端**：1401 → Crash report → **Send logs**
- **Windows 端 1401**：**Scan for logs and send them**

它会带上驱动状态（PCI 设备 ID、BAR 地址）、内核日志、插件日志、崩溃报告和 OpenCore 启动日志，并自动去掉机器名、序列号和内存地址，最后给一个 report ID。把 report ID 贴进 issue，比贴手机拍的屏幕清楚得多。

---

## 版权与归属


- `OpenCore.efi`、`OpenRuntime.efi`、`OpenCanopy.efi`、`ResetNvramEntry.efi`、`HfsPlus.efi` 来自 [Acidanthera OpenCore](https://github.com/acidanthera/OpenCorePkg) 与 [OcBinaryData](https://github.com/acidanthera/OcBinaryData)，遵循 BSD-3-Clause。
- 各第三方 kext 版权归其作者所有，遵循各自的 LICENSE（Lilu / VirtualSMC / WhateverGreen / AppleALC / RestrictEvents / CPUFriend / NVMeFix 为 BSD-3；AirportItlwm / IntelBluetoothFirmware 为 GPL-2.0；USBToolBox 为 MIT）。
- `IOSkywalkFamily.kext`、`IO80211FamilyLegacy.kext`、`AirPortBrcmNIC.kext` 是 **Apple 原始二进制文件**，版权归 Apple Inc. 所有，此处仅作互操作性研究用途。若权利人提出异议，将立即移除这些文件。
- **本仓库不包含** [nullmoth/nvidia-macos-driver](https://github.com/nullmoth/nvidia-macos-driver) 的任何文件。该项目的 kext 与插件以 **PolyForm Noncommercial 1.0.0** 授权（© NullMoth Systems），允许个人、研究、教育与非营利使用，**禁止任何商业用途**；其翻译器部分为 LGPL-3.0-or-later，NVK 部分为 Mesa 的 MIT 授权，GSP 固件遵循 NVIDIA 的可再分发固件许可。下载与安装请走[官方 Releases](https://github.com/nullmoth/nvidia-macos-driver/releases)。
- `config.plist` 与文档由本仓库维护者整理，可自由参考。

## 免责声明

修改引导配置有风险，可能导致系统无法启动、数据丢失或硬件异常。使用前请备份重要数据。本仓库按「现状」提供，不附带任何担保，使用后果由使用者自行承担。

## 参考

- [OpenCorePkg 官方文档](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/Configuration.tex)
- [**nullmoth/nvidia-macos-driver**](https://github.com/nullmoth/nvidia-macos-driver) —— NVIDIA Metal 驱动本体（含 `docs/CARD-SUPPORT.md`、`docs/TROUBLESHOOTING.md`、`docs/SLEEP-WAKE.md`）
- [nullmoth/1401](https://github.com/nullmoth/1401) —— 配套安装器，可从 Windows 或 macOS 安装
- [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/)
- [OpenCore Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher)
- [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS)
- [USBToolBox](https://github.com/USBToolBox/tool)
