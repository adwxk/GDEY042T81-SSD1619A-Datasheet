# GDEY042T81 (SSD1619A) 4.2寸墨水屏 技术资料

> 商家未提供任何技术文档，本文档通过逆向工程 + 实测整理而成，供所有使用者参考。
> 测试平台：ESP32-C3 + GxEPD2 库，2026 年实测验证。

---

## 目录

1. [屏幕基本信息](#1-屏幕基本信息)
2. [引脚定义](#2-引脚定义)
3. [与 SSD1683 的关键差异](#3-与-ssd1683-的关键差异)
4. [命令集](#4-命令集)
5. [0x22 Mode Byte](#5-0x22-mode-byte)
6. [LUT 格式（70字节）](#6-lut-格式70字节)
7. [差分驱动机制](#7-差分驱动机制)
8. [裸 SPI 驱动代码](#8-裸-spi-驱动代码)
9. [GxEPD2 集成](#9-gxepd2-集成)
10. [刷新性能数据](#10-刷新性能数据)
11. [LUT 调参指南](#11-lut-调参指南)
12. [常见问题排错](#12-常见问题排错)
13. [移植到其他 MCU](#13-移植到其他-mcu)
14. [与其他控制器对比](#14-与其他控制器对比)

---

## 1. 屏幕基本信息

| 项目 | 参数 |
| --- | --- |
| 厂家标签 | GDEY042T81（大连佳显 Good Display） |
| 实际控制器 | Solomon Systech **SSD1619A**（非 SSD1683） |
| 尺寸 | 4.2 英寸 |
| 分辨率 | 400 x 300 |
| 颜色 | 黑白（B/W） |
| 接口 | SPI 4 线（SCK, MOSI, CS, DC）+ RST + BUSY |
| FPC 排线 | 9Pin（含 CS2，通常不接） |
| 工作电压 | 3.3V |
| BUSY 极性 | HIGH = 忙，LOW = 空闲 |
| SPI 频率 | 10 MHz（实测稳定） |

## 2. 引脚定义

### 9Pin FPC 排线

| Pin | 信号 | 说明 |
| --- | --- | --- |
| 1 | GND | 地 |
| 2 | 3V3 | 电源 3.3V |
| 3 | SCK | SPI 时钟 |
| 4 | SDA | SPI MOSI（不是 I2C SDA） |
| 5 | RST | 复位（低电平复位） |
| 6 | DC | 数据/命令选择（0=命令, 1=数据） |
| 7 | CS1 | 片选（低有效） |
| 8 | BUSY | 忙信号（HIGH=忙） |
| 9 | CS2 | 第二片选（通常不接） |

### 接线示例（ESP32-C3）

| 屏幕引脚 | GPIO | 说明 |
| --- | --- | --- |
| GND | GND | |
| 3V3 | 3V3 | |
| SCK | GPIO4 | SPI 时钟 |
| SDA(MOSI) | GPIO6 | SPI MOSI |
| RST | GPIO0 | 复位 |
| DC | GPIO1 | 数据/命令 |
| CS1 | GPIO7 | 片选 |
| BUSY | GPIO10 | 忙信号 |
| CS2 | 不接 | 悬空即可 |

> 其他 MCU 只需改引脚号，驱动代码不变。

## 3. 与 SSD1683 的关键差异

这块屏幕经常被误认为 SSD1683 屏（命令集兼容）。但两者有重要差异：

| 特性 | SSD1619A（本屏） | SSD1683 |
| --- | --- | --- |
| LUT 大小 | **70 字节** | 227 字节 |
| 自定义LUT Mode Byte | **0xC5** | 0xDC |
| OTP 全刷 Mode Byte | 0xF7 | 0xF7 |
| 命令集 | 兼容 SSD1683 | - |

**用 SSD1683 的 227 字节 LUT 或 0xDC mode byte，屏幕不会更新。** 这是最常见的"屏幕点不亮"原因。

### 如何判断你的屏幕是 SSD1619A

1. 用 SSD1683 的 227 字节 LUT 刷新，屏幕不更新 -> 可能是 SSD1619A
2. 用 0xDC mode byte 不工作 -> SSD1619A
3. 用 0xC5 mode byte + 70 字节 LUT 工作正常 -> 确认 SSD1619A
4. 查看 FPC 标签 GDEY042T81 通常配 SSD1619A

## 4. 命令集

| 命令 | 名称 | 说明 |
| --- | --- | --- |
| 0x01 | Driver Output Control | 设置 MUX（300行）。参数: 0x2B 0x01 0x00 |
| 0x03 | VGH Voltage | 设置 VGH |
| 0x04 | VSH1/VSH2/VSL | 设置源极驱动电压 |
| 0x0C | Booster Soft Start | 升压器软启动 |
| 0x10 | Deep Sleep | 参数 0x01 进入深度睡眠 |
| 0x11 | RAM Entry Mode | 0x03 = X+ Y+ 正常方向 |
| 0x12 | SWRESET | 软复位 |
| 0x18 | Temperature Sensor | 0x80 = 内置温度传感器 |
| 0x20 | Master Activation | 执行刷新（触发 BUSY） |
| 0x22 | Display Update Control | 设置 mode byte 触发不同刷新类型 |
| 0x24 | Current RAM | 当前帧图像数据（15000字节） |
| 0x26 | Previous RAM | 上一帧图像数据（差分驱动用） |
| 0x32 | Waveform LUT | 写入 70 字节自定义波形 LUT |
| 0x3A | Dummy Line Period | 每帧空行数（推荐 0x1A=26） |
| 0x3B | Gate Line Period | 每行驱动时间（推荐 0x0B=11） |
| 0x3C | Border Waveform | 边框波形（0x01=跟随LUT） |
| 0x44 | RAM X Address | 起始/结束列（字节单位，每字节=8像素） |
| 0x45 | RAM Y Address | 起始/结束行（像素单位，低8位+高8位） |
| 0x4E | RAM X Counter | 当前列指针 |
| 0x4F | RAM Y Counter | 当前行指针 |

## 5. 0x22 Mode Byte

| Mode Byte | 用途 | 时间 | 闪烁 | 残影清除 |
| --- | --- | --- | --- | --- |
| **0xC5** | **自定义LUT刷新（推荐）** | 取决于T1 | 不闪 | 否 |
| 0xF7 | OTP 标准全刷 | ~3175ms | 多次 | 是 |
| 0xD7 | OTP 快速全刷 | ~1802ms | 2次 | 部分 |
| 0xFC | OTP 局刷 | 不可靠 | - | - |

> SSD1683 的 0xDC 在 SSD1619A 上**不工作**。必须用 0xC5。

## 6. LUT 格式（70字节）

SSD1619A 使用 **70 字节 LUT**（不是 SSD1683 的 227 字节）：

```
bytes  0..34 : 5 planes x 7 bytes (VCOM, WW, BW, WB, BB)
bytes 35..69 : 7 phases x 5 bytes (T1, T2, T3, T4, Repeat)
```

### 6.1 Plane Section（前35字节）

每个 plane 有 7 字节，**首字节是电压**，后 6 字节为 0x00：

| 电压值 | 含义 |
| --- | --- |
| 0x00 | VSS（无驱动） |
| 0x40 | VSH1（驱动黑） |
| 0x80 | VSL（驱动白） |

| 顺序 | Plane | 含义 | 推荐值 | 注意 |
| --- | --- | --- | --- | --- |
| 0 | VCOM | 公共电极 | **0x40** | 0x80 驱动整屏含边框，0x00 无输出 |
| 1 | WW | 白->白 | 0x80 | |
| 2 | BW | 黑->白 | **0x80** | 0x00 会导致散点 |
| 3 | WB | 白->黑 | **0x40** | 0x00 会导致散点 |
| 4 | BB | 黑->黑 | **0x00** | 0x40 会导致全白（与VCOM同值无压差） |

### 6.2 Timing Section（后35字节）

7个相位，每个5字节（T1,T2,T3,T4,Repeat）。通常只用第一个：

```c
T1, 0x00, 0x00, 0x00, 0x00,  // phase 0: T1=驱动帧数
0x00, 0x00, 0x00, 0x00, 0x00,  // phase 1-6: 空
...
```

### 6.3 推荐的 LUT 配置

**快速刷新 LUT（T1=10，237ms，用于翻页）：**

```c
static const uint8_t FAST_LUT[70] = {
    0x40, 0x00,0x00,0x00,0x00,0x00,0x00,  // VCOM
    0x80, 0x00,0x00,0x00,0x00,0x00,0x00,  // WW
    0x80, 0x00,0x00,0x00,0x00,0x00,0x00,  // BW
    0x40, 0x00,0x00,0x00,0x00,0x00,0x00,  // WB
    0x00, 0x00,0x00,0x00,0x00,0x00,0x00,  // BB(必须0x00)
    0x0A, 0x00,0x00,0x00,0x00,  // T1=10帧
    0x00,0x00,0x00,0x00,0x00, 0x00,0x00,0x00,0x00,0x00,
    0x00,0x00,0x00,0x00,0x00, 0x00,0x00,0x00,0x00,0x00,
    0x00,0x00,0x00,0x00,0x00, 0x00,0x00,0x00,0x00,0x00,
};
```

**清残影 LUT（T1=31，808ms）：** 同上，只把 T1 从 `0x0A` 改为 `0x1F`。

## 7. 差分驱动机制

SSD1619A 比较两帧数据来决定驱动哪些像素：

- **0x24**：当前帧（要显示的新内容）
- **0x26**：上一帧（屏幕当前显示的内容）

### 7.1 全刷时的 RAM 管理

全刷前**必须清空 0x26 为全白**，制造"旧=全白"的差分：

```
全刷前: 0x26 = 全白(0xFF)
全刷:   0x24 = 新内容, 0x26 = 全白
         白->黑: WB=0x40 驱动黑
         白->白: WW=0x80 驱动白
```

如果不清 0x26，差分方向错误，导致显示反色或变淡。

### 7.2 局刷时的 RAM 管理

局刷后需同步 0x26 = 0x24，为下次局刷准备正确的差分基准。

## 8. 裸 SPI 驱动代码

以下代码不依赖任何第三方库，可移植到任意 MCU：

### 8.1 基础通信

```c
#define EPD_SCK   4
#define EPD_MOSI  6
#define EPD_CS    7
#define EPD_DC    1    // 0=命令, 1=数据
#define EPD_RST   0
#define EPD_BUSY  10   // HIGH=忙
#define BUF_SIZE  15000  // 400*300/8

void epdCmd(uint8_t cmd) {
  digitalWrite(EPD_DC, LOW);
  digitalWrite(EPD_CS, LOW);
  SPI.transfer(cmd);
  digitalWrite(EPD_CS, HIGH);
}

void epdData(uint8_t data) {
  digitalWrite(EPD_DC, HIGH);
  digitalWrite(EPD_CS, LOW);
  SPI.transfer(data);
  digitalWrite(EPD_CS, HIGH);
}

void epdData(const uint8_t *data, size_t len) {
  digitalWrite(EPD_DC, HIGH);
  digitalWrite(EPD_CS, LOW);
  SPI.writeBytes((uint8_t *)data, len);
  digitalWrite(EPD_CS, HIGH);
}

void epdWaitBusy() {
  while (digitalRead(EPD_BUSY) == HIGH) { delay(1); }
}
```

### 8.2 初始化

```c
void epdInit() {
  digitalWrite(EPD_RST, LOW);  delay(10);
  digitalWrite(EPD_RST, HIGH); delay(10);
  epdWaitBusy();

  epdCmd(0x12);  // SWRESET
  epdWaitBusy();

  epdCmd(0x01);  // Driver Output: 300行
  epdData(0x2B); epdData(0x01); epdData(0x00);

  epdCmd(0x3C); epdData(0x01);  // Border
  epdCmd(0x18); epdData(0x80);  // Temp sensor
  epdCmd(0x11); epdData(0x03);  // RAM Entry Mode
}
```

### 8.3 加载 LUT + 设置窗口

```c
void epdLoadLUT(const uint8_t *lut) {
  epdCmd(0x32); epdData(lut, 70);
  epdCmd(0x3A); epdData(0x1A);
  epdCmd(0x3B); epdData(0x0B);
}

void epdSetFullWindow() {
  epdCmd(0x44); epdData(0x00); epdData((400-1)/8);
  epdCmd(0x45); epdData(0x00); epdData(0x00);
  epdData(0x2B); epdData(0x01);
  epdCmd(0x4E); epdData(0x00);
  epdCmd(0x4F); epdData(0x00); epdData(0x00);
}
```

### 8.4 写入数据 + 触发刷新

```c
void epdWriteImage(const uint8_t *bmp) {
  epdSetFullWindow();
  epdCmd(0x24); epdData(bmp, BUF_SIZE);
}

void epdClearPrevRAM() {  // 全刷前必须
  epdSetFullWindow();
  epdCmd(0x26);
  digitalWrite(EPD_DC, HIGH); digitalWrite(EPD_CS, LOW);
  for (int i = 0; i < BUF_SIZE; i++) SPI.transfer(0xFF);
  digitalWrite(EPD_CS, HIGH);
}

void epdRefreshFast() {  // 自定义LUT刷(237ms)
  epdClearPrevRAM();
  epdCmd(0x22); epdData(0xC5);
  epdCmd(0x20); epdWaitBusy();
}

void epdRefreshOTP() {  // OTP全刷(3175ms,清残影)
  epdCmd(0x22); epdData(0xF7);
  epdCmd(0x20); epdWaitBusy();
}

void epdDeepSleep() {
  epdCmd(0x10); epdData(0x01); delay(100);
}
```

### 8.5 完整使用示例

```c
void setup() {
  SPI.begin(EPD_SCK, -1, EPD_MOSI, EPD_CS);
  epdInit();

  // 开机OTP全刷建立基线(3175ms)
  uint8_t white[BUF_SIZE]; memset(white, 0xFF, BUF_SIZE);
  epdWriteImage(white);
  epdSetFullWindow(); epdCmd(0x26); epdData(white, BUF_SIZE);
  epdCmd(0x22); epdData(0xF7); epdCmd(0x20); epdWaitBusy();

  // 加载快速LUT并显示内容(237ms)
  epdLoadLUT(FAST_LUT);
  uint8_t img[BUF_SIZE]; memset(img, 0xFF, BUF_SIZE);
  // TODO: 在img上绘制内容
  epdWriteImage(img);
  epdRefreshFast();
}

void loop() {
  delay(30000);  // 30秒后清残影
  epdLoadLUT(CLEAR_LUT);  // T1=31的LUT
  epdWriteImage(img); epdRefreshFast();  // 808ms清残影
  epdLoadLUT(FAST_LUT);  // 换回快速LUT
}
```

## 9. GxEPD2 集成

GxEPD2 库默认用 SSD1683 驱动（GxEPD2_420_GDEY042T81），需要继承并覆盖关键方法：

### 9.1 继承关系

```
GxEPD2_EPD
  -> GxEPD2_420_GDEY042T81 (SSD1683命令族)
       -> 自定义驱动 (SSD1619A 70字节LUT + 0xC5)
```

### 9.2 需要覆盖的方法

| 方法 | 作用 |
| --- | --- |
| `writeImageForFullRefresh` | 只写0x24，不写0x26（保持差分） |
| `writeImageAgain` | 全刷时跳过（保持0x26旧值） |
| `refresh(bool)` | fast=true用自定义LUT(T1=10)，false用清残影LUT(T1=31) |
| `powerOff()` | 覆盖为空操作，省81ms |

### 9.3 GxEPD2 使用示例

```cpp
#include <GxEPD2_BW.h>
#include <SPI.h>

// 自定义驱动类定义在 test2.ino 中
static GxEPD2_BW<SSD1619AFastPartialDriver, 300> display(
    SSD1619AFastPartialDriver(CS, DC, RST, BUSY));

void setup() {
  SPI.begin(SCK, -1, MOSI, CS);  // ss=-1 禁用硬件SS!
  display.epd2.selectSPI(SPI, SPISettings(10000000, MSBFIRST, SPI_MODE0));
  display.init(115200, true, 10, false);
  display.epd2.selectFastFullUpdate(true);
  display.clearScreen();  // OTP全刷建立基线

  display.epd2.prepareFastPartial();  // 加载快速LUT

  // 快速显示(237ms)
  display.setFullWindow();
  display.firstPage();
  do {
    display.fillScreen(GxEPD_WHITE);
    display.setTextColor(GxEPD_BLACK);
    display.setCursor(10, 50);
    display.print("Hello");
  } while (display.nextPage());
}
```

> **重要**：`SPI.begin()` 的 SS 参数必须传 **-1**。否则 SPI 硬件会在每次传输时自动拉低 SS 引脚，导致屏幕和共享 SPI 总线的其他设备被同时选中。

## 10. 刷新性能数据

| 操作 | Mode Byte | T1 | BUSY时间 | 闪烁 | 残影清除 |
| --- | --- | ---: | ---: | --- | --- |
| OTP 标准全刷（开机） | 0xF7 | - | 3175ms | 多次 | 是 |
| 自定义LUT快速刷（翻页） | 0xC5 | 10 | **237ms** | 不闪 | 否 |
| 自定义LUT清残影刷 | 0xC5 | 31 | **808ms** | 不闪 | 较好 |

### 优化历程

| 阶段 | 翻页时间 | 优化点 |
| --- | ---: | --- |
| 原始LUT(T1=31) | 808ms | 基准 |
| 减少帧数(T1=10) | 237ms | 快3.4倍，轻微残影 |
| 跳过PowerOff | 237ms | 再省81ms(总省962ms) |
| 清残影全刷替代OTP | 808ms | 替代3175ms的OTP，快4倍 |

## 11. LUT 调参指南

### 11.1 关键参数

| 参数 | 寄存器 | 推荐值 | 作用 |
| --- | --- | --- | --- |
| T1 (驱动帧数) | LUT[35] | 10或31 | 帧数越多驱动越充分 |
| Dummy Line | 0x3A | 0x1A(26) | 每帧空行 |
| Gate Line | 0x3B | 0x0B(11) | 每行驱动时间 |

### 11.2 刷新时间计算

```
帧时间 = (300 + 26) x 11 x T_OSC = 3586 x T_OSC
刷新时间 = T1 x 帧时间
实测: T1=31 时 808ms => T_OSC = 7.27us (OSC约137kHz)
```

### 11.3 T1 实测对照

| T1 | 帧数 | 时间 | 效果 | 推荐 |
| ---: | ---: | ---: | --- | --- |
| 7 | 7 | 167ms | 残影很重 | 不推荐 |
| **10** | 10 | **237ms** | 轻微残影 | **翻页** |
| 15 | 15 | 351ms | 残影少 | 折中 |
| **31** | 31 | **808ms** | 无残影 | **清残影** |

### 11.4 0x3A/0x3B 调参

| 0x3A | 0x3B | 效果 |
| ---: | ---: | --- |
| **0x1A** | **0x0B** | **推荐**，显示清晰 |
| 0x0A | 0x08 | 残影加重，不推荐 |

> 保持 0x1A/0x0B，只调 T1 是最佳策略。

### 11.5 Plane 电压禁忌

| Plane | 禁止值 | 后果 |
| --- | --- | --- |
| VCOM | 0x80 | 驱动整屏（含边框），屏幕全闪 |
| VCOM | 0x00 | 无输出，屏幕不更新 |
| BW | 0x00 | 黑->白不驱动，出现散点 |
| WB | 0x00 | 白->黑不驱动，出现散点 |
| BB | 0x40 | 与VCOM同值无压差，**屏幕全白** |

## 12. 常见问题排错

| 问题 | 原因 | 解决 |
| --- | --- | --- |
| 屏幕完全不更新 | 用了SSD1683的227字节LUT或0xDC | 换70字节LUT + 0xC5 |
| 局刷出现散点 | BW或WB=0x00 | BW=0x80, WB=0x40 |
| 全刷后屏幕全白 | BB=0x40 | BB改回0x00 |
| VCOM=0x80整屏闪烁 | VCOM过高驱动边框 | VCOM=0x40 |
| 全刷后变淡/反色 | 0x26未清为全白 | 全刷前调用_clearPreviousRamWhite |
| T1=10帧残影重 | 驱动帧数不足 | 保持0x3A=0x1A/0x3B=0x0B，增大T1 |
| 0x3A/0x3B减小后残影重 | 每帧驱动时间不足 | 恢复0x1A/0x0B |
| 与SD卡共享SPI异常 | SPI硬件SS自动拉低其他设备 | SPI.begin传ss=-1 |
| 翻页有81ms额外延迟 | PowerOff等待 | 覆盖powerOff()为空操作 |

## 13. 移植到其他 MCU

只需修改引脚定义，驱动代码完全不变：

```c
// ESP32 (原版) - VSPI引脚
#define EPD_SCK   18  #define EPD_MOSI  23  #define EPD_CS  5
#define EPD_DC     2  #define EPD_RST    4  #define EPD_BUSY 15

// ESP32-S3 - 任意GPIO
#define EPD_SCK   12  #define EPD_MOSI  11  #define EPD_CS  10
#define EPD_DC     9  #define EPD_RST   14  #define EPD_BUSY 13

// ESP32-C3 (本项目)
#define EPD_SCK    4  #define EPD_MOSI   6  #define EPD_CS   7
#define EPD_DC     1  #define EPD_RST    0  #define EPD_BUSY 10

// STM32 - HAL SPI
#define EPD_SCK   PB3 #define EPD_MOSI  PB5 #define EPD_CS  PB0
#define EPD_DC   PB4 #define EPD_RST   PB6 #define EPD_BUSY PB7
```

### 与其他设备共享 SPI 总线

SSD1619A 是只写设备（不需要MISO），可与SD卡共享SPI：

```c
// 关键: SPI.begin 的 ss 参数传 -1
SPI.begin(SCK, MISO, MOSI, -1);  // 禁用硬件SS!
// 否则SPI硬件在每次传输时自动拉低SS引脚
// 导致屏幕和SD卡被同时选中，MISO信号冲突
```

## 14. 与其他控制器对比

| 控制器 | LUT大小 | Mode Byte | 备注 |
| --- | ---: | --- | --- |
| **SSD1619A** | **70字节** | **0xC5** | **本文档（GDEY042T81）** |
| SSD1683 | 227字节 | 0xDC | GxEPD2默认支持 |
| SSD1681 | OTP(无LUT) | 0xFC/0xF7 | GxEPD2_154_D67 |
| SSD1675B | 105字节 | 0xC4/0xC7 | GxEPD2_213_B73 |
| IL3829 | 30字节 | 0x04/0xC4 | GxEPD2_154 |

---

## 致谢

本文档基于以下实测环境整理：
- 硬件：ESP32-C3 + GDEY042T81 4.2寸墨水屏
- 库：GxEPD2 1.6.9 + Adafruit GFX 1.12.6
- ESP32 Arduino Core: 4.0.0-alpha1

如果你也使用这块屏幕，欢迎基于本文档改进和分享。如果有任何问题或补充，请提 Issue。
