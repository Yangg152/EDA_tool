---
weight: 11
---

# VCS与Verdi

## VCS与Verdi简介

* VCS全称Verilog Compiler Simulator，是Synoposys家的编译型Verilog模拟器，可编译 C、C++、Verilog、SystemVerilog 等文件，编译后生成 simv 可执行文件进行仿真。
* Verdi 最开始是由 novas 公司设计的，2012 年由 Synopsys 公司间接收购。除了源代码浏览器的标准功能（原理图、状态机图和波形比较），Verdi 平台还具有自动跟踪信号活动的高级功能。Verdi 主要用于仿真波形的查看，有助于快速定位和解决设计错误，加速 IC 设计流程。

## Verilog测试激励
在硬件设计中，Verilog测试激励（Testbench）用于验证设计的正确性。通过创建一个模拟环境，给设计提供输入并捕获输出结果。测试激励通常包含时钟、复位、输入信号的驱动，输出信号的监控，以及用于调试的打印信息等。
Verilog中的测试激励可以使用各种系统函数和任务进行调试和输出。以下是一些常见的系统任务和函数。

### 常见系统任务和函数
* `$timescale`  
  `$timescale`用于指定时间单位和时间精度的系统任务。它决定仿真中时间步长的单位，比如`1ns/1ps`表示时间单位为1纳秒，精度为1皮秒。它通常位于模块定义的开头。例如：
  ```verilog
  `timescale 1ns / 1ps
  ```

* `$display`  
  `$display`在仿真时打印消息到控制台，类似于C语言中的`printf`。它可以用于输出信号的状态、调试信息等。例如：
  ```verilog
  $display("Time = %0t, Signal = %b", $time, signal);
  ```

* `$monitor`  
  `$monitor`是用于持续监视信号变化的系统任务。它在仿真中每当监控的信号变化时自动打印出相应的消息。例如：
  ```verilog
  $monitor("Time = %0t, Signal = %b", $time, signal);
  ```

* `$finish`  
  `$finish`用于结束仿真。当仿真达到某个特定条件时，调用`$finish`可以终止仿真。例如：
  ```verilog
  if (done) $finish;
  ```

* `$time`  
  `$time`是返回当前仿真时间的系统函数。通常与`$display`、`$monitor`结合使用，以打印出时间信息。例如：
  ```verilog
  $display("Current time: %0t", $time);
  ```

* `$stop`  
  `$stop`暂停仿真，通常用于手动调试。与`$finish`不同的是，`$stop`不会完全结束仿真，只是暂时中断，可以通过仿真器继续。例如：
  ```verilog
  if (error_condition) $stop;
  ```
#### 示例
以下是一个简单的Verilog测试激励示例，它测试一个简单的计数器模块。

```verilog
`timescale 1ns / 1ps

module tb_counter();
  reg clk;
  reg rst;
  wire [3:0] count;

  // 实例化被测模块
  counter dut (
    .clk(clk),
    .rst(rst),
    .count(count)
  );

  // 生成时钟
  always #5 clk = ~clk;

  initial begin
    // 初始化信号
    clk = 0;
    rst = 1;
    #10 rst = 0;  // 复位信号拉低

    // 仿真持续100ns后结束
    #100 $finish;
  end

  // 监控输出
  initial begin
    $monitor("Time: %0t, Reset: %b, Count: %d", $time, rst, count);
  end
endmodule
```

这个测试激励模块使用了``$timescale``设置时间单位和精度，`$monitor`来监视输出信号的变化，并通过`$finish`在100纳秒后终止仿真。

### 检查Verilog仿真结果
* $display
项目后期可以用log进行debug比较方便
* 看波形（VCD或FSDB）
项目早期使用，各种信号比较详细

波形方式：

* VCD波形
   - **概述**：VCD（Value Change Dump）是IEEE标准的波形文件格式，广泛支持，适用于各种仿真工具。
   - **生成方法**：
     - 使用 `$dumpfile` 指定输出的VCD文件名，例如：`$dumpfile("waveform.vcd");`。
     - 使用 `$dumpvars` 指定需要记录的信号，一般在仿真开始时调用，例：`$dumpvars(0, top_module);`。
   - **优点**：
     - 标准化格式，兼容性好，支持多种仿真工具查看波形。
     - 生成较为简单，适合小型设计及快速仿真验证。
   - **缺点**：
     - 文件体积较大，仿真时间长时会消耗大量磁盘空间。
     - 支持的信号类型和压缩率有限，难以处理大规模设计。
   - **使用建议**：
     - 在项目早期阶段或较小规模设计中，使用VCD格式方便快速查看和分析波形变化。
     - 通过限制 `$dumpvars` 中的层级或具体信号范围来控制文件大小。

* FSDB波形
   - **概述**：FSDB（Fast Signal Database）是Synopsys的专有波形格式，通常用于较大规模设计和复杂仿真，配合Verdi等工具使用。
   - **生成方法**：
     - 使用 `$fsdbDumpfile` 来设置输出文件，例如：`$fsdbDumpfile("waveform.fsdb");`。
     - 使用 `$fsdbDumpvars` 指定需要记录的信号，例如：`$fsdbDumpvars(0, top_module);`。
   - **优点**：
     - 文件体积较小，支持高效的信号压缩，能够记录更多信息，适合大规模设计。
     - 支持更复杂的信号类型和详细的信号数据，查看精度更高。
     - 与Verdi等调试工具高度集成，能够使用强大的波形浏览、信号分析、故障定位等功能。
   - **缺点**：
     - 是专有格式，需要专用工具（如Verdi）才能查看，兼容性相对较差。
     - 生成FSDB文件可能需要消耗较多资源，仿真速度可能稍有影响。
   - **使用建议**：
     - 在项目后期和复杂设计中使用FSDB格式，结合Verdi工具分析细致波形和交互关系。
     - 利用Verdi的过滤和筛选功能查看关键信号，减少不必要的波形数据
