# NH3 CNN-LSTM LoongArch SoC 触屏预测演示工程

本目录是一个可独立构建的 LoongArch SoC 工程。NH3 CNN-LSTM 加速器以普通
SystemVerilog RTL 接入 AXI 总线，目标器件为 `xc7a200tfbg676-1`，不使用
Vivado IP Packager 或 `.xci` 封装。

模型仍采用“过去 72 小时数据预测下一小时”的方式，本次修改没有改变模型输出
维度，也不需要重新训练。工程主要扩展了完整测试集选择、物理量还原、预测结果
对比和 800x480 触屏界面。

## 功能概览

| 项目 | 当前实现 |
|---|---|
| 输入窗口 | 连续 72 小时，每小时 51 个特征，共 `72 x 51 = 3672` 个 Q4.12 word |
| 预测目标 | 输入窗口结束后的下一小时 NH3 浓度 |
| 可选测试窗口 | `#000` 至 `#655`，共 656 组 |
| 组号输入 | 点击组号区域弹出数字键盘，可直接输入 `000` 至 `655` |
| 快速导航 | 首组、`-24时`、前一组、后一组、`+24时`、末组 |
| 曲线 | 72 小时历史浓度曲线，以及下一小时预测值、实测值和两者误差线 |
| 结果 | `预测`、`实测`、`偏差` |
| 显示方式 | 800x480 固定单屏，主要中文使用 14x14 点阵，无上下滚动，无全帧缓冲 |
| 板载指示器 | 16 个 LED 和 8 位数码管固定熄灭，运行状态只在 LCD 显示 |
| 运算资源 | 整个 SoC 的 DSP48 使用量为 0 |

界面中的三项结果含义如下：

- `预测`：模型预测的下一小时 NH3 浓度。
- `实测`：测试集记录的下一小时实际 NH3 浓度。
- `偏差`：`预测 - 实测`，保留正负号。负值表示预测偏低。

预测值与实测值不完全相等是正常的模型误差，不表示系统失败。只有固件、DMA、
数据或计算链路发生异常时，界面才显示 `SOC 错误`。

## 触屏操作

1. 下载 `output/nh3_demo_soc_dma_tft_touch.bit` 并释放复位。
2. 等待顶部状态显示 `SOC 就绪`，右上角显示 `触控 正常`。
3. 使用 `<<`、`-24时`、`<`、`>`、`+24时`、`>>` 浏览测试窗口。
4. 要直接跳转，点击中间的 `#000 / 655` 组号区域，在数字键盘输入任意组号并确认。
5. 点击绿色 `运行`。界面依次显示 `SOC 加载`、`SOC 运行` 和 `SOC 完成`；完成后更新
   三项结果并在底部显示 `SOC 计算正常`。
6. 完成后查看历史曲线、预测/实测标记以及三项数值结果。

组号在边界处会被钳位，不会循环越界。`-24时` 和 `+24时` 分别移动 24 个相邻的
一小时滑动窗口。

## 示例结果

完整 SoC 回归测试默认运行窗口 `#000`，结果为：

```text
CORE_Q4_12=0x0526
PREDICTED_MILLI=8107
OBSERVED_MILLI=8824
BIAS_MILLI=-717
```

对应界面显示 `8.107`、`8.824` 和 `-0.717 mg/m3`。

## 数据组织

数据来自量化导出流程的完整保留测试集 `x_test`，不是训练集。原始形状为
`(656, 72, 51)`。相邻窗口重叠 71 小时，因此固件头文件将其无损压缩为
`727 x 51 = 37077` 个输入 word；选择组 `g` 时，DMA 源地址指向连续数据的
第 `g x 51` 个 word，并固定传输 3672 个 word。

固件使用每组的 lag1 基线和固定点逆缩放系数，将 CNN-LSTM 的残差输出还原为
物理浓度。测试窗口、历史浓度、实测值和缩放参数由
`scripts/generate_vectors.py` 统一生成到
`software/nh3_demo/nh3_vectors.h`。

## 验证结果

| 验证项 | 结果 |
|---|---|
| 数据形状及 71 小时窗口重叠 | PASS |
| 旧 8 组向量逐 word 兼容性 | PASS |
| LCD/触摸单元测试 | PASS |
| 任意组号数字键盘输入 `655` | PASS |
| 800x480 中文 SOC 文案及曲线像素 | PASS |
| 中文字体放大为 14x14 | PASS |
| 绝对差结果面板已移除 | PASS |
| 完整 CPU/DMA/CNN-LSTM/LCD 行为仿真 | PASS |
| 板载 LED 和数码管固定熄灭 | PASS |
| DMA 读/写数量 | `3672 / 3672` words |
| NH3 核固定点输出 | `0x0526` |
| 综合后 DSP48 | `0 / 740` |
| 最终静态时序 | WNS `+0.210 ns`，WHS `+0.018 ns` |
| 最终路由 | `61960 / 61960` 可布线网络完成，路由错误 0 |
| 最终 DRC | Error 0，Critical Warning 0 |
| 最终资源 | LUT 49859（37.26%），寄存器 19837（7.41%），BRAM tile 268.5（73.56%） |

完整 SoC 仿真的关键日志：

```text
UART: TEST_WINDOWS=656
UART: GROUP=0
UART: DMA_WORDS=3672
UART: CORE_Q4_12=0x0526
UART: PREDICTED_MILLI=8107
UART: OBSERVED_MILLI=8824
UART: BIAS_MILLI=-717
UART: DIFFERENCE_MILLI=717
UART: SOC_COMPUTE_OK
NH3_SOC_SIM_RESULT=PASS
LCD_SOC_COMPLETED_STATE=PASS
BOARD_INDICATORS_OFF=PASS
DMA_READ_WORDS=3672
DMA_WRITE_WORDS=3672
```

## 重新构建

在 WSL `Ubuntu-22.04` 中构建 LoongArch 固件：

```powershell
wsl.exe -d Ubuntu-22.04 -- bash -lc `
  "cd /mnt/d/loongarch_soc/nh3_soc_dma_touch_demo && ./scripts/build_firmware.sh"
```

重新创建 Vivado 工程：

```powershell
& 'D:\Xilinx\Vivado\2019.2\bin\vivado.bat' -mode batch `
  -source scripts\create_project.tcl
```

运行 LCD/触摸单元测试：

```powershell
& 'D:\Xilinx\Vivado\2019.2\bin\vivado.bat' -mode batch `
  -source scripts\run_lcd_touch_test.tcl
```

运行完整 SoC 行为仿真：

```powershell
& 'D:\Xilinx\Vivado\2019.2\bin\vivado.bat' -mode batch `
  -source scripts\run_simulation.tcl
```

执行综合、布局布线并生成 bitstream：

```powershell
& 'D:\Xilinx\Vivado\2019.2\bin\vivado.bat' -mode batch `
  -source scripts\synth_impl_direct.tcl
```

烧录当前 bitstream：

```powershell
& 'D:\Xilinx\Vivado\2019.2\bin\vivado.bat' -mode batch `
  -source scripts\program_board.tcl
```

## 主要文件

| 路径 | 用途 |
|---|---|
| `software/nh3_demo/demo.c` | 组号读取、DMA、推理、物理量还原及 SOC UI 数据输出 |
| `software/nh3_demo/nh3_vectors.h` | 656 组测试窗口的压缩输入及显示数据 |
| `scripts/generate_vectors.py` | 从完整测试集生成固件向量头文件 |
| `rtl/ip/confreg/confreg.v` | SOC UI MMIO 寄存器和 640 列曲线存储器 |
| `rtl/display/nh3_lcd_ui_renderer.v` | 800x480 固定单屏界面渲染器 |
| `rtl/display/gt9147_touch.v` | GT9147 I2C 和组号/数字键盘触摸控制 |
| `rtl/nh3_demo_soc_top.v` | 板级顶层、跨时钟同步和显示链路 |
| `sim/tb_nh3_lcd_touch.sv` | 任意组选择、文案和像素级 LCD 单元测试 |
| `sim/tb_nh3_demo_soc.sv` | 完整 SoC 回归测试 |
| `output/nh3_soc_touch_preview.png` | 当前 800x480 界面预览 |
| `output/nh3_demo_soc_dma_tft_touch.bit` | 最终可烧录 bitstream |

当前固件 `firmware/firmware.memh` 的 SHA-256：

```text
28d659e5decf6dbc40b98eccb9b665103fb0353fae196837a3a81113381876ec
```

当前 bitstream `output/nh3_demo_soc_dma_tft_touch.bit` 的 SHA-256：

```text
3684b997bbe5f3e22262463c1f74e87eab2ba75a460c7de0a91d56e84dc1cfad
```

实板烧录不在自动仿真范围内。烧录后仍需确认屏幕排线、触点方向、按钮命中，
以及 GT9147 的 `SCL/SDA/INT/RST` 接线和上拉。
