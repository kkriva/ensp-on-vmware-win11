# 在 VMware 虚拟机中安装 eNSP 完整教程

**Win11 主机 + VMware + Win10 虚拟机 + 华为 eNSP，含 AR 路由器启动失败（嵌套虚拟化 / VBS）全解**

> 本教程是一次真实安装过程的完整记录：从下载 VMware 开始，一步步装好 Win10 虚拟机、VMware Tools 和 eNSP，并最终解决“AR 路由器设备启动失败”这个最折磨人的问题。所有步骤均在文末标注的环境中实测通过，踩过的坑按分类整理在「常见问题」一章，赶时间可以直接跳过去查。

---

## 关于引用与致谢（请先看这一段）

本教程不是凭空写的，它站在几个开源/公开项目的肩膀上。写作时遵循三条原则，也请转载、引用本文的人一起遵守：

1. **注明出处**：每一处借用其他项目的方法、安装包或结论的地方，正文都会就近标注来源，文末「参考项目与致谢」有完整清单。
2. **不二次分发他人的文件**：本文不打包、不镜像任何安装包。eNSP 等安装包请到原项目的 Releases 下载，给原作者一个 Star，是对分享者最实在的感谢。
3. **版权归原作者**：eNSP 版权归华为技术有限公司；VMware Workstation 归 Broadcom；VirtualBox、Wireshark 等各自遵循其开源许可。本文只记录安装与排障过程，不主张对任何软件本身的权利。

特别感谢 [yuchen-ya/eNSP-install-win](https://github.com/yuchen-ya/eNSP-install-win) 的作者整理并公开了 eNSP 安装包与避坑指南——本教程第四部分的安装顺序与注意事项直接来自该项目。

## 目录

- [一、教程概览](#一教程概览)
- [二、环境与版本清单](#二环境与版本清单)
- [三、准备工作](#三准备工作)
- [四、第一部分：安装 VMware Workstation Pro](#四第一部分安装-vmware-workstation-pro)
- [五、第二部分：创建并安装 Win10 虚拟机](#五第二部分创建并安装-win10-虚拟机)
- [六、第三部分：虚拟机装后收尾](#六第三部分虚拟机装后收尾)
- [七、第四部分：安装 eNSP](#七第四部分安装-ensp)
- [八、第五部分：开启嵌套虚拟化（AR 启动失败的根源与解法）](#八第五部分开启嵌套虚拟化ar-启动失败的根源与解法)
- [九、完成验证清单](#九完成验证清单)
- [十、常见问题（分类整理）](#十常见问题分类整理)
- [十一、代价、恢复与安全说明](#十一代价恢复与安全说明)
- [十二、参考项目与致谢](#十二参考项目与致谢)
- [十三、免责声明](#十三免责声明)

## 一、教程概览

### 1.1 最终要搭起来的结构

```
物理机（Win11，AMD-V / Intel VT-x）
└─ VMware Workstation 里的 Win10 虚拟机     ← 嵌套虚拟化：把 CPU 虚拟化能力传进虚拟机
   └─ eNSP
      ├─ 交换机 / PC 等设备                  ← eNSP 自带模拟，不依赖虚拟化
      └─ AR 路由器                           ← 本质是 VirtualBox 虚拟机，必须用到传进来的 AMD-V
```

理解这个三层结构是全篇的钥匙：**交换机能启动、AR 不能启动**，就是典型的“虚拟机本身没问题、嵌套虚拟化没打通”的特征，不要去重装 eNSP。

### 1.2 整体流程图

```
主机准备（BIOS 开 SVM/VT-x）
  → 安装 VMware Workstation Pro
  → 新建 Win10 虚拟机（50 GB / 4 核 / 8 GB）
  → 安装 Win10 系统（选“自定义”）
  → 安装 VMware Tools
  → 主机侧打通嵌套虚拟化（第五部分，本教程的核心难点）
  → 按顺序装 eNSP 四个组件
  → 启动 AR 验证完工
```

### 1.3 适合谁看

- 课程/实验要求用 eNSP，又不想（或不能）在物理机上直接装老版本 VirtualBox 的同学
- 物理机是 Win11 新系统、装 eNSP 后 AR 死活启动不了的人
- 想把 eNSP 关进虚拟机、装坏了能靠快照还原的人

## 二、环境与版本清单

以下为作者实测环境。版本不是随便写的，标 ⚠️ 的是**版本选错必出问题**的地方。

| 层 | 项目 | 版本 / 规格 | 说明 |
|---|---|---|---|
| 物理机 | 华硕天选 4 锐龙版 | AMD Ryzen | Intel 平台同理，把 AMD-V/SVM 换成 VT-x 即可 |
| 主机系统 | Windows 11 家庭版 24H2 | 内部版本 26100 | 第五部分的 VBS 问题主要出现在 Win11 24H2/25H2 |
| 虚拟化软件 | VMware Workstation Pro | 最新版即可 | 现已对个人/商业/教育用途免费，无需许可证 |
| 客户机系统 | Windows 10 64 位 | 22H2 | eNSP 官方年代的操作系统，兼容性最好 |
| 客户机配置 | CPU / 内存 / 磁盘 | 4 核 / 8 GB / 50 GB | 按 16 GB 内存的物理机给出；磁盘理由见问题 B-1 |
| 抓包驱动 | WinPcap | 4.1.3 | eNSP 依赖它，不是 Npcap |
| 协议分析 | Wireshark | 4.4.6 | ⚠️ 安装时**不要**勾选它附带的 Npcap |
| 设备虚拟化 | VirtualBox | ⚠️ **5.2.44，必须是此版本** | 高版本与 eNSP 不兼容，装完不要升级 |
| 模拟器本体 | eNSP | 华为 eNSP 主程序 | 安装包来源见 [yuchen-ya/eNSP-install-win](https://github.com/yuchen-ya/eNSP-install-win) 的 Releases |

## 三、准备工作

开始前备齐这三样，后面会顺很多：

1. **Win10 镜像（ISO）**：到微软官网下载，用媒体创建工具选“为另一台电脑创建安装媒体 → ISO 文件”，得到约 5～6 GB、以 `.iso` 结尾的文件。注意十几 MB 的 `MediaCreationTool.exe` 只是下载工具，不是镜像。
2. **eNSP 安装包（4 个 exe）**：到 [yuchen-ya/eNSP-install-win](https://github.com/yuchen-ya/eNSP-install-win) 的 Releases 页面下载 `WinPcap_4_1_3.exe`、`Wireshark-4.4.6-x64.exe`、`VirtualBox-5.2.44-139111-Win.exe`、`eNSP-Setup.exe`（Source code 压缩包不用下）。请从原仓库下载，不要用来路不明的转载包。
3. **一个干净的存放位置**：虚拟机文件放空间充足的 SSD 分区（如 `D:\VM\Win10`），不要堆在 C 盘。

另外确认物理机 BIOS 里虚拟化已开启：任务管理器 → 性能 → CPU，若“虚拟化”显示已启用即可；未启用则进 BIOS（华硕开机连按 F2，F7 进高级模式）到 Advanced → CPU Configuration，把 **SVM Mode**（Intel 平台为 Intel Virtualization Technology）设为 Enabled，F10 保存。

## 四、第一部分：安装 VMware Workstation Pro

1. 打开 Broadcom Support Portal，注册免费基础账号并登录（不需要企业邮箱或付费订阅）。
2. 进入 My Dashboard → My Downloads → Free Software Downloads，搜索 **VMware Workstation Pro**，选择 **for Windows** 的最新版。
3. 先点开 **Terms and Conditions** 链接看完，再勾选同意——不点开链接的话，同意框勾不上，下载按钮不会亮，这是最多人卡住的一步。
4. 下载完成后，右键安装包 → 以管理员身份运行 → 一路 Next。安装路径可改到其他盘；建议勾选 Enhanced Keyboard Driver，其余保持默认。
5. 安装完成后**重启物理机**再继续。首次打开不需要输入许可证密钥。

## 五、第二部分：创建并安装 Win10 虚拟机

### 5.1 新建虚拟机

打开 VMware → 创建新的虚拟机 → 选“典型”，然后：

| 步骤 | 选什么 |
|---|---|
| 安装来源 | 安装程序光盘映像文件 (ISO)，选准备好的 Win10 ISO |
| 客户机系统 | Microsoft Windows → Windows 10 x64 |
| 名称与位置 | 如 `Win10`，位置选空间大的分区，如 `D:\VM\Win10` |
| 磁盘大小 | **50 GB**，选“将虚拟磁盘存储为单个文件”，**不要勾**“立即分配所有磁盘空间” |
| 完成前 | 先点“自定义硬件”，按下表改完再完成 |

自定义硬件参数（物理机 16 GB 内存时）：

| 硬件 | 设置 | 备注 |
|---|---|---|
| 内存 | 8 GB | 物理机 32 GB 可给 12 GB；留一半给主机 |
| 处理器 | 总数 4（1 个处理器 × 4 核） | 同时勾选“虚拟化 AMD-V/RVI”是第五部分的事，也可现在先勾上 |
| 网络适配器 | NAT | 默认即可上网 |
| 显示器 | 勾选“加速 3D 图形”，显存 1～2 GB | 不用拉满 |
| CD/DVD | 保持挂载 Win10 ISO，勾 Connected 与 Connect at power on | 漏勾会遇到问题 A-1 |

### 5.2 安装 Win10 系统

开启虚拟机，出现 VMware 标志后点进窗口连按回车，赶上 `Press any key to boot from CD or DVD` 进入安装界面：

1. 语言、键盘保持默认 → 现在安装
2. 密钥页选“我没有产品密钥”（VMware 不附带 Windows 授权，激活需自备 Win10 密钥，版本要和密钥对应）
3. 安装类型选 **“自定义：仅安装 Windows（高级）”**——新建的空白虚拟机不要选“升级”，选了会报错
4. 磁盘选“磁盘 0 未分配的空间”→ 下一步，不用手动新建或格式化，安装程序会自动分区
5. 等待自动重启数次，完成初始设置（用户名、密码、隐私选项）进入桌面

## 六、第三部分：虚拟机装后收尾

### 6.1 安装 VMware Tools（必做，先做它）

作用：分辨率自适应、鼠标不卡、主机与虚拟机之间可拖拽文件/复制粘贴。没装好之前先别往下走。

1. VMware 顶部菜单 `虚拟机 (VM)` → `安装 VMware Tools`
2. 在客户机里打开“此电脑”，双击多出来的 VMware Tools 光驱，运行 **`setup64.exe`**（64 位系统不要运行 setup.exe）
3. 安装类型选 **Typical（典型）**，完成后按提示重启虚拟机

> 点了菜单没反应、或者光驱里还是 Win10 安装盘（显示 ESD-ISO）怎么办？看问题 A-2 的手动挂载 `windows.iso` 方法。

### 6.2 其他收尾

- **联网检查**：客户机浏览器能打开网页即可（NAT 默认可上网）。
- **Windows 更新**：设置 → 更新和安全 → 检查更新，跑一遍并重启。50 GB 磁盘下更新占几个 GB 属于正常。
- **客户机内部也要“干净”**：客户机 Win10 里保持 Hyper-V 相关功能关闭、内存完整性关闭（Windows 安全中心 → 设备安全性 → 内核隔离）。否则 VirtualBox 5.2 在客户机内部同样会报虚拟化不可用，这一点和主机侧的问题是同一个原理、不同楼层。
- **打快照**：系统干净、Tools 装好后，关机或开机状态下 `VM → 快照 → 拍摄快照`，命名如“装eNSP前”。快照很占空间，留 1 个即可。

## 七、第四部分：安装 eNSP

> 本部分的安装顺序与注意事项来自 [yuchen-ya/eNSP-install-win](https://github.com/yuchen-ya/eNSP-install-win)，该项目还打包了全部安装文件，请到其 Releases 下载，并给作者一个 Star。

**总原则：严格按顺序、全部右键“以管理员身份运行”、装完一个再装下一个。**

```
① WinPcap 4.1.3  →  ② Wireshark 4.4.6  →  ③ VirtualBox 5.2.44  →  ④ eNSP
```

### 7.1 WinPcap 4.1.3

运行 `WinPcap_4_1_3.exe`，一路下一步，无特殊设置。

### 7.2 Wireshark 4.4.6 ⚠️ 全篇最容易翻车的一步

运行 `Wireshark-4.4.6-x64.exe` 正常安装，但安装过程中会弹出/询问是否安装 **Npcap**：

- **必须取消勾选，不要安装 Npcap**
- 原因（原项目说明）：Npcap 与 WinPcap 冲突，会导致 eNSP 抓包功能异常（表现为能开拓扑、抓不到包）
- 其余选项保持默认

### 7.3 VirtualBox 5.2.44 ⚠️ 版本必须对

运行 `VirtualBox-5.2.44-139111-Win.exe`：

- **必须是 5.2.44 这一个版本**，更高版本与 eNSP 不兼容（原项目明确要求）
- 安装时提示安装网络/驱动组件、短暂断网，点同意即可
- 装完**不要打开它，也不要接受它的升级提示**，以后弹窗让升级一律忽略

### 7.4 eNSP 主程序

运行 `eNSP-Setup.exe`：

- 它会**自动检测**前面三个依赖是否就位，所以必须放在最后装；检测不过先回头补，不要强行继续
- 安装路径保持默认或使用纯英文路径（如 `C:\eNSP`），不要带中文和空格，这个年代的软件对中文路径容易出问题
- 安装/首次运行时防火墙弹窗选“允许访问”

### 7.5 首次运行与验证

右键 eNSP 图标 → 以管理员身份运行 → 新建拓扑：

- 先拖一台**交换机**（如 S5700）启动，能打开命令行 → 说明本体安装成功
- 再拖一台 **AR 路由器**（如 AR1220/AR2220）启动——这一步才是真正的分水岭：
  - AR 能启动并打开命令行 → 全部完工，去看第九部分清单
  - AR 启动失败、交换机却正常 → 不要重装 eNSP，直接去第八部分，那是嵌套虚拟化的问题

## 八、第五部分：开启嵌套虚拟化（AR 启动失败的根源与解法）

### 8.1 先理解问题出在哪一层

```
物理 CPU（AMD-V）✓ 有
  └ 主机 Win11 的 VBS 独占了 AMD-V        ← 十有八九卡在这里
      └ VMware 只能用兼容模式，传不进 AMD-V
          └ 客户机里的 VirtualBox 拿不到 AMD-V
              └ AR 启动失败（交换机不走这条路，所以没事）
```

定位方法（主机上）：`Win + R` 输入 `msinfo32`，看系统摘要底部：

| 看到什么 | 结论 |
|---|---|
| 基于虚拟化的安全性 = 正在运行 | VBS 还在占着 AMD-V，按 8.3 / 8.4 处理 |
| 有一行“检测到虚拟机监控程序。将不会显示 Hyper-V 所需的功能” | 微软虚拟机监控程序还在跑，同上 |
| 基于虚拟化的安全性 = 未启用，且 Hyper-V 四项要求全是“是” | 这一层已通，去客户机侧排查（6.2、问题 D-1） |

### 8.2 VMware 侧的开关（必要条件）

虚拟机**关机**状态下：`虚拟机设置 → 处理器` → 勾选 **“虚拟化 Intel VT-x/EPT 或 AMD-V/RVI”**。

勾上后开机若弹窗 `Virtualized AMD-V/RVI is not supported on this platform`，说明主机侧还没放行（即 8.1 的第二层），弹窗点 **No**，先做完 8.3 再回来。

### 8.3 主机侧常规步骤（多数机器做到这一步就好）

全部在**主机（物理机）**上操作：

1. Windows 安全中心 → 设备安全性 → 内核隔离详细信息 → **内存完整性：关**
2. `Win + R` 输入 `optionalfeatures`，**取消勾选**：Hyper-V、虚拟机平台、Windows 虚拟机监控程序平台、适用于 Linux 的 Windows 子系统、Windows 沙盒（有哪项关哪项，家庭版没有 Hyper-V 属正常），点确定
3. 管理员命令提示符（CMD）运行：

```bat
bcdedit /set hypervisorlaunchtype off
bcdedit /set vsmlaunchtype off
```

4. 用开始菜单的**重启**重启主机（不要关机再开机，快速启动可能让设置不完全生效）
5. 回到 8.1 的定位方法复查 msinfo32

### 8.4 若常规步骤无效：查 DeviceGuard 的 Scenarios（作者实测的真凶）

如果 8.3 全做完、重启后 msinfo32 仍显示 VBS“正在运行”，不要再重复改相同的开关。VBS 由一组“场景（Scenarios）”各自独立撑起，一般教程只提内存完整性那一个，而 Scenarios 下面藏着界面里看不到的消费者，其中 **Windows Hello 增强登录安全**是作者机器上的真凶——它和这套诊断/修复思路来自 GitHub 项目 [brhoom98x/vmware-nested-virtualization-windows-11](https://github.com/brhoom98x/vmware-nested-virtualization-windows-11)（该仓库专门写“该关的都关了、虚拟机监控程序还在跑”的情况），防复活标记来自 [AoTofu/wsl2-vbs-off](https://github.com/AoTofu/wsl2-vbs-off)，特此致谢，详细出处见第十二部分。

**（1）只读诊断（不改任何东西）**，主机管理员 CMD 运行：

```bat
reg query "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios" /s
```

看每个子项的 `Enabled` 值，谁是 `0x1` 谁就在撑着 VBS。作者机器的结果：`WindowsHello → 0x1`（元凶），其余如 KernelShadowStacks 为 `0x0`；`KeyGuard\Status` 里 `IsSecureKernelRunning = 0x1` 证实安全内核在跑，而 `CredGuardEnabled = 0x0` 说明 Credential Guard 根本没开。

**（2）对症关闭**（动手前确认记得**主机账户密码**：关闭 Windows Hello 场景会使 PIN/指纹/人脸失效，需用密码登录后重新录入），主机管理员 CMD 逐条运行：

```bat
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello" /v Enabled /t REG_DWORD /d 0 /f
reg add "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard" /v HyperVVirtualizationBasedSecurityOptout /t REG_DWORD /d 1 /f
bcdedit /set hypervisorlaunchtype off
bcdedit /set vsmlaunchtype off
```

说明：

- 第 1 行关掉亮着的场景；诊断里若有其他场景也是 `0x1`，用同样格式逐个改 `0`——原则是 Scenarios 下所有 `Enabled` 都应为 `0`
- 第 2 行是**防复活标记**（来自 wsl2-vbs-off 项目的说明）：不设它，虚拟机监控程序一启动，Windows 可能自动重新打开 VBS
- 第 3、4 行**必须放在最后再执行一遍**：增删 Windows 功能会把 `hypervisorlaunchtype` 悄悄重置回 Auto，这是很多“明明设过却没生效”的原因

**（3）改完先验证再重启**：

```bat
reg query "HKLM\SYSTEM\CurrentControlSet\Control\DeviceGuard\Scenarios\WindowsHello"
bcdedit /enum {current} | findstr hypervisorlaunchtype
```

确认 `Enabled = 0x0`、`hypervisorlaunchtype = Off` 后重启主机，复查 msinfo32 达到 8.1 表格的第三行状态，即为打通。

### 8.5 关于 DG Readiness Tool 的重要提醒

很多指南会推荐微软官方工具 DG Readiness Tool（`DG_Readiness_Tool_v3.6.ps1 -Disable`，靠重启时 SecConfig.efi 让你按 F3 解除 Credential Guard/VBS 锁定）。作者实测提醒：

- 它针对的是 **Credential Guard 被 UEFI 锁定**的情况；先用 8.4 的诊断确认 `CredGuardEnabled` 是否为 1，不是这个病就不要吃这个药（作者机器为 0，走这条路纯属绕远）。
- 作者机器上它的 SecConfig 启动项两次加载失败（启动管理器报错 `0xc043000c`），未出现 F3 界面。遇到这个报错**不要重装系统**：在报错界面按 `ENTER` 选回正常 Windows 即可开机，一次性启动项尝试一次后会自动失效。
- 它日志里的 `Disabling Hyper-V failed` 在家庭版上多半是 DISM 报错 `0x800f080c`（家庭版没有 Hyper-V 功能名），属无害误报，不要被它误导。

> 本节只保留结论与避坑要点。这段弯路的完整记录（工具日志逐条解读、SecConfig.efi 文件与启动项参数的逐项核对）单独成文，需要深挖的读者请看专题文档：[VBS 关不掉的完整排查记录](docs/nested-virtualization-vbs-fix.md)。

## 九、完成验证清单

逐项打勾，全过才是真的完工：

- [ ] 主机 `msinfo32`：基于虚拟化的安全性 = **未启用**，Hyper-V 四项要求均为“是”
- [ ] VMware 启动 Win10 虚拟机：**不再弹** `Virtualized AMD-V/RVI is not supported` 的提示
- [ ] 客户机内 VMware Tools 正常（分辨率可自适应、能拖文件进去）
- [ ] eNSP（管理员运行）新建拓扑，交换机可启动并打开命令行
- [ ] **AR 路由器可启动并打开命令行** ← 最终标准
- [ ] （可选）Wireshark 能抓到接口流量，验证 WinPcap 与 Npcap 没有装错

## 十、常见问题（分类整理）

按出现的阶段分四类：A 安装阶段 / B 资源与性能 / C 嵌套虚拟化与 VBS / D eNSP 运行。每条给「症状 → 原因 → 解法」。

### A 类：安装阶段

**A-1 虚拟机开机黑屏，反复出现 `Network media not present` / `EFI Network ... Time out`**

- 原因：虚拟机没找到 ISO，退到网络启动。常见于没挂上镜像，或光驱的 Connected 没勾。
- 解法：虚拟机关机 → 设置 → CD/DVD (SATA) → 勾选 Connected 与 Connect at power on → 确认挂的是 5～6 GB 的 Win10 ISO（不是 MediaCreationTool.exe）→ 开机后点进窗口连按回车。

**A-2 点了“安装 VMware Tools”没反应，光驱里还是 `ESD-ISO`**

- 原因：Win10 安装镜像还占着虚拟光驱，Tools 的镜像挂不进来；或只勾了 Connect at power on、没勾 Connected。
- 解法（手动挂载，实测最稳）：
  1. 客户机里右键弹出原 DVD 盘
  2. 主机上打开虚拟机设置 → CD/DVD → 勾选 **Connected** 和 Connect at power on → “使用 ISO 映像文件”浏览到 **VMware 安装目录**里的 `windows.iso`（默认在 VMware 安装文件夹内，非默认安装路径去你安装 VMware 的文件夹找）
  3. 回客户机“此电脑”，双击变成 VMware Tools 的盘，运行 `setup64.exe`，选 Typical，装完重启虚拟机

**A-3 Win10 安装时选“升级”后报错**

- 原因：空白新虚拟机没有旧系统可升。
- 解法：回到安装类型页选“自定义：仅安装 Windows（高级）”，磁盘页选“磁盘 0 未分配的空间”直接下一步。

**A-4 VMware 下载页同意框勾不上、下载按钮不亮**

- 原因：Broadcom 门户要求先点开 Terms and Conditions 链接浏览条款。
- 解法：先点链接再勾选同意，下载按钮随即可用。全程免费，无需信用卡和许可证。

### B 类：资源与性能

**B-1 虚拟磁盘给多大才不浪费、又不红盘？**

- 微软对较新 Win10 的官方最低要求是 32 GB，但那只是“能装上”：系统约 20 GB + 更新与分页 5～8 GB + eNSP 全套约 3～5 GB，32 GB 很快告急。
- 结论：**50 GB 合适，40 GB 是下限。** VMware 里填的是上限，创建时不勾“立即分配所有磁盘空间”，用多少占多少，填 50 GB 实际只占二十多 GB，不存在浪费。

**B-2 客户机 C 盘莫名少几个 GB**

- 原因：Windows 更新缓存与休眠文件。
- 解法：管理员终端运行 `powercfg /h off` 关闭休眠可省数 GB；快照只保留 1 个，用不上的及时删除。

**B-3 主机内存不够，虚拟机很卡**

- 原因：主机 16 GB 时虚拟机给 8 GB 是平衡点；同时开浏览器多标签会抢内存。
- 解法：做实验时关掉主机上占内存的大户；物理机本身 8 GB 内存的，建议虚拟机只给 4 GB 且只跑 eNSP。

### C 类：嵌套虚拟化与 VBS

**C-1 开机弹窗 `Virtualized AMD-V/RVI is not supported on this platform`**

- 原因：主机 Win11 的 VBS 独占 AMD-V，VMware 退到兼容模式，无法开启嵌套。四层链条见 8.1。
- 解法：8.2 勾选开关 → 8.3 常规步骤 → 仍不行走 8.4 查 Scenarios。不要只勾开关就反复开机试。

**C-2 内存完整性、功能、bcdedit 全关了，msinfo32 还显示 VBS“正在运行”**

- 原因：还有未被关闭的 DeviceGuard 场景在请求 VBS，作者案例是界面里看不见的 Windows Hello（`Scenarios\WindowsHello\Enabled = 1`）；也可能是 24H2 以后 Windows Hello 的 VBS 保护自行把 VBS 拉起。
- 解法：按 8.4 的只读查询逐项排查，`Enabled = 1` 的逐个改 0，并补上防复活标记，最后重跑一遍 bcdedit 再重启验证。

**C-3 用了 DG Readiness Tool，重启出现启动管理器报错 `0xc043000c`，没有 F3 界面**

- 原因：SecConfig.efi 一次性启动项加载失败（作者机器复现两次，文件本身与启动项参数核对无误，原因疑似与固件策略有关）。
- 解法：在报错界面按 **ENTER** 选回正常 Windows 开机即可；一次性启动项（bootsequence）用过即失效，不会循环。之后改走 8.4 路线。切记不要因此重装系统或“修复安装”。

**C-4 DG 工具日志里报 `Disabling Hyper-V failed` / DISM 错误 `0x800f080c`**

- 原因：家庭版没有 `Microsoft-Hyper-V-Online` 这个功能名，DISM 找不到就报错。
- 解法：忽略即可，它不是 VBS 关不掉的原因；先确认自己是不是 Credential Guard 的问题（看 `CredGuardEnabled`）再决定用不用这个工具。

**C-5 主机 PIN / 指纹突然不能登录**

- 原因：8.4 关闭了 Windows Hello 场景，Hello 凭据原本由 VBS 密封，容器拆除后需重建。
- 解法：用**账户密码**登录（所以动手前务必确认记得密码），再到 设置 → 账户 → 登录选项 重新录入 PIN/指纹。

### D 类：eNSP 运行

**D-1 AR 路由器“设备启动失败”，交换机却正常**

- 定位顺序（从外到内逐层查，不要一上来重装）：
  1. 主机 msinfo32 是否达到第九部分的标准（VBS 未启用）→ 没达到回第八部分
  2. VMware 处理器设置里 AMD-V/RVI 是否勾上、开机是否弹过不支持的窗 → 弹过说明主机层没通
  3. 客户机内 Hyper-V 功能/内存完整性是否开着 → 开着就关掉（6.2）
  4. VirtualBox 是不是 5.2.44 → 版本不对，卸掉重装为 5.2.44
  5. eNSP 是否以管理员身份运行、路径是否含中文 → 改之
- 仍不行时看报错代码（如 40/41 多为 VirtualBox 组件问题），按代码单独查，不要盲目全量重装系统。

**D-2 拓扑能跑但抓不到包 / 抓包为空**

- 原因：多半是 7.2 里误装了 Npcap，与 WinPcap 冲突。
- 解法：卸载 Npcap（及 Wireshark），确认只保留 WinPcap 4.1.3 后重装 Wireshark 且这次不勾 Npcap。

**D-3 设备启动时 VirtualBox 报错（如 VT-x/AMD-V 被其他软件占用类提示）**

- 原因：客户机内部有东西在占虚拟化，常见为客户机的 Hyper-V 功能、内存完整性，或杀毒软件的硬件虚拟化防护。
- 解法：按 6.2 把客户机内部该关的关掉并重启客户机；仍不行时在 eNSP 里右键设备“检查/清理”后重新注册设备再启动。

## 十一、代价、恢复与安全说明

第八部分的主机改动不是无代价的，动手前与完工后都请知悉：

- **二选一关系**：主机关闭虚拟机监控程序期间，主机上的 WSL2、Docker Desktop（WSL2 后端）、Windows 沙盒无法使用；需要它们时按下面恢复，与嵌套虚拟化轮流用。
- **安全功能关闭中**：VBS、内存完整性、Credential Guard 在关闭期间不提供保护。实验电脑可接受，公司/办公主力机请谨慎评估。
- **Windows Hello**：PIN/指纹会失效（问题 C-5），用密码登录后重新录入。
- **大致恢复方法**：课程结束后，把 `Scenarios\WindowsHello\Enabled` 改回 `1`（或删除该值）、删除 `HyperVVirtualizationBasedSecurityOptout` 值、管理员 CMD 执行 `bcdedit /set hypervisorlaunchtype auto`、在 Windows 安全中心重新打开内存完整性、`optionalfeatures` 里按需勾回虚拟机平台等功能，重启后用 msinfo32 确认 VBS 回到“正在运行”。
- **动手前先备份**：修改注册表与 BCD 前建议创建系统还原点；若 C 盘启用了 BitLocker/设备加密，先暂停 BitLocker 一个重启周期，避免开机被要求输入恢复密钥。

## 十二、参考项目与致谢

本教程的方法、安装包与结论分别来自以下项目与官方来源，分四类列出。**每个项目只引用了其公开说明中的思路或其发布的安装文件，未复制其仓库内容**；受益于人，理当致谢，也请读者顺手给他们 Star。

### A. 安装包与安装顺序

- **[yuchen-ya/eNSP-install-win](https://github.com/yuchen-ya/eNSP-install-win)**（作者：yuchen-ya）——本教程第四部分的主要来源：提供了 WinPcap 4.1.3、Wireshark 4.4.6、VirtualBox 5.2.44 与 eNSP 的完整安装包及安装顺序，并明确了三条关键经验：顺序不可乱、Wireshark 不要装 Npcap、VirtualBox 必须是 5.2.44 且虚拟机中需开嵌套虚拟化。本文的 eNSP 安装步骤均按其指南执行，特此首要致谢。

### B. 问题诊断思路

- **[brhoom98x/vmware-nested-virtualization-windows-11](https://github.com/brhoom98x/vmware-nested-virtualization-windows-11)**（作者：brhoom98x）——为“常规方法全部做完、虚拟机监控程序仍在运行”这一疑难场景提供了关键诊断法：枚举 `DeviceGuard\Scenarios` 全部子项逐项排查，并指出 Windows Hello 增强登录安全是界面不可见、却能单独维持 VBS 的隐藏消费者。第 8.4 节的定位方法源于此，其只读诊断脚本（无需提权、不改系统）的设计也值得学习。

### C. 系统设置细节

- **[AoTofu/wsl2-vbs-off](https://github.com/AoTofu/wsl2-vbs-off)**（作者：AoTofu）——提供了两个关键细节：防复活标记 `HyperVVirtualizationBasedSecurityOptout = 1`（“不设它，虚拟机监控程序启动时 Windows 会自动重新打开 VBS”），以及“24H2 起 Windows Hello 的 VBS 保护可自行拉起 VBS”的版本背景说明。该项目以 Unlicense 发布，其 README 的对照表与“What it changes”写法也启发了本文的组织方式。
- **[keshabakumar/Disable_HyperV_VBS](https://github.com/keshabakumar/Disable_HyperV_VBS)**（作者：keshabakumar）——常规关闭 Hyper-V/VBS 的一键脚本项目，与本文 8.3 的方法同源，可作交叉对照。
- **[AbelardoCarcamo/win11-nested-virtualization-fix](https://github.com/AbelardoCarcamo/win11-nested-virtualization-fix)**——按 Win11 24H2/25H2 编写的嵌套虚拟化修复手册（DISM 关闭功能 + bcdedit + DeviceGuard 注册表），结构完整，可作为英文对照资料。

### D. 官方工具与软件

- **[Microsoft Device Guard and Credential Guard hardware readiness tool](https://www.microsoft.com/en-us/download/details.aspx?id=53337)**（微软官方）——Credential Guard/VBS 被 UEFI 锁定时的官方解除工具；本文 8.5 与问题 C-3、C-4 记录了它在“非 Credential Guard 问题”机器上的不适用情形与日志误读，供后人避坑，这无损于它在对症场景下的价值。
- 软件本体：eNSP © 华为技术有限公司；VMware Workstation © Broadcom；VirtualBox © Oracle（GPL）；Wireshark © Wireshark 基金会（GPL）；WinPcap © Riverbed。安装与使用请遵循各自许可与条款。

如有遗漏的出处或引用有误，欢迎在 Issues 中指出，会第一时间补正——尊重原作者也是这个仓库的规矩。

## 十三、免责声明

本文是作者在特定环境（Win11 家庭版 24H2 + AMD 锐龙平台 + 上述软件版本）下的个人实测记录，不构成对任何组织/个人的技术承诺。不同 Windows 版本、OEM 固件与组策略环境下，VBS 的占用者与最优解法可能不同：请先诊断、再动手（第 8.4 节的只读查询不会改动系统）。修改注册表、启动配置与安全功能存在风险，请提前备份数据、创建还原点，并自行承担操作后果。

## 实测环境（Tested on）

- 华硕天选 4 锐龙版，Windows 11 家庭版 24H2（26100）
- VMware Workstation Pro + Windows 10 22H2 客户机（4 核 / 8 GB / 50 GB）
- eNSP + VirtualBox 5.2.44：交换机与 AR 路由器均启动成功（2026-11-03 实测）

*最后更新：2026-11-03。祝一次装通，AR 一把启动成功。*
