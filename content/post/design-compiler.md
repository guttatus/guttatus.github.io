+++
author = "guttatus"
title = "Design Compiler入门"
date = "2023-12-19"
description = "Design Compiler基础使用笔记"
tags = [
    "synthesis",
]
categories = [
    "IC",
]
toc = true
autonumbering = true
+++
# Design Compiler

## Interface to DC

### Design Vision(GUI)

  ``` shell
  $ design_vision -topographical_mode
  ```

### DC Shell

  ``` shell
  $ dc_shell -topographical_mode
  ```

### Batch mode

  ``` shell
  $ dc_shell -topo -f run.tcl | tee -i run.log
  ```

  


## Workflow

``` shell
$ cd workdir
$ dc_shell -topo
dc_shell-topo> read_verilog <verilog file>
dc_shell-topo> source <constraint file>
dc_shell-topo> check_timing
dc_shell-topo> set_app_var target_library <target library>
dc_shell-topo> compile_ultra
dc_shell-topo> write -format verilog -output <netlist.v>
dc_shell-topo> exit
```

### Constraint report

``` shell
$ cd workdir
$ dc_shell -topo
dc_shell-topo> read_verilog <netlist.v>
dc_shell-topo> source <constraint file>
dc_shell-topo> set_app_var target_library [list <target library>]
dc_shell-topo> report_constraint -all_violators
dc_shell-topo> exit
```

### Add file search path

``` shell
# set_app_var search_path "$search_path rtl cons libs"
dc_shell-topo> set_app_var search_path [list . $search_path <serch path>]
```

### Library

``` tcl
dc_shell-topo> set_app_var target_library [list <target library>]
dc_shell-topo> set_app_var link_library   [list * $target_library IP.db]
dc_shell-topo> set_app_var symbol_library "* target.db IP.db"
```

### Set current design

``` tcl
dc_shell-topo> read_verilog [list A.v B.v Top.v]
dc_shell-topo> current_design MY_TOP
dc_shell-topo> link
```

### Check design

check_design checks your current design for connectivity and hierarchy issues

``` tcl 
dc_shell-topo> read_verilog {A.v B.v Top.v}
dc_shell-topo> current_design MY_TOP
dc_shell-topo> link
dc_shell-topo> check_design
## html format output 
dc_shell-topo> check_design -html check_design.html
```

### Load design by `analyze` and `elaborate`

``` tcl
dc_shell-topo> analyze -format verilog [list A.v Top.v]
dc_shell-topo> elaborate  MY_TOP -parameters "A_WIDTH=9, B_WIDTH=16"
```

### Load design form directory

``` tcl
dc_shell-topo> read_file [list $RTL_PATH] -autoread -recursive -format verilog -top MyTopModule
```

### Save the ddc design before compile

``` tcl
$ cd workdir
$ dc_shell -topo
dc_shell-topo> read_verilog <verilog file>
dc_shell-topo> current_design MY_TOP
dc_shell-topo> link
dc_shell-topo> check_design
dc_shell-topo> write -format ddc -hier -output unmaped/MY_TOP.ddc
dc_shell-topo> source <constraint file>
dc_shell-topo> check_timing
dc_shell-topo> set_app_var target_library [list <target library>]
dc_shell-topo> compile_ultra
```

### Save the ddc design after compile

``` tcl
dc_shell-topo> compile_ultra
dc_shell-topo> change_names -rule verilog -hier
dc_shell-topo> write -format verilog -hier -output netlist.v
dc_shell-topo> write -format ddc -hier -output maped/MY_TOP.ddc
```

## Timing Constraints

### Create Clock

``` tcl
create_clock -period 2 [get_ports clk]
```

### Setup Time

![set_clock_uncertainty](/img/posts/dc/set_clock_uncertainty.png)

``` tcl
set_clock_uncertainty -setup 0.14 [get_clocks clk]
```

### Model latency or insertion delay

![](/img/posts/dc/set_clock_latency.png)

``` tcl
set_clock_latency -source -max 3 [get_clocks clk]
set_clock_latency -max 1 [get_clocks clk]
```

### Model Transition Time

Transition models the rise and fall time of the clock waveform at the register clock pins

``` tcl
set_clock_transition 0.08 [get_clocks clk]
```

###  Constraining input  paths

![](/img/posts/dc/set_input_delay.png)

``` tcl 
set_input_delay -max 0.6 -clock clk [get_ports A]
```

To constraining all inputs the same, except for the clock port

``` tcl
set_input_delay -max 0.5 -clock clk [remove_from_collection [all_inputs] [get_ports clk]]
```

To remove the constrain of a port

``` tcl
remove_input_delay [get_ports clk]
```

### Constraining output paths

![](/img/posts/dc/set_output_delay.png)

``` tcl
set_output_delay -max 0.8 -clock clk [get_ports B]
```

To constraining all outputs the same

``` tcl
set_output_delay -max 0.5 -clock clk [all_outputs]
```

### Constraining a pure combinational design

![](/img/posts/dc/vir_clk.png)

we should create a virtual clock

``` tcl
create_clock -name vclk -period 2
```

### Constraining clk to q

``` tcl
set_clk_to_q_max 1.5
set_clk_to_q_min 0.9
```

