**如果是在centOS系统中运行，**

**setup.vcs中的csh文件格式需要改为bash格式，并且修改为自己的文件路径。比如：**
```bash
#!/bin/csh

setenv VCS_HOME /opt/vcs/E-2011.03 
setenv UVM_HOME ~/uvm/uvm-1.1d
setenv WORK_HOME `pwd`
setenv SIM_TOOL VCS 
set path = (/opt/vcs/E-2011.03/bin ${WORK_HOME}/bin $path)
```

**需要改为：**
```bash
#!/bin/bash
export UVM_HOME=/home/autumn/testDemo/project/example_and_uvm_source_code/uvm-1.1d
export WORK_HOME=$(pwd)
export SIM_TOOL=VCS 
export PATH=/opt/Synopsys/vcs_202003/Q-2020.03-SP2-7/bin:${WORK_HOME}/bin:$PATH
```

**run文件也需一并修改，如：**
```bash
#!/bin/csh
if ( $SIM_TOOL == "QUESTA" ) then
vlib work
vlog -f filelist.f
vsim -sv_lib $UVM_DPI_DIR/uvm_dpi -do $WORK_HOME/bin/vsim.do -c top_tb
endif

if ( $SIM_TOOL == "VCS" ) then
vcs -sverilog $UVM_HOME/src/dpi/uvm_dpi.cc -CFLAGS -DVCS -timescale=1ns/1ps -f filelist.f 
./simv 
endif

if ( $SIM_TOOL == "NCSIM" ) then
ncverilog +sv -f filelist.f -licqueue -timescale 1ns/1ps -uvm -uvmhome $UVM_HOME 
endif
```
**改为：**
```bash
#!/bin/bash

if [ "$SIM_TOOL" = "QUESTA" ]; then
    vlib work
    vlog -f filelist.f
    vsim -sv_lib "$UVM_DPI_DIR/uvm_dpi" -do "$WORK_HOME/bin/vsim.do" -c top_tb
fi

if [ "$SIM_TOOL" = "VCS" ]; then
    vcs -R -full64 +v2k -fsdb -sverilog "$UVM_HOME/src/dpi/uvm_dpi.cc" -CFLAGS -DVCS -timescale=1ns/1ps -f filelist.f 
    ./simv 
fi

if [ "$SIM_TOOL" = "NCSIM" ]; then
    ncverilog +sv -f filelist.f -licqueue -timescale 1ns/1ps -uvm -uvmhome "$UVM_HOME" 
fi

endif
```


**（选用）可以在testbench中添加以下代码提取波形：**

```systemverilog
// ===========================================================================
// ** 新增的波形生成代码 **
// ===========================================================================
initial begin
    // 1. 指定VCD波形文件的名称。仿真结束后，会在当前目录生成一个名为 "waveform.vcd" 的文件。
    $dumpfile("waveform.vcd");

    // 2. 指定要 dump 的信号范围。
    //    - 第一个参数 0 表示 dump 所有层次（从顶层模块开始）。
    //    - 第二个参数 top_tb 是要 dump 的顶层模块实例名。
    //    这会记录 top_tb 模块及其内部所有实例（包括 my_dut, input_if, output_if 等）的所有信号。
    $dumpvars(0, top_tb);
    
    $display("INFO: VCD waveform dumping started. File: waveform.vcd");
end
// ===========================================================================
```



**VCS仿真编译**

```bash
vcs -R -full64 +v2k -fsdb +define+FSDB -sverilog -ntb_opts uvm top_tb.sv -debug_all -elab -lca -kdb
```

**VCS查看波形**

```bash
./simv -gui & 
```

**verdi观察波形**

```bash
# fsdb文件名和top_tb.sv中设置的保持相同
verdi -ssf tb.fsdb & 
```





**或者通过makefile的方式来统一管理仿真编译**：
**makefile文件**
```bash
simulate ?= vcs
FILELIST = ./../tb/top.f
 
tc ?= test_case
DUMP_EN ?= 1
 
LOG_DIR = ${shell mkdir -p ./log}
WAVE_DIR = ${shell mkdir -p ./wave}
 
ifeq ($(DUMP_EN), 1)
	WAVE = +define+WAVE_DUMP -fsdb
else
	WAVE =
endif
 
comp:
	vcs \
	-f $(FILELIST) \
	-kdb -lca \
	-full64 \
	-sverilog -v2k \
	-ntb_opts uvm-1.1 \
	-debug_access+all -debug_region+cell+encrypt \
	+lint=TFIPC-L +warn=all -error=IWNF \
	-top top_tb \
	-Mupdate \
	+memcbk \
	+libext+.v+.V+.sv+.vp \
	+systemverilogext+.sv+.SV+ \
	+nospecify +notimingcheck \
	-timescale=1ns/100ps \
	$(LOG_DIR) \
	$(WAVE_DIR) \
	$(WAVE) \
	+tc_name=$(tc) \
	-l ./log/$(tc)_compile.log
 
sim:
	./simv \
	-l ./log/$(tc)_sim.log \
	-ucli -i do.ucli \
	+UVM_MAX_QUIT_COUNT=6,NO \
	+UVM_VERBOSITY=UVM_LOW \
	+UVM_TESTCASE=${tc} \
	+UVM_TESTNAME=${tc} \
	+TC_NAME=$(tc) \
	+vpdfileswitchsize=300
 
run: comp sim
 
verdi:
	verdi \
	-f $(FILELIST) \
	-sverilog -v2k -sv \
	-ntb_opts uvm-1.1 \
	+libext+.v+.V+.sv+.vp \
	+nospecify +notimingcheck \
	-timescale=1ns/100ps \
	-ssf ./wave/$(tc)_wave.vf
 
clean:
	rm -f ./wave/* \
	rm -f *.log \
	rm -rf simv* \
	rm -rf simv.daidir
```
**运行代码**
```bash
make run tc=my_case0
```
**查看波形**
```bash
make verdi tc=my_case0 
```  


### 关于 “+UVM_TESTNAME=” 使用方法：

当启动仿真时，通过在命令行中添加 `+UVM_TESTNAME=测试用例名称` 来指定要运行的UVM 测试用例(test case)。

例如，在你的脚本中已经包含了这样的用法：



```
# 在Questa仿真中

vsim ... +UVM_TESTNAME=$1

# 在VCS仿真中

./simv +UVM_TESTNAME=$1

# 在NCSIM仿真中

ncverilog ... +UVM_TESTNAME=$1
```

这里的 `$1` 表示脚本接收的第一个命令行参数，也就是你要运行的测试用例名称。

### 实际操作示例：

假设你有一个名为 `my_test_case` 的测试用例，运行仿真时可以这样使用：



```
# 直接运行脚本并指定测试用例

./run_tc my_test_case
```

此时脚本会将 `my_test_case` 传递给 `+UVM_TESTNAME=` 参数，UVM 框架会自动找到并运行这个测试用例。

### 注意事项：



1. 测试用例名称必须与你在代码中定义的测试类名称完全一致（区分大小写）

2. 如果不指定 `+UVM_TESTNAME`，UVM 会尝试运行默认测试用例（通常需要在代码中特别指定）

3. 这是 UVM 标准的命令行参数，所有主流的仿真工具（Questa、VCS、Xcelium 等）都支持

在你的脚本中，还注释了一些其他常用的 UVM 命令行参数，例如：



* `+UVM_OBJECTION_TRACE`：跟踪 objection 的变化

* `+UVM_PHASE_TRACE`：跟踪 UVM phases 的执行过程

* `+UVM_CONFIG_DB_TRACE`：跟踪 config\_db 的设置和获取过程

这些参数可以和 `+UVM_TESTNAME` 一起使用，用于调试和分析仿真过程。

> 

**UVM框架源码：**


[Download UVM (Universal Verification Methodology) - Accellera Systems Initiative](https://accellera.org/downloads/standards/uvm)




