# -YingSui-FPGA-Real-time-Tracking-Prediction-System
# FPGA Multi-Target Color Recognition and Intelligent Obstacle-Avoidance Vision System

## 项目简介

本项目基于正点原子达芬奇Pro开发板和 AMD/Xilinx Artix-7 `xc7a200tfbg484-2`，实现摄像头采集、RGB 颜色目标检测、CNN 目标分类、A* 避障路径规划、轨迹记录和 HDMI 显示。

当前数据流：

```text
OV5640 RGB565 采集
  -> DDR3 帧缓存
  -> RGB 颜色阈值与连通域定位
  -> 最多 16 个目标 ROI
  -> 16 路 INT8 CNN 分类
  -> CAR / BLOCK / destination 标签
  -> 64x48、16x16 像素栅格 A*
  -> HDMI 路径、目标框、OSD 和轨迹叠加
```

主要功能：

- 1024x768 OV5640 彩色图像采集与 HDMI 显示；
- 蓝、红、粉、绿四类 RGB 颜色候选；
- 最多 16 个颜色连通目标；
- 16 路 CNN 分类；
- CNN CAR 作为路径起点，绿色目标作为终点；
- 当前 A* 路径层使用 64x48 栅格，每格 16x16 像素，支持八方向搜索；
- All 模式路径显示、Snapshot 冻结、Live 逐帧更新和路径清除；
- All 模式绿色标签为 `destination`，Green 模式为 `block`；
- KEY0 控制轨迹记录，KEY1 切换颜色模式；
- CH340 图形上位机与实体按键并行工作；
- HDMI OSD 显示记录时间和颜色模式。

路径只用于视觉规划显示，不向实体小车输出运动控制命令。

## 目录结构

```text
FPGA_MultiTarget_Color_Avoidance_V3/
|-- README.md
|-- SUBMISSION_GUIDE.md
|-- FILE_MANIFEST.csv
|-- src/
|   |-- color_detect/
|   |-- cnn_accel/
|   |-- traj_prediction/
|   |-- trajectory/
|   |-- driver/
|   |-- top/
|   |-- interaction/
|   |-- display/
|   |-- common/
|   |-- constraints/
|   `-- ip/
|-- sim/
|   |-- unit/
|   |-- system/
|   |-- scripts/
|   |-- results/
|   `-- assets/
|-- build/
|   |-- tcl/
|   |-- reports/
|   `-- bitstream/
|-- board/
|   |-- vivado_project/
|   |-- host_gui/
|   |-- flashing/
|   |-- captures/
|   `-- test_data/
|-- data/
|   |-- cnn_weights/
|   |-- test_images/
|   |-- reference_results/
|   `-- performance/
|-- skill/
|   |-- fpga-astar-vision-maintenance/
|   |-- llm_collaboration_log.md
|   |-- llm_records/
|   `-- reusable_modules/
`-- report/
    |-- design_report.md
    |-- scoring_matrix.md
    |-- optimization_comparison.md
    |-- astar_design_notes.md
    |-- demo_video_script.md
    `-- assets/
```

`src/` 是便于评审核对的分类源码视图。`board/vivado_project/` 是可直接打开的工程副本，其中 RTL 与 `src/` 对应文件内容一致。

提交包同时保留 `synth_1` 和 `impl_1` 两个完整运行目录。打开工程后可以直接查看已完成的综合、布局布线和位流生成结果。

## 环境要求

| 项目 | 要求 |
| --- | --- |
| FPGA | `xc7a200tfbg484-2` |
| 开发板 | 正点原子达芬奇Pro |
| Vivado | 2026.1 |
| 摄像头 | OV5640，1024x768 RGB565 |
| 显示 | HDMI 1024x768 |
| 串口 | CH340C，115200 8N1，无流控 |
| 上位机 | Python 3.10+、Tkinter，随包提供 pyserial |
| 仿真 | Icarus Verilog 11+；可选 Vivado XSim |

## 快速复现

### 1. 打开 Vivado 工程

打开：

```text
board/vivado_project/prj/ov5640_hdmi_median_filter.xpr
```

顶层：

```text
ov5640_hdmi_median_filter
```

全部 73 个工程文件引用已检查，均可由提交包内的相对路径解析。
综合与实现运行目录随包提供，Vivado 工程不应再显示为 `NoStart`。

### 2. 一键生成比特流

在设置好 Vivado 环境变量后运行：

```powershell
cd build
.\run_build.bat
```

或者从 Vivado Tcl Console 执行：

```tcl
source build/tcl/build_submission.tcl
```

构建脚本将重新运行综合、实现、时序检查、DRC 和比特流生成，并覆盖 `build/reports/` 与 `build/bitstream/` 中的本包副本。

### 3. 运行仿真

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File .\sim\scripts\run_iverilog_tests.ps1
```

日志写入 `sim/results/`。

### 4. 上板

参考 [board/README.md](board/README.md)。烧录完成后连接 OV5640、HDMI 和 CH340，再运行：

```text
board/host_gui/launch_host_gui.bat
```

## 当前提交状态

- 当前路径缓冲区同步修复版已完成综合、实现和比特流生成。
- 实现时序：WNS `+0.120 ns`，WHS `+0.026 ns`，TNS/THS 为 0。
- 资源：LUT 90,646；寄存器 45,177；Block RAM Tile 261；DSP 101。
- 路径规划计算结果在 `frame_pulse` 边界提交，避免规划完成时直接切换显示缓冲造成帧内路径混合。
- 修复版位流生成于 2026-11-01 20:47，提交前仍需按板级测试清单复核。
- 上板验证前应先运行最新比特流并复测 Snapshot、Live、Clear、颜色模式和轨迹功能。

当前比特流：

```text
build/bitstream/fpga_vision_route_fix.bit
```

SHA-256：

```text
58DC7FD748019EF113D13F024328F917EEFEDF05F8F8A4498BBD5311AD5DEDB0
```

## 注意事项

- RGB 阈值对光照、阴影、反光和摄像头白平衡敏感。
- 相互接触或重叠的同色物体会合并为一个连通域。
- 当前最多向下游输出 16 个候选目标。
- 路径显示与实体车辆控制相互独立。
- 报告中未填写的实测数据应以正式上板结果为准，不应把仿真或综合结果表述为板级识别率。
- 第三方摄像头、DDR3 和 HDMI 驱动来源应在正式报告中按赛事要求注明。
