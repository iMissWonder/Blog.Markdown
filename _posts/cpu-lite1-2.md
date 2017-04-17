---
title: 如何用VerilogHDL编写一个单周期MIPS32CPU（二）
date: 2016-05-02 13:34:20
tags: 计算机原理与汇编
---
第二篇复习Verilog HDL的行为语句（非常重要）。
## Verilog HDL行为语句
### 过程语句
Verilog定义的模块一般包括有过程语句，过程语句有两种：initial、always。其中initial常用于仿真中的初始化，其中的语句只执行一次，而always中语句则是不断重复执行的。此外，always过程语句是可综合的，initial过程语句是不可综合的。
> 综合（Synthesis）是将较高级抽象层次的设计描述自动转化为较低层次描述的过程。有以下几种综合形式。
将算法表示、行为描述转换到寄存器传输级（RTL），即从行为描述到结构描述。
将RTL级描述转换到逻辑门级，称为逻辑综合。
将逻辑门表示转换到PLD器件的配置网表表示，有了配置网表就可完成基于PLD器件的系统实现。
综合器就是能够自动实现上述转换的软件工具，其能够将原理图或HDL语言表达、描述的电路编译成由与或阵列、RAM、触发器、寄存器等逻辑单元组成的电路网表。
<!-- more -->

#### **always过程语句**

```格式
always @(<敏感信号表达式>)
begin

    // 语句序列

end
```

always过程语句通常是带有触发条件的，触发条件写在敏感信号表达式中，敏感信号表达式又称为事件表达式或敏感信号列表，当该表达式中变量的值改变时，就会引发其中语句序列的执行。因此，敏感信号表达式中应列出影响块内取值的所有信号。

##### 敏感信号表达式的格式
如果有两个或两个以上的敏感信号时，它们之间使用“or”连接，此处还是以32位加法器为例，之前使用assign直接赋值的，其实也可以使用always过程语句实现，如下。只要被加数in1、加数in2中的任何一个改变，都会触发always过程语句，在其中进行加法运算。这里有两个敏感信号in1、in2，使用“or”连接。

```
module add32(input  wire[31:0]  in1,   
             input  wire[31:0]  in2,  
        output reg[31:0]  out);  

always @ (in1 or in2)             //使用always过程语句实现加法  
begin  
      out = in1 + in2;  
end  

endmodule  
```

敏感信号列表中的多个信号也可以使用逗号隔开，上面的32位加法器可以修改为如下形式。

```
module add32(input  wire[31:0]  in1,   
             input  wire[31:0]  in2,  
        output reg[31:0]  out);  

always @ (in1, in2)             //多个敏感信号使用逗号分隔  
begin  
      out = in1 + in2;  
end  

endmodule  
```

敏感信号列表也可以使用通配符*，表示在该过程语句中的所有输入信号变量，上面的32位加法器可以修改为如下形式。

```
module add32(input  wire[31:0]  in1,   
             input  wire[31:0]  in2,  
        output reg[31:0]  out);  

always @ (*)                   //使用通配符表示过程语句中的所有输入信号变量  
begin  
      out = in1 + in2;  
end  

endmodule  
```

##### 组合电路与时序电路
敏感信号可以分为两种类型：一种为电平敏感型，一种为边沿敏感型。前一种一般对应组合电路，如上面给出的加法器的例子，后一种一般对应时序电路。对于时序电路，敏感信号通常是时钟信号，Verilog HDL提供了 **posedge**、negedge 两个关键字来描述时钟信号。**posedge** 表示以时钟信号的上升沿作为触发条件，negedge表示以时钟信号的下降沿作为触发条件。还是以32位加法器为例，可以为其添加一个时钟同步信号，如下。
```
module add32(input wire        clk,     //增加了一个时钟输入信号  
             input wire[31:0]  in1,   
             input wire[31:0]  in2,  
output reg[31:0] out);  

always @ (posedge clk)              //在时钟信号的上升沿会触发always中的语句  
begin  
       out = in1 + in2;  
end  

endmodule  
```
在时钟信号的上升沿，才会进行加法运算，这一点与前面的加法器不同，也就是当被加数in1、加数in2变化时，并不会立即改变输出out，而是要等待时钟信号的上升沿。

#### initial过程语句

```格式
inital
begin

    //语句序列

end
```
initial过程语句不带触发条件，并且其中的语句序列只执行一次。initial过程语句通常用于仿真模块中对激励向量的描述，或用于给寄存器赋初值，它是面向模拟仿真的过程语句，通常不能被综合。如下是initial过程语句的一个例子，用于给存储器mem赋初值。

```
initial  
  begin  
    for(addr = 0; addr < size; addr = addr+1)  // for是一种循环语句，下文会介绍  
     mem[addr] = 0;  
  end  
```
### 赋值语句
赋值语句有两种： **持续赋值语句、过程赋值语句**。

#### 持续赋值语句
assign为持续赋值语句，主要用于对wire型变量的赋值。如上文中加法器的例子。

#### 过程赋值语句
在always、initial过程中的赋值语句称为过程赋值语句，多用于对reg型变量进行赋值，分为非阻塞赋值和阻塞赋值两种方式。

##### 非阻塞赋值（Non-Blocking）
赋值符号为“<=”
```
b <= a  
```
非阻塞赋值在整个过程语句结束时才会完成赋值操作，即b的值并不是立刻改变的。

##### 阻塞赋值（Blocking）
赋值符号为“=”
```
b = a
```
阻塞赋值在该语句结束时就立即完成赋值操作，即b的值在这条语句结束后立刻改变。如果在一个块语句中，有多条阻塞赋值语句，那么在前面的赋值语句没有完成之前，后面的语句就不能被执行，仿佛被阻塞了一样，因此称为阻塞赋值方式。

在always过程块中，阻塞赋值可以理解为赋值语句是顺序执行的，而非阻塞赋值可以理解为赋值语句是并发执行的。如图2-12所示。在一个过程块中，阻塞式赋值与非阻塞式赋值只能使用其中一种。

![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/2/1.png)

### 条件语句
条件语句有if-else、case两种，应放在always块内。分别介绍如下。

#### if-else语句
 if-else语句的格式有如下三种。

```
 (1) if(表达式)                语句序列1;              // 非完整性IF语句  

(2) if(表达式)                语句序列1;              // 二重选择的IF语句  
else                         语句序列2;  

(3) if(表达式1)               语句序列1;             // 多重选择的IF语句  
    else if(表达式2)           语句序列2;  
    else if(表达式3)          语句序列3;  
      ......  
else if(表达式n)              语句序列n;  
else                         语句序列n+1;  
```

上述格式中的“表达式”一般为逻辑表达式或关系表达式，也可能是1位的变量。系统对表达式的值进行判断，如果为0、X、Z，则按“假”处理，如果为1，则按“真”处理。语句序列可以是单句，也可以是多句，多句时需使用begin-end块语句括起来。

还是以32位加法器为例，为其添加一个复位信号rst，如果rst为高电平，那么复位信号有效，输出out为0，反之，复位信号无效，输出out为两个输入信号之和。
```
module add32(input wire        rst,    // 增加了一个复位信号  
             input wire[31:0]  in1,   
             input wire[31:0]  in2,  
        output reg[31:0] out);  

always @ (*)  
begin  
      if(rst == 1'b1)  
        out <= 32'h0;       // 如果复位信号有效，那么输出out为0  
else  
        out <= in1 + in2;   // 反之，输出out为两个输入信号之和  
end  

endmodule  
```

#### case语句
相对于if-else语句只有两个分支而言，case语句是一种多分支语句，所以case语句多用于多条件译码电路，如译码器、数据选择器、状态机及微处理器的指令译码等。

```
case(敏感表达式)  
值1: 语句序列1;  
值2: 语句序列2;  
......  
值n: 语句序列n;  
default: 语句序列n+1;  
endcase  
```
当敏感表达式的值等于“值1”时，执行语句序列1；当等于“值2”时，执行语句序列2；依次类推。如果敏感表达式的值与上面列出的值都不符，那么执行default后面的语句序列。如下代码是一个简单的运算单元，可执行加法或减法运算，如果输入变量type的值为1，那么执行加法运算，如果type的值为0，那么执行减法运算。

```
module add_sub32(input  wire        type,   // type决定运算类型  
                 input  wire[31:0]  in1,   
                 input  wire[31:0]  in2,  
            output reg[31:0]  out);  

always @ (*)  
begin  
       case(type)  
      1'b1 : out <= in1 + in2;  // type为1，执行加法运算  
      1'b0 : out <= in1 - in2;  // type为0，执行减法运算  
      default : out <= 32'h0;  
endcase  
    end  

endmodule  
```

case语句中，敏感表达式与值1-n之间的比较是一种全等比较，必须保证两者的对应位全等。casez、casex语句是case语句的两种扩展。

在casez语句中，如果比较的双方某些位的值为高阻Z，那么对这些位的比较结果就不予考虑，只需考虑其它位的比较结果。
在casex语句中，如果比较的双方某些位的值为Z或X，那么对这些位的比较结果就不予考虑，只需考虑其它位的比较结果。
此外，还有一种表示X或Z的方式，即用表示无关值的符号“?”来表示。

```
case(a)  
2'b1x : out <= 1;   //只有a等于2'b1x时，out才等于1  

casez(a)  
2'b1x : out <= 1;   //a等于2'b1x、2'b1z时，out等于1  

casex(a)  
2'b1x : out <= 1;   //a等于2'b10、2'b11、2'b1x、2'b1z时，out等于1  

case(a)  
2'b1? : out <= 1;   //a等于2'b10、2'b11、2'b1x、2'b1z时，out等于1  
```

### 循环语句
Verilog HDL中存在四种类型的循环语句：for、forever、repeat、while，用来控制语句的执行次数，分别介绍如下。

#### for语句
for语句与C语言是相似的。
一个使用for语句实现7人表决器的例子如下。通过for循环统计赞成的人数，若超过4人（含4人）赞成则通过，其中vote[7:1]表示7个人的投票情况，vote[i]为1，表示第i个人投的是赞成票，反之是反对票，pass是输出，超过4个人赞成，pass为1，反之为0。
```
module vote7(vote, pass);  

input  wire[7:1] vote;  
output reg       pass;  
reg[2:0]         sum;  
integer          i;  

always @ (vote)  
begin  
     sum = 0;  
     for(i=1; i<7; i=i+1)  
if(vote[i])  
       sum = sum+1;      //如果vote[i]为1，那么sum加1，注意此处使用阻塞赋值  
if(sum[2] == 1'b1)   //如果sum大于等于4，那么输出pass为1  
       pass = 1;  
else  
       pass = 0;  
end  

endmodule  
```

#### forever语句
forever语句的格式如下。
```
forever begin  

语句序列  

end  
```
forever循环语句连续不断的执行其中的语句序列，常用来产生周期性的波形。在2.8节编写仿真用的Test Bench文件时，会给出forever语句的例子。

#### repeat语句
repeat语句的格式如下。
```
repeat(循环次数表达式) begin  

语句序列  

end
```

#### while语句
while语句的格式如下。
```
while(循环执行条件表达式) begin  

语句序列  

end
```
while语句在执行时，首先判断循环执行条件表达式是否为真，若为真，则执行其中的语句序列，然后再次判断循环执行条件表达式是否为真，若为真，则再次执行其中的语句序列，如此反复，直到循环执行条件表达式不为真。

### 编译指示语句（非常重要）
Verilog HDL和C语言一样提供了编译指示功能，允许在程序中使用编译指示（Compiler Directives）语句，在编译时，通常先对这些指示语句进行预处理，然后再将预处理的 结果和源程序一起进行编译。

编译指示语句以`开始，以区别其它语句。常用的编译指示语句有：`define、`include、`ifdef、`else、`endif，分别介绍如下。

#### 宏替换

`define可以用一个简单的名字或有意义的标识（也称为宏名）代替一个复杂的名字或变量，其格式如下。
```
`define 宏名 变量或名字  
```

例如：一般在时序电路中会有一个复位信号，当该复位信号为高电平时表示复位信号有效，当该复位信号为低电平时，表示复位信号无效。分别执行不同的代码，如下。

```
always @ (clk)  
begin  
     if(rst == 1'b1)  
       //复位有效  
     else  
       //复位无效  
end  
```
一种更为友好的书写方式，是使用宏定义，如下。
```友好的书写方式
 // 定义宏RstEnable表示复位信号有效，这个名字对读者而言更有意义  
`define RstEnable 1'b1  

 ......  

always @ (clk)  
begin  
     if(rst == `RstEnable)  // 在编译的时候会自动将`RstEnable替换成1'b1  
       //复位有效  
     else  
       //复位无效  
end  
```

#### include语句
`include是文件包含语句，它可将一个文件全部包含到另一个文件中，使用格式如下。
```
`include "文件名"  
```
在OpenMIPS处理器的实现过程中，我们定义了很多宏，这些宏都集中在文件defines.v中，如果某一程序需要使用其中的宏定义，就可以在程序文件的开始使用`include语句将defines.v文件包含进来即可，如下。
```
`include "defines.v"  
```
#### 条件编译语句ifdef、else、endif

条件编译语句`ifdef、`else、`endif可以指定仅对程序中的部分内容进行编译，有两种使用形式。
第一种使用形式如下。当指定的宏在程序中已定义，那么其中的语句序列参与源文件的编译，否则，其中的语句序列不参与源文件的编译。
```
`ifdef 宏名  

   语句序列  

 `endif  
```

 第二种使用形式如下。当指定的宏在程序中已定义，那么其中的语句序列1参与源文件的编译，否则，其中的语句序列2参与源文件的编译。
```
`ifdef 宏名  

   语句序列1  

`else  

   语句序列2  

`endif  
```
### 行为语句的可综合性
多种行为语句中，有些语句是不可综合的，也就是说综合器无法将这些语句转变为对应的硬件电路。
![statement](http://7xt50p.com1.z0.glb.clouddn.com/post/cpu/2/2.png)
