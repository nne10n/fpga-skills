---
name: vivado-ip
description: Vivado IP 核创建（打包自定义 IP）与例化使用（IP Catalog 官方 IP、XPM 宏）的完整工作流。凡涉及 Vivado IP 核、create_ip、set_property CONFIG、generate_target、.xci、ipx:: 打包、IP Packager、IP Catalog、IP repo、.veo/.vho 例化模板、XPM CDC/FIFO/Memory、clk_wiz/fifo_generator/blk_mem_gen/axi 外设等场景都应加载本 skill —— 即使用户没有明说"IP 核"三个字。
---

# Vivado IP 核创建与例化使用

以 Vivado 2022.2 为基准验证（安装路径示例 `C:\Xilinx\Vivado\2022.2`，下称 `<vivado>`）。其他版本命令一致，个别 CONFIG 参数名可能有差异，用 `report_property` 现场核对。

## 三种 IP 形态（先判断用户要做什么）

| 形态 | 是什么 | 关键操作 |
|------|--------|----------|
| **IP Catalog 官方 IP** | Xilinx 打包好的 IP（618 个，位于 `<vivado>\data\ip\xilinx\`），项目里体现为 `.xci` 文件 | `create_ip` → `set_property CONFIG.*` → `generate_target` → 按模块名例化 |
| **XPM 宏** | 纯 SystemVerilog 模块（`<vivado>\data\ip\xpm\`），免生成、直接例化 | `set_property XPM_LIBRARIES` → 直接写 HDL 例化 |
| **用户自定义 IP** | 把自己的 RTL 用 IP Packager 打包成可复用 IP（产出 `component.xml` + 源码，可存入 IP repo 或 zip） | `ipx::package_project` → 属性/总线接口 → `ipx::check_integrity` → `ipx::archive_core` |

**决策路由（按用户意图进入对应参考文档）：**

- 要**使用**官方 IP（时钟、FIFO、BRAM、DMA、AXI 外设、 SerDes…）→ 读 `references/consume-catalog-ip.md`
- 要把自己的 RTL/Block Design **封装成 IP** 给别人或给 Block Design 用 → 读 `references/package-custom-ip.md`
- 要做 **CDC 打拍、异步 FIFO、片上 RAM/ROM** 等轻量原语 → 读 `references/xpm-macros.md`
- 只是要排查 IP 报错/升级 → 直接看下面「命令速查」+ `references/consume-catalog-ip.md` 的「故障排查」节

## 命令速查（最常用 12 条）

```tcl
# ---- 官方 IP 消费 ----
get_ipdefs -regexp {.*clk_wiz.*}                     ;# 在 catalog 里搜 IP（拿 VLNV）
create_ip -name clk_wiz -vendor xilinx.com -library ip -version 6.0 -module_name clk_wiz_0
set_property -dict [list CONFIG.PRIM_IN_FREQ {100.000} \
  CONFIG.CLKOUT1_REQUESTED_OUT_FREQ {50.000}] [get_ips clk_wiz_0]   ;# 批量配置参数
report_property [get_ips clk_wiz_0] CONFIG.*          ;# 查看全部可配置参数名与当前值
generate_target all [get_ips clk_wiz_0]               ;# 生成输出产物(综合/仿真/例化模板)
# ---- XPM ----
set_property XPM_LIBRARIES {XPM_CDC XPM_FIFO XPM_MEMORY} [current_project]
# ---- 自定义 IP ----
ipx::package_project -root_dir <dir> -import_files -vendor myco.com -library user -taxonomy {/MyIP} -set_current true -force
ipx::check_integrity [ipx::current_core]              ;# 打包完整性检查
ipx::archive_core <out.zip> [ipx::current_core]       ;# 打成可分发的 zip
# ---- IP repo ----
set_property ip_repo_paths {<repo_dir>} [current_project] ; update_ip_catalog
# ---- 状态与升级 ----
report_ip_status ; upgrade_ip [get_ips *]
```

## 高频踩坑（动手前先看）

1. **例化模板在 `.gen` 不在 `.srcs`**（2021.1+）：`.veo`(Verilog)/`.vho`(VHDL) 位于
   `<proj>/<proj>.gen/sources_1/ip/<module>/<module>.veo`；`.xci` 仍在 `<proj>.srcs/...`。例化端口/参数名**永远抄 .veo 模板**，不要凭记忆写。
2. **模块名 vs 例化名**：`-module_name clk_wiz_0` 决定生成模块的名字。项目里有同名 IP 例化多次时，每个实例要用不同 module_name（先 `copy_ip` 或重新 `create_ip`），否则端口配置互相覆盖。
3. **CONFIG 参数名不靠猜**：先 `report_property [get_ips xxx] CONFIG.*` 核对，参数值多为字符串（频率要带引号如 `{100.000}`）。GUI 右键 IP → Copy IP Configuration Tcl 可直接拿到全部参数的等价脚本。
4. **批量用 `-dict`**：`set_property -dict [list CONFIG.A {...} CONFIG.B {...}]`，一次赋值避免多次触发参数联动校验；赋值顺序无关。
5. **XPM 必须显式启用**（非项目模式/部分脚本流）：`set_property XPM_LIBRARIES {XPM_CDC XPM_FIFO XPM_MEMORY} [current_project]`，否则 elaborate 找不到 `xpm_*` 模块。
6. **IP repo 改了不生效**：`update_ip_catalog`（顽固时 `-rebuild`）；repo 路径用绝对路径，相对路径相对工程目录解析。
7. **OOC 综合是默认**：官方 IP 默认 Out-of-Context，顶层综合时是黑盒。手动预综合用 `synth_ip [get_ips xxx]`；要全局综合改 `set_property GENERATE_SYNTH_CHECKPOINT false [get_files xxx.xci]`。
8. **版本锁死**：`create_ip` 不带 `-version` 会取最新版，团队协作建议显式锁版本，避免各人生成结果不一致。
9. **官方 IP 目录只读**：`<vivado>\data\ip\` 是安装目录里的 catalog 数据（含 618 个 IP 的 `component.xml`、ttcl 模板、bd.tcl），只作参考，用户的生成产物/自定义 IP 一律放工程或自己的 repo 目录。
10. **升级 IP**：换版本打开工程后先 `report_ip_status`，再 `upgrade_ip [get_ips all]`，然后 `reset_target all [get_ips xxx]` + `generate_target all` 重生成。

## 官方 IP 内部结构（读懂 catalog 数据，便于仿写）

`<vivado>\data\ip\xilinx\<ip>_v<ver>\` 下每个 IP 的核心是 `component.xml`（IP-XACT/SPIRIT 1685-2009 描述文件）：

- **VLNV**：`vendor=xilinx.com, library=ip, name=clk_wiz, version=6.0` —— `create_ip` 用 `-name/-vendor/-library/-version` 四元组定位。
- **busInterfaces + portMaps**：逻辑信号（如 `AXI` 的 `ARADDR`）到物理端口（`s_axi_araddr`）的映射，Block Design 里连线靠它。
- **model/views + fileSets**：综合视图（`xilinx_anylanguagesynthesis`）、仿真视图、例化模板视图分别引用哪些文件、什么 fileType（`verilogSource`/`vhdlSource`/`tclSource`/`xdc`）。
- **parameters**：GUI 上每个 CONFIG 参数的定义、类型、默认值、取值范围。
- 其他目录：`ttcl/`（按参数生成 HDL 的模板）、`bd/bd.tcl`（Block Design 集成）、`xgui/`（自定义 GUI）。

自己打包 IP 时 `ipx::` 命令族生成的正是同一套 `component.xml` 结构，所以官方 IP 是最好的仿写样本。
