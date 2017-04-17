---
title: 如何用VerilogHDL编写一个单周期MIPS32CPU（三）
date: 2016-05-02 17:46:22
tags: 计算机原理与汇编
---

第三篇为设计取指令模块(IFU)。

本节将设计一个简化的处理器取指令电路，通过这个例子体会Verilog HDL的使用。

处理器内部一般有一个PC寄存器，其中存储指令地址，正常运行过程中，PC的值会随时间增加，同时从指令存储器中取出对应地址的指令。所以，本节实现的处理器取指令电路，包含两部分：PC模块、指令存储器。

## 电路设计举例
### PC模块的设计与实现

PC模块的功能就是给出取指令地址，同时每个时钟周期取指令地址递增。其接口设计如图所示。采用左边是输入接口，右边是输出接口的方式绘制，这样便于理解。
![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/3/1.png)
![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/3/2.png)
<!-- more -->
此处定义指令地址pc的宽度为6，PC模块的主要代码如下。
```
module pc_reg(  

    input   wire         clk,  
    input  wire      rst,  

    output reg[5:0]    pc,  
    output reg         ce  

);  

    always @ (posedge clk) begin  //在时钟信号上升沿触发  
       if (rst == 1'b1) begin  
        ce <= 1'b0;      //复位信号有效的时候，指令存储器使能信号无效  
       end else begin  
        ce <= 1'b1;      //复位信号无效的时候，指令存储器使能信号有效  
       end  
    end  

always @ (posedge clk) begin  //在时钟信号上升沿触发  
       if (ce == 1'b0) begin  
        pc <= 6'h00;     //指令存储器使能信号无效的时候，pc保持为0  
       end else begin  
        pc <= pc + 1'b1; //指令存储器使能信号有效的时候，pc在每个时钟加1  
       end  
    end  

endmodule  
```
### 指令存储器ROM的设计与实现
指令存储器ROM的作用是存储指令，并依据输入的地址，给出对应地址的指令。其接口如图2-14所示，还是采用左边是输入接口，右边是输出接口的方式绘制，这样便于理解。
![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/3/3.png)
![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/3/4.png)

此处定义指令的宽度为32，指令存储器ROM的主要代码如下。

```
module rom(  

    input  wire          ce,  
    input  wire[5:0]     addr,  
    output reg[31:0]     inst  

);  

    reg[31:0]  rom[63:0];      //使用二维向量定义存储器  

    always @ (*) begin  
       if (ce == 1'b0) begin  
        inst <= 32'h0;      //使能信号无效时，给出的数据是0  
       end else begin  
           inst <= rom[addr];  //使能信号有效时，给出地址addr对应的指令  
       end  
    end  

endmodule  
```
其中使用了一个二维向量定义存储器，深度是64，每个元素的宽度是32，这也是使用6位地址即可的原因。

### 顶层文件
先介绍元件例化的知识，在一个复杂电路的实现过程中，可以将其分割成多个功能单元分别实现，然后在一个顶层文件中通过调用各个功能单元，将其按照一定方式连接在一起，从而实现最终电路。其中调用功能单元的过程就称为元件例化。
```
<模块名> <实例名>(

    <相连的端口名>(相连的信号名),
    <相连的端口名>(相连的信号名),
    ……
);
```
经过上面两步，我们分别实现了PC模块、指令存储器ROM，现在可以编写顶层文件将两者连接起来。
![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/3/5.png)

PC模块的输出pc连接到指令存储器ROM的地址接口addr，PC模块输出的使能信号ce连接到ROM的使能信号接口ce。顶层模块对应的模块名为IFU，有三个接口。

![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/3/6.png)

IFU模块的主要代码如下，其中例化了PC模块、指令存储器ROM。
```
module inst_fetch(  

    input   wire        clk,  
    input  wire     rst,  
    output wire[31:0]   inst_o  

);  

    wire[5:0] pc;  
    wire      rom_ce;  


       //PC模块的例化  
    pc_reg pc_reg0(.clk(clk),   .rst(rst),  
                .pc(pc),    .ce(rom_ce));  

       //指令存储器ROM的例化  
       rom rom0( .ce(rom_ce),   .addr(pc),  .inst(inst_o));  

endmodule
```
PC模块的输出pc、ROM模块的输入addr都连接到变量pc，所以两者连接在一起；PC模块的输出ce、ROM模块的输入ce都连接到rom_ce，所以两者连接在一起。
