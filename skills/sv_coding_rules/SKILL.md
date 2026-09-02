---
name: sv_coding_rules
description: >-
  Use this when splitting FPGA modules or writing/reviewing/refactoring
  synthesizable SystemVerilog RTL after architecture is chosen
  (sv_coding_rules). Defaults: sys_clk, sys_rst_n, asynchronous active-low
  reset. Three-process FSM with cs_state/ns_state (never bare cs_/ns_).
  Multi-FSM: cs_<machine>/ns_<machine>. Explicit if/else if arcs,
  i_/o_/w_/r_/s_/m_ names, SPEC comments, o_dbg_cs_state keep ports, vendor IP
  for async FIFO/RAM/switch. Use on
  模块划分、编码、重构、状态机、三段式、cs_state、sys_clk、sys_rst_n、异步复位、命名、注释、IP核. Do not use
  during open-ended architecture exploration, XDC/timing closure, UVM, or board
  bring-up.
---
# sv_coding_rules

在**架构已经选定之后**，做模块拆分、写 SystemVerilog DUT、审 RTL、按家规改代码时用本 skill。架构阶段不要用本 skill 去卡层次模板或八件套交付。

编码风格细枝可与 [FPGA RTL Style](sand-workflow:fpga-rtl-style) 对照；**与本文件冲突时以本文件为准**。尤其 FSM 信号名：禁止 `state_ff` / `state_ns`，必须用 `cs_state` / `ns_state`（或多机带机名）。

## 铁律

1. 不为千分之一的功能边角堆状态机；异常可观测、复位能空管即可。CDC、握手死锁、写冲突丢事件不算边角。
2. 同一 `logic` 只在一个过程块里赋值。
3. 跳转条件要能读回规格原文，禁止用远离 FSM 的 `*_trig` 把弧藏起来。
4. 不假绿：监控口不能被综合吃掉；不靠 `false_path` 代替同步器。
5. 有状态机就必须 **三段式**：第一段只改 `cs_*`，第二段只改 `ns_*`，其余全在第三段。禁止影子次态、禁止第二段夹带 sticky/计数/数据。
6. FSM 信号必须带完整名字：单机 `cs_state` / `ns_state`；多机 `cs_<机名>` / `ns_<机名>`。禁止光秃秃的 `cs_` / `ns_`，禁止 `state_q` / `st` / `w_ns_` / `state_ff` / `state_ns`。
7. **时钟/复位默认：** 时钟端口 `sys_clk`，复位端口 `sys_rst_n`，**异步低电平复位**。时序块写成 `always_ff @(posedge sys_clk or negedge sys_rst_n)`，复位分支 `if (!sys_rst_n)`。不要默认 `clk` / `rst_n`，不要默认同步复位。

## 不要约束架构

架构只要求能回答：几个时钟域、数据面/控制面怎么切、接口是脉冲还是握手、吞吐延迟是否算得过来。不要在架构阶段强制三层目录、完整文件清单或过早锁 IP 型号。

下面全部是**拆模块 + 编码**的硬规矩。

---

## 1. 复杂模块先画 FSM

- 单个模块里有多步动作时，先写状态与弧，再写第三段数据通路。
- **命名（写死）：**
  - 模块内只有一个状态机：必须 `cs_state` / `ns_state`。
  - 模块内多个状态机：`cs_<机名>` / `ns_<机名>`，例如 `cs_ddr_wr` / `ns_ddr_wr`。机名小写蛇形，表明职责。
  - `cs_` / `ns_` 只是前缀，不是完整信号名。
- 只画核心路径。异常：第三段里 sticky 可观测 + 复位空管。不要为极低概率功能场景加一串恢复态。
- 非法编码：第二段 `default` **只**把次态拉回默认态（通常 IDLE）；`err_sticky` 放第三段，禁止在第二段里写 sticky。
- **禁止第二套状态变量**（`st`、`state_q`、`state_ff` 与 `cs_state` 并行）。次态唯一来源是 `ns_state`（或多机对应 `ns_<机名>`）；第三段按当前态分支（需要提前量时可读次态）。

## 2. 三段式 FSM（硬性）

下列示例按**单机**写 `cs_state` / `ns_state`。多机时把名字换成 `cs_<机名>` / `ns_<机名>`，每台机自己的第一段、第二段，第三段可按机拆块。

### 第一段 — 时序，只更新当前态

一个 `always_ff`，**除该机的 `cs_*` 外不准写任何别的信号**。默认异步低有效：

```systemverilog
always_ff @(posedge sys_clk or negedge sys_rst_n) begin
    if (!sys_rst_n)
        cs_state <= ST_IDLE;
    else
        cs_state <= ns_state;
end
```

本段仍然只能出现该机的 `cs_*`。

### 第二段 — 组合，只更新次态

一个 `always_comb`，**除该机的 `ns_*` 外不准写任何别的信号**（包括 `w_illegal`、计数使能、`w_go_drop`、数据通路）。

- 开头 `ns_state = cs_state`（保持）。
- `case (cs_state)` 分态；每个态内有效弧用 `if` / `else if` 显式写出。
- `else` / `default` 只允许保持或回默认态。禁止从 `else` 飞进新业务态。
- 跳转条件尽量保留规格原始组合（如 `i_s_valid && i_s_last && !r_err`）。禁止用 `s_fire`、`*_trig` **整段取代**弧条件。
- 允许：紧挨第二段上方用连续赋值写 `w_sof = i_s_valid && i_s_ready && i_s_last`（规格词 = 原始项），弧上用 `w_sof`。该赋值不属于第二段过程块。
- 禁止：远处边沿检测做出 `*_trig`，第二段只吃脉冲。
- `w_s_fire` 给第三段装载/计数，不要当弧的唯一条件。

### 第三段 — 状态下对其余信号的控制

不要求只有一个过程块。凡在当前态下对**其他**信号的控制都算第三段：数据通路、计数、握手、`dbg`、`err_sticky`、skid 等。按「逻辑相似、同一套复位/使能」拆成多个 `always_ff` / `always_comb`。

- 用 `cs_state`（或已算好的 `ns_state`；多机用对应名字）做分支，不要另造 `st`。
- sticky / 非法态检测写在这里。
- 第三段里每个 `always_ff` 同样默认 `@(posedge sys_clk or negedge sys_rst_n)`，`if (!sys_rst_n)`。

## 3. always 分组（第三段 + 无 FSM 的模块）

- 禁止一个 `always_ff` / `always_comb` 塞进全部信号。
- 逻辑相似、同一套复位/使能语义的信号放同一块。
- 差别大的拆开。
- **绝对禁止**同一信号在多个块中赋值。当前态与次态是两个信号；第一段和第二段各写各的，不算违规。
- 拆块后每块复位必须一致（默认都是异步低有效 `sys_rst_n`）；同一级流水使能必须一致。
- 第一、二段的「只准一个目标信号」优先于本条的「相似信号可以同块」——不要把 sticky 塞进第一或第二段凑分组。

## 4. 命名

| 对象 | 前缀 / 全名 | 说明 |
|------|-------------|------|
| 默认时钟 | `sys_clk` | 不要默认 `clk` |
| 默认复位 | `sys_rst_n` | 异步、低有效；不要默认 `rst_n` 或同步复位 |
| 模块输入 | `i_` | `sys_clk` / `sys_rst_n` 用专用名，不加 `i_` |
| 模块输出 | `o_` | |
| 组合网 | `w_` | 表示本拍组合，不是 Verilog `wire` 关键字 |
| 时序寄存 | `r_` | 家规未另指定寄存前缀时的默认，与 `w_` 必须可区分 |
| 单机当前态 / 次态 | `cs_state` / `ns_state` | 禁止光秃 `cs_` / `ns_` |
| 多机当前态 / 次态 | `cs_<机名>` / `ns_<机名>` | 如 `cs_ddr_wr`；机名小写蛇形 |
| slave / master 接口 | `s_` / `m_` | 只表示角色 |

组合顺序：**方向 → 角色 → 名字**，例如 `i_s_valid`、`o_m_data`。内部连线用 `w_` / `r_`，不要再叠 `i_`。

信号**集中声明**，用分块注释标明这一组的大致功能。

端口顺序：`sys_clk`、`sys_rst_n` 永远是前两个。

## 5. 注释

- 每个过程块上方写**功能目标**，不要把代码翻译成中文。
- 有文档约束时必须可追溯，格式：`// SPEC: <doc-id> §x.y` 或需求/issue 号。无文档写 `// SPEC: none`。只追约束（拍数、协议、复位），不抄整章 SRS。
- 例化上方一行说明该实例干什么。
- 例化的每个接口旁标注形态：`握手` 或 `脉冲`（脉冲须注明相对哪路时钟、是否单拍）。需要时加 `电平` / `sticky`。`sys_clk` / `sys_rst_n` 标 `时钟` / `复位(异步低)`。只标模块边界。

## 6. 监控口

- 每个模块保留一组状态监控输出，至少：`o_dbg_cs_state`（多机则 `o_dbg_cs_<机名>`）、`o_dbg_err_sticky`、关键溢出/满。
- 上层例化**可以先不接功能逻辑**，但这些网必须保留：`(* keep = "true" *)` 或 `mark_debug`，禁止被综合优化掉。
- 不要每模块甩超宽杂项总线；先小束标准口，需要再加。

## 7. IP 核

- **必须用 IP（或仓库唯一封装）：** 异步 FIFO、BRAM/URAM、跨时钟桥、多主 switch / 交叉开关。业务模块禁止手写灰码指针 FIFO。
- **允许 RTL：** 同钟 1～2 深 skid、同钟小缓冲、很小的 LUT 表、二选一 mux。
- 例化 IP 时同样写功能注释 + 接口形态（FIFO 几乎都是握手）。

## 8. 模块边界其他

- 一个模块一个时钟域做运算；跨时钟只实例化第 7 节的桥/IP。
- 流接口默认握手；脉冲必须在边界标清，且置位优先于同拍清除（必达事件）。
- 默认异步低有效复位，释放后由本域时序采样。跨域复位仍要每域同步释放（可在 `_top` 用复位同步器 IP/封装），不要把第二套运算钟散进业务模块。

---

## 改已有代码时怎么做

1. 只选**一段**（一个模块或一条清晰数据路径），不要整仓格式化。
2. 先列违规对照表（哪条家规、现在怎么写、改成什么）。
3. 有 FSM 的先改成三段式：第一段只当前态，第二段只次态；单机改名为 `cs_state` / `ns_state`（多机带机名）；时钟复位改成 `sys_clk` / `sys_rst_n` 异步低有效；再拆第三段。
4. 删掉影子状态（`st`、`state_q`、`cs_`/`ns_` 光秃名、`state_ff`）。
5. 不改架构分层，除非当前文件把两个时钟域的运算写在同一模块里（编码违规，允许把桥拎出去）。
6. 交付：对照表 + 改后关键片段 + 未改项（例如上层仍不接 `o_dbg_*`）。

## 审查打回

- 巨型 always；同一信号多块驱动
- 第一段写了当前态以外的信号；第二段写了次态以外的信号（含 `w_illegal`、计数、数据）
- FSM 用光秃 `cs_` / `ns_`，或 `state_q` / `st` / `w_ns_` / `state_ff` / `state_ns`
- 存在与 `cs_state`（或多机 `cs_<机名>`）并行的影子状态变量
- 默认时钟不是 `sys_clk`、默认复位不是 `sys_rst_n`，或时序块写成同步复位 / 高有效
- 跳转靠 `else` 进业务态；弧上只剩 `*_trig`
- 端口无 `i_`/`o_`（`sys_clk` / `sys_rst_n` 除外）；接口角色与方向前缀乱序
- 例化无功能注释或未标脉冲/握手
- 手写异步 FIFO；监控口无 keep 被优化
- 为极罕见功能边角加一串恢复态，却没有 sticky 可观测错误
