---
title: 如何用VerilogHDL编写一个单周期MIPS32CPU（六）
date: 2016-05-07 19:11:46
tags: 计算机原理与汇编
---
第六篇实现译码、执行、访存、回写四个阶段的功能。
数据通路、顶层文件以及SOPC将在下一篇中构建。

请注意一些细节，比如reg和wire类型的区别。赋值时reg类型要用always语句和非阻塞赋值（阻塞也可以），wire类型要用assign，否则会报错。

之前没有提到的一点语法细节是条件赋值语句。
```
//条件赋值语句
//当条件为1时结果为a，条件为0时结果为b
    assign 结果 = (条件) ? a : b
```
使用条件赋值语句可以大大简化代码，不用写那么长的if或case语句了。
不过只适用于一位的条件。两位的还是用case吧。
<!-- more -->

## 译码阶段
### 寄存器组的实现
```
module rf(
    //输入
    input wire          clk,
    input wire          rst,
    input wire          RegWrite,   //寄存器写使能
    input wire[4:0]     RegAddr1,   //读取寄存器1地址
    input wire[4:0]     RegAddr2,   //读取寄存器2地址
    input wire[4:0]     WriteAddr,  //写入寄存器地址
    input wire[31:0]    BusW,       //写入寄存器

    //输出
    output wire[31:0]   BusA,       //读取寄存器A
    output wire[31:0]   BusB,       //读取寄存器B
    output wire[31:0]   Reg31       //$31中的内容
    );

    reg[31:0] RegFile[0:31];       //二维向量定义寄存器组

    integer i;

    //使用条件赋值语句保证0号寄存器读取内容始终为0
    assign BusA = (RegAddr1 != 5'b0) ? RegFile[RegAddr1] : 0;    //根据地址从寄存器1读取
    assign BusB = (RegAddr2 != 5'b0) ? RegFile[RegAddr2] : 0;    //根据地址从寄存器2读取
    assign Reg31 = RegFile[31];                                  //从寄存器31中读取返回地址（jr）

    always @ (posedge clk or negedge rst) begin
        if(rst) begin
            for (i = 0; i < 32; i = i + 1) begin
                RegFile[i] <= 0;
            end
        end else begin
            if (RegWrite && WriteAddr != 0) begin               //写使能有效时
                RegFile[WriteAddr] <= BusW ;                    //进行写入
            end
        end
    end
endmodule
```

### 控制器的具体实现
#### 控制器的宏定义
```
//Controller中的宏定义

`define ADD   3'b000              //加法
`define SUB   3'b001              //减法
`define OR    3'b010              //或
`define LUI   3'b011              //lui
`define SLT   3'b100              //slt

`define R     6'b00_0000        //R型指令的opcode
`define addu  6'b10_0001        //以下为R型指令的funct
`define subu  6'b10_0011
`define slt   6'b10_1010
`define jr    6'b00_1000

`define addi  6'b00_1000        //以下为I型指令的opcode
`define addiu 6'b00_1001
`define ori   6'b00_1101
`define lw    6'b10_0011
`define sw    6'b10_1011
`define beq   6'b00_0100
`define lui   6'b00_1111

`define j     6'b00_0010       //以下为J型指令的opcode
`define jal   6'b00_0011

`define ALU       2'b00        //寄存器组写入数据
`define DMout     2'b01
`define JalAddr   2'b10

`define Rt        2'b00        //寄存器组写入内容
`define Rd        2'b01
`define LAST      2'b10

`define Plus      2'b00        //跳转指令的判断
`define Beq       2'b01
`define J         2'b10
`define Jr        2'b11
```

#### 控制器
```
`include  "define.v"
module control (
    //输入
    input wire[5:0] op,
    input wire[5:0] funct,

    //输出
    //寄存器组
    output reg RegWr,          //寄存器组写使能
    output reg[1:0] RegDst,    //寄存器组写入地址
                                //2'b00：rt         2'b01：rd
                                //2'b10：$31
    output reg[1:0] MemtoReg,  //选择寄存器组写入数据
                                //2'b00：ALU计算结果  2'b01：数据储存器的输出
                                //2'b10：jal指令的地址

    //运算器
    output reg AluSrc,         //ALU运算数         0：rt            1：立即数符号扩展的结果
    output reg[2:0] ALUctr,    //ALU的运算模式
                                //3'b000：加法运算    3'b010：减法运算
                                //3'b010：或运算      3'b011：lui运算      3'b100:slt运算
    output reg ExtOp,          //判断是否为符号扩展

    //数据储存器
    output reg MemWr,          //数据储存器写使能

    //指令控制信号
    output reg[1:0] nPCsel     //判断是否为beq或j型命令
    );

    always @ ( * )  begin
        if (op==`R) begin
            case (funct)
                `addu : begin
                    //寄存器组
                    RegWr   <=1;        //寄存器组写使能
                    RegDst  <=`Rd;      //寄存器组写入地址：Rt，Rd，LAST
                    MemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                    //运算器
                    AluSrc  <=0;        //ALU运算数         0：rt           1：extimm
                    ALUctr  <=`ADD;     //运算模式
                    //数据储存器
                    MemWr   <=0;        //数据储存器写使能
                    //PC
                    nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                end

                `subu : begin
                    //寄存器组
                    RegWr   <=1;        //寄存器组写使能
                    RegDst  <=`Rd;      //寄存器组写入地址：Rt，Rd，LAST
                    MemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                    //运算器
                    AluSrc  <=0;        //ALU运算数        0：rt           1：extimm
                    ALUctr  <=`SUB;     //运算模式
                    //数据储存器
                    MemWr   <=0;        //数据储存器写使能
                    //PC
                    nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                end

                `slt : begin
                    //寄存器组
                    RegWr   <=1;        //寄存器组写使能
                    RegDst  <=`Rd;      //寄存器组写入地址：Rt，Rd，LAST
                    MemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                    //运算器
                    AluSrc  <=0;        //ALU运算数        0：rt           1：extimm
                    ALUctr  <=`SLT;     //运算模式
                    //数据储存器
                    MemWr   <=0;        //数据储存器写使能
                    //PC
                    nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                end

                `jr : begin
                    //寄存器组
                    RegWr   <=0;        //寄存器组写使能
                    //运算器

                    //数据储存器
                    MemWr   <=0;        //数据储存器写使能
                    //PC
                    nPCsel  <=`Jr;      //跳转指令的判断：Plus，Beq，J，Jr
                end
            endcase
        end else begin
            case (op)
                `addi : begin
                    //寄存器组
                    RegWr   <=1;        //寄存器组写使能
                    RegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                    MemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                    //运算器
                    AluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                    ALUctr  <=`ADD;     //运算模式
                    ExtOp   <=1;        //是否为符号扩展
                    //数据储存器
                    MemWr   <=0;        //数据储存器写使能
                    //PC
                    nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                end

                `addiu : begin
                    //寄存器组
                    RegWr   <=1;        //寄存器组写使能
                    RegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                    MemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                    //运算器
                    AluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                    ALUctr  <=`ADD;     //运算模式
                    ExtOp   <=1;        //是否为符号扩展
                    //数据储存器
                    MemWr   <=0;        //数据储存器写使能
                    //PC
                    nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                end

                `ori : begin
                    //寄存器组
                    RegWr   <=1;        //寄存器组写使能
                    RegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                    MemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                    //运算器
                    AluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                    ALUctr  <=`OR;      //运算模式
                    ExtOp   <=0;        //是否为符号扩展
                    //数据储存器
                    MemWr   <=0;        //数据储存器写使能
                    //PC
                    nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                end

                `lw : begin
                    //寄存器组
                    RegWr   <=1;        //寄存器组写使能
                    RegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                    MemtoReg<=`DMout;   //寄存器组写入数据：ALU，DMout，JalAddr
                    //运算器
                    AluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                    ALUctr  <=`ADD;   //运算模式
                    ExtOp   <=1;        //是否为符号扩展
                    //数据储存器
                    MemWr   <=0;        //数据储存器写使能
                    //PC
                    nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                end

                `sw : begin
                    //寄存器组
                    RegWr   <=0;        //寄存器组写使能
                    RegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                    //运算器
                    AluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                    ALUctr  <=`ADD;     //运算模式
                    ExtOp   <=1;        //是否为符号扩展
                    //数据储存器
                    MemWr   <=1;        //数据储存器写使能
                    //PC
                    nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                end

                `beq : begin
                    //寄存器组
                    RegWr   <=0;        //寄存器组写使能
                    //运算器
                    AluSrc  <=0;        //ALU运算数       0：rt           1：extimm
                    ALUctr  <=`SUB;   //运算模式
                    //数据储存器
                    MemWr   <=0;        //数据储存器写使能
                    //PC
                    nPCsel  <=`Beq;     //跳转指令的判断：Plus，Beq，J，Jr
                end

                `lui : begin
                    //寄存器组
                    RegWr   <=1;        //寄存器组写使能
                    RegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                    //运算器
                    AluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                    ALUctr  <=`LUI;     //运算模式
                    ExtOp   <=1;        //是否为符号扩展
                    //数据储存器
                    MemWr   <=1;        //数据储存器写使能
                    //PC
                    nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                end

                `j : begin
                    //寄存器组
                    RegWr   <=0;        //寄存器组写使能
                    //运算器

                    //数据储存器
                    MemWr   <=0;        //数据储存器写使能
                    //PC
                    nPCsel  <=`J;       //跳转指令的判断：Plus，Beq，J，Jr
                end

                `jal : begin
                    //寄存器组
                    RegWr   <=1;        //寄存器组写使能
                    RegDst  <=`LAST;    //寄存器组写入地址：Rt，Rd，LAST
                    MemtoReg<=`JalAddr; //寄存器组写入数据：ALU，DMout，JalAddr
                    //运算器

                    //数据储存器
                    MemWr   <=0;        //数据储存器写使能
                    //PC
                    nPCsel  <=`J;       //跳转指令的判断：Plus，Beq，J，Jr
                end
            endcase
        end
    end
 endmodule
 ```

## 执行阶段
### 运算器运算数的选择器
```
//ALU运算数的选择器
module AluSrcMux(
    //输入
    input wire[31:0] rd2,
    input wire[31:0] ext,
    input wire AluSrc,
    //输出
    output reg[31:0] result
    );

    //0：将rt作为运算数
    //1：将立即数扩展结果作为运算数
    always @ ( * ) begin
        result <= AluSrc ? ext : rd2;
    end
endmodule
```

### 运算器
```
module alu(
    //输入
    input wire[31:0] A,         //运算数A
    input wire[31:0] B,         //运算数B
    input wire[2:0] AluCtr,     //ALU运算模式控制信号

    //输出
    output reg[31:0] result,    //运算结果
    output wire zero            //判断beq
    );

    always @ ( * ) begin
        case (AluCtr)
            3'b000 : result <= A + B ;
            3'b001 : result <= A + ~B + 1 ;
            3'b010 : result <= A | B ;
            3'b011 : result <= { B [15:0] , 16'b0 };
            3'b100 : begin
                    //SLT指令，首先比较符号位，根据符号判断大小
                    if (A[31] == 1 && B[31] == 0) begin
                        result <= 1;
                    end else if(A[31] == 0 && B[31] == 0) begin
                        result <= (A[30:0] < B[30:0]) ? 1 : 0;
                    end else begin
                        result <= (A[30:0] < B[30:0]) ? 0 : 1;
                    end
                end
        endcase
    end

    //若结果为零（即按位或结果为1），则zero=0，否则为1
    assign zero = ( result[31:0] == 32'b0 ) ? 0 : 1 ;

endmodule
```
### 扩展器
```
module ext(
    //输入
    input wire[15:0] imm,           //输入的16位立即数
    input wire extop,               //0为无符号扩展，1为符号扩展

    //输出
    output reg[31:0] extimm        //扩展后32位
    );

    //无符号扩展在高位补16个0
    //符号扩展将符号位（也就是第15位）重复16次扩展在高位上
    always @ ( * ) begin
        extimm <= extop ? { {16{imm[15]}} , imm} : {16'b0 , imm};
    end
endmodule
```
## 访存阶段
### 数据储存器的实现
```
module dm(
    //输入
    input wire clk,
    input wire[11:2] addr,                  //数据地址
    input wire[31:0] din,                   //输入数据
    input wire MemWr,                       //数据储存器写使能

    //输出
    output reg[31:0] dout                  //输出数据
    );

    reg [31:0] DataMem[1023:0];             //二维向量定义数据储存器，大小为4kb

    always @ (*) begin
        dout = DataMem[addr[11:2]];           //读取数据
    end

    always @ (posedge clk) begin
        if (MemWr)                          //写使能有效时
            DataMem[addr[11:2]] <= din;     //进行写入
    end

endmodule
```
## 回写阶段
### 寄存器组写入地址的选择器
```
module RegDstMux(
    //输入
    input wire[4:0] rt,
    input wire[4:0] rd,
    input wire[1:0] RegDst,
    //输出
    output reg[4:0] result
    );

    //00：将rt作为写入地址
    //01：将rd作为写入地址（R型指令）
    //10：将$31作为写入地址（Jr）

    always @ (*) begin
        case(RegDst)
            2'b00: result <= rt;
            2'b01: result <= rd;
            2'b10: result <= 5'b11111;
        endcase
    end
endmodule
```
## 寄存器组写入内容的选择器
```
module MemtoRegMux(
    //输入
    input wire[31:0] ALUResult,
    input wire[31:0] DataMem,
    input wire[31:0] jalAddr,
    input wire[1:0] MemtoReg,
    //输出
    output reg[31:0] result
    );

    //00：将ALU计算结果写入寄存器
    //01：将数据储存器读取结果写入寄存器（lw）
    //10：将jal指令的返回地址写入$31中

    always @ ( * ) begin
        case(MemtoReg)
            2'b00: result <= ALUResult;
            2'b01: result <= DataMem;
            2'b10: result <= jalAddr;
        endcase
    end
endmodule
```
