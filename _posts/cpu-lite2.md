---
title: 多周期MIPS32CPU的实现
date: 2016-05-15 21:52:07
tags: 计算机原理与汇编
---
## 何为多周期？
>多周期CPU的基本思想是把每条指令的执行分成多个大致相等的阶段，每个阶段在一个时钟周期内完成；各阶段功能相对完整，产生下一阶段需要的数据并保存；比如完成一次存储器访存、一次寄存器读写或一次ALU操作等；各阶段的执行结果在下一个时钟到来时保存到相应存储单元或稳定地保持在组合电路中；时钟的宽度以最复杂阶段所用时间为准，通常取一次存储器读写时间。每阶段的执行由控制器输出不同控制信号，用于控制器中各单元电路的有序运行。控制器由有限状态机实现，状态机在每个时钟改变一个状态，输出相应控制信号，控制一个特定的执行阶段的具体操作。

<!-- more -->
## 仿真结果
仿真结果比较直观，首先先来看一下。
### Mars运行结果
![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu-2/1.png)

### ModelSim仿真结果
![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu-2/2.png)
从仿真结果能直观的看出来，每条指令会经过多个周期来完成。每个周期都对应一个阶段，不同指令经过的阶段可能是不一样的。
我在这里定义0为取指阶段，1为译码阶段，2为执行阶段，3为访存阶段，4为回写阶段。

### 前三条ori指令的仿真结果
![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu-2/3.png)
如图，ori指令没有3阶段。

## 修改方式
多周期MIPS32CPU完全可以依照单周期修改。我在这里做了以下几点修改：
① 彻底修改控制器，加入阶段控制
② 给PC加写使能
③ 将RF（GPR）的输出改为reg型，相当于在输出后加了两个寄存器
④ 将ALU的输出改为reg型，相当于在输出后加了一个寄存器
⑤ 加入lb，sb，lh，sh指令，修改了数据储存器

## 具体实现
这里只贴出控制器和数据储存器的实现，剩下的和单周期基本相同。
### 控制器
```
`include  "define.v"
module control (
    //输入
    input wire clk,
    input wire rst,
    input wire[5:0] op,
    input wire[5:0] funct,

    //输出
    //PC
    output reg pcWr,           //指令储存器写使能
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
    output reg[1:0] INMode,    //数据储存器输入模式
    output reg[1:0] OUTMode,   //数据储存器输出模式
    output reg MemWr,          //数据储存器写使能

    //指令控制信号
    output reg[1:0] nPCsel     //判断是否为beq或j型命令
    );

    //定义阶段变量
    integer currentstate;     //表示当前状态
    integer nextstate;        //五个阶段状态     0：取指    1：译码    2：执行    3：访存    4：回写
    reg[1:0] state;            //state[0]:访存阶段      state[1]：回写阶段

    //定义中间变量
    //寄存器组
    reg[1:0] TRegDst;       //寄存器组写入地址：Rt，Rd，LAST
    reg[1:0] TMemtoReg;     //寄存器组写入数据：ALU，DMout，JalAddr

    //运算器
    reg      TAluSrc;       //ALU运算数       0：rt           1：extimm
    reg[2:0] TALUctr;       //运算模式
    reg      TExtOp;        //是否为符号扩展

    //数据储存器
    reg      TMemWr;        //数据储存器写使能
    reg[1:0] TINMode;       //数据储存器输入模式  2'b00:32位       2'b01:16位      2'b11:8位
    reg[1:0] TOUTMode;      //数据储存器输出模式  2'b00:32位       2'b01:16位      2'b11:8位

    //重置阶段
    always @ (*) begin
        if ( rst == 1 ) begin
            currentstate <= 0;
            nextstate <= 1;
            state[0] <= 0;
            state[1] <= 0;
        end
    end

    //取指阶段
    always @ ( posedge clk ) begin
        if ( nextstate == 0 && rst == 0) begin
            //设置当前阶段
            currentstate <= 0;
            //设置下一阶段
            nextstate <= 1;
            //取指单元
            pcWr    <= 1;        //pc写使能
            //寄存器组
            RegWr   <= 0;        //寄存器组写使能
            //数据储存器
            MemWr   <= 0;        //数据储存器写使能
        end
    end

    //译码阶段前
    always @ ( posedge clk )  begin
        if ( nextstate == 1 && rst == 0)    begin
            //设置当前阶段
            currentstate <= 1;
            //设置下一阶段
            nextstate <= 2;
            //取指单元
            pcWr    <= 0;        //pc写使能
            //寄存器组
            RegWr   <= 0;        //寄存器组写使能
            //数据储存器
            MemWr   <= 0;        //数据储存器写使能
        end
    end

    //译码阶段
    always @ ( * )  begin
        if ( currentstate == 1 && rst == 0)    begin
            //译码操作
            if (op ==`R) begin
                case (funct)
                    `addu : begin
                        //判断阶段
                        state[0] <= 0;       //访存阶段
                        state[1] <= 1;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rd;      //寄存器组写入地址：Rt，Rd，LAST
                        TMemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                        //运算器
                        TAluSrc  <=0;        //ALU运算数         0：rt           1：extimm
                        TALUctr  <=`ADD;     //运算模式
                    end

                    `subu : begin
                        //判断阶段
                        state[0] <= 0;       //访存阶段
                        state[1] <= 1;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rd;      //寄存器组写入地址：Rt，Rd，LAST
                        TMemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                        //运算器
                        TAluSrc  <=0;        //ALU运算数        0：rt           1：extimm
                        TALUctr  <=`SUB;     //运算模式
                    end

                    `slt : begin
                        //判断阶段
                        state[0] <= 0;       //访存阶段
                        state[1] <= 1;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rd;      //寄存器组写入地址：Rt，Rd，LAST
                        TMemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                        //运算器
                        TAluSrc  <=0;        //ALU运算数        0：rt           1：extimm
                        TALUctr  <=`SLT;     //运算模式
                    end

                    `jr : begin
                        //判断阶段
                        state[0] <= 0;       //访存阶段
                        state[1] <= 0;       //回写阶段
                        //PC
                        nPCsel  <=`Jr;      //跳转指令的判断：Plus，Beq，J，Jr
                    end
                endcase
            end else begin
                case (op)
                    `addi : begin
                        //判断阶段
                        state[0] <= 0;       //访存阶段
                        state[1] <= 1;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                        TMemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                        //运算器
                        TAluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                        TALUctr  <=`ADD;     //运算模式
                        TExtOp   <=1;        //是否为符号扩展
                    end

                    `addiu : begin
                        //判断阶段
                        state[0] <= 0;       //访存阶段
                        state[1] <= 1;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                        TMemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                        //运算器
                        TAluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                        TALUctr  <=`ADD;     //运算模式
                        TExtOp   <=1;        //是否为符号扩展
                    end

                    `ori : begin
                        //判断阶段
                        state[0] <= 0;       //访存阶段
                        state[1] <= 1;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                        TMemtoReg<=`ALU;     //寄存器组写入数据：ALU，DMout，JalAddr
                        //运算器
                        TAluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                        TALUctr  <=`OR;      //运算模式
                        TExtOp   <=0;        //是否为符号扩展
                    end

                    `lb : begin
                        //判断阶段
                        state[0] <= 1;       //访存阶段
                        state[1] <= 1;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                        TMemtoReg<=`DMout;   //寄存器组写入数据：ALU，DMout，JalAddr
                        //运算器
                        TAluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                        TALUctr  <=`ADD;     //运算模式
                        TExtOp   <=1;        //是否为符号扩展
                        //数据储存器
                        TMemWr   <=0;        //数据储存器写使能
                        TOUTMode  <=`Byte;    //数据储存器输出模式
                    end

                    `lh : begin
                        //判断阶段
                        state[0] <= 1;       //访存阶段
                        state[1] <= 1;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                        TMemtoReg<=`DMout;   //寄存器组写入数据：ALU，DMout，JalAddr
                        //运算器
                        TAluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                        TALUctr  <=`ADD;     //运算模式
                        TExtOp   <=1;        //是否为符号扩展
                        //数据储存器
                        TMemWr   <=0;        //数据储存器写使能
                        TOUTMode  <=`HalfWord;//数据储存器输出模式
                    end

                    `lw : begin
                        //判断阶段
                        state[0] <= 1;       //访存阶段
                        state[1] <= 1;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                        TMemtoReg<=`DMout;   //寄存器组写入数据：ALU，DMout，JalAddr

                        //运算器
                        TAluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                        TALUctr  <=`ADD;   //运算模式
                        TExtOp   <=1;        //是否为符号扩展
                        //数据储存器
                        TMemWr   <=0;        //数据储存器写使能
                        TOUTMode  <=`Word;        //数据储存器输出模式  0:32位    1:16位
                    end

                    `sb : begin
                        //判断阶段
                        state[0] <= 1;       //访存阶段
                        state[1] <= 0;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                        //运算器
                        TAluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                        TALUctr  <=`ADD;     //运算模式
                        TExtOp   <=1;        //是否为符号扩展
                        //数据储存器
                        TINMode  <=`Byte;    //数据储存器输入模式
                        TMemWr   <=1;        //数据储存器写使能
                    end

                    `sh : begin
                        //判断阶段
                        state[0] <= 1;       //访存阶段
                        state[1] <= 0;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                        //运算器
                        TAluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                        TALUctr  <=`ADD;     //运算模式
                        TExtOp   <=1;        //是否为符号扩展
                        //数据储存器
                        TINMode  <=`HalfWord;//数据储存器输入模式
                        TMemWr   <=1;        //数据储存器写使能
                    end

                    `sw : begin
                        //判断阶段
                        state[0] <= 1;       //访存阶段
                        state[1] <= 0;       //回写阶段
                        //PC
                        nPCsel  <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                        //运算器
                        TAluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                        TALUctr  <=`ADD;     //运算模式
                        TExtOp   <=1;        //是否为符号扩展
                        //数据储存器
                        TINMode  <=`Word;    //数据储存器输入模式
                        TMemWr   <=1;        //数据储存器写使能
                    end

                    `beq : begin
                        //判断阶段
                        state[0] <= 0;       //访存阶段
                        state[1] <= 0;       //回写阶段
                        //PC
                        nPCsel  <=`Beq;     //跳转指令的判断：Plus，Beq，J，Jr
                        //运算器
                        TAluSrc  <=0;        //ALU运算数       0：rt           1：extimm
                        TALUctr  <=`SUB;   //运算模式
                    end

                    `lui : begin
                        //判断阶段
                        state[0] <= 0;       //访存阶段
                        state[1] <= 1;       //回写阶段
                        //PC
                        nPCsel   <=`Plus;    //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`Rt;      //寄存器组写入地址：Rt，Rd，LAST
                        //运算器
                        TAluSrc  <=1;        //ALU运算数       0：rt           1：extimm
                        TALUctr  <=`LUI;     //运算模式
                        TExtOp   <=1;        //是否为符号扩展
                    end

                    `j : begin
                        //判断阶段
                        state[0] <= 0;       //访存阶段
                        state[1] <= 0;       //回写阶段
                        //PC
                        nPCsel  <=`J;       //跳转指令的判断：Plus，Beq，J，Jr
                    end

                    `jal : begin
                        //判断阶段
                        state[0] <= 0;       //访存阶段
                        state[1] <= 1;       //回写阶段
                        //PC
                        nPCsel  <=`J;       //跳转指令的判断：Plus，Beq，J，Jr
                        //寄存器组
                        TRegDst  <=`LAST;    //寄存器组写入地址：Rt，Rd，LAST
                        TMemtoReg<=`JalAddr; //寄存器组写入数据：ALU，DMout，JalAddr
                    end
                endcase
            end
        end
    end

    //执行阶段前
    always @ ( posedge clk ) begin
        if ( nextstate == 2 && rst == 0) begin
            //设置当前阶段
            currentstate <= 2;
            //取指单元
            pcWr    <= 0;        //pc写使能
            //寄存器组
            RegWr   <= 0;        //寄存器组写使能
        end
    end

    //执行阶段
    always @ ( * ) begin
        if ( currentstate == 2 && rst == 0) begin
            //执行阶段运算器进行运算
            AluSrc  <= TAluSrc;        //ALU运算数       0：rt           1：extimm
            ALUctr  <= TALUctr;        //运算模式
            ExtOp   <= TExtOp;        //是否为符号扩展
            //数据储存器
            MemWr   <= 0;        //数据储存器写使能
            //根据state判断下一阶段
            if ( state[0] == 1 ) begin
                nextstate <= 3;
            end else if ( state[1] == 1) begin
                nextstate <= 4;
            end else begin
                nextstate <= 0;
            end
        end
    end

    //访存阶段前
    always @ ( posedge clk ) begin
        if ( nextstate == 3 && state[0] == 1 && rst == 0) begin
            //设置当前阶段
            currentstate <= 3;
            //取指单元
            pcWr    <= 0;        //pc写使能
            //寄存器组
            RegWr   <= 0;        //寄存器组写使能
        end
    end

    //访存阶段
    always @ ( * ) begin
        if ( currentstate == 3 && state[0] == 1 && rst == 0) begin
            //数据储存器
            MemWr   <= TMemWr;        //数据储存器写使能
            INMode  <= TINMode;      //数据储存器输入模式
            OUTMode  <= TOUTMode;     //数据储存器输出模式
            //根据state判断下一阶段
            if ( state[1] == 1 ) begin
                nextstate <= 4;
            end else begin
                nextstate <= 0;
            end
        end
    end

    //回写阶段前
    always @ ( posedge clk ) begin
        if ( nextstate == 4 && state[1] == 1 && rst == 0) begin
            //设置当前阶段
            currentstate <= 4;
            //取指单元
            pcWr    <= 0;        //pc写使能
        end
    end

    //回写阶段
    always @ ( * ) begin
        if ( currentstate == 4 && state[1] == 1 && rst == 0) begin
            //寄存器组
            RegWr   <= 1;            //寄存器组写使能
            RegDst  <= TRegDst;      //寄存器组写入地址：Rt，Rd，LAST
            MemtoReg<= TMemtoReg;     //寄存器组写入数据：ALU，DMout，JalAddr
            //数据储存器
            MemWr   <= 0;           //数据储存器写使能
            //设置下一阶段
            nextstate <= 0;

        end
    end

 endmodule
```

### 数据储存器
```
module dm(
    //输入
    input wire clk,
    input wire[31:0] addr,                  //数据地址
    input wire[31:0] din,                   //输入数据
    input wire[1:0] INMode,                 //数据储存器输入模式
    input wire[1:0] OUTMode,                //数据储存器输出模式
    input wire MemWr,                       //数据储存器写使能

    //输出
    output reg[31:0] dout                  //输出数据
    );

    reg[31:0] DataMem[1023:0];            //二维向量定义数据储存器，大小为4kb
    reg[31:0] currentmem;                 //读取中间变量

    integer i;
    initial begin
        for ( i = 0 ; i < 1024 ; i = i + 1 ) begin
            DataMem[i] = 32'h00000000;
        end
    end

    always @ (*) begin
        currentmem <= DataMem[addr[11:2]];        //从内存中读取内容
        if( OUTMode == 2'b11 ) begin              //32位输出
            dout <= currentmem;
        end else if( OUTMode == 2'b01 ) begin     //16位输出
            if(addr[1] == 0) begin
                dout <= {{16{currentmem[15]}},currentmem[15:0]};
            end else begin
                dout <= {{16{currentmem[31]}},currentmem[31:16]};
            end
        end else if ( OUTMode == 2'b00 ) begin    //8位输出
            if( addr[1:0] == 2'b00 ) begin
                dout <= {{24{currentmem[7]}},currentmem[7:0]};
            end else if ( addr[1:0] == 2'b01 ) begin
                dout <= {{24{currentmem[15]}},currentmem[15:8]};
            end else if ( addr[1:0] == 2'b10 ) begin
                dout <= {{24{currentmem[23]}},currentmem[23:16]};
            end else if ( addr[1:0] == 2'b11 ) begin
                dout <= {{24{currentmem[31]}},currentmem[31:24]};
            end
        end
    end

    always @ (posedge clk) begin
        if ( MemWr ) begin                              //写使能有效时
            currentmem <= DataMem[addr[11:2]];          //从内存中读取内容
            if ( INMode == 2'b11 ) begin                //32位输入
                DataMem[addr[11:2]] <= din;
            end else if ( INMode == 2'b01 ) begin       //16位输入
                if (addr[1] == 0)begin
                    DataMem[addr[11:2]] <= {currentmem[31:16],din[15:0]};
                end else begin
                    DataMem[addr[11:2]] <= {din[15:0],currentmem[15:0]};
                end
            end else if ( INMode == 2'b00 ) begin       //8位输入
                if (addr[1:0] == 2'b00 ) begin
                    DataMem[addr[11:2]] <= {currentmem[31:8],din[7:0]};
                end else if ( addr[1:0] == 2'b01 ) begin
                    DataMem[addr[11:2]] <= {currentmem[31:16],din[7:0],currentmem[7:0]};
                end else if ( addr[1:0] == 2'b10 ) begin
                    DataMem[addr[11:2]] <= {currentmem[31:24],din[7:0],currentmem[15:0]};
                end else if ( addr[1:0] == 2'b11 ) begin
                    DataMem[addr[11:2]] <= {din[7:0],currentmem[23:0]};
                end
            end
        end
    end

endmodule
```
