# STM32U575 智能家居项目 SWD 连接失败保姆级排查+全片擦除教程
> 解决 `No STM32 target found!` / `No ST-LINK detected` 等SWD连接问题，适配STM32U5系列智能家居开发

## 📌 问题背景
最近在做**基于STM32U575的智能家居控制终端**项目，烧录/调试时遇到SWD连接失败的致命问题：
- STM32CubeProgrammer 持续报错 `No STM32 target found!`
- Keil 下载/调试直接报错，完全连不上芯片
- 排查接线、供电、芯片好坏，踩了无数坑，最终彻底解决
- 整理完整排查流程、踩坑记录、解决方案，帮做智能家居/嵌入式开发的小伙伴少走弯路

---

## ⚠️ 问题现象（快速定位）
### 1. Keil 端报错
- 「No ST-LINK detected」：Keil 无法识别 ST-Link 烧录器
- 「No STM32 target found!」：无法找到STM32目标芯片，提示Debug Authentication问题

![Keil-No-ST-LINK-detected](1-keil-no-stlink.png)
![Keil-No-target-found](2-keil-no-target.png)

### 2. CubeProgrammer 端报错
无论怎么调整参数，都无法连接芯片，日志持续报错：

![CubeProgrammer-connect-fail](10-cube-connect-fail.png)

### 3. 命令行（CLI）擦除失败报错
尝试用 `STM32_Programmer_CLI.exe` 命令行擦除，同样连接失败：

![CLI-error](11-cli-error.png)

---

## 🔧 完整排查&解决流程（按顺序执行，不踩坑）
### 1. 硬件基础排查（必做）
- **接线检查**：确认5根核心线接对（智能家居硬件必查，避免接线混乱）
  - ST-Link SWDIO → 芯片PA13
  - ST-Link SWCLK → 芯片PA14
  - ST-Link GND → 板子GND（必须共地，抗传感器干扰）
  - ST-Link 3.3V → 板子3.3V
  - ST-Link NRST → 板子NRST（U5系列必接，解决时序问题）
- **芯片好坏判断**：拔掉ST-Link，测PA13/PA14为0V（高阻态），证明芯片未损坏
- **BOOT0检查**：确认BOOT0接GND（主Flash模式），避免进入ISP模式锁死SWD
- **NRST检查**：确认NRST为3.3V高电平，无被传感器电路拉低/短路

### 2. 升级ST-Link固件+CubeProgrammer（解决U5兼容性）
旧版固件（V2J37S7）对STM32U5的Debug Authentication支持不全，是核心原因之一：
- 操作：打开CubeProgrammer → 右侧「固件升级」，升级到最新版（v2.22.0），同步升级ST-Link固件到V2J46S7+

![CubeProgrammer-update](3-cube-update.png)

### 3. Keil 关键配置（with Pre-reset 自动复位）
U5系列对SWD时序要求极高，智能家居代码运行快，手动复位很难卡准窗口，必须开启自动预复位：
1.  打开Keil → 魔法棒 → Debug 选项卡
2.  选择ST-Link Debugger → 点击Settings
3.  `Connect & Reset Options` 选择 `with Pre-reset`
4.  Max Clock 从10MHz降到1.8MHz（抗传感器干扰）
5.  勾选 `Reset after Connect`，点击确定

![Keil-Debug-setting](4-keil-debug-tab.png)
![Keil-ST-Link-setting](5-keil-stlink-setting.png)

### 4. CubeProgrammer 终极连接配置（抄作业）
| 参数项 | 必须设置 | 作用 |
|--------|----------|------|
| 端口 | SWD | 禁用JTAG，避免GPIO复用冲突 |
| 频率(kHz) | 4000（兜底100kHz） | 降低速率，最大化握手窗口 |
| 模式 | Under reset | 复位下连接，抢占SWD总线 |
| 复位模式 | Hardware reset | 硬件复位，精准控制时序 |
| 速度 | Reliable | 可靠通信，避免丢包 |
| 共享 | Disabled | 独占端口，避免冲突 |

### 5. 全片擦除（解决SWD锁死，恢复芯片）
- 连接成功后，进入「擦除和编程」界面，勾选 `Full chip erase`
- 点击 `Start automatic mode`，执行全片擦除
- 日志显示 `Mass erase successfully achieved`，证明擦除成功，芯片恢复出厂状态

![CubeProgrammer-erase](7-cube-erase.png)
![CubeProgrammer-erase-success](8-cube-erase-success.png)

---

## 🛟 兜底方案：SWD彻底锁死终极解决
如果以上操作仍无法连接，大概率是代码误关闭了SWD/JTAG接口：
- 操作：将开发板BOOT0引脚接3.3V，重新上电，芯片进入系统Bootloader模式，SWD默认开启，可正常连接烧录；烧录完成后将BOOT0改回GND

![BOOT0-fix](6-boot0-fix.png)

---

## ❌ 智能家居项目专属踩坑&避坑指南
1.  **坑1：不接NRST，手动按复位键**
    - 后果：智能家居代码逻辑复杂，芯片上电瞬间锁死SWD，手动复位90%概率连不上
    - 解决：必须接NRST硬件复位引脚，让ST-Link自动控制时序
2.  **坑2：用10MHz高频SWD连接**
    - 后果：U5对高频兼容性差，智能家居传感器干扰导致握手失败
    - 解决：频率降到4MHz以下，兜底100kHz，抗干扰拉满
3.  **坑3：命令行（CLI）擦除**
    - 后果：CLI时序控制差，U5的Debug Authentication直接拒绝连接
    - 解决：优先用图形化CubeProgrammer，稳定性远高于CLI
4.  **坑4：误以为芯片被智能家居电路烧坏**
    - 误区：PA13有3.3V=芯片击穿
    - 真相：3.3V是ST-Link上拉的，断电后测0V就是正常
5.  **坑5：传感器电路干扰SWD**
    - 后果：继电器、WiFi模块电磁干扰导致通信异常
    - 解决：烧录时断开传感器/继电器电源，仅保留主控供电

---

## 📌 后续操作：Keil烧录智能家居程序
擦除成功后，Keil按以下配置，一键烧录调试：
1.  魔法棒 → Debug → Settings
2.  端口选SW，Max Clock设为1MHz
3.  `Connect & Reset Options` 选 `with Pre-reset`
4.  勾选 `Reset after Connect`，Flash Download勾选`Program/Verify/Reset and Run`
5.  点击确定，直接点Debug/下载，一键完成烧录

---

## 🎯 总结
STM32U5系列在智能家居项目中SWD连接失败，99%的原因是**时序不匹配+硬件干扰+兼容性问题**，核心解决思路：
1.  硬件：接NRST，保证接线正确，烧录时断开传感器电源
2.  软件：升级ST-Link+CubeProgrammer，降频+硬件复位模式
3.  操作：先连接，再全片擦除，最后烧录程序，分步操作不踩坑

---

## 📄 许可证
本教程采用 MIT License 开源，可自由转载、修改，注明出处即可。
