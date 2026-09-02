# FPGA / 数字 IC 可复制 Skill 盘点

- 盘点日期：2026-09-02（Asia/Shanghai）
- 方法：打开 GitHub raw SKILL.md / README、官方 UG HTML；GitHub API 本次 403，星标取自当日 GitHub 仓库页可见数字。未拿到精确星标则不写。
- 原则：不编造仓库名、文档号、星标。sunburst-design.com/papers 当日被重定向到 Paradigm Works 登录墙，Cummings 原文改用仍可打开的镜像/第三方 PDF。
- 未找到：Cursor Marketplace / awesome-cursor-skills 中独立 FPGA skill（spencerpauly/awesome-cursor-skills 无 FPGA 条目）；Windsurf / GitHub Copilot / Continue 无同等 SKILL.md 包。官方 Cursor 技能规范见 https://docs.cursor.com/agent/skills 与 https://agentskills.io 。

---

## RTL 编码风格

### 1. lowRISC Verilog Coding Style Guide
- **来源 URL**：https://github.com/lowRISC/style-guides/blob/master/VerilogCodingStyle.md ；仓库 https://github.com/lowRISC/style-guides （493★，GitHub 页 last push 2025-11-06）
- **适用场景**：Comportable IP / OpenTitan 风格 SV RTL；团队统一命名、禁止特性、Verible 格式化基线。
- **值得借鉴的步骤结构**
  - 默认 C-like / 对齐 Google C++ 风格。
  - 优先 SystemVerilog-2017，列出禁止特性。
  - 模块声明、参数、信号命名、always_ff / always_comb 分离。
  - 可综合 vs testbench 分流。
- **缺陷**：ASIC/硅根信任项目导向，不是 FPGA 时序/XDC skill；无 SKILL.md。

### 2. OpenTitan 硬件方法论 + 风格强制
- **来源 URL**：https://opentitan.org/book/doc/contributing/hw/methodology.html ；https://github.com/lowRISC/opentitan （3,621★）；Comportability https://opentitan.org/book/doc/contributing/hw/comportability/index.html
- **适用场景**：把风格写成「PR 前必须过 lint」的工程纪律。
- **值得借鉴的步骤结构**
  - SV 限制到 lowRISC style guide。
  - Verilator 语义 lint（优先改代码，waiver 要审）。
  - Verible 风格 lint + 自动 format。
  - CDC：维护已验证跨时钟子模块清单 → 强制使用 → 签核级 CDC 工具扫全设计（文档写明工具未在开源仓定稿）。
- **缺陷**：CDC/RDC 签核工具离线专有（AscentLint 等），开源侧无 Spyglass 替代 playbook。

### 3. rtl-skills CodingStyle.md（可直接当 agent 合同）
- **来源 URL**：https://raw.githubusercontent.com/phamcuong21478/rtl-skills/main/skills/shared/CodingStyle.md ；仓库 https://github.com/phamcuong21478/rtl-skills （10★）
- **适用场景**：Claude/Cursor skill 库的「编码合同」；Verilog-2005 DUT + SV TB 拆分。
- **值得借鉴的步骤结构（文件真实条目）**
  - §0 语言：RTL 仅 IEEE 1364-2005；TB 可用 SV。
  - §1 ANSI 端口；`clk`/`rst_n` 前两端口；`RS_LV` 末参数，默认同步低有效。
  - §3 `_ff` / `_ns` / pipeline `sN_` / bus `m_*` `s_*`。
  - §5 同步复位 `if (rst_n == RS_LV)` + 非阻塞。
  - §14 `default_nettype none`、禁 latch、禁混用阻塞、one-shot 清零优先级。
  - §15 CDC：单 bit 2FF；多 bit gray/FIFO/握手；禁总线逐 bit 2FF。
  - §16 分离 `*_state_ff` / `*_state_ns`。
  - §18 `//@` 方向标记生命周期（vdesign 写、vfill 吃掉）。
  - §19 TB 唯一裁决 token：`[FINISH] PASS|FAIL`。
  - §21 用 verible-format + verible-lint + Verilator 进 CI。
- **缺陷**：Verilog-2005 过严（FPGA 团队常用 SV）；命名带项目前缀 `fgbw_`；星标低。

### 4. mindrally `fpga` SKILL.md
- **来源 URL**：https://raw.githubusercontent.com/mindrally/skills/main/fpga/SKILL.md ；https://github.com/mindrally/skills
- **适用场景**：给 Cursor/Claude 的短系统提示（`npx skills add https://github.com/mindrally/skills --skill fpga`）。
- **值得借鉴的步骤结构**
  - 模块化 → 同步设计（偏好同步复位）→ 早写 XDC → STA → 流水/多周期。
  - 资源：LUT/FF/BRAM、厂商 IP、综合策略。
  - 调试：自检 TB、SVA、行为/后仿、ILA。
  - 进阶：CDC 同步器/FIFO、AXI 握手、DMA burst、延迟 vs 吞吐。
- **缺陷**：条目级原则，无 Tcl/命令、无验证门；`reg []` 推断 RAM 的写法过时。

---

## CDC / 复位 / 时钟

### 5. oh-my-fpga `cdc-audit` SKILL.md（最完整的 CDC agent 剧本）
- **来源 URL**：https://raw.githubusercontent.com/LNC0831/oh-my-fpga/main/skills/cdc-audit/SKILL.md
- **适用场景**：Vivado + SynthPilot MCP 上做跨时钟审计与最小修复。
- **值得借鉴的步骤结构（摘自文件）**
  - 先决：`test_connection` → 打开工程 → `get_all_clocks`（缺时钟当 blocker）→ 有综合网表。
  - 1 库存时钟：`get_all_clocks` / `report_clock_interaction`。
  - 2 RTL `check_cdc_lint`；网表 `report_cdc`。
  - 3 分类 C1–C9：无同步器 1bit、已有 2FF、总线被逐 bit sync（视为 bug）、async FIFO、RDC、再汇聚、准静态、实际同源、IP 内部 CDC。
  - 4 修复阶梯：已安全只加约束 → 缺同步器推荐 RTL（不偷偷改）→ 准静态才可 waive 并写假设。
  - 5 同一工具复测；RTL 改完必须再综合。
  - 6 签核：post-impl `report_cdc` + `report_methodology` + `set_bus_skew` 过时序。
  - 硬规则：**约束 ≠ 正确性**；禁止用 `set_false_path` 掩盖未同步交叉。
- **缺陷**：绑定 SynthPilot 工具名；无 MCP 则步骤不能原样执行，但分类表可独立复用。

### 6. ZipCPU 异步 FIFO / 跨时钟（可操作工程文）
- **来源 URL**：
  - https://zipcpu.com/blog/2018/07/06/afifo.html
  - https://zipcpu.com/blog/2020/10/03/tfrvalue.html
  - https://zipcpu.com/formal/2018/05/31/clkswitch.html
  - https://zipcpu.com/blog/2017/07/29/fifo.html
- **适用场景**：自己写 AFIFO、2FF、两相握手；用形式化检查多时钟。
- **值得借鉴的步骤结构**
  - 读写指针分域；full/empty 必须跨时钟 → Gray + 2/3FF。
  - 流数据用 AFIFO；低频控制字用两相握手（约 10 拍，非低延迟吞吐方案）。
  - 形式化：多时钟环境、复位同步释放假设、BMC/归纳。
  - 作者明确：一般反对异步复位，但跨时钟两侧常需要异步复位以保证同时复位。
- **缺陷**：博客非 SKILL.md；部分观点与「FPGA 数据通路禁异步复位」需按场景取舍。

### 7. Cliff Cummings 复位 / FIFO / 多时钟（工程清单级论文）
- **来源 URL（当日仍可打开的 PDF）**：
  - 复位 Part Deux：https://www.trilobyte.com/pdf/CummingsSNUG2003Boston_Resets_rev1_2.pdf
  - Async FIFO（高校镜像，搜索命中）：http://staff.ustc.edu.cn/~wyu0725/FPGA/snug_collection/Clifford%20E.%20Cummings%27%20Paper/03.FIFO/Simulation%20and%20Synthesis%20Techniques%20for%20Asynchronous%20FIFO%20Design%20with%20Asynchronous%20Pointer%20Comparisons.pdf
  - 目录页（需登录）：https://www.sunburst-design.com/papers/
- **适用场景**：复位同步器、Gray FIFO、多异步时钟脚本；HFT 编码规则的原始来源。
- **值得借鉴的步骤结构**
  - 使用异步复位的 ASIC/**每个**设计应含复位同步器：异步置位、同步释放。
  - FIFO Style #1：Gray 指针同步后再比 full/empty；复位同时清两端指针。
  - 多时钟：同步器 + 正确 SDC 例外，而不是靠 STA 碰巧过。
- **缺陷**：sunburst 官网当日登录墙；镜像稳定性未知；ASIC 色彩浓，FPGA LUT 异步复位有额外注意点（文中亦提到 FPGA LUT）。

### 8. OpenTitan lint/CDC 页
- **来源 URL**：https://opentitan.org/book/hw/lint/index.html
- **适用场景**：FuseSoC `lint` target 模板（Verilator `-Wall` + Verible + AscentLint waiver 文件集）。
- **值得借鉴的步骤结构**
  - 同一 `lint` target 切换工具。
  - waiver 分 `files_verilator_waiver` / `files_veriblelint_waiver`。
  - CDC 文案：DV 理想时序掩盖真实亚稳态 → 必须独立 CDC 策略。
- **缺陷**：开源仓无 CDC 工具 runbook（TODO）。

---

## 时序闭合 / XDC-SDC

### 9. oh-my-fpga `timing-closure` SKILL.md
- **来源 URL**：https://raw.githubusercontent.com/LNC0831/oh-my-fpga/main/skills/timing-closure/SKILL.md
- **适用场景**：Vivado 迭代关时序的 agent。
- **值得借鉴的步骤结构**
  - Step0 基线：`run_synthesis`/`impl` → `report_timing_summary` → `extract_timing_metrics`；签核必须 post-impl WNS≥0。
  - Step1 诊断：关键路径拆 logic/net/skew；`report_clock_interaction`。
  - Step2 分类 A–E：缺例外 / 缺 I/O delay / 高扇出 / 真实逻辑深度 / 最后一公里策略。
  - Step3 最小安全改动：约束/策略先于 RTL；一类改动一轮。
  - Step4 复测；Step5 停止条件：闭合 / 无安全招 / 连续 2 轮无改善 / 预算用尽。
  - 禁止假绿：不得用 false_path 藏真实路径。
- **缺陷**：SynthPilot 专有工具名。

### 10. FPGA-Agent-skills `vivado-constraints`（UG903 决策树 SKILL）
- **来源 URL**：https://raw.githubusercontent.com/Shinei-Nouzen-Arch/FPGA-Agent-skills/main/vivado-constraints/SKILL.md ；仓库 https://github.com/Shinei-Nouzen-Arch/FPGA-Agent-skills （158★）
- **适用场景**：写 XDC 的 agent；三层 SKILL.md + REFERENCE.md + examples。
- **值得借鉴的步骤结构（摘自文件）**
  - 综合/实现 XDC 分离：`USED_IN_SYNTHESIS` / `USED_IN_IMPLEMENTATION`。
  - IP 约束 `SCOPED_TO_REF`。
  - 主时钟建在 **输入端口** 而非 BUFG 输出；差分只在 P 脚建时钟。
  - MMCM/PLL 输出自动派生，用户分频才手写 `create_generated_clock`。
  - `set_clock_groups -asynchronous` 优先于双向 `set_false_path`。
  - 多周期：同频 / 慢→快 / 快→慢 的 setup/hold 配对表。
  - CDC：`set_max_delay -datapath_only`；握手 `set_bus_skew = N_sync * dst_period`；Gray FIFO skew = dst_period。
  - 例外优先级：clock_groups > false_path > max/min_delay > multicycle。
  - XDC 书写顺序 1–6（先 prune 再时钟再 I/O 再例外）。
  - 校验：`check_timing`、`report_exceptions -coverage`、`report_clock_interaction`、`report_methodology`。
- **缺陷**：基于 UG903 v2025.2 摘要，需对照当前 Vivado 版本文档；不驱动工具。

### 11. AMD UltraFast UG949 + 检查表
- **来源 URL**：
  - UG949：https://docs.amd.com/r/en-US/ug949-vivado-design-methodology
  - About：https://docs.amd.com/r/en-US/ug949-vivado-design-methodology/About-the-UltraFast-Design-Methodology
  - DRC：https://docs.amd.com/r/en-US/ug949-vivado-design-methodology/Using-the-UltraFast-Design-Methodology-DRCs
- **文档 ID**：UG949；配套 UG1231（Quick Reference）、UG1292（Timing Closure Quick Reference）、XTP301（Checklist 表格）。
- **适用场景**：官方最佳实践总纲；`report_methodology` 规则库。
- **值得借鉴的步骤结构（官方列出）**
  - 系统级流程图（Documentation Navigator 可点步骤）。
  - `report_methodology` 三阶段：综合前 RTL 结构；综合后网表+约束；实现后约束与时序。
  - UG1292 要点（UG949 引用）：初始检查 → baseline → 消违例。
- **缺陷**：docs.amd.com 门户页对部分抓取器只返回壳页，应用浏览器打开正文；XTP301 是 spreadsheet，不是 markdown skill。

### 12. AMD UG903 Using Constraints
- **来源 URL**：https://docs.amd.com/r/en-US/ug903-vivado-using-constraints
- **适用场景**：XDC/SDC 权威语法与约束方法论。
- **值得借鉴**：与条目 10 的决策树对应；skill 库应「SKILL 决策 + UG903 为 REFERENCE」。
- **缺陷**：长手册，agent 不能整本塞进上下文。

### 13. Intel Quartus Timing Analyzer Cookbook（文档 683081）
- **来源 URL**：https://www.intel.com/content/www/us/en/docs/programmable/683081/current/timing-analyzer-cookbook.html
  - 资源中心：https://www.intel.com/content/www/us/en/support/programmable/support-resources/design-software/sof-qts-timinganalyzer.html
  - 历史 PDF：https://www.intel.com/content/dam/www/programmable/us/en/pdfs/literature/manual/mnl_timequest_cookbook.pdf
  - SDC API：资源中心链到 SDC and Timing Analyzer API Reference
  - Quick Start 683588：https://www.intel.com/content/www/us/en/docs/programmable/683588/17-1/step-2-specify-clock-constraints.html
- **适用场景**：QSF/SDC、`create_clock`/`create_generated_clock`、I/O delay、clock groups；`quartus_sta` Tcl。
- **值得借鉴的步骤结构（Quick Start 实测条目）**
  - 打开示例 → Compiler 出网表 → Timing Analyzer 选 snapshot/delay model。
  - 编辑 `.sdc`：`create_clock -period` + `-waveform`。
  - Read SDC File → Update Timing Netlist。
  - 后续步骤（教程 TOC）：看时钟、再加 input/output delay、报 top failing paths。
- **缺陷**：Cookbook 与 Pro/Standard 版本文档分裂；部分旧 PDF 路径仍可用但版本滞后。

---

## 仿真验证 (Verilator / cocotb / UVM)

### 14. Gateflow `gf-lint` + `gf` 编排
- **来源 URL**：
  - https://raw.githubusercontent.com/codejunkie99/Gateflow-Plugin/main/plugins/gateflow/skills/gf-lint/SKILL.md
  - https://raw.githubusercontent.com/codejunkie99/Gateflow-Plugin/main/plugins/gateflow/skills/gf/SKILL.md
  - 仓库 https://github.com/codejunkie99/Gateflow-Plugin （109★）；Cursor 变体 https://github.com/DGGua/Gateflow-Plugin-Cursor
- **适用场景**：开源 SV：问需求 → 规划 → 并行 codegen → Verilator lint → sim → 最多 3 次修复。
- **值得借鉴的步骤结构**
  - `which verilator` 否则 `verible-verilog-lint`，都没有则 STATUS:ERROR，禁止假装 lint。
  - `verilator --lint-only -Wall`；解析 `%Error` / `%Warning-*`。
  - 固定块 `---GATEFLOW-RESULT---`：STATUS/ERRORS/WARNINGS/FILES/DETAILS。
  - 编排：AskUserQuestion → sv-planner → sv-orchestrator → gf-lint → gf-sim；FAIL 只派 sv-refactor/sv-debug，禁止 orchestrator 直接改码。
  - bug 修复：先写必失败的复现 TB，再改 DUT。
- **缺陷**：面向 Yosys/Verilator 小模块，不是 Vivado 时序签核；强制「永远用子 agent」在 Cursor 里成本高。

### 15. Gateflow `gf-formal`
- **来源 URL**：https://raw.githubusercontent.com/codejunkie99/Gateflow-Plugin/main/plugins/gateflow/skills/gf-formal/SKILL.md
- **适用场景**：自然语言 → SVA + SymbiYosys。
- **值得借鉴的步骤结构**
  - 检测 `sby`；BMC/Prove/Cover `.sby` 模板。
  - 属性库：FIFO 溢出/下溢、valid/ready 稳定、one-hot、liveness、复位。
  - 反例：BMC 可达则改设计；prove 失败 BMC 过则加 invariant 或 `abc pdr`。
- **缺陷**：`pip install symbiyosys` 的安装提示不完整（通常需 OSS CAD Suite）。

### 16. rtl-skills `vfill`（实现→lint→仿真门）
- **来源 URL**：https://raw.githubusercontent.com/phamcuong21478/rtl-skills/main/skills/vfill/SKILL.md
- **适用场景**：从已批准 proposal 填 Verilog，双工具 lint，自检 TB。
- **值得借鉴的步骤结构**
  - 缺工具则 BLOCKED，不伪造结果。
  - 有 `//@` 才全量填；已填充则增量改。
  - Pass1 `verilator --lint-only --top-module … -y …`
  - Pass2 `xvlog` + `xelab`（显式 filelist，缺子模块硬错误）。
  - TB：`$error` + 唯一 `[FINISH] PASS/FAIL`；`xvlog/xelab/xsim --runall`；写 `run.sh`。
  - Self-check 清单：无 `//@`、Verilog-2005、`default_nettype`、双 lint、仿真、文档 as-built。
- **缺陷**：Xilinx xsim 依赖；DUT 禁 SV。

### 17. 开源仿真/验证栈（高星，作工具层而非 SKILL.md）
| 名称 | URL | 星标（仓库页 2026-09-02） | 借鉴点 | 缺陷 |
| Verilator | https://github.com/verilator/verilator | 3,890★ | `--lint-only -Wall`；CI 语义 lint | 非完整 IEEE SV；时序仿真弱 |
| Icarus Verilog | https://github.com/steveicarus/iverilog | 3,617★ | Corundum/cocotb 常用事件仿真 | SV 支持有限 |
| cocotb | https://github.com/cocotb/cocotb | 2,486★ | Python TB；与 cocotbext-axi/eth/pcie 组成 NIC 验证 | 非 UVM 全功能 |
| pyuvm | https://github.com/pyuvm/pyuvm | 568★ | Python UVM 分层 | 生态小于 SV UVM |
| UVVM | https://github.com/UVVM/UVVM | 462★ | VHDL FPGA 验证库 | VHDL-only |
| uvm-core | https://github.com/accellera-official/uvm-core | 148★ | Accellera UVM 源 | 不是 FPGA skill；chipsalliance/uvm-core 当日 404 |
| svlint | https://github.com/dalance/svlint | 391★ | 规则文件可提交 | 风格为主 |
| Verible | https://github.com/google/verible | 1,926★ | format + style lint + LSP | 不管 CDC/时序 |
| Yosys | https://github.com/YosysHQ/yosys | 4,728★ | 开源综合/形式化前端 | UltraScale 时序签核不能替代 Vivado |

OpenTitan 的 Verilator lint 目标示例见 https://opentitan.org/book/hw/lint/index.html （`mode: lint-only` + `-Wall`）。

---

## 综合实现 / 工程流 Tcl

### 18. oh-my-fpga `full-flow-demo`
- **来源 URL**：https://raw.githubusercontent.com/LNC0831/oh-my-fpga/main/skills/full-flow-demo/SKILL.md
- **适用场景**：RTL→bit 编排；**签核在写 bitstream 之前**。
- **值得借鉴的步骤结构**
  - 先决：连接、工程、top、**约束数>0**（0 时钟禁止当 PASS）、license。
  - Gate0 语法/lint → Gate1 sim → Gate2 CDC lint → Gate3 synth（时钟存在、利用率<100%、网表 CDC）→ Gate4 impl 完成 → **Gate5 SIGNOFF：WNS 且 WHS≥0、100% routed、DRC、methodology** → Gate6 `generate_bitstream` 再确认。
  - 失败路由表：lint / sim-debug / cdc / timing-closure / 用户（架构/waiver）。
  - `run_full_flow` 快路径仍必须补跑 Gate5。
- **缺陷**：SynthPilot 绑定。

### 19. FPGA-Agent-skills 八技能（按官方 UG 拆分）
- **来源 URL**：https://github.com/Shinei-Nouzen-Arch/FPGA-Agent-skills ；README https://raw.githubusercontent.com/Shinei-Nouzen-Arch/FPGA-Agent-skills/main/README.md
- **技能 ↔ 文档**：vitis-hls-synthesis/UG1399；vivado-synth/UG901；vivado-constraints/UG903；vivado-impl/UG904；vivado-analysis/UG906；vivado-debug/UG908；vivado-sim/UG900；vivado-tcl/UG835+UG892。
- **适用场景**：把 AMD 手册变成 agent 决策表；Tcl 命令声称对 Vivado 2025.2 `-help` 核对过。
- **值得借鉴**：三层加载（决策 / 语法 / 按需 examples）；vivado-synth 带 UG901 HDL 模板 64 个文件。
- **缺陷**：作者 README 写账号被封；无 MCP 执行层。

### 20. Project F Vivado Tcl 非工程流
- **来源 URL**：https://projectf.io/posts/vivado-tcl-build-script/
- **适用场景**：CI/`vivado -mode batch -source build.tcl`；无 .xpr。
- **值得借鉴的步骤结构**
  - `read_verilog -sv` → `read_xdc` → `synth_design` → `opt_design` → `place_design` → `route_design` → `write_bitstream -force`
  - 烧录建议 openFPGALoader 而非 GUI。
- **缺陷**：教学级，无 QoR 循环、无 CDC。

### 21. 官方 Tcl
- **UG835 Tcl Command Reference**：https://docs.amd.com/r/en-US/ug835-vivado-tcl-commands
- **UG894 Using Tcl Scripting**：搜索命中 https://www.xilinx.com/support/documents/sw_manuals/xilinx2022_2/ug894-vivado-tcl-scripting.pdf （版本钉在 2022.2）
- **UG908 Programming and Debugging**：https://docs.amd.com/r/en-US/ug908-vivado-programming-debugging
- **适用场景**：batch、hook、ILA/VIO、hw_server。
- **缺陷**：命令百科，不是 workflow skill。

### 22. FuseSoC + Edalize
- **来源 URL**：https://github.com/olofk/fusesoc （1,453★）；用户指南 https://fusesoc.readthedocs.io/en/stable/user ；https://github.com/olofk/edalize （771★）
- **适用场景**：CAPI2 `.core` 包管理；同一描述打 Vivado/Quartus/Yosys/Verilator。
- **值得借鉴的步骤结构**
  - `fusesoc library add` → `fusesoc run --target=sim <core>` → `--tool=` 换仿真器。
  - Edalize：files + parameters + 工具选项 → 生成工程并 build/run。
- **缺陷**：不是编码风格 skill；Vivado IP/BD 深度不如厂商 Tcl。

### 23. 其它 Tcl/CMake 流水线
- TripRichert/viv-prj-gen：https://github.com/TripRichert/viv-prj-gen — CMake 包 Tcl，.xpr 不入库。
- aiclab-official/vscode-vivado-fpga：https://github.com/aiclab-official/vscode-vivado-fpga — VSCode tasks 调 Tcl。

---

## 板级调试 (ILA / SignalTap)

### 24. oh-my-fpga `ila-hw-debug`
- **来源 URL**：https://raw.githubusercontent.com/LNC0831/oh-my-fpga/main/skills/ila-hw-debug/SKILL.md
- **适用场景**：上板抓波形；sim 能复现则不要上 ILA。
- **值得借鉴的步骤结构**
  - 先决：MCP、工程、插入路径（HDL MARK_DEBUG vs BD System ILA）、hw_server、器件在链。
  - Phase A：最小探针集 → mark_debug → synth → `setup_debug` → 核对 cores → impl → DRC+时序 → bitstream + `.ltx`。
  - Phase B：program 必须 bit 与 ltx 配对；`hw_ila_list` 为空则停。
  - Phase C：读真实探针名 → set trigger → 回读条件 → arm → wait → `hw_ila_read_data` CSV 才算证据。
  - 决策表：连不上 / 无 ILA / 信号被优化 / 永不触发 / 错时钟域 / ILA 打断时序。
- **缺陷**：SynthPilot；SignalTap/Quartus 不在此 skill（见 fpga-mcp `q_set_signal_probe`）。

### 25. FPGA-Agent `vivado-debug`（UG908）
- **来源 URL**：仓库 README 表；技能目录应对应 `vivado-debug/SKILL.md`（与同仓 vivado-constraints 同结构）。
- **适用场景**：ILA/VIO/JTAG-to-AXI 选型、mark_debug、Versal debug。
- **缺陷**：本次完整打开的是 constraints SKILL；debug 正文需 clone 后读，不要凭 README 臆造步骤。

---

## IP / AXI / 流式接口

### 26. alexforencich Verilog AXI / AXIS / Ethernet / PCIe
- **来源 URL**：
  - https://github.com/alexforencich/verilog-axi （2,125★）
  - https://github.com/alexforencich/verilog-ethernet （3,078★）
  - https://github.com/alexforencich/verilog-pcie （1,654★）
  - AXIS 在 Corundum 文档引用 https://github.com/alexforencich/verilog-axis
- **适用场景**：可综合 AXI4/AXI-Stream、10G MAC/PCS、PCIe DMA 的开源实现与 cocotb 测试。
- **值得借鉴**：valid/ready、寄存器输出、参数化宽度；与 Corundum 同源，可当接口「黄金模型」。
- **缺陷**：无 SKILL.md；要自己抽编码规则。

### 27. AMD AXI4-Stream Infrastructure PG085
- **来源 URL**：https://docs.amd.com/r/en-US/pg085-axi4stream-infrastructure/Control-Register （OpenNIC README 引用）
- **适用场景**：AXIS switch/register slice；Alveo NIC shell 插用户逻辑。
- **缺陷**：IP 产品手册，不是编码风格。

### 28. oh-my-fpga `zynq-bringup`
- **来源 URL**：https://github.com/LNC0831/oh-my-fpga/blob/main/README.md （表内编排：PS7 BD → automation → AXI 外设 → 地址图 → validate → wrapper）
- **适用场景**：Zynq PS+PL；完整 SKILL 在 `skills/zynq-bringup/SKILL.md`（与 timing-closure 同仓，README 已列编排）。
- **缺陷**：未在本次把该 SKILL.md 全文拉下（应 clone 后收录）；依赖 SynthPilot BD API。

---

## 工具链 / MCP / agent skills

### 29. oh-my-fpga + SynthPilot（13 个命名 skill）
- **来源 URL**：https://github.com/LNC0831/oh-my-fpga （18★，MIT skill 包）；README https://raw.githubusercontent.com/LNC0831/oh-my-fpga/main/README.md
  - SynthPilot：https://github.com/LNC0831/SynthPilot ；站点 https://synthpilot.dev ；文档 https://www.synthpilot.dev/docs.html ；PyPI https://pypi.org/project/synthpilot/
- **13 个 skill（README v1，已用 raw 核实仓库含 skills/）**：timing-closure, cdc-audit, constraints-authoring, full-flow-demo, sim-bringup, coverage-closure, lint-triage, qor-report, utilization-reduction, power-optimization, zynq-bringup, ila-hw-debug, bitstream-program。
- **适用场景**：Claude Code plugin：`/plugin marketplace add LNC0831/oh-my-fpga`；Cursor 等用 SynthPilot 1.3.0+ 内置 MCP prompts（与 SKILL 同源）。
- **设计原则（可当所有 FPGA skill 的 meta-rail）**
  1. Verification-first：无新鲜工具输出不得宣称闭合。
  2. Never fake-pass。
  3. 最小安全改动：约束/策略先于 RTL。
  4. 安全选项用尽则停，交还人类。
- **缺陷**：MCP 主体专有（Free/Pro/Max，工具数文档在 414/500/510 间不一致，以当日 synthpilot.dev/platform-vivado.html 的 510 目录为准）；Windows 主线更成熟；技能编排绑定其工具名。

### 30. wmm246/fpga-mcp（多厂商开源 MCP + 7 个 methodology md）
- **来源 URL**：https://github.com/wmm246/fpga-mcp （仓库页当日显示 0★）；https://pypi.org/project/fpga-mcp/
- **适用场景**：一个 MCP 打 Vivado + Quartus + 安路；`methodology/*.md` 作 MCP prompts。
- **7 个 workflow**：full_flow, timing_closure, cdc_audit, resource_budgeting, sim_signoff, bitstream_handoff, soc_bringup。
- **步骤（README）**：`pip install fpga-mcp` → `fpga-mcp setup` → 在 EDA 内 source `tcl/*_server.tcl`（Vivado :9999, Quartus :9998, Anlogic :9997）。
- **缺陷**：新项目、星标 0；README 工具数 689 vs 文末自动索引 816 不一致；声明受 SynthPilot 启发，需自行审计许可/质量。

### 31. coreyhahn/vivado_mcp
- **来源 URL**：https://github.com/coreyhahn/vivado_mcp （60★）
- **适用场景**：持久 Vivado Tcl 会话（避免每条命令 30s 启动）。
- **步骤**：start_session → open_project → run synth/impl → get_timing_summary / get_utilization → run_tcl。
- **缺陷**：无 methodology SKILL；原始 Tcl 逃逸面大。

### 32. lcapossio/fpgaZeroMCP
- **来源 URL**：https://github.com/lcapossio/fpgaZeroMCP
- **适用场景**：OSS CAD Suite（Yosys/nextpnr/iverilog/Verilator）+ FuseSoC 核注册。
- **步骤（搜索命中 README）**：lint → sim → synth → pnr → program；`list_ip_cores` / `import_fusesoc_core` / GitHub MIT 核导入。
- **缺陷**：不覆盖 Vivado 签核；需读仓库确认许可白名单。

### 33. NellyW8/MCP4EDA 与 ssql2014/mcp4eda
- **来源 URL**：https://github.com/NellyW8/MCP4EDA/ ；论文 https://arxiv.org/abs/2507.19570 ；另一实现 https://github.com/ssql2014/mcp4eda
- **适用场景**：Yosys/Icarus/GTKWave/OpenLane 的 LLM 桥；偏 ASIC RTL-to-GDS。
- **缺陷**：FPGA 深度弱；克隆 URL 在 README 里写成 `mcp-EDA` 需小心；**ASIC-first**。

### 34. AMD 官方 AI / vivado-doc-server MCP
- **来源 URL**：
  - UG1400 AI Chat：https://docs.amd.com/r/en-US/ug1400-vitis-embedded/AI-Assistance-in-the-Vitis-Unified-IDE （文档 ID UG1400，页内日期 2026-07-31，版本 2026.1）
  - UG1702 Theia agents：https://docs.amd.com/r/en-US/ug1702-vitis-accelerated-reference/Using-AI-Features-of-the-Vitis-Unified-IDE
  - MCP：`https://vivado.amd.com/mcp/doc-search`（UG1702 示例 settings.json 的 `vivado-doc-server`）
  - 实践说明：https://bard0.com/insights/vivado-chatbot-practical-guide.html
- **适用场景**：IDE 内问文档；agent 检索 UG 而非编造 Tcl。
- **UG1400 步骤**：Settings 搜 AI → 填供应商 API key → View > AI Chat → 采用前人工审查、重建验证。
- **UG1702 agents**：Architect（只读）、Coder、Command、Orchestrator、Universal、User_Guide（走 AMD MCP）。
- **缺陷**：不跑综合/烧录；文档检索不是执行器；勿把专有 RTL 送进远程 MCP。

### 35. rtl-skills 全流（10 skill + 8 agent yaml）
- **来源 URL**：https://github.com/phamcuong21478/rtl-skills
- **流**：ip_req → varch → vdesign → vfill → vtestgen/vtestrun → vdebug → vsynth → vdoc；vflow 编排。
- **缺陷**：Verilog-2005；星标 10；Vivado synth 报告级。

### 36. 其它 Verilog generator skill
- Eriemon/verilog-generator：https://github.com/Eriemon/verilog-generator/ — Codex skill，Verilog-2001，强调「没跑过的检查不得报完成」（README）。
- SKILL.md 规范：https://github.com/agentskills/agentskills/blob/main/docs/specification.mdx

---

## 金融低延迟 / 网络 / PCIe

### 37. Corundum 开源 NIC（首选可克隆工程）
- **来源 URL**：https://github.com/corundum/corundum （仓库元数据 2,265★，组织页列表曾显示 2,358★；last push 2024-07-05）；文档 https://docs.corundum.io/en/latest/gettingstarted.html ；站点 https://corundum.io/
- **适用场景**：10/25/100G MAC+PCS、PCIe DMA、多队列、RSS、校验和卸载；Alveo 类板级。
- **值得借鉴的步骤结构（getting started）**
  - 依赖：cocotb + cocotbext-axi/eth/pcie + scapy + iverilog。
  - pytest / tox / cocotb makefile 跑全系统含驱动与 PCIe 模型。
  - 构建：Linux + 本机 Vivado（明确不建议 VM，RAM 大）。
  - 内部复用 verilog-axi/axis/ethernet/pcie。
- **缺陷**：主仓 2024-07 后较久未推；不是 HFT 订单簿；无 SKILL.md。**不要把 STAC 宣传当编码规则。**

### 38. OpenNIC shell（AMD Alveo 100G 壳）
- **来源 URL**：https://github.com/Xilinx/open-nic （334★）；shell https://github.com/Xilinx/open-nic-shell （148★）；手册 PDF https://github.com/Xilinx/open-nic/blob/main/OpenNIC_manual.pdf
- **适用场景**：Alveo U50/U55C/U200/U250/U280 等，最多两路 100G + 多 PF QDMA；用户逻辑挂 AXIS。
- **值得借鉴**
  - 250MHz box 插件 + AXIS switch（PG085）。
  - 构建脚本拉 Xilinx Board Store。
  - 100G CMAC 需免费 license。
- **缺陷**：README 钉 Vivado 2020.x–2022.1；偏吞吐壳，不是 tick-to-trade 延迟优化指南。

### 39. AMD Alveo 平台 / 以太网教程
- **来源 URL**：
  - UG1120 平台：https://docs.amd.com/r/en-US/ug1120-alveo-platforms/U250-Gen3x16-XDMA-4_1-Platform
  - XD099 GT+以太网：https://docs.amd.com/r/en-US/Vitis-Tutorials-Vitis-Hardware-Acceleration/Using-GT-Kernels-and-Ethernet-IPs-on-Alveo
- **适用场景**：QSFP 10/25/40/100G、XDMA/QDMA、GTY 在 RTL kernel 中的连接。
- **步骤（教程自述）**：RTL kernel 含 GTY → AXI-Stream 到其它 kernel/DRAM → v++ connectivity 把 `gt_refclk`/`gt_port` 接到平台 QSFP。
- **缺陷**：教程写明无真实功能；需 xxv_eth_mac_pcs 等 license；Vitis overlay 路径延迟通常差于纯 RTL 直连。

### 40. Intel Low Latency Ethernet 10G MAC
- **来源 URL**：用户指南文档 **683426** https://www.intel.com/content/www/us/en/docs/programmable/683426/23-3-22-0-3/about-ll-ethernet-10g-mac.html
  - Stratix 10 例程 **683026**
  - AN 849 超低延迟参考 **683671** https://www.intel.com/content/www/us/en/docs/programmable/683671/18-0/ultra-low-latency-ethernet-10g-reference.html （文内给出 round-trip 171.0 ns vs 普通例程 246.5 ns）
- **适用场景**：Intel 器件上 10GbE MAC+PHY 低延迟参考，含时钟/复位/Avalon-ST 接口说明。
- **缺陷**：专有 IP；AN849 版本 18.0 偏旧；不是开源 RTL。

### 41. 低延迟编码规则（从上述清单综合，供 skill 作者固化）

下列每条都能回溯到本盘点中的可打开文件，而不是经纪商宣传页：

1. **数据通路避免异步复位**；控制/跨时钟可用异步置位+同步释放（Cummings 复位文；rtl-skills §5 默认同步；ZipCPU 指出跨时钟例外）。
2. **输出寄存**；HFT 流水线每级寄存器输出，组合只在一级内（UG949/时序闭合 skill 的 class D：深组合只能流水，不能 false_path）。
3. **CDC**：1bit 控制 2FF；总线 **Gray / async FIFO / req-ack**；禁止总线逐 bit 2FF（oh-my-fpga C3；rtl-skills §15；ZipCPU afifo）。
4. **约束**：async 时钟 `set_clock_groups`；同步器入口 `set_max_delay -datapath_only`；Gray 指针 `set_bus_skew` ≈ 目的时钟周期（FPGA-Agent UG903 skill）。
5. **10GbE 时钟**：线路侧常为 156.25/322.265 MHz 量级（取决于 32b/64b 数据通路）；CDC 到 PCIe/DMA 域必须显式 FIFO，不能靠「差不多同源」。Corundum/OpenNIC/Intel LL MAC 都是分时钟域架构。
6. **PCIe DMA**：用成熟核（Corundum/verilog-pcie 或厂商 QDMA），scatter-gather + 寄存器门铃；不要在 DMA 数据路上混异步复位。
7. **测量**：ILA 只在 sim 无法复现时用；bit 与 ltx 必须配对（ila-hw-debug）。延迟数字必须以板上测量/仿真周期计，而不是 STAC 营销。

**刻意不收入**：STAC 基准产品页（无工程步骤）；Exablaze 闭源产品手册（非可克隆 skill）。NetFPGA 组织仓存在但本次未把 SUME 主 README 全文核到与 Corundum 同级，优先 Corundum/OpenNIC。

---

## 建议 skill 专家优先克隆的「骨架」

若目标是 **Cursor/Claude 可落地的 FPGA skill 库**，建议按层组装，而不是从论文重新发明：

| 层 | 建议源 | 作用 |
| Meta 安全轨 | oh-my-fpga README 四原则 | 禁止假绿、证据门 |
| 风格合同 | rtl-skills CodingStyle.md 精简 + lowRISC | 命名/复位/CDC/default_nettype |
| XDC 决策 | FPGA-Agent vivado-constraints | UG903 可执行表 |
| 时序/CDC/ILA/全流 | oh-my-fpga 对应 SKILL.md（去掉 SynthPilot 工具名，换成 Tcl） | 已有循环与分类表 |
| 开源执行 | Gateflow gf-lint/gf-sim/gf-formal | Verilator/sby 结构化结果 |
| 工程包 | FuseSoC + Project F Tcl | CI |
| 网络/PCIe | Corundum + verilog-ethernet/pcie | 10G/DMA 参考实现 |
| 官方检查表 | UG949 `report_methodology` + Quartus 683081 | 厂商签核 |

把厂商工具调用做成适配器：Vivado 用 UG835 Tcl 或 fpga-mcp/vivado_mcp；文档问答用 `vivado-doc-server`；禁止把「timing met」写进 skill 输出除非刚跑过 post-route 报告。
