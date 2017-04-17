---
title: 如何用VerilogHDL编写一个单周期MIPS32CPU（五）
date: 2016-05-07 12:40:38
tags: 计算机原理与汇编
---
第五篇介绍各阶段的功能分析与具体实现，

*前四篇为VerilogHDL和ModelSim的操作，熟悉的话可以选择跳过。*

本篇将实现十二条指令的功能。
R型指令：addu subu slt jr
I型指令：addi(addiu) ori lw sw beq liu
J型指令：j jal
（没有实现溢出功能，在不计addi溢出的情况下addi和addiu是一样的，imm都需要进行符号扩展）

在开始前我们先来分析一下各阶段的功能。
①取指 ②译码 ③执行 ④访存 ⑤回写
<!-- more -->

## 各阶段功能分析
### 取指
取指阶段要从指令存储器中取指令，并把指令传出。
同时需要改变PC寄存器中的值，正常指令PC+4，beq和J型指令跳转到相应的PC地址。

### 译码
这里我们没有在数据通路中定义具体的译码器，采用将指令的高六位（opcode）和低六位（funct）传入控制器（controller），进行指令的分析操作，并输出相应的控制信号，相当于译码功能在控制器实现。控制器根据32位指令中的opcode进行指令识别，并改变控制信号。

### 执行
根据译码结果从寄存器组中读出相应的rs、rt的值后，不同的指令具有不同的操作。

R型指令中rd是寄存器的写入地址，rs是运算器的第一个操作数，rt是** 第二个**操作数（jr除外）。
I型指令中rt是寄存器的写入地址，rs是运算器的第一个操作数，** 扩展**后的立即数作为** 第二个**操作数。
J型指令进行地址的跳转，opcode外剩下26位均为地址值。

所以现在需要：
①区分第二个运算器操作数为rt或立即数的选择器。
当控制信号及操作数输入完毕后，通过控制运算器的信号在运算器中进行相应的操作，并将结果输出。

### 访存
只有lw和sw指令会对数据储存器进行访问。
lw指令会输出数据储存器中储存内容，并写入相应寄存器。
sw指令会将rt写入数据储存器相应的地址（rs+偏移量）。

### 回写
寄存器写入时需要两个值，一个是内容，一个是地址。
所以还需要：
②区分写入内容的选择器。
③区分写入地址的选择器。

### 文件结构分析
>单周期处理器由**datapath(数据通路)和controller(控制器)** 组成。
> a)	数据通路由如下module组成：
> ** PC(程序计数器)、NPC(NextPC计算单元)、GPR (通用寄存器组，也称为寄存器文件、寄存器堆)、ALU(算术逻辑单元)、EXT(扩展单元)、IM(指令存储器)、DM(数据存储器)。**
> b)	IM：容量为4KB(32bit×1024字)。
> c)	DM：容量为4KB(32bit×1024字)。

根据以上的分析，我们的数据通路内需要添加三个选择器。
需要注意的是，IM和DM是不包含在数据通路的定义内的，也不包含在mips中，而是在顶层sopc中。
![修改后的文件结构](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/5/1.png)

接下来将分阶段实现各模块的功能，一些指令细节的实现不具体说明，注释已经很清楚了。
## 取指阶段
### PC寄存器的实现
```
module pc(
    //输入
    input wire[31:2] npc,
    input wire clk,
    input wire rst,

    //输出
    output reg[31:2] pc
    );

    reg[31:0] pcreg;                  //定义PC寄存器

    always @ (*) begin
        pc <= pcreg[31:2];            //为PC寄存器赋值
    end

    always @ (posedge clk) begin
        if (rst) begin
            pcreg <= 32'h00003000;    //pc的初始值，与Mars中相同
        end else begin
            pcreg <= {npc,2'b00};
        end
    end
endmodule
```

### 计算下一个PC地址（NPC）的实现
```
module npc(
    //输入
    input wire[31:2] pc,        //30位pc
    input wire[31:0] im,        //指令储存器输出的指令
    input wire[1:0] nPCsel,     //跳转指令的判断
                                //00:不跳转      01：beq指令
                                //10:J型指令     11:jr指令
    input wire[31:0] jalInput,  //$31的内容
    input wire zero,            //beq指令判断是否相等的依据，zero为0时进行跳转

    //输出
    output reg[31:2] npc,       //下一个pc的值
    output reg[31:0] jaladdr    //储存到$31的返回地址
    );


    reg [29:0] beqaddr ;                                     //beq指令的30位pc码
    reg [29:0] jaddr;                                        //J型指令的30位pc码

    always @ (*) begin
        beqaddr <= {{14{im[15]}} ,im[15:0]} + pc + 1'b1;    //beq指令的跳转地址
        jaddr   <= {pc[31:28], im[25:0]};                   //J型指令的跳转地址
        jaladdr <= {pc, 2'b00}+ 4;                          //储存到$31的返回地址
    end

    always @ (*) begin
        case (nPCsel)
            2'b00 : npc <= pc + 1'b1;             //不跳转，pc+4
            2'b01 : begin
                if (zero == 0 ) begin
                    npc <= beqaddr;               //跳转到beq中的跳转地址
                end else begin
                    npc <= pc + 1'b1;             //zero不为0，pc+4
                end
            end
            2'b10 : npc <= jaddr;                 //跳转到J型指令的pc跳转地址
            2'b11 : npc <= jalInput[31:2];        //跳转到$31寄存器储存内容的地址
        endcase
    end
endmodule

```

### 指令储存器（IM）的实现
```
module im(
    //输入
    input wire[11:2] addr,

    //输出
    output reg[31:0] imout
    );

    reg [31:0] im[1023:0];               //二维向量定义指令储存器，大小为4KB

    //使用initial初始化指令储存器
    initial begin
        $readmemh("code.txt" , im) ;     //使用$readmemh函数加载文件中的32位指令
    end

    always @ (*) begin
        imout <= im[addr];               //根据地址输出指令
    end

endmodule
```
