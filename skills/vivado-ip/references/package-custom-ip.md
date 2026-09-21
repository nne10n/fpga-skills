# 自定义 IP 创建与打包（IP Packager / ipx:: 命令族）

目标：把自己的 RTL（或 Block Design）封装成符合 IP-XACT 规范的可复用 IP，产出 `component.xml` + 源码，可放入 IP repo 或 zip 分发。基准 Vivado 2022.2，**本页 TCL 全流程已在本机实测通过**（含从 repo 再消费）。

## 1. 概念与产出物

- **VLNV**：`vendor / library / name / version` 四元组唯一标识 IP。自定义 IP 常用 `vendor=你的域名（如 example.com）`、`library=user`。
- **component.xml**：IP-XACT（SPIRIT 1685-2009）描述文件，是 IP 的"身份证"——端口、总线接口、参数、文件组、GUI 全在里面。官方 IP `<vivado>\data\ip\xilinx\<ip>_v<ver>\component.xml` 就是最好的参照样本。
- **taxonomy**：IP Catalog 里的分类树路径（如 `/MyIP`），决定别人在 catalog 哪个分类下看到它。
- **IP repo**：放 IP 目录（含 component.xml 的目录）或 zip 的仓库目录，工程通过 `ip_repo_paths` 注册。

## 2. GUI 流程（UG1118 主线）

1. **入口**：菜单 `Tools → Create and Package New IP...`，四种来源：
   - **Package your current project**（最常用：当前工程顶层 RTL 打包）
   - Package a directory of sources（无工程，纯源码目录）
   - Package a block design（把 BD 打包成 IP，如自定义 PS 子系统）
   - Package an IP core from a Xilinx IP（克隆/派生官方 IP）
2. 选 IP 存放目录；工程含 .xci 时会询问是否包含。
3. 进入 **Package IP 窗口**，按 tab 从左到右检查：
   - **Identification**：vendor/library/name/version、display name、description、**Categories(taxonomy)**
   - **Compatibility**：支持的器件族
   - **File Groups**：综合/仿真文件是否齐、每个组设置正确 top；缺文件点 Add Files
   - **Customization Parameters**：哪些参数暴露到 GUI
   - **Ports and Interfaces**：RTL 改动后点 **Merge Changes from Top** 同步端口；把信号归组成总线接口（clock/reset/AXI）
   - **Customization GUI**：布局参数控件，点 Create GUI files 生成 xgui
   - **Review and Package**：看 integrity 检查结果 → **Package IP**（选 zip 或 repo 目录）
4. GUI 里点一次 Package 等价于 TCL 的 `set_property core_revision`+`ipx::update_checksums`+`ipx::archive_core`。

## 3. TCL 全流程（实测通过的完整脚本）

```tcl
# ── 准备：工程里要打包的顶层 my_adder.v 已 add_files 并设为 top ──
create_project val_pkg ./proj2 -part xc7a35tcsg324-1 -force
add_files ../myip_src/my_adder.v
set_property top my_adder [current_fileset]
update_compile_order -fileset sources_1

# ── ① 打包：导入文件到目标目录，生成 component.xml ──
ipx::package_project -root_dir ./iprepo/my_adder -import_files \
  -vendor example.com -library user -taxonomy {/MyIP} -set_current true -force

# ── ② 编辑 IP 属性（current_core 即刚打包的 core 对象）──
set core [ipx::current_core]
set_property name         my_adder  $core
set_property display_name {My Adder} $core
set_property description  {Registered adder demo IP} $core
set_property version      1.1       $core          ;# 每次发布递增
set_property core_revision 1        $core          ;# 源码变了要递增

# ── ③ 总线接口关联（强烈建议，消除 clock 未关联告警）──
ipx::associate_bus_interfaces -busif clk -clock clk [ipx::current_core]
# AXI IP 再加: ipx::associate_bus_interfaces -busif s_axi -clock s_axi_aclk -reset s_axi_aresetn [ipx::current_core]

# ── ④ 生成定制 GUI、校验、打包 ──
ipx::create_xgui_files  [ipx::current_core]
ipx::update_checksums   [ipx::current_core]
ipx::check_integrity    [ipx::current_core]        ;# 返回 1 = 通过(可有 warning)
ipx::save_core          [ipx::current_core]        ;# 改动落盘到 component.xml
file mkdir ./dist
ipx::archive_core ./dist/my_adder.zip [ipx::current_core]   ;# 分发用 zip
```

## 4. 消费打包好的 IP（IP repo 用法，实测通过）

```tcl
# 注册 repo（可以多个目录；用绝对路径）
set_property ip_repo_paths [list ./iprepo] [current_project]
update_ip_catalog                    ;# repo 内容变了就再跑一次；顽固缓存用 update_ip_catalog -rebuild

# 像官方 IP 一样创建、配置、生成、例化
create_ip -name my_adder -vendor example.com -library user -version 1.1 -module_name my_adder_0
generate_target all [get_ips my_adder_0]
# 之后按 <proj>.gen/sources_1/ip/my_adder_0/my_adder_0.veo 模板例化即可
```

zip 分发：把 zip 放进 repo 目录后 `update_ip_catalog` 自动解包索引。

## 5. 进阶

### 5.1 AXI 外设 IP
- GUI：`Tools → Create and Package New IP → Create a new AXI4 peripheral`，选接口类型（AXI4-Lite/AXI4-Stream/AXI4 Full）、寄存器数量，向导生成带 `axi_lite_ipif` 模板的工程，在 `user_logic` 里填业务逻辑再打包。
- TCL 补充（手动给普通 IP 加 AXI 从接口）：

```tcl
ipx::add_bus_interface s_axi [ipx::current_core]
set_property abstraction_type_vlnv xilinx.com:interface:aximm_rtl:1.0 [ipx::get_bus_interfaces s_axi -of_objects [ipx::current_core]]
set_property interface_mode slave [ipx::get_bus_interfaces s_axi -of_objects [ipx::current_core]]
# 端口映射：把 s_axi_araddr 等物理端口映射到逻辑信号 ARADDR……（ipx::add_port_map）
# 地址映射（AXI slave 必须有，否则 BD 连不上主设备）
ipx::add_memory_map s_axi [ipx::current_core]
set_property slave_memory_map_ref s_axi [ipx::get_bus_interfaces s_axi -of_objects [ipx::current_core]]
ipx::add_address_block axi_lite [ipx::get_memory_maps s_axi -of_objects [ipx::current_core]]
set_property range 4096 [ipx::get_address_blocks axi_lite -of_objects [ipx::get_memory_maps s_axi -of_objects [ipx::current_core]]]
```

- 快捷路：让 Vivado 从 RTL 推断 —— `ipx::infer_core`（对源码目录做自动识别），或在 Package IP 窗口 Ports and Interfaces 页用 inferring 工具。

### 5.2 暴露用户参数到 GUI
```tcl
ipx::add_parameter WIDTH [ipx::current_core]
set_property value_resolve_type user       [ipx::get_parameters WIDTH -of_objects [ipx::current_core]]
set_property value_format    long          [ipx::get_parameters WIDTH -of_objects [ipx::current_core]]
set_property value           8             [ipx::get_parameters WIDTH -of_objects [ipx::current_core]]
ipx::create_xgui_files [ipx::current_core]   ;# 之后重新生成 GUI 控件
```

### 5.3 文件组补充（模拟/例化模板视图）
```tcl
ipx::add_file_group -type xilinx_anylanguagebehavioralsimulation [ipx::current_core]   ;# 仿真视图
ipx::add_file <sim_sources.v> [ipx::get_file_groups xilinx_anylanguagebehavioralsimulation -of_objects [ipx::current_core]]
# 打包含子 IP 的设计：把子 IP 的 .xci 一并加入综合/仿真文件组（勾选 Include .xci）
```

## 6. 版本迭代规范

- 源码每次修改：`core_revision` +1 → `ipx::update_checksums` → `ipx::save_core` → `ipx::archive_core` 出新 zip。
- 接口/参数不兼容改动：递增 `version` 主版本号；compatible 版本列表用 Compatibility 页（`set_property supported_families`）。
- repo 里同名不同版 IP 并存是正常的，`create_ip` 不带 `-version` 取最新，**团队脚本要显式锁版本**。

## 7. 故障排查（含实测踩坑）

| 症状 | 原因与处置 |
|------|-----------|
| `WARNING: [IP_Flow 19-5661] Bus Interface 'clk' does not have any bus interfaces associated with it`（实测出现） | 时钟/复位没关联到任何接口：`ipx::associate_bus_interfaces -busif <if> -clock <clk> -reset <rst>`；纯逻辑 IP 可忽略该 warning |
| check_integrity 报端口未映射 | RTL 端口没归组/映射：Package IP 窗口 Ports and Interfaces 里映射，或 `ipx::add_port_map` |
| 打包后 IP 里例化的子 IP 丢失 | 未包含 .xci：打包时勾选 Include .xci，或把 .xci add 进文件组 |
| 别人 catalog 里看不到新 IP | repo 路径没注册或没刷新：`set_property ip_repo_paths` + `update_ip_catalog`（必要时 `-rebuild`） |
| 改了 RTL 但 IP 端口没变 | 没做 Merge Changes from Top（GUI）或重新 `ipx::package_project`；core_revision 别忘了 +1 |
| `create_ip` 报版本不存在 | repo 里的 version 与 create_ip 指定值不一致：`get_ipdefs -quiet example.com*` 查实际 VLNV |
| taxonomy 不生效 | taxonomy 是列表属性：`set_property taxonomy {{/MyIP}} $core` 或 package_project 的 `-taxonomy {/MyIP}` |
