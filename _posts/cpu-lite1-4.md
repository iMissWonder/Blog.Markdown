---
title: 如何用VerilogHDL编写一个单周期MIPS32CPU（四）
date: 2016-05-02 19:50:25
tags: 计算机原理与汇编
---

第四篇为介绍仿真，ModelSim的使用。

这里默认已经安装了ModelSim(需要破解)。

## 两个问题
1、如何在指令存储器中存储指令，也就是指令存储器初始化问题。
————使用$readmemh函数

2、如何给出时钟信号？
————使用Test Bench

## 两个重要函数
### $stop
$stop用于对仿真过程进行控制，暂停仿真，其使用格式如下。
```
$stop();       // 使用格式一，不带参数  
$stop(n);      // 使用格式二，带参数n，n可以等于0、1、2等值，含义如下：  
               // 0:不输出任何信息；  
               // 1：给出仿真时间和位置  
               // 2：给出仿真时间和位置，还有其它一些运行统计数据  
```

<!-- more -->
### $readmemh
$readmemh函数用于读取文件，其作用是从外部文件中读取数据并放入存储器中。使用
```
$readmemh("数据文件名", 存储对象);  
```
将第1个参数指定文件的数据读入第2个参数指定的存储器中。
```
reg[31:0]  rom[63:0];  

initial $readmemh ( "code.txt", rom );  // 读入文件rom.data的数据到rom中
```
此处对数据文件的格式有一定要求，要求使用十六进制记录数据，且每一行记录一个地址的数据。例如：rom.data的内容如下，每一行是一个32位的数据。
```
rom[0]： 32'h00000000;   //存储器rom的第0个元素初始化为0x00000000  
rom[1]： 32'h01010101;   //存储器rom的第1个元素初始化为0x01010101  
rom[2]： 32'h02020202;   //存储器rom的第2个元素初始化为0x02020202  
rom[3]： 32'h03030303;   //存储器rom的第3个元素初始化为0x03030303  
......  
```
回到本节最开始提出的两个问题，现在可以回答第一个问题了，为了实现对指令存储器的初始化，只需要创建一个数据文件，其内容如上面的code.txt所示，然后在指令存储器rom.v中，增加代码$readmemh ("code.txt", rom )即可。

**因此在之前创建的rom中要一定要添加$readmemh函数，否则寄存器始终为x（未定值）**

##Test Bench

我们通过创建Test Bench文件以给出时钟信号。

> Test Bench为测试或仿真一个Verilog HDL程序搭建了一个平台，我们给被测试的模块施加激励信号，通过观察被测试模块的输出响应，从而判断其逻辑功能实现的正确与否。

![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/4/1.png)

Test Bench的结构如图所示，与Verilog HDL模块的结构没有根本区别，但有自身的一些特点。

Test Bench只有模块名，没有端口列表；激励信号（输入到待测试模块的信号）必须定义为reg类型，以保持信号值；从待测试模块输出的信号（用户观察的信号）必须定义为wire类型。
在Test Bench中要调用被测试模块，也就是元件例化。
Test Bench中一般会使用initial、always过程块来定义、描述激励信号。
```
module <Test Bench名>;
       <数据类型说明> //激励信号使用reg类型，显示信号使用wire类型
       <激励向量定义> //always、initial过程块等
       <待测试模块例化>
endmodule
```
为简单取指令电路（IFU）设计的Test Bench如下
```
module inst_fetch_tb;          // Test Bench名为inst_fetch_tb  

/****************************************************************
***********              第一段：数据类型说明             *********
*****************************************************************/  

  reg           CLOCK;         // 激励信号CLOCK，这是时钟信号  
  reg           rst;           // 激励信号rst，这是复位信号  
  wire[31:0]    inst;          // 显示信号inst，取出的指令  

/****************************************************************
***********              第二段：激励向量定义             *********
*****************************************************************/  

  // 定义CLOCK信号，每隔10个时间单位，CLOCK的值翻转，由此得到一个周期信号。  
  // 在仿真的时候，一个时间单位默认是1ns，所以CLOCK的值每10ns翻转一次，对应  
  // 就是50MHz的时钟  
  initial begin  
    CLOCK = 1'b0;  
    forever #10 CLOCK = ~CLOCK;  
  end  

  // 定义rst信号，最开始为1，表示复位信号有效，过了195个时间单位，即195ns，  
  // 设置rst信号的值为0，复位信号无效，复位结束，再运行1000ns，暂停仿真  
  initial begin  
    rst = 1'b1;  
    #195 rst= 1'b0;  
    #1000 $stop;  
  end  

/****************************************************************
***********             第三段：待测试模块例化            *********
*****************************************************************/  

  inst_fetch inst_fetch0(  
        .clk(CLOCK),  
        .rst(rst),  
        .inst_o(inst)     
    );  

endmodule  
```

*ModelSim仿真过程参照*
http://blog.csdn.net/leishangwen/article/details/38040203