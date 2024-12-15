---
bookToc: false
weight: 24
---

# 实验2 同步 FIFO 设计

## 实验内容
完成同步 FIFO（First-In-First-Out 数据缓存器）的设计，包括 Verilog 代码和测试文件验证。

## 1. FIFO 介绍
FIFO 是一种**先进先出**的数据缓存器，与普通存储器的区别在于：
- 没有外部读写地址线，操作简单。
- 只能**顺序读写**，不能随机访问。

### 1.1 FIFO 的应用场景
1. **数据缓冲**：
   - 当数据写入速率过快或写入不连续时，FIFO 用于暂存数据，使后续处理流程平滑。
2. **时钟域隔离**（主要是异步 FIFO）：
   - 用于跨时钟域数据传输，避免设计和约束复杂度。
   - 例如：AD 端与 PCI 总线之间数据传输。AD 采样速率为 16 位 100KSPS（1.6Mbps），而 PCI 总线速度为 33MHz，32 位宽。
3. **不同数据宽度的接口适配**：
   - 例如：单片机（8 位）与 DSP（16 位）之间的数据传输。

### 1.2 同步 FIFO
同步 FIFO 是指**读时钟**和**写时钟**为同一个时钟源。在时钟沿到来时，可同时进行读写操作。

#### 同步 FIFO 结构
同步 FIFO 的核心是**双口 RAM**和**读写控制逻辑**。

```
                    +------------------+
      i_w_en -----> |                  |          o_buf_full
      i_data -----> |     FIFO 内核    | ------------------->
                    |    (双口 RAM)    |
                    |                  |       o_buf_empty
      i_r_en -----> |                  | ------------------->
                    +------------------+
                          ^     ^
                          |     |
                       写指针   读指针
```
### 1.3 双口 RAM
双口 RAM（Dual-Port RAM）是一种支持两个独立端口的存储器，可以同时进行读操作和写操作。在同步 FIFO 设计中，双口 RAM 主要用于实现高效的数据存储和访问，其主要作用包括：
- 同时读写：读和写操作可以在同一个时钟周期内独立进行，互不干扰。写指针控制写入数据，读指针控制读取数据。
- 避免冲突：双口 RAM 提供了两个独立的地址端口（一个用于写入、一个用于读取），因此能够避免传统单端口 RAM 只能分时读写的问题。
高效的数据缓存：FIFO 中的数据通过写指针依次写入双口 RAM，通过读指针顺序读出，确保数据先进先出（FIFO）的特性。
双端RAM的代码：
```verilog
module ram_dual(RST, CLK_R, CLK_W, RD_EN, WRT_EN, ADDR_R, ADDR_W, DATA_WRT, DATA_RD);

parameter   DATA_WIDTH = 8;
parameter   RAM_DEEP   = 128;
parameter   ADDR_WIDTH = 7;

input		CLK_R;
input		CLK_W;
input		RST;
input		RD_EN;
input		WRT_EN;

input	[ADDR_WIDTH-1:0]    ADDR_R;
input	[ADDR_WIDTH-1:0]    ADDR_W;
input	[DATA_WIDTH-1:0]    DATA_WRT;

output	[DATA_WIDTH-1:0]    DATA_RD;

reg	[DATA_WIDTH-1:0]    DATA_RD;
reg	[DATA_WIDTH-1:0]    MEM	[0:RAM_DEEP-1];

// reg	[ADDR_WIDTH:0]    i;
// always @(posedge CLK_W or posedge RST) begin
//     if (RST) begin
// 	for(i=0; i<RAM_DEEP; i=i+1) begin
// 	    MEM[i] <= 0;
// 	    // #0.001;
// 	    // $display("iteration is %d", i);
// 	end
//     end
//     else if (WRT_EN)
// 	MEM[ADDR_W] <= DATA_WRT;
// end

generate
genvar i;
for (i=0; i<RAM_DEEP; i=i+1)
begin: MEM_GEN
    always @(posedge CLK_W or posedge RST) begin
	if (RST)
	    MEM[i] <= 0;
	else if (WRT_EN)
	    MEM[i] <= (ADDR_W == i) ? DATA_WRT : MEM[i];
    end
end
endgenerate


always @(posedge CLK_R or posedge RST) begin
    if (RST)
	DATA_RD <= 0;
    else if (RD_EN)
	DATA_RD <= MEM[ADDR_R];
end

endmodule
```


### 1.4 FIFO 参数与信号定义
| 信号         | 说明                                     |
|--------------|------------------------------------------|
| **clk**      | 系统时钟                                 |
| **rstn**     | 系统复位信号（低电平有效）               |
| **wr_en**    | 写使能信号，控制数据写入                 |
| **wr_data**  | 写入数据                                 |
| **fifo_full**| FIFO 满标志位，表示 FIFO 已满            |
| **rd_en**    | 读使能信号，控制数据读出                 |
| **rd_data**  | 读出数据                                 |
| **fifo_empty**| FIFO 空标志位，表示 FIFO 为空           |
| **读指针**   | 指向下一个要读出的地址                  |
| **写指针**   | 指向下一个要写入的地址                  |

### 1.5 FIFO 设计原则
1. **写入溢出（Overflow）**：
   - 禁止在 FIFO 满时继续写入数据。
2. **读取溢出（Underflow）**：
   - 禁止在 FIFO 为空时继续读取数据。
3. **指针管理**：
   - FIFO 初始化时，读指针和写指针均指向地址 0。
   - 每写入一个数据，写指针自动加 1；每读出一个数据，读指针自动加 1。
   - 通过比较读、写指针状态，判断 FIFO 的**空**和**满**状态。

---

## 2. 同步 FIFO 基本接口定义
```verilog
module sync_fifo#(parameter BUF_SIZE=？, BUF_WIDTH=？) (
    input                      i_clk,       // 系统时钟
    input                      i_rst,       // 复位信号
    input                      i_w_en,      // 写使能信号
    input                      i_r_en,      // 读使能信号
    input      [BUF_WIDTH-1:0] i_data,      // 写入数据
   
    output reg [BUF_WIDTH-1:0] o_data,      // 读出数据
    output                     o_buf_empty, // FIFO 空标志位
    output                     o_buf_full   // FIFO 满标志位
);

    // 在此添加设计逻辑

endmodule
```

---

## 3. 设计实现要求
1. 完成 `sync_fifo` 模块的 Verilog 代码设计，宽度为8，深度为8。
2. 确保以下功能：
   - 写入数据时，写指针自动更新。
   - 读取数据时，读指针自动更新。
   - 判断 FIFO 的空（`o_buf_empty`）和满（`o_buf_full`）状态。
3. 编写**测试文件**（Testbench）验证设计。
4. 最终检查：使用给定的 TB 文件验证设计的正确性。

---

## 4. 验证测试
给出测试文件的基本框架，可以使用以下 TB 文件来验证设计逻辑。

### 示例 Testbench
```verilog
module tb_sync_fifo;
    reg         clk;
    reg         rst;
    reg         wr_en;
    reg         rd_en;
    reg  [7:0]  wr_data;
    wire [7:0]  rd_data;
    wire        fifo_empty;
    wire        fifo_full;

    // 实例化同步 FIFO 模块
    sync_fifo uut (
        .i_clk       (clk),
        .i_rst       (rst),
        .i_w_en      (wr_en),
        .i_r_en      (rd_en),
        .i_data      (wr_data),
        .o_data      (rd_data),
        .o_buf_empty (fifo_empty),
        .o_buf_full  (fifo_full)
    );

    // 时钟生成
    always #5 clk = ~clk;

    initial begin
        // 初始化
        clk = 0;
        rst = 1;
        wr_en = 0;
        rd_en = 0;
        wr_data = 8'h00;
        
        #10 rst = 0; // 释放复位

        // 测试写入操作
        repeat (10) begin
            @(posedge clk);
            wr_en = 1;
            wr_data = wr_data + 1;
        end
        wr_en = 0;

        // 测试读取操作
        repeat (10) begin
            @(posedge clk);
            rd_en = 1;
        end
        rd_en = 0;

        #20 $finish;
    end
endmodule
```

---

## 5. 任务总结
1. 完成同步 FIFO 设计的基本逻辑。
2. 实现了空、满状态的判断。
3. 使用测试文件验证设计是否符合要求。

