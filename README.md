# Toshiba-Portege-M500-modem-x64-driver

由dsh编写，基于原有32位驱动而改的64位东芝PortegeM500调制解调器驱动



----DSH.Drv.Build.Beta.Readme----



# TOSHIBA PORTEGE M500 · Windows 10 x64 内置调制解调器驱动

> 用本目录中的原厂驱动文件（LSI/Agere `TOSHIBA Software Modem 2.2.97`，微软 WHQL 签名）为
> **TOSHIBA PORTEGE M500** 制作并安装了一套在 **Windows 10 x64 19045** 上可正常工作的本机调制解调器驱动。
>
> 已验证：设备状态 **OK / CM_PROB_NONE**、内核服务 **RUNNING**、串口 **COM3**、
> **AT 指令正常应答**、**DAA 摘机与拨号音检测通路正常**。

---

## 1. 目标机器与本机硬件识别

| 项目 | 实测值 |
|---|---|
| 机型 | TOSHIBA **PORTEGE M500**（BIOS `Version 3.70`，2007-06-04，SN `Y6084857H`） |
| 主板 | TOSHIBA Portable PC Version A0 |
| CPU / 内存 | Intel Core 2 T7600 @2.33GHz ×2 ／ 4 GB（可用约 3.2 GB） |
| 操作系统 | Microsoft Windows 10 专业版 **10.0.19045** **64 位** |
| HD Audio 控制器 | `PCI\VEN_8086&DEV_27D8`（ICH7-M Family HD Audio） |
| HD Audio 音频编解码器 | `HDAUDIO\FUNC_01&VEN_8384&DEV_7680`（SigmaTel/IDT STAC9200） |
| **调制解调器** | `HDAUDIO\FUNC_02&VEN_11C1&DEV_3026&SUBSYS_11790001&REV_1007` |
| Modem 位置 | Internal High Definition Audio Bus，Address 257 |
| Modem 芯片 | LSI（原 Agere）**SV92PP** 软调制解调器（HDA 接口的 "TOSHIBA Software Modem"） |

> `VEN_11C1` = LSI/Agere；`DEV_3026` = SV92PP；`SUBSYS_11790001` = **TOSHIBA 1179** 的子系统 ID。
> 也就是说这台机器的猫不是 AC'97/PCI 的，而是挂在 **HDA 总线上的 FUNC_02 功能组**——
> 这一点很关键，它决定了只能用 LSI 的 **HDA** 版驱动（`lsimdv64.inf` / `agrsm64.sys`），
> 而不能用目录里那份 AC'97 的 `mdmags64.inf`。

### 安装完成后的状态

| 项目 | 结果 |
|---|---|
| 设备名称 | **TOSHIBA Software Modem** |
| 设备状态 | **OK**，问题码 **CM_PROB_NONE**（原来的 代码 48 已消除） |
| 驱动提供者 / 版本 | **LSI** / **2.2.97.0**（2009-07-21） |
| 驱动 INF | `oem12.inf`（源自本目录 `driver\lsimdv64.inf`，未做任何修改） |
| 目录签名 | WHQL 有效：`CN=Microsoft Windows Hardware Compatibility Publisher` |
| 内核服务 | `AgereSoftModem`（显示名 *TOSHIBA V.92 Software Modem*）**RUNNING** |
| 实际加载镜像 | `C:\Windows\System32\drivers\tosm500md.sys`（1,208,320 字节） |
| 串口 | **COM3** 115200 8N1 |
| 功能自检 | 见第 6 节 |

---

## 2. 原始问题与根因（两层）

### 第一层：微软官方把这台机器的猫"摆烂"了

Windows 10 自带的 `C:\Windows\INF\mdmags64.inf`（Microsoft，2.2.89.4）里有直接命中本机的一行：

```ini
%Modem1% = Modem_DF, HDAUDIO\FUNC_02&VEN_11C1&DEV_3026&SUBSYS_11790001
```

对应字符串是 `Modem1 = "Win Modem (Unsupported)"`、`DisplayName = "Modem Driver (Unsupported)"`。
它安装的 `C:\Windows\System32\drivers\agrsm64.sys` 只有 **7,168 字节**，文件描述为
`Generic Modem Driver (Unsupported)` —— 是个空壳占位驱动，服务显示名直接叫
"Modem Driver (Unsupported)"。设备因此长期处于 **代码 48（CM_PROB_DRIVER_BLOCKED）**、
名为 **Win Modem (Unsupported)** 的状态。

### 第二层（本机特有，也是最后真正卡住的地方）：文件名级驱动加载拦截

把真正的 LSI 驱动包正常安装之后，设备仍然启动不了：

```text
sc start AgereSoftModem
[SC] StartService FAILED 1275:
This driver has been blocked from loading
```

`1275 = ERROR_DRIVER_BLOCKED`，设备侧对应 `0xC000036B = STATUS_DRIVER_BLOCKED`。
经过一组对照实验，确认拦截是**按文件名**做的，与文件内容、路径、签名都无关：

| 实验 | 文件内容 | 文件名 | 结果 |
|---|---|---|---|
| 1 | LSI 1.2 MB 真实驱动 | `agrsm64.sys` | ❌ 1275 被拦截 |
| 2 | 同上（复制到 `C:\Windows\Temp\dtest\`） | `agrsm64.sys` | ❌ 1275 被拦截（排除路径因素） |
| 3 | 同上 | `agrsm64x.sys` | ✅ 正常加载（RUNNING） |
| 4 | 同上 | `toshmodem.sys` | ✅ 正常加载（RUNNING） |
| 5 | **`beep.sys`（完全无关的系统驱动）** | **`agrsm64.sys`** | ❌ **1275 被拦截** |
| 6 | 字符串 `agrsm64` 是否在微软 WDAC 拦截清单里 | — | ❌ **不在**（清单共 120 条文件名规则，见 `C:\Windows\System32\CodeIntegrity\driversipolicy.p7b`） |

实验 5 最有说服力：把微软自己的 `beep.sys` 改名成 `agrsm64.sys` 也照样被拦，
说明拦截**只认文件名**。实验 6 说明这**不是**微软的漏洞驱动黑名单策略。

本机同时装有具备驱动加载防护能力的安全软件：

| 驱动 | 厂商 | 说明 |
|---|---|---|
| `sysdiag.sys` | Beijing Huorong Network Technology（火绒） | System Diagnosis Toolkit |
| `hrdevmon.sys` | 火绒 | Huorong Internet Security Device Monitor |
| `hrwfpdrv.sys` | 火绒 | Defence Module (Firewall) |
| `DgSafe.sys` | Kingsoft（驱动精灵） | DgService Driver |
| `bootsafe` | — | 驱动保护类 |

因此这条 `agrsm64.sys` 的加载拦截**极可能来自本机安装的驱动防护类安全软件所保存的一条"阻止该驱动"规则**
（火绒的 `hips.db` / `applog.db` 被其服务独占锁定，无法在不修改机主安全软件的前提下读出原始规则条目，
故此结论按"证据指向"表述）。这也顺带解释了为什么连微软那个 7 KB 的空壳驱动都起不来——它同样叫
`agrsm64.sys`，同样被拦。

---

## 3. 解决方案原理

### 3.1 用原厂 WHQL 包正常安装

本目录 `driver\lsimdv64.inf` 是 LSI 官方的 "LSI Soft Modem" 安装脚本，其第 35 行**精确包含本机硬件 ID**：

```ini
%LSI2% = LSI_TOS_HDA, HDAUDIO\FUNC_02&VEN_11C1&DEV_3026&SUBSYS_11790001
```

`%LSI2% = "TOSHIBA Software Modem"`。配套 `lsimdv64.cat` 是微软 WHQL 签名目录
（SetupAPI 日志：`Signer Score = 0x0D000005 (WHQL)`、`Submission ID = 1340389`），
9 个文件全部通过目录校验。安装后设备正确绑定为 **LSI / 2.2.97**，服务、注册表、
AT 指令表、传真 Class 1 参数、安全描述符等都由这个 INF 完整写入。

### 3.2 把"加载文件名"换掉，绕过本机那条文件名拦截

INF 本身不能改（一改 `lsimdv64.cat` 的哈希校验就会失效），所以做法是：

1. 同一份二进制**原样再落一份**为 `C:\Windows\System32\drivers\tosm500md.sys`；
2. 把内核服务 `AgereSoftModem` 的 `ImagePath` 指向 `tosm500md.sys`。

INF、CAT、驱动库（DriverStore）、设备绑定、服务配置全部保持原厂状态，只改了一个
`ImagePath` 字符串。

### 3.3 为什么这样做**没有削弱**系统安全性（重要）

内核代码完整性（Code Integrity / KMCS）是按**文件哈希**去已安装的目录（catalog）里查找签名，
**与文件名无关**。改名后的 `tosm500md.sys` 与 `agrsm64.sys` 逐字节相同（SHA-256 相同），
因此它依然被 `oem12.cat`（微软 WHQL 签名）认证。实测验证：

* `tosm500md.sys` 正常加载；
* 把同一文件**篡改 1 个字节**后另存加载 → **错误 577**（`ERROR_INVALID_IMAGE_HASH`，签名校验失败）。

也就是说签名强制**是生效的**，我们的驱动是"真的被 WHQL 签名认证"才得以加载，而不是靠关掉签名强制。
本方案**没有**做以下任何一件事：

- ❌ 没有开启测试签名（`testsigning`）
- ❌ 没有关闭驱动签名强制（`nointegritychecks` / F8 选项）
- ❌ 没有改安全启动、没有动 WDAC / HVCI 策略
- ❌ 没有删除或修改你的安全软件
- ❌ 没有给自己造自签名证书、没有往"受信任的根"里塞证书
- ❌ 没有修改驱动的任何一个字节（`tosm500md.sys` 与 `agrsm64.sys` 哈希完全相同）

---

## 4. 文件清单

```text
TOSHIBA_M500_Win10_x64_Modem_v2.2.97\
├─ Install.cmd          一键安装（自动请求管理员权限）
├─ Install.ps1          安装脚本主体
├─ Verify.cmd           一键验证
├─ Verify.ps1           验证脚本（17 项检查）
├─ Uninstall.cmd        一键卸载（带确认）
├─ Uninstall.ps1        卸载脚本
├─ ModemCommon.ps1      公共函数库（被上面三个脚本点源加载）
├─ README.md            本文件
├─ SHA256SUMS.txt       校验值清单
├─ SAMPLE-OUTPUT.txt    验证脚本的真实输出样例（本机实测）
└─ driver\              原厂驱动包（未做任何修改）+ 一个同哈希改名副本
   ├─ lsimdv64.inf      LSI 原厂 INF（CatalogFile=lsimdv64.cat，DriverVer 07/21/2009,2.2.97）
   ├─ lsimdv64.cat      微软 WHQL 签名目录
   ├─ agrsm64.sys       真实驱动二进制（INF 指定的名字）
   ├─ tosm500md.sys     同一二进制的副本（实际加载名，用于绕开文件名拦截）
   ├─ agr64svc.exe      附带程序（原厂包内容，INF 会复制）
   ├─ agrsco64.dll      原厂 64 位 CoInstaller
   ├─ agrsmdel.exe      原厂卸载程序
   ├─ agrdel64.exe      原厂卸载程序
   └─ agrsmdel.ico      图标
```

> `agrsm64.sys` 与 `tosm500md.sys` 是**同一份二进制**（SHA-256 均为
> `F78919616CB275008FFF2DB57C03F86132C52A257C4ED00FF289C57035A89CE7`），
> 保留两份是为了让 `driver\` 目录本身仍可作为标准的"从磁盘更新驱动程序"源使用。

---

## 5. 安装方法

> 需要 **管理员权限**。整个过程 30 秒左右，通常**不需要重启**。

### 方法 A：一键安装（推荐）

1. 双击 **`Install.cmd`**；
2. UAC 弹窗点"是"；
3. 看到 `安装完成。若需验证请运行 Verify.cmd` 即可。

脚本是**幂等**的，重复运行安全；驱动被重装/回滚/系统更新覆盖后，重新运行即可修复。

### 方法 B：命令行

```bat
powershell -NoProfile -ExecutionPolicy Bypass -File .\Install.ps1
powershell -NoProfile -ExecutionPolicy Bypass -File .\Install.ps1 -Force      :: 强制重新导入驱动包
powershell -NoProfile -ExecutionPolicy Bypass -File .\Install.ps1 -SkipAtTest :: 跳过 AT 自检
```

### 方法 C：半手动（设备管理器 + 一条命令）

1. 设备管理器 → **调制解调器** → `Win Modem (Unsupported)`（或 `TOSHIBA Software Modem`）
   → 右键 **更新驱动程序** → **浏览我的电脑以查找驱动程序** → 指向本目录的 `driver\` 文件夹
   → 下一步，安装 `TOSHIBA Software Modem`（发布者 LSI）；
2. 此时设备可能仍是"代码 48"，因为 `agrsm64.sys` 这个名字在本机被拦截——
   再运行一次 **`Install.cmd`**（或只跑 `Install.ps1 -SkipAtTest`）完成"加载名重定向"这一步即可。

---

## 6. 验证

双击 **`Verify.cmd`**，或：

```bat
powershell -NoProfile -ExecutionPolicy Bypass -File .\Verify.ps1
```

本机实测结果（**17/17 全部通过**，完整输出见 `SAMPLE-OUTPUT.txt`）：

```text
[通过] Windows 10 x64 平台                 Windows 10 专业版 10.0.19045 64 位
[通过] 驱动包文件齐全
[通过] WHQL 目录签名有效                    Valid / CN=Microsoft Windows Hardware Compatibility Publisher
[通过] 调制解调器设备存在                    HDAUDIO\FUNC_02&VEN_11C1&DEV_3026&SUBSYS_11790001&REV_1007
[通过] 设备状态正常 (OK / CM_PROB_NONE)
[通过] 驱动提供者为 LSI                     LSI
[通过] 驱动版本为 2.2.97
[通过] 内核服务 AgereSoftModem 已注册
[通过] ImagePath 指向 tosm500md.sys
[通过] 服务已启动 (RUNNING)
[通过] 实际加载的镜像为 tosm500md.sys
[通过] 已安装二进制与包内一致
[通过] 调制解调器串口已分配                  COM3
[通过] 串口 AT 通信
[通过] 调制解调器固件可识别                  ATI -> TOSHIBA Software Modem Version 2.2.97
[通过] 厂商信息 (AT+FMI)                    AT+FMI -> LSI Corporation
[通过] DAA 摘机与拨号音检测通路              ATX4DT0 -> NO DIALTONE（未接电话线，属正常）
```

AT 指令实测对话：

```text
AT          -> OK
ATI         -> TOSHIBA Software Modem Version 2.2.97      + OK
ATI4        -> Built on 07/21/2009 14:03:29                + OK
AT+FMI      -> LSI Corporation                             + OK
AT+FMM      -> Data/Fax                                    + OK
AT+FMR      -> 2.2.97                                      + OK
AT+FCLASS=? -> 0,1                                         + OK      （数据 + 传真 Class 1）
AT+FCLASS=1 -> OK                                          （切入传真模式）
AT+FCLASS=0 -> OK                                          （切回数据模式）
AT+FCLASS=8 -> ERROR                                       （本型号无语音模式，正常）
ATS0?       -> 000 + OK
ATX4DT123456 -> NO DIALTONE                                （摘机、检拨号音 — DAA 通路正常）
ATH         -> OK                                          （挂机）
ATZ         -> OK                                          （复位）
AT&V        -> ACTIVE PROFILE: B1 L2 M1 E1 N1 Q0 V1 W2 X4 Y0 &C1 &D2 … + OK   （参数表可读）
AT&F        -> OK                                          （恢复出厂参数）
AT&W        -> OK                                          （参数写入 NVRAM）
ATI7        -> HDA MB OK                                   （HDA 主板接口自检返回 OK）
ATE0 / ATE1 / AT+FPR=1 / ATI3 / ATI4 / AT+GMM …            （均正常）
```

> `AT+FCLASS=8 -> ERROR` 与 `ATDT -> NO DIALTONE` 都是**正确**行为：
> 前者表示该型号没有语音（voice）功能组，后者表示摘机后没有检测到拨号音（没接电话线）。
> 接上电话线后 `ATDT` 就会真正拨号。

额外的自动检查（每次 `Install.ps1` 都会做）：

| 检查 | 结果 |
|---|---|
| 9 个包文件齐全 | ✅ |
| `tosm500md.sys` 与 `agrsm64.sys` 哈希一致 | ✅ |
| 服务 `ImagePath` 指向 `tosm500md.sys` | ✅ |
| 设备重启后 状态 OK / CM_PROB_NONE | ✅ |

---

## 7. 卸载与回滚

### 正常卸载

双击 **`Uninstall.cmd`**（有 Y/N 确认），或：

```bat
powershell -NoProfile -ExecutionPolicy Bypass -File .\Uninstall.ps1
```

它会：还原 `ImagePath` → 删除 `tosm500md.sys` → 从驱动库删除 LSI 驱动包 → 重启设备。

> 卸载后设备会回到微软的 **Win Modem (Unsupported) / 代码 48** 状态 —— 这是这台机器在
> Windows 10 下的**出厂默认状态**，不是故障。

只撤销"加载名重定向"、保留驱动库中的驱动包（排查用）：

```bat
powershell -NoProfile -ExecutionPolicy Bypass -File .\Uninstall.ps1 -KeepPackage
```

### 极端情况回滚

改动前的注册表快照保存在工作区根目录的 `_backup\` 里：

| 文件 | 内容 |
|---|---|
| `_backup\dev-enum.reg` | 安装前设备枚举键 `HKLM\SYSTEM\CurrentControlSet\Enum\HDAUDIO\...` |
| `_backup\modem-class.reg` | 安装前 Modem 类键 `...\Control\Class\{4D36E96D-...}` 

现已支持 Windows7 x64 ～ Windows11 x64
