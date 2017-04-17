---
title: 如何用VerilogHDL编写一个单周期MIPS32CPU（七）
date: 2016-05-07 20:06:30
tags: 计算机原理与汇编
---
第七篇为构建数据通路、顶层文件以及SOPC。写Testbench并用ModelSim进行仿真验证。*这是最后一篇*

## 将各个模块组装起来
首先需要组装的是数据通路。数据通路包括除指令储存器、数据储存器和控制器以外的所有文件。
然后要组装MIPS，也就是将数据通路和控制器组装起来。
这里要注意到MIPS并不是最顶层的文件。最终我们要将MIPS和IM与DM进行连接，构成最小SOPC，这才是真正的顶层文件。
因此最后的仿真也是在最小SOPC上进行的。

> SOPC：System-on-a-Programmable-Chip，可编程片上系统。 用可编程逻辑技术把整个系统放到一块硅片上，即由单个芯片完成整个系统的主要逻辑功能。
<!-- more -->

### 数据通路的构建
```
module datapath(
    //输入
    input wire clk,
    input wire reset,
    //指令
    input wire[31:0] Inst,
    //寄存器组
    input wire RegWr,
    input wire[1:0] RegDst,
    input wire[1:0] MemtoReg,
    //运算器
    input wire AluSrc,
    input wire[3:0] AluCtr,
    input wire extOp,

    //数据储存器
    input wire MemWr,
    input wire [31:0] ReadData,

    //
    input wire[1:0] nPCsel,

    //输出
    output wire[31:0] Pc,         //下一个pc
    output wire[31:0] ALU,        //ALU计算结果
    output wire[31:0] WriteData   //写入数据储存器的内容（sw）
    );

    wire[31:0] Npc;               //下一个pc的值
    wire[31:0] jalInput;          //$31中的返回地址
    wire[31:0] jalAddr;           //jal指令的返回地址
    wire zero;                    //判断beq

    wire[31:0] R1 , R2;           //两个进行计算的数
    wire[4:0] BusWAddr;           //BusW的输入地址
    wire[31:0] BusWInput;         //BusW的输入内容

    wire[31:0] Ext;               //imm16扩展后的数

    //计算下一条PC指令
    pc PC(Npc , clk , reset , Pc) ;
    npc NPC(Pc , Inst , nPCsel , jalInput , zero , Npc, jalAddr) ;

    //寄存器组
    rf RF(clk , reset, RegWr , Inst[25:21] , Inst[20:16] , BusWAddr , BusWInput , R1 ,WriteData, jalInput);
    RegDstMux regdstmux( Inst[20:16] , Inst[15:11] , RegDst , BusWAddr );
    MemtoRegMux MtRMux( ALU , ReadData ,jalAddr ,MemtoReg , BusWInput);

    //运算器
    ext EXT(Inst[15:0], extOp , Ext) ;
    AluSrcMux alusrcmux( WriteData , Ext , AluSrc , R2 );
    alu alu( R1 , R2 , AluCtr , ALU , zero) ;


endmodule
```
### MIPS的构建
```
module mips(
    //输入
    input wire clk,
    input wire reset,
    input wire[31:0] Inst,              //指令储存器输入的指令
    input wire[31:0] ReadData,          //数据储存器读取

    //输出
    output wire[31:0] ALU,              //运算器输出结果
    output wire MemWr,                  //数据储存器写使能
    output wire[31:0] WriteData,        //数据储存器写入内容
    output wire[31:2] pc                //指令地址
    );

    //寄存器组信号
    wire RegWr;
    wire[1:0] RegDst;
    wire[1:0] MemtoReg;

    //运算器信号
    wire AluSrc;
    wire[2:0] AluCtr;
    wire extop;

    //指令控制信号
    wire[1:0] nPCsel;

    control control( Inst[31:26] , Inst[5:0] , RegWr , RegDst, MemtoReg, AluSrc,
        AluCtr, extop, MemWr, nPCsel);

    datapath datapath( clk , reset , Inst, RegWr, RegDst, MemtoReg, AluSrc, AluCtr,
        extop , MemWr, ReadData , nPCsel, pc, ALU, WriteData);

endmodule
```

### SOPC的构建
```
module mips_sopc(
    //输入
    input wire CLK,
    input wire RESET ,

    //输出
    output wire[31:0] DATA,         //运算器输出结果
    output wire MEMWRITE,           //数据储存器写使能
    output wire[31:0] WRITEDATA     //数据储存器器写入内容
    );

    wire [31:2] PC ;
    wire [31:0] INST , READDATA ;

    //连接mips,IM、DM，构成SOPC（System-on-a-Programmable-Chip）
    mips MIPS(CLK, RESET,  INST, READDATA, DATA, MEMWR, WRITEDATA, PC);

    im IM(PC[11:2] , INST);

    dm DM(CLK, DATA[11:2]  , WRITEDATA , MEMWR , READDATA);

endmodule
```

## 仿真
**进行仿真的目的是要看到寄存器组的内容和Mars运行结果相符**，这样就验证了代码的正确性。

### 指令内容
![指令](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/6/1.png)

### 运行结果对比
![Mars运行结果](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/6/2.png)
![仿真结果](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/6/3.png)
注意到$28和$29中内容不一致。这两个寄存器分别是全局指针和堆栈指针，我们的指令并没有使用到，因此我认为可以忽略。

### 仿真文件及调试
进行到这一步，别以为算是大功告成了。DEBUG的时间很可能会比写代码的时间更长。
建议在仿真时保留波形，遇到BUG从指令出发逐个信号分析。
$stop函数最好要有，不然仿真起来比较麻烦。timescale也建议调成1ns/1ns，方便查看。
```
`timescale  1ns/1ns
module mips_sopc_tb();

    reg clk;
    reg rst;
    wire[31:0] data;
    wire memwrite;
    wire[31:0] writedata;

    initial begin
        clk = 1'b0;
        forever #10 clk = ~clk;
    end

    initial begin
        rst = 1'b1;
        #195 rst= 1'b0;
        #2000 $stop;
    end

    mips_sopc SOPC(clk , rst, data, memwrite, memwrite);

endmodule
```

### 测试用例
即为code.txt内容
```
34100001
34110003
34080001
3c0c0001
3c0d000a
00102021
00082821
0c000c2e
00028021
02288823
1211fffa
34080004
20090004
240afff8
ad0c0000
8d0e0000
ad0d0004
8d0f0004
ad0efffc
8d12fffc
00082021
00092821
0c000c2e
00024021
0148c82a
13200016
01484023
11000001
3c0cffff
34000001
3c080005
3c090032
00082021
00092821
0c000c2e
00024021
00082021
00092821
0c000c2e
00024821
01094821
01284823
3c0a0069
112a0001
1000fff3
08000c33
00851021
03e00008
3c011234
34215678
0001d020
```

## 结语
无，这玩意真是做吐血了！
不过下一个应该就不会这么困难了，毕竟这次是从语法学起。
但愿吧！
