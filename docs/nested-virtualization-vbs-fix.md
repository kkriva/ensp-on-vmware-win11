# 在 VMware 虚拟机里跑通 eNSP：AR 路由器启动失败 / 嵌套虚拟化被 VBS 锁死 完整踩坑记录

> 一台华硕天选 4 锐龙版，从装 VMware 开始一路踩坑，最后发现关不掉 VBS 的真凶是一个所有教程都不提的注册表项。把全过程记录下来，希望你不用再走一遍这些弯路。

## TL;DR（一句话结论）

在 VMware 的 Win10 虚拟机里装 eNSP，AR 路由器启动失败，根源是**主机 Win11 的 VBS（基于虚拟化的安全性）把 CPU 的 AMD-V 独占了，VMware 传不进虚拟机**。

常规方法（关内存完整性、关虚拟机平台、关 hypervisorlaunchtype）全做完 VBS 还在运行的话，去查这个注册表位置：

```
HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios
```

把下面所有子项里 `Enabled = 1` 的改成 `0`——本机上亮着的是 **WindowsHello**（Windows Hello 增强登录安全），它不在任何设置界面里出现，却能独立把 VBS 撑起来。再补一个防复活标记，最后重新执行一遍 `bcdedit /set hypervisorlaunchtype off` 并重启，VBS 变为“未启用”，嵌套虚拟化恢复正常。

## 环境

| 项目 | 内容 |
|---|---|
| 物理机 | 华硕天选 4 锐龙版（AMD Ryzen） |
| 主机系统 | Windows 11 家庭版 24H2（内部版本 26100.4770） |
| 虚拟机软件 | VMware Workstation Pro（已免费，无需许可证） |
| 客户机系统 | Windows 10 64 位，4 核 / 8 GB 内存 / 50 GB 虚拟磁盘 |
| 目标软件 | 华为 eNSP + VirtualBox 5.2.44 + WinPcap 4.1.3 + Wireshark 4.4.6 |

目标结构是三层嵌套，这一点决定了后面所有的坑：

```
物理机 (AMD-V)
└─ VMware 里的 Win10 虚拟机        ← 需要 VMware 把 AMD-V 传进来（嵌套虚拟化）
   └─ eNSP 里的 AR 路由器          ← 本质是 VirtualBox 虚拟机，必须用到 AMD-V
      （交换机不走 VirtualBox，所以交换机能启动、AR 不能，是典型特征）
```

## 症状

1. VMware 里勾上「虚拟机设置 → 处理器 → 虚拟化 Intel VT-x/EPT 或 AMD-V/RVI」后，开机弹窗：

   > Virtualized AMD-V/RVI is not supported on this platform.
   > Continue without virtualized AMD-V/RVI?

2. 点 Yes 也能开机，但虚拟机里没有 AMD-V。eNSP 里拖入 AR 路由器 → 设备启动失败；拖交换机（S5700 等）却能正常启动。

## 原理（四层链条，坏在第二层）

1. **物理 CPU**：锐龙支持 AMD-V，没问题（虚拟机能跑、主机自己的虚拟机监控程序能启动，都证明它可用）。
2. **主机 Windows 的 VBS**：Win11 开机时通过微软虚拟机监控程序独占 AMD-V。这一层不放行，下面全白搭。
3. **VMware**：拿不到 AMD-V，只能退到 Windows Hypervisor Platform 兼容模式（ULM 模式）运行普通虚拟机，这个模式**不支持嵌套虚拟化**，所以出现上面的弹窗。
4. **eNSP**：只是受害者。AR 是 VirtualBox 虚拟机，检查不到 AMD-V 就启动失败。

判断卡点在哪一层，只看一个地方：主机运行 `msinfo32`（Win+R 输入 msinfo32），在系统摘要底部看：

- 「基于虚拟化的安全性」= **正在运行** → VBS 还在占着，就是第 2 层的问题
- 底部有「检测到虚拟机监控程序。将不会显示 Hyper-V 所需的功能」→ 微软的虚拟机监控程序还在跑

## 排查过程：这些常规方法本机全部无效

以下操作本机全部做过并重启验证，VBS 依然“正在运行”，列出来是为了让你先快速排除——如果对你有效，就不需要往下走了：

| 方法 | 本机结果 |
|---|---|
| Windows 安全中心 → 设备安全性 → 内核隔离 → 关闭内存完整性 | ❌ VBS 仍在运行 |
| 关闭 Windows 功能：Hyper-V / 虚拟机平台 / Windows 虚拟机监控程序平台 / 适用于 Linux 的 Windows 子系统 / Windows 沙盒 | ❌ 同上 |
| 管理员终端 `bcdedit /set hypervisorlaunchtype off` | ❌ 同上 |
| 管理员终端 `bcdedit /set vsmlaunchtype off` | ❌ 同上 |
| 注册表 `DeviceGuard\EnableVirtualizationBasedSecurity = 0`、`Scenarios\HypervisorEnforcedCodeIntegrity\Enabled = 0` | ❌ 同上 |

## 弯路警告：DG Readiness Tool + SecConfig 在本机走不通

微软官方的 Device Guard and Credential Guard hardware readiness tool（`DG_Readiness_Tool_v3.6.ps1 -Disable`）是很多指南给出的“终极方案”：它设置一个一次性的 SecConfig.efi 启动项，重启时按 F3 确认解除 Credential Guard / VBS 的 UEFI 锁定。

本机实测结论：**不适用，且会吓你一跳**，细节供参考：

- 工具日志里 `Disabling Hyper-V failed` 不是真失败：原因是 DISM 报错 `0x800f080c 功能名称 Microsoft-Hyper-V-Online 未知`——家庭版根本没有 Hyper-V 这个功能，找不到名字而已，无害。
- 工具日志里删注册表、写启动项（`loadoptions DISABLE-LSA-ISO,DISABLE-VBS`）都显示成功。
- 但重启后没有出现 F3 确认界面，而是 Windows 启动管理器报错：`文件 \EFI\Microsoft\Boot\SecConfig.efi，状态 0xc043000c，所需文件丢失或包含错误`。
  - 事后逐项核对过：`SecConfig.efi` 在 `C:\Windows\System32` 和 EFI 分区的 `\EFI\Microsoft\Boot\` 里都存在（2,564,512 字节）；用 `bcdedit /enum {0cb3b571-2f2e-4343-a879-d86a476d7215}` 检查启动项的 `device`（partition=W:）、`path`、`loadoptions` 也全部正确，手动重设后再次重启仍然加载失败，原因未完全查明（怀疑与本机固件/Secure Boot 策略有关）。
  - **恢复方法很简单，别慌**：在这个报错界面按 `ENTER`（操作系统选择）选回正常的 Windows 11 即可正常开机。一次性启动项（bootsequence）尝试一次后自动失效，不会循环卡住。不要按界面提示去“插入安装盘修复”，更不要重装系统。
- 更关键的是，本机后来查明 `CredGuardEnabled = 0`——**Credential Guard 压根没开**。这个工具治的是 Credential Guard 的锁，而本机的锁根本不在它身上，自然不对症。

## 真正的解决方法：查 DeviceGuard 的 Scenarios

思路转变：不再猜“哪个功能没关”，而是直接枚举 VBS 的所有“场景（Scenarios）”——每个场景都是一个能独立把 VBS 撑起来的消费者，谁的 `Enabled = 1`，谁就在占着 AMD-V。一般教程只查内存完整性那一个场景（HypervisorEnforcedCodeIntegrity），而 Scenarios 下面还有好几个，其中 **WindowsHello** 在任何设置界面里都看不到。

### 第 1 步：只读诊断（不改任何东西）

在**主机**上以管理员身份打开命令提示符（CMD），运行：

```bat
reg query "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios" /s
```

本机的输出关键部分：

```
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\KernelShadowStacks
    Enabled    REG_DWORD    0x0

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\KeyGuard\Status
    IsSecureKernelRunning    REG_DWORD    0x1     ← 安全内核确实在跑
    KeyGuardEnabled          REG_DWORD    0x1
    CredGuardEnabled         REG_DWORD    0x0     ← Credential Guard 没开，DG 工具不对症

HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello
    Enabled    REG_DWORD    0x1                   ← 元凶：Windows Hello 增强登录安全还开着
```

### 第 2 步：关掉亮着的场景 + 防复活 + 最后重设启动项

> ⚠️ 动手前先确认你记得**主机账户密码**。关闭 Windows Hello 场景后，PIN / 指纹 / 人脸登录会失效（它们共用一个被 VBS 密封的容器），重启后要用密码登录，之后可以重新录入 PIN。

仍在主机的管理员 CMD 里逐条运行（每条应显示“操作成功完成”）：

```bat
:: 关掉 Windows Hello 场景（哪个场景 Enabled=1 就关哪个，不止这一项的话同理逐个改 0）
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello" /v Enabled /t REG_DWORD /d 0 /f

:: 防复活标记：不设它，虚拟机监控程序一启动，Windows 可能自动把 VBS 重新打开
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v HyperVVirtualizationBasedSecurityOptout /t REG_DWORD /d 1 /f

:: 必须放在最后再执行一次：增删 Windows 功能会把 hypervisorlaunchtype 悄悄重置回 Auto
bcdedit /set hypervisorlaunchtype off
bcdedit /set vsmlaunchtype off
```

前提条件（前面常规步骤里已完成的要保持）：内存完整性关闭；Hyper-V、虚拟机平台、Windows 虚拟机监控程序平台、WSL 等功能处于关闭状态。

### 第 3 步：改完先验证再重启

```bat
reg query "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello"
bcdedit /enum {current} | findstr hypervisorlaunchtype
```

确认 `Enabled` 变成 `0x0`、`hypervisorlaunchtype` 是 `Off` 后，用开始菜单的**重启**（不要用关机再开机，快速启动可能让设置不完全生效）重启主机。

### 第 4 步：验证结果

重启后在主机运行 `msinfo32`，成功标准是：

- 「基于虚拟化的安全性」= **未启用**
- 「检测到虚拟机监控程序」那一行**消失**，转而显示 Hyper-V 要求四项（虚拟机监视模式扩展、二级地址转换、固件中启用的虚拟化、数据扩展保护）全部为“是”——说明 AMD-V 已经完整释放出来

本机结果（2026-11-03 实测确认）：以上标准全部达成；重启后打开 VMware 启动 Win10 虚拟机，不再弹出 `is not supported on this platform` 的提示；在虚拟机里装完 eNSP 后，AR 路由器成功启动，整条链路全部打通。

## 附带小坑（同一趟里踩过的其他坑，顺手记录）

### 1. 虚拟机开机黑屏 `EFI Network... Time out`

不是系统坏了，是虚拟机没找到 ISO 镜像、跑去网络启动了。关机 → 虚拟机设置 → CD/DVD (SATA) → 勾选 Connected 和 Connect at power on → 重新选择 Win10 的 ISO（`.iso` 结尾、约 5～6 GB；十几 MB 的 `MediaCreationTool.exe` 是下载工具，不是镜像）→ 开机后点进虚拟机窗口连按回车，赶上 `Press any key to boot from CD or DVD`。

### 2. VMware Tools 点了没反应 / 此电脑里没有 Tools 光驱

先看清光驱里的盘叫什么：如果显示的是 `ESD-ISO`，那还是 Win10 安装镜像占着光驱，Tools 自然挂不进来。手动挂载法最稳：

1. 在客户机里右键弹出原来的 DVD 盘
2. 主机上开虚拟机设置 → CD/DVD (SATA) → 勾选 **Connected**（这个“立即连接”最容易漏，只勾 Connect at power on 要等下次开机才生效）和 Connect at power on
3. 选择“使用 ISO 映像文件”，浏览到 **VMware 安装目录**里的 `windows.iso`（不在 C 盘默认路径的人，去你实际安装 VMware 的文件夹里找），打开
4. 回客户机“此电脑”，D 盘变成 VMware Tools 后，运行 `setup64.exe`，选 **Typical（典型）** 安装，完成后重启客户机

### 3. 只做 eNSP，虚拟磁盘给多大

- 微软对较新版本 Win10 的官方最低要求是 32 GB，但那只是“能装上”：系统约 20 GB + 更新和分页约 5～8 GB + eNSP 及依赖（WinPcap、Wireshark、VirtualBox）约 3～5 GB，32 GB 很快红盘。
- **给 50 GB 合适，40 GB 是能用的下限。**
- 关键认知：VMware 里填的数字是**上限**，新建时不要勾“立即分配所有磁盘空间”，虚拟磁盘用多少占多少。填 50 GB，装完系统实际只占 25 GB 左右，不会浪费。
- 快照很吃空间，空间紧张时不要存多个快照。装完系统可在客户机里管理员终端运行 `powercfg /h off` 关休眠，再省几 GB。

### 4. eNSP 本体的安装顺序（缺一不可）

安装包和顺序参考 [yuchen-ya/eNSP-install-win](https://github.com/yuchen-ya/eNSP-install-win)，必须严格按序、全部以管理员身份运行：

1. **WinPcap 4.1.3**：直接装。
2. **Wireshark 4.4.6**：安装中途弹出的 **Npcap 必须取消勾选，千万别装**，Npcap 与 WinPcap 冲突会导致 eNSP 抓包异常。
3. **VirtualBox 5.2.44**：**必须是这个版本**，高版本与 eNSP 不兼容；装完提示升级也不要升。
4. **eNSP**：最后装，它会自动检测前面三个依赖。安装路径用英文（如默认路径或 `C:\eNSP`），不要带中文；防火墙弹窗选允许。

装依赖和 eNSP 之前，建议先在 VMware 里给干净的 Win10 打一个快照（如命名“装eNSP前”），装坏了一键还原，不用重装系统。另外，客户机 Win10 内部也要保持 Hyper-V 相关功能关闭、内存完整性关闭，否则 VirtualBox 5.2 在客户机里同样会报虚拟化不可用。

## 代价与恢复

关闭 VBS / 虚拟机监控程序不是无代价的，动手前请知悉：

- 主机上的 **WSL2、Docker Desktop（WSL2 后端）、Windows 沙盒**在关闭期间无法使用，与嵌套虚拟化只能二选一。
- **Windows Hello 的 PIN / 指纹 / 人脸会失效**，需用密码登录后重新录入。
- 内存完整性、Credential Guard 等安全功能处于关闭状态，实验机可以接受，办公主力机请自行权衡。

大致恢复方法（课程结束后）：把 `Scenarios\WindowsHello\Enabled` 改回 `1`（或删除该值）、删除 `HyperVVirtualizationBasedSecurityOptout` 值、管理员 CMD 执行 `bcdedit /set hypervisorlaunchtype auto`，在 Windows 安全中心重新打开内存完整性、按需重新勾选虚拟机平台等功能，重启后用 `msinfo32` 确认 VBS 回到“正在运行”，并重新录入 PIN。涉及启动配置的修改前，建议先创建一个系统还原点；若 C 盘开了 BitLocker / 设备加密，改 BCD 前先暂停 BitLocker 一个重启周期，避免被要求输恢复密钥。

## 参考与致谢

- eNSP 安装包与顺序：[yuchen-ya/eNSP-install-win](https://github.com/yuchen-ya/eNSP-install-win)
- “该关的都关了、虚拟机监控程序还在跑”的诊断思路（枚举全部 Scenarios、Windows Hello ESS 这个隐藏消费者）：[brhoom98x/vmware-nested-virtualization-windows-11](https://github.com/brhoom98x/vmware-nested-virtualization-windows-11)
- 防复活标记 `HyperVVirtualizationBasedSecurityOptout` 与 Windows Hello 在 24H2 上会自行拉起 VBS 的说明：[AoTofu/wsl2-vbs-off](https://github.com/AoTofu/wsl2-vbs-off)
- 微软官方 DG Readiness Tool 下载页（本机不适用，但对 Credential Guard 被 UEFI 锁定的机器仍是正道）：[Device Guard and Credential Guard hardware readiness tool](https://www.microsoft.com/en-us/download/details.aspx?id=53337)
- 一键脚本类项目（与本文方法同源，可作对照）：[keshabakumar/Disable_HyperV_VBS](https://github.com/keshabakumar/Disable_HyperV_VBS)

## 免责声明

本文是个人在上述特定环境（Win11 家庭版 24H2 + 锐龙平台）的一次实测记录，不是通用教程。不同 Windows 版本、品牌固件、组策略环境下，VBS 的占用者可能不同——先用第 1 步的只读查询定位你机器上亮着的是哪一项，再对症下药。修改注册表与启动配置有风险，操作前请备份重要数据并创建还原点，后果自负。

*记录于 2026-11-03，祝你的 AR 一次启动成功。*
