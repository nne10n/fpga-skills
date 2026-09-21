# 官方 IP Catalog IP 的创建、配置、生成与例化

基准：Vivado 2022.2（7 系列为例）。所有命令在 `vivado -mode batch -source xxx.tcl` 与 GUI Tcl Console 下均可运行。

## 1. 在 Catalog 里找到目标 IP

```tcl
# 模糊搜索（返回 VLNV 全名，如 xilinx.com:ip:clk_wiz:6.0）
get_ipdefs -regexp {.*clk_wiz.*}
get_ipdefs -regexp {.*fifo.*}

# 看某 IP 的详细信息（版本、可配置参数默认值）
report_property [get_ipdefs xilinx.com:ip:clk_wiz:6.0]

# GUI: Window → IP Catalog，左侧按分类树浏览，右上角搜索框
```

常用 IP 对应关系（2022.2 安装目录实测存在的目录名 = catalog 里 name + 版本）：

| 需求 | IP name | 备注 |
|------|---------|------|
| 时钟管理（PLL/MMCM） | `clk_wiz` | 7 系用 6.0；含 AXI 版用 `clk_wizard` |
| 复位同步 | `proc_sys_reset` | BD 外也常用 |
| 同步/异步 FIFO | `fifo_generator` | 13.2 |
| 块存储 BRAM | `blk_mem_generator` | |
| 分布式 RAM | `dist_mem_gen` | 8.0 |
| 内存控制器 DDR3/DDR4 | `mig` | MIG 7 系 |
| ILA/VIO 调试 | `ila`(6.2) / `vio`(3.0) / `jtag_axi` | |
| DSP：FIR/FFT/DDS/CORDIC/乘加/除法 | `fir_compiler`(7.2) / `xfft`(9.1) / `dds_compiler`(6.0) / `cordic`(6.0) / `mult_gen` / `c_addsub` / `div_gen` | |
| AXI 互连 | `axi_interconnect` / `smartconnect` / `axi_protocol_converter` / `axi_clock_converter` / `axi_dwidth_converter` | |
| AXI 外设 | `axi_gpio` / `axi_uartlite` / `axi_quad_spi` / `axi_iic` / `axi_timer` / `axi_dma` / `axi_bram_ctrl` | |
| 视频 | `v_tc`(6.2) | 视频时序 |
| 以太网/高速串行 | `ten_gig_eth_pcs_pma` / `gig_ethernet_pcs_pma` / `gtwizard` / `aurora_8b10b` | |
| PCIe | `pcie4_uscale_plus` | UltraScale+；7 系用 `pcie_7x` |

## 2. TCL 完整流程（已在本机 2022.2 实测通过）

```tcl
# ① 建工程（或复用已有工程）
create_project demo ./demo -part xc7a35tcsg324-1 -force

# ② 从 catalog 实例化一个 IP（得到项目里的 .xci）
create_ip -name clk_wiz -vendor xilinx.com -library ip -version 6.0 -module_name clk_wiz_0

# ③ 配置参数：先核对参数名，再 -dict 批量赋值
report_property [get_ips clk_wiz_0] CONFIG.*          ;# 核对参数名/类型/默认值
set_property -dict [list \
  CONFIG.PRIM_IN_FREQ                {100.000} \
  CONFIG.CLKOUT1_REQUESTED_OUT_FREQ  {50.000} \
  CONFIG.CLKOUT2_USED                {true} \
  CONFIG.CLKOUT2_REQUESTED_OUT_FREQ  {25.000} \
] [get_ips clk_wiz_0]

# ④ 生成输出产物：all = 综合+仿真+例化模板(+XDC 等)
generate_target all [get_ips clk_wiz_0]

# ⑤ （可选）提前做 OOC 综合出 checkpoint，减少顶层综合时间
synth_ip [get_ips clk_wiz_0]

# ⑥ （可选，非交互/脚本流需要）导出 IP 用户文件
export_ip_user_files -of_objects [get_ips clk_wiz_0] -no_script -sync -force -quiet
```

GUI 等价操作：IP Catalog 双击 IP → 自定义窗口填参数 → Generate → Output Products 窗口选 Global/Synthesized（OOC）→ Generate。

## 3. 例化（关键：抄生成的 .veo/.vho 模板）

生成产物位置（2021.1+ 起生成文件在 `.gen`，源 `.xci` 在 `.srcs`）：

```
<proj>/
├── demo.xpr
├── demo.srcs/sources_1/ip/clk_wiz_0/clk_wiz_0.xci      ← 配置源（入库）
└── demo.gen/sources_1/ip/clk_wiz_0/
    ├── clk_wiz_0.veo   ← Verilog 例化模板（含真实参数值与端口）
    ├── clk_wiz_0.vho   ← VHDL 例化模板
    ├── clk_wiz_0.xci / clk_wiz_0.xml
    └── (综合/仿真产物)
```

在顶层 RTL 里按 `.veo` 模板例化（模板就是官方替你写好的端口对齐代码）：

```verilog
// 以下内容直接取自 clk_wiz_0.veo —— 勿凭记忆手写端口
clk_wiz_0 instance_name (
  .reset(reset),            // input wire reset
  .clk_in1(clk_in1),        // input wire clk_in1
  .clk_out1(clk_out1),      // output wire clk_out1
  .clk_out2(clk_out2),      // output wire clk_out2
  .locked(locked)           // output wire locked
);
```

VHDL 用 `.vho` 里的 component 声明 + 端口 map。

## 4. 拿到任意 IP 的 CONFIG 参数名的三种办法

1. `report_property [get_ips xxx] CONFIG.*` —— 全量列出（TCL 流首选）。
2. GUI：选中 IP 例化 → 右键 **Copy IP Configuration Tcl** —— 直接得到完整 `set_property -dict` 脚本，照改数值即可。
3. 打开 `.xci`（XML 文本）搜 `spirit:configurableElementValue`，能看到每个参数的当前值。

注意：参数值几乎都是**字符串**，频率 `{100.000}`、布尔 `true/false`、枚举 `"distributed"` 等；数值范围/单位以 report_property 的 Value 列为准。

## 5. 复用与多实例

```tcl
# 已有 .xci 直接加入工程（版本控制推荐：提交 .xci，不提交 .gen）
read_ip -quiet <path>/clk_wiz_0.xci
# 或 add_files（GUI: Add Sources → Add or create constraints/... 选 .xci）

# 同一 IP 要两种配置 → 建两个实例，module_name 必须不同
create_ip -name fifo_generator -vendor xilinx.com -library ip -version 13.2 -module_name fifo_adapt
create_ip -name fifo_generator -vendor xilinx.com -library ip -version 13.2 -module_name fifo_pkt
# 复制现有实例改配置（保留原配置作底子）
copy_ip -name fifo_adapt2 [get_ips fifo_adapt]
```

## 6. 状态、重生成与版本升级

```tcl
report_ip_status                                  ;# 看锁定/过期/可升级状态
upgrade_ip [get_ips all]                          ;# 换 Vivado 版本打开旧工程后执行
reset_target all [get_ips clk_wiz_0]              ;# 清掉输出产物
generate_target all [get_ips clk_wiz_0]           ;# 再重新生成
set_property IS_LOCKED true [get_ips clk_wiz_0]   ;# 锁定防止误改
```

## 7. 故障排查

| 症状 | 原因与处置 |
|------|-----------|
| `create_ip` 报 ip not found | VLNV 写错或 repo 没刷新：`get_ipdefs` 核对；自定义 IP 先 `set_property ip_repo_paths` + `update_ip_catalog` |
| `set_property` 报 invalid property/parameter | 参数名不对：`report_property [get_ips xxx] CONFIG.*` 核对；注意 IP 版本差异 |
| 顶层综合报端口不匹配/黑盒 | 例化没抄 `.veo`；或 `generate_target` 没跑/产物被清：重跑 generate_target |
| elaborate 找不到 `xpm_*` | 没启用 XPM 库：`set_property XPM_LIBRARIES {...} [current_project]` |
| 生成产物丢失/路径错乱 | 2021.1+ 在 `.gen`；老脚本写死 `.srcs` 会找不到 `.veo`，用 `glob` 递归找 |
| 团队综合结果不一致 | `create_ip` 未锁版本，各人取到不同版本；显式 `-version` |
| IP 生成很慢 | OOC 默认开启；纯仿真用 `generate_target simulation`；或 Global 输出 `set_property SYNTH_CHECKPOINT_MODE none [get_files xxx.xci]` |

## 8. 脚本化项目模板（可直接改造复用）

```tcl
# build_ip_demo.tcl —— 一键建工程+配 IP+生成+例化验证
set part xc7a35tcsg324-1
create_project demo ./demo -part $part -force

create_ip -name clk_wiz -vendor xilinx.com -library ip -version 6.0 -module_name clk_wiz_0
set_property -dict [list CONFIG.PRIM_IN_FREQ {100.000} \
  CONFIG.CLKOUT1_REQUESTED_OUT_FREQ {50.000}] [get_ips clk_wiz_0]
generate_target all [get_ips clk_wiz_0]

# 顶层文件例化 clk_wiz_0 后：
add_files -fileset sources_1 ./top.sv
set_property top top [current_fileset]
update_compile_order -fileset sources_1
launch_runs synth_1 -jobs 4
wait_on_run synth_1
puts "synth: [get_property STATUS [get_runs synth_1]] [get_property PROGRESS [get_runs synth_1]]"
```

> 实测记录（本机 2022.2，Artix-7）：clk_wiz 6.0 按 100MHz 输入 → 50/25MHz 双输出配置，`generate_target all` 后在 `demo.gen/sources_1/ip/clk_wiz_0/` 生成 `.veo`；顶层例化 + `xpm_cdc_single`，`launch_runs synth_1` 进度 100% 通过。
