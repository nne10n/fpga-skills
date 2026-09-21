# XPM 宏（Xilinx Parameterized Macros）例化指南

XPM 是纯 SystemVerilog 行为级宏，**免 create_ip、免生成、免 .xci**——直接在 RTL 里例化即可。本地完整源码（含每个参数的范围约束 DRC）在：
`<vivado>\data\ip\xpm\{xpm_cdc|xpm_fifo|xpm_memory}\hdl\*.sv`（UG974 文档化）。

## 1. 模块清单（2022.2 实测）

| 库 | 模块 | 用途 |
|----|------|------|
| XPM_CDC | `xpm_cdc_single` | 单 bit 电平同步（2~10 级打拍） |
| | `xpm_cdc_array_single` | 多 bit 总线（各 bit 独立同步，仅源数据稳定时用） |
| | `xpm_cdc_gray` | 格雷码（计数器跨域） |
| | `xpm_cdc_pulse` | 脉冲跨域（展宽+同步+还原） |
| | `xpm_cdc_handshake` / `xpm_cdc_low_latency_handshake` | 握手式多 bit 数据跨域 |
| | `xpm_cdc_sync_rst` / `xpm_cdc_async_rst` | 同步/异步复位同步 |
| XPM_FIFO | `xpm_fifo_sync` | 同时钟 FIFO |
| | `xpm_fifo_async` | 跨时钟域 FIFO（最常用） |
| | `xpm_fifo_axis` / `xpm_fifo_axif` / `xpm_fifo_axil` | AXI-Stream/AXI4-Full/AXI4-Lite FIFO |
| XPM_MEMORY | `xpm_memory_spram` / `xpm_memory_sprom` | 单口 RAM/ROM |
| | `xpm_memory_dprom` | 双口 ROM |
| | `xpm_memory_sdpram` | 简单双口（A 写 B 读，BRAM 常用） |
| | `xpm_memory_tdpram` | 真双口 |
| | `xpm_memory_dpdistram` | 双口分布式 RAM |

## 2. 启用（必须）

```tcl
# GUI: Project Settings → General → Verilog options / XPM libraries 勾选
set_property XPM_LIBRARIES {XPM_CDC XPM_FIFO XPM_MEMORY} [current_project]
```

- **项目模式**：Vivado 综合时通常能自动识别例化并补库，但**脚本化/非项目模式必须显式 set**，否则 elaborate 报 `xpm_* not found`。
- 仿真：Vivado 仿真器自动编译；第三方仿真器需把上述 hdl/*.sv 加进编译序。

## 3. 例化模板（签名取自本地 2022.2 源码）

### 3.1 xpm_cdc_single —— 单 bit 电平跨域

```verilog
xpm_cdc_single #(
  .DEST_SYNC_FF   (2),   // 2~10, 打拍级数
  .SIM_ASSERT_CHK (0),
  .SRC_INPUT_REG  (1)    // 1=源端先打一拍(推荐,改善时序)
) u_cdc (
  .src_clk  (clk_src),
  .src_in   (sig_in),
  .dest_clk (clk_dst),
  .dest_out (sig_out)
);
```

> 注意：`xpm_cdc_single`/`array_single` 只适合**源端数据稳定后目标域一定能采到**的准静态信号；多 bit 相关总线用 `xpm_cdc_gray`/`handshake`/异步 FIFO。

### 3.2 xpm_fifo_async —— 跨时钟域 FIFO（实测签名）

```verilog
xpm_fifo_async #(
  .FIFO_MEMORY_TYPE  ("block"), // "block"|"distributed"|"auto"|"ultra"
  .FIFO_WRITE_DEPTH  (512),     // 深度, 2 的幂; 实际可用容量 = DEPTH - 4(标准模式)
  .WRITE_DATA_WIDTH  (32),
  .READ_DATA_WIDTH   (32),
  .READ_MODE         ("std"),   // "std"=标准(读延迟1拍) | "fwft"=首字直通(读延迟0)
  .FIFO_READ_LATENCY (1),       // 必须与 READ_MODE 一致: std=1, fwft=0
  .USE_ADV_FEATURES  ("0707"),   // 默认0707=overflow/prog_full/wr_data_count/
                                 // underflow/prog_empty/rd_data_count; 位映射见下文
  .CDC_SYNC_STAGES   (2),
  .PROG_FULL_THRESH  (10),
  .PROG_EMPTY_THRESH (10)
) u_fifo (
  .rst        (~rst_n),          // 高有效异步复位
  .wr_clk     (clk_wr),
  .wr_en      (wr_en),
  .din        (din),             // [WRITE_DATA_WIDTH-1:0]
  .full       (full),
  .prog_full  (prog_full),       // 仅 USE_ADV_FEATURES 对应位打开时有效
  .rd_clk     (clk_rd),
  .rd_en      (rd_en),
  .dout       (dout),            // [READ_DATA_WIDTH-1:0]
  .empty      (empty),
  .sleep      (1'b0),
  .wr_rst_busy(wr_rst_busy),     // 复位期间必须憋住读写
  .rd_rst_busy(rd_rst_busy)
  // injectsbiterr/injectdbiterr, overflow/underflow, wr_data_count/rd_data_count,
  // almost_full/almost_empty, wr_ack/prog_full..., 按需连接, 未用可悬空
);
```

**FWFT 陷阱**：想要"读使能前数据已在 dout 上"（AXI-Stream 风格）必须 `READ_MODE("fwft")` **且** `FIFO_READ_LATENCY(0)`，两者不一致会 DRC 报错。

**USE_ADV_FEATURES 位映射**（hex 字符串，每位一个开关，默认 "0707"）：bit0=overflow、bit1=prog_full、bit2=wr_data_count、bit3=almost_full、bit4=wr_ack、bit8=underflow、bit9=prog_empty、bit10=rd_data_count、bit11=almost_empty、bit12=data_valid。默认 "0707" 已打开前六个（两个 prog、两个计数、两个溢出标志）。

### 3.3 xpm_memory_sdpram —— 简单双口 BRAM（A 写 B 读）

```verilog
xpm_memory_sdpram #(
  .MEMORY_SIZE        (16384),     // 总比特数 = WRITE_DATA_WIDTH_A * 2^ADDR_WIDTH_A
  .MEMORY_PRIMITIVE   ("block"),   // "block"|"distributed"|"ultra"
  .CLOCKING_MODE      ("common_clock"), // "independent_clock" 跨域读写口
  .WRITE_DATA_WIDTH_A (32),
  .READ_DATA_WIDTH_B  (32),
  .ADDR_WIDTH_A       (9),         // 2^9 * 32 = 16384, 与 MEMORY_SIZE 自洽
  .ADDR_WIDTH_B       (9),
  .READ_LATENCY_B     (2),         // >=1; 输出寄存器级数, BRAM 常用 2
  .WRITE_MODE_B       ("no_change"), // 读地址=写地址时 doutb 保持
  .MEMORY_INIT_FILE   ("none")     // 用 .mem 初始化时填文件名(不带扩展名)
) u_ram (
  .clka  (clk_wr), .ena (wr_en),  .wea  (wr_en), .addra (waddr), .dina (wdata),
  .clkb  (clk_rd), .rstb (~rst_n), .enb (rd_en),    .regceb (1'b1),
  .addrb (raddr),  .doutb (rdata),
  .sleep (1'b0)
  // injectsbiterra/injectdbiterra, sbiterrb/dbiterrb: 仅 ECC 模式
);
```

> `wea` 位宽 = `WRITE_DATA_WIDTH_A / BYTE_WRITE_WIDTH_A`：默认 `BYTE_WRITE_WIDTH_A=WRITE_DATA_WIDTH_A` 时 **wea 只有 1 bit**（整字写）；要按字节写则设 `BYTE_WRITE_WIDTH_A(8)`，此时 32 位字对应 4 bit wea。

### 3.4 xpm_cdc_sync_rst / async_rst —— 复位同步

```verilog
xpm_cdc_sync_rst #(.DEST_SYNC_FF(2)) u_rst_sync (
  .src_rst  (~rst_n),      // 高有效输入
  .dest_clk (clk_dst),
  .dest_rst (rst_dst)      // 高有效输出, 直接连 always @(posedge clk) if (rst_dst)
);
```

## 4. XPM vs IP Catalog 选型

| 场景 | 推荐 | 理由 |
|------|------|------|
| 简单 CDC/FIFO/RAM，脚本化构建 | **XPM** | 无生成步骤、参数化即时生效、版本控制干净 |
| 需要 AXI 接口的 FIFO、带纠错/共享逻辑、BD 里连线 | IP Catalog | GUI 参数向导 + 总线接口 + BD 集成 |
| 需要独立 OOC checkpoint 复用 | IP Catalog | synth_ip 产物可缓存 |
| 大规模 BRAM 拼接/ECC/mmi 初始化 | `blk_mem_generator` | 支持coe/mmi、ECC、字节写等高级特性 |

## 5. 排错

| 报错 | 处置 |
|------|------|
| `xpm_fifo_async` not found | `set_property XPM_LIBRARIES` 漏了；第三方仿真器没编译 hdl/*.sv |
| DRC 报 READ_LATENCY 与 READ_MODE 不一致 | std→FIFO_READ_LATENCY(1)，fwft→(0) |
| FIFO 复位后异常 | 等 `wr_rst_busy`/`rd_rst_busy` 拉低后才能读写 |
| 深度/阈值超范围 | 每个 XPM 源文件头部有 DRC 注释列出合法范围（如 PROG_FULL_THRESH 相对深度），直接查源码 |
| 综合出来不是 BRAM | MEMORY_PRIMITIVE="auto" 时由工具选；显式指定 "block"；检查 READ_LATENCY_B>=1 |
