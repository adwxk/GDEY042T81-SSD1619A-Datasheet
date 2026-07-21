# GDEY042T81 (SSD1619A) 4.2寸黑白墨水屏技术资料

## 背景

在淘宝**鑫盛液晶**买了一块 4.2 寸黑白墨水屏，型号 FPC-194。屏幕上没有任何标签，商家也没给任何资料。

商家没有提供任何技术资料——没有数据手册，没有驱动代码，连引脚定义都得自己猜。用 SSD1683 的标准驱动点不亮，折腾了很久才发现实际控制器是 **SSD1619A**，和 SSD1683 命令集兼容但 LUT 格式完全不同。

后来在评论区看到有大哥分享了关键信息（70 字节 LUT、0xC5 mode byte），才终于点亮。在此基础上用 AI 辅助整理了完整的驱动文档，包含裸 SPI 驱动代码、GxEPD2 集成、LUT 调参指南等。

希望这份资料能帮到同样买了这块屏但一头雾水的人。

## 屏幕参数

| 项目 | 参数 |
| --- | --- |
| 购买渠道 | 淘宝鑫盛液晶 |
| 型号 | FPC-194 |
| 控制器 | SSD1619A（非 SSD1683） |
| 尺寸 | 4.2 英寸 |
| 分辨率 | 400 x 300 |
| 颜色 | 黑白 |
| 接口 | SPI |
| 工作电压 | 3.3V |

## 最关键的信息

如果你只想知道一件事，那就是：

> **这块屏的控制器是 SSD1619A，不是 SSD1683。**
>
> - LUT 是 **70 字节**，不是 227 字节
> - 刷新 Mode Byte 是 **0xC5**，不是 0xDC
> - 用 GxEPD2 的 SSD1683 驱动能初始化但**屏幕不更新**

## 文档内容

详细的技术文档在 [GDEY042T81_SSD1619A_Datasheet.md](GDEY042T81_SSD1619A_Datasheet.md)，包含：

- 引脚定义与接线
- 完整命令集
- LUT 格式详解（70 字节结构）
- 推荐的 LUT 配置（快速刷 237ms / 清残影刷 808ms）
- 裸 SPI 驱动代码（不依赖第三方库）
- GxEPD2 集成方法
- LUT 调参指南
- 常见问题排错
- 移植到其他 MCU

## 快速上手

### 接线（ESP32-C3）

```
屏幕 SCK  -> GPIO4
屏幕 SDA  -> GPIO6  (MOSI)
屏幕 RST  -> GPIO0
屏幕 DC   -> GPIO1
屏幕 CS1  -> GPIO7
屏幕 BUSY -> GPIO10
屏幕 VCC  -> 3V3
屏幕 GND  -> GND
屏幕 CS2  -> 不接
```

### 最简驱动代码

```c
// 初始化
epdInit();                    // 复位 + 寄存器配置
epdLoadLUT(FAST_LUT);         // 加载70字节LUT

// 显示画面（237ms快速刷新）
epdWriteImage(your_bitmap);   // 写入15000字节图像数据
epdRefreshFast();              // 触发刷新，237ms完成

// 定期清残影（808ms）
epdLoadLUT(CLEAR_LUT);        // 换T1=31帧的LUT
epdWriteImage(your_bitmap);
epdRefreshFast();              // 808ms完成
epdLoadLUT(FAST_LUT);          // 换回快速LUT
```

完整代码见文档第 8 节。

## 致谢

- 评论区分享关键信息的大哥
- AI 辅助整理和验证

## License

MIT - 随意使用，能帮到人就好。
