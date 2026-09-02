---
name: FPGA RTL Style
description: >-
  Use this when writing or reviewing synthesizable FPGA/ASIC Verilog or
  SystemVerilog at the code level: ports, reset, always_ff/always_comb,
  valid/ready wiring, lint. For FSM signal names and three-process structure,
  defer to FPGA Module House Rules (cs_state/ns_state, not state_ff/state_ns).
  Do not use for architecture, module partitioning, clock-domain maps, or block
  diagrams. Do not use for XDC, timing closure, UVM benches, or board bring-up.
---
# FPGA RTL 编码风格

写、改、审可综合 DUT **代码**时按本 skill。架构、层次、时钟域图走 [FPGA Architecture Partition](sand-workflow:fpga-architecture-partition)。验证、XDC、时序、上板另走对应 skill。模块拆分与 FSM 家规走 [FPGA Module House Rules](sand-workflow:fpga-module-house-rules)；**FSM 信号名以家规为准**。

默认：**可综合 SystemVerilog**。仓库锁 Verilog-2005 时 DUT 去掉 SV 语法，TB 仍可用 SV。

## 铁律

1. **先验证再声称。** 没有仿真/lint 输出，不说“能综合、没 CDC 问题”。
2. **不假绿。** waiver 必须写原因并贴到行上。
3. **最小安全改动。** 先修边界与握手，再动算法。
4. **权衡后停手。** 面积/延迟/跨时钟无法两全时列出选项。

层次已在架构 skill 里定死：`_top` 接线，`_core` 单时钟数据面，跨时钟只实例化桥。这里只约束**怎么写**。

## 端口与复位

- ANSI 端口。`clk`、`rst_n` 永远前两个。
- 同步复位；数据通路不用异步复位。跨域复位释放在 `_top` 做异步置位、同步释放。
- 复位覆盖 FSM、valid、计数、握手。被 valid 限定的宽数据可选择不复位，必须注释。
- 模块边界输出优先寄存。
- 只用命名连接。

## 命名

| 对象 | 规则 | 例 |
|------|------|----|
| 模块 | 小写蛇形 | `pkt_parser` |
| 参数 | `UPPER_SNAKE`，宽度 `*_W`，个数 `*_N` | `ADDR_W` |
| 时序寄存器 | 后缀 `_ff` | `r_count` 或 `foo_ff` |
| 组合下一拍 | `_ns` | 非 FSM 用；FSM 不用这个 |
| 同步器 | `_meta_ff` 然后 `_sync_ff` | `req_meta_ff` |
| 端口 | `s_*` 从属，`m_*` 主 | `s_valid` |

家规另有 `i_`/`o_`/`w_`/`r_`。与本表冲突时跟家规。

## 时序 / 组合

- `always_ff @(posedge clk)` 用 `<=`；`always_comb` 用 `=`。禁止混块。
- `always_comb` 全路径赋值，`case` 必须 `default`。禁止锁存。
- 文件头 `` `default_nettype none ``。
- RAM/DSP 进叶子包装，用可推断模板或厂商原语。

## 握手

- `fire = valid && ready`。ready 链过长就插 skid。
- 单周期脉冲：置位优先于清除。
- `valid` 不得在 `ready==0` 时无故拉低（除非协议允许并注释）。

## CDC（写代码时）

审计走 cdc-audit。硬规则：1 bit 用 2FF；多 bit 禁止逐 bit 2FF；用灰码 FIFO 或 req/ack。约束代替不了硬件。

## FSM

**不要用 `state_ff` / `state_ns`。** 以 [FPGA Module House Rules](sand-workflow:fpga-module-house-rules) 为准：三段式；单机 `cs_state` / `ns_state`；多机 `cs_<机名>` / `ns_<机名>`。`typedef enum`，禁止裸数字。

## 审查打回

DUT 混 TB-only 语法；位置端口；推断锁存；混用 `<=`/`=`；总线逐 bit 同步；数据通路异步复位；魔数。

## 交付

1. 对照架构 skill 的模块树，只改本文件职责内的代码
2. 端口契约
3. 跨时钟只实例化已有桥
4. 建议 Verilator `-Wall`；waiver 行内写原因
