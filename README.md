# Hackintosh EFI · ASUS TX GAMING B760M WIFI D4

> OpenCore **1.0.8** 引导配置 · Intel i5-12400F · macOS Tahoe (26)
> 已通过官方 `ocvalidate` 1.0.8 校验：**No issues found.**

这是一份经过逐项审计的 OpenCore EFI。所有配置项都能在官方 `Configuration.tex` 中找到依据，没有来路不明的键，也没有靠「抄别人的 EFI」堆出来的参数。

---

## ⚠️ 先读这一段

**这个仓库里的 SMBIOS 三码是占位符，直接拿来用是启动不了的。**

```xml
<string>MLB-GOES-HERE</string>          <!-- 主板序列号 -->
<string>SERIAL-GOES-HERE</string>       <!-- 整机序列号 -->
<string>UUID-GOES-HERE</string>         <!-- 系统 UUID -->
<data>AAAAAAAA</data>                    <!-- ROM，6 字节 0x00 -->
```

请用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 为机型 **MacPro7,1** 生成你自己的三码，填进 `EFI/OC/config.plist` 的 `PlatformInfo → Generic`：

| 字段 | 怎么填 |
| --- | --- |
| `SystemProductName` | 保持 `MacPro7,1` |
| `SystemSerialNumber` | GenSMBIOS 生成的 Serial |
| `MLB` | GenSMBIOS 生成的 MLB |
| `SystemUUID` | GenSMBIOS 生成的 SmUUID |
| `ROM` | 你网卡的 MAC 地址（6 字节），GenSMBIOS 会自动生成 |

> 为什么不直接放作者的：公开发布真实三码会让别人也能用同一个身份登录 iMessage / FaceTime，最终导致该序列号被 Apple 拉黑，对谁都没好处。

---

## 硬件

| 部件 | 型号 | 状态 |
| --- | --- | --- |
| 主板 | ASUS TX GAMING B760M WIFI D4 | ✅ |
| CPU | Intel Core i5-12400F (Alder Lake, 6P/12T) | ✅ 需 `CPUFriend` |
| 核显 | 无（F 后缀无 iGPU） | — |
| 独显 | NVIDIA 独立显卡（nvkms 方案，见下方说明） | ⚠️ 需根补丁 |
| 有线网卡 | Realtek RTL8125 2.5GbE | ✅ `RTL812xLucy` |
| 无线网卡 | Intel Wi-Fi 6 AX201 | ✅ `AirportItlwm` + OCLP-Mod |
| 蓝牙 | Intel 蓝牙（AX201 集成） | ✅ `IntelBluetoothFirmware` |
| 声卡 | Realtek ALC897 | ✅ `layout-id = 77` |
| 存储 | NVMe SSD | ✅ `NVMeFix` |

---

## 软件版本

| 组件 | 版本 |
| --- | --- |
| OpenCore | 1.0.8（x64，PE Machine `0x8664`） |
| Lilu | 1.7.3 |
| VirtualSMC | 1.3.9（含 SMCProcessor / SMCSuperIO） |
| WhateverGreen | 1.7.2 |
| AppleALC | 1.9.9 |
| RestrictEvents | 1.1.7 |
| CPUFriend + DataProvider | 1.3.1 / 1.0.1 |
| NVMeFix | 1.1.4 |
| AMFIPass | 1.4.1 |
| AirportItlwm | 2.3.0（**Ventura 构建**） |
| IntelBluetoothFirmware / BTPatcher | 2.5.0 |
| BlueToolFixup | 2.6.9 |
| USBToolBox + UTBMap | 1.2.0 / 1.1 |
| RTL812xLucy | 1.1.1 |

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

### 2. `Kernel > Block` 的三条屏蔽

| 被屏蔽的 kext | 原因 |
| --- | --- |
| `com.apple.iokit.IOSkywalkFamily` | 用仓库里的改版替换，不屏蔽的话系统原版会抢先加载，Intel 网卡起不来 |
| `com.apple.driver.AppleEthernetRL` | 系统自带 RTL 驱动与本机的 `RTL812xLucy` 冲突 |
| `com.apple.iokit.IONDRVSupport` | 防止 boot framebuffer 抢占 index 0，给 nvkms 让位 |

### 3. Wi-Fi 的三段式依赖

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

### 4. boot-args 逐条

```
keepsyms=1 amfi_get_out_of_my_way=0x1 amfi=0x80 ipc_control_port_options=0
nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth -lilubetaall gfxrst=4
```

| 参数 | 作用 |
| --- | --- |
| `keepsyms=1` | 内核崩溃时保留函数符号，否则 panic 日志全是裸地址。**不要删** |
| `amfi_get_out_of_my_way=0x1` `amfi=0x80` | 关闭 AMFI 校验，配合 `AMFIPass` + 根补丁 |
| `ipc_control_port_options=0` | 修复部分 App 的 XPC 通信 |
| `nvfb=1` `nvaccel=1` `nvfbheads=4` | nvkms 显示输出：启用 framebuffer、硬件加速、4 个显示头 |
| `-nvkmsnosmooth` | 关闭 nvkms 的平滑缩放（避免画面撕裂） |
| `-lilubetaall` | 允许 Lilu 及其插件在新系统上加载（Tahoe 必需） |
| `gfxrst=4` | 显卡复位策略 |

### 5. 日志已关闭

`Misc > Debug > Target` 设为 **0**。官方定义：`0x01` 位是总开关，关掉后「all log is discarded」，ESP 分区根目录不会再生成 `opencore-*.txt`；`AppleDebug`、`ApplePanic`、`DisableWatchDog` 一并置 `false`。

需要抓日志排查时：

```xml
<key>Target</key><integer>67</integer>   <!-- 0x01|0x02|0x40 = 总开关 + 屏幕 + 写文件 -->
<key>AppleDebug</key><true/>
<key>ApplePanic</key><true/>
```

再在 boot-args 前面加回 `debug=0x100`。

### 6. 其他关键项

| 键 | 值 | 说明 |
| --- | --- | --- |
| `Misc > Security > ScanPolicy` | `0` | 不限制扫描范围，方便从任意分区启动 |
| `Misc > Security > Vault` | `Optional` | 不做文件保管库校验 |
| `csr-active-config` | `0x0FFF` | SIP 基本全关，根补丁需要 |
| `DeviceProperties` | `layout-id = 77` | ALC897 的可用 layout |
| `PlatformInfo > UpdateSMBIOSMode` | `Custom` | 配合 `SpoofVendor = true` |
| `UEFI > Output > Resolution` | `Max` | 用显示器原生分辨率 |
| `Misc > Boot > PickerVariant` | `Acidanthera\GoldenGate` | 图形化选单主题 |

---

## 这台机器上做过什么修复

本 EFI 经过完整审计，修掉了以下问题（记录见 commit 历史）：

| 问题 | 处理 |
| --- | --- |
| `SSDT-USBX.aml` 在配置里 `Enabled=true` 但文件不存在 | 补回文件，USB 供电属性恢复注入 |
| `FixupAppleEfiImages` 缺失（`SecureBootModel=Disabled` 下的必需项） | 补为 `true` |
| 15 处 schema 缺键，`ocvalidate` 报 16 项 | 全部补齐，校验清零 |
| `Misc > Debug` 开着文件日志（`Target=67`） | 关闭 |
| 一个未被引用的孤儿 kext（15.4 MB） | 从仓库剔除 |

---

## 已知限制

- **图形需要根补丁。** nvkms 方案意味着每次系统小版本更新后可能要重打 OCLP-Mod 补丁，否则显示异常。
- **Wi-Fi 需要根补丁。** 同上，`AirportItlwm` 装进 `/Library/Extensions` 后依赖补丁链。
- **`i5-12400F` 无核显**，无法使用 Sidecar / 随航的硬件编码，也无法在无独显时启动 GUI。
- **不保证移植到其他主板可用。** ACPI 中的 `SSDT-RTCAWAC`、`SSDT-SBUS` 是针对这块板子生成的；换主板请重新生成。
- **macOS 大版本升级前请先备份 EFI**，并确认 kext 已跟进。

---

## 版权与归属

- `OpenCore.efi`、`OpenRuntime.efi`、`OpenCanopy.efi`、`ResetNvramEntry.efi`、`HfsPlus.efi` 来自 [Acidanthera OpenCore](https://github.com/acidanthera/OpenCorePkg) 与 [OcBinaryData](https://github.com/acidanthera/OcBinaryData)，遵循 BSD-3-Clause。
- 各第三方 kext 版权归其作者所有，遵循各自的 LICENSE（Lilu / VirtualSMC / WhateverGreen / AppleALC / RestrictEvents / CPUFriend / NVMeFix 为 BSD-3；AirportItlwm / IntelBluetoothFirmware 为 GPL-2.0；USBToolBox 为 MIT）。
- `IOSkywalkFamily.kext`、`IO80211FamilyLegacy.kext`、`AirPortBrcmNIC.kext` 是 **Apple 原始二进制文件**，版权归 Apple Inc. 所有，此处仅作互操作性研究用途。若权利人提出异议，将立即移除这些文件。
- `config.plist` 与文档由本仓库维护者整理，可自由参考。

## 免责声明

修改引导配置有风险，可能导致系统无法启动、数据丢失或硬件异常。使用前请备份重要数据。本仓库按「现状」提供，不附带任何担保，使用后果由使用者自行承担。

## 参考

- [OpenCorePkg 官方文档](https://github.com/acidanthera/OpenCorePkg/blob/master/Docs/Configuration.tex)
- [Dortania OpenCore Install Guide](https://dortania.github.io/OpenCore-Install-Guide/)
- [OpenCore Legacy Patcher](https://github.com/dortania/OpenCore-Legacy-Patcher)
- [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS)
- [USBToolBox](https://github.com/USBToolBox/tool)
