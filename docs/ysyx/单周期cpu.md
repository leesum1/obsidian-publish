![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220705183402.png)
单周期
+ pc
地址自增、跳转、暂停、时序逻辑电路
[Index of /~cs61c/fa14/lec (berkeley.edu)](https://inst.eecs.berkeley.edu/~cs61c/fa14/lec/)
[Ibex: An embedded 32 bit RISC-V CPU core — Ibex Documentation 0.1.dev50+g5c49fad.d20220629 documentation (ibex-core.readthedocs.io)](https://ibex-core.readthedocs.io/en/latest/index.html)
[riscv/riscv_decoder.v at master · ultraembedded/riscv (github.com)](https://github.com/ultraembedded/riscv/blob/master/core/riscv/riscv_decoder.v)
## 测试
+ 寄存器读写
+ 译码指令是否正确
+ ALU功能的测试



## 译码模块
> 译码模块说简单也简单说复杂也复杂，这几天我查询了很多资料，参考了许多开源CPU译码模块的设计，想找出比较好译码的方式

### 行为建模方式
>这种译码方式最为直接明了，直接采用 case 语句，对不同的 opcode 进行分支处理，在每一条分支后面进行 alu 运算单元的信号处理等等。
>主要参考代码：[ibex/ibex_decoder.sv at master · lowRISC/ibex (github.com)](https://github.com/lowRISC/ibex/blob/master/rtl/ibex_decoder.sv)
>[tinyriscv: 一个从零开始写的极简、非常易懂的RISC-V处理器核。 (gitee.com)](https://gitee.com/liangkangnan/tinyriscv)

![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220707134627.png)
可以从上图看到，这种译码方式编写起来很简单，只要不断的往下加分支就行了，但在代码可读性上就差很多，需要深入每一条指令才能理解代码。面积优化也不够好。
### 模板匹配方式
>这种译码方式比行为建模要优雅，可读性也比较好。每条指令的 opcode、fun3、fun7 都是固定的，可以为每条指令创建一个 mask，用 inst&mast 就可以判断当前指令属于哪一条指令。与 pa 中 nemu 解析指令的方式一样。
>主要参考代码：[riscv/riscv_decoder.v at master · ultraembedded/riscv (github.com)](https://github.com/ultraembedded/riscv/blob/master/core/riscv/riscv_decoder.v)

`nemu` 中指令解析方式如下
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220707135451.png)

参考的 `riscv` 设计中的 `decoder` 模块如下
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220707135814.png)
可以看到 `exec_o` 控制信号由指令 `ANDI`、`ADDI` 等生成，采用了一个或逻辑。简洁明了。
```verilog
// andi
`define INST_ANDI 32'h7013      //指令
`define INST_ANDI_MASK 32'h707f //指令mask
```
这样的设计方式，添加一条指令十分的方便，只需要设计对应的 `mask` 就行了，和 `nemu` 是一模一样的。但缺点也十分明显，这样子是否资源消耗太大了呢？

### 逐层分析方式
>说实话这个名称是我自己瞎编的，因为我不太清楚用什么去描述。早些时候在群里看见有人在讨论 `蜂鸟e203` 的设计方式，最近去了解了一下。最开始在 `github` 上看见它的 `decoder` 的设计方式时，确实被惊艳到了，基本是按照手册设计出来的，很细节，和其他方式都不一样。
>主要参考代码：[e203_hbirdv2/e203_exu_decode.v at master · riscv-mcu/e203_hbirdv2 (github.com)](https://github.com/riscv-mcu/e203_hbirdv2/blob/master/rtl/e203/core/e203_exu_decode.v)

花了一天读完了的该 `riscv` 设计的配套书籍 `手把手教你设计CPU——RISC-V处理器篇` 想深入了解一下 `decoder` 的设方式，发现介绍的很少。没办法，只能去研究源码。
#### 第一步：分解 opcode
```verilog
  wire [6:0]  opcode = rv32_instr[6:0];

  wire opcode_1_0_00  = (opcode[1:0] == 2'b00);
  wire opcode_1_0_01  = (opcode[1:0] == 2'b01);
  wire opcode_1_0_10  = (opcode[1:0] == 2'b10);
  wire opcode_1_0_11  = (opcode[1:0] == 2'b11);
  // We generate the signals and reused them as much as possible to save gatecounts
  wire opcode_4_2_000 = (opcode[4:2] == 3'b000);
  wire opcode_4_2_001 = (opcode[4:2] == 3'b001);
  wire opcode_4_2_010 = (opcode[4:2] == 3'b010);
  wire opcode_4_2_011 = (opcode[4:2] == 3'b011);
  wire opcode_4_2_100 = (opcode[4:2] == 3'b100);
  wire opcode_4_2_101 = (opcode[4:2] == 3'b101);
  wire opcode_4_2_110 = (opcode[4:2] == 3'b110);
  wire opcode_4_2_111 = (opcode[4:2] == 3'b111);
  wire opcode_6_5_00  = (opcode[6:5] == 2'b00);
  wire opcode_6_5_01  = (opcode[6:5] == 2'b01);
  wire opcode_6_5_10  = (opcode[6:5] == 2'b10);
  wire opcode_6_5_11  = (opcode[6:5] == 2'b11);
```
可以看到，他将 `opcdoe` 各个位的情况直接枚举出来了，用 `wire` 信号线表示是否出现了对应信号。从 `riscv` 手册得知，`opcode` 的低两位一定是 `11` ，`opcode[6:5]` 与 `opcode[4:2]` 的不同排列组合表示不同的指令类型。如下图：
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220707142054.png)
在代码中分别指令类型具体操作如下
```verilog
  wire rv32_nmadd    = opcode_6_5_10 & opcode_4_2_011 & opcode_1_0_11; 
  wire rv32_jal      = opcode_6_5_11 & opcode_4_2_011 & opcode_1_0_11; 

  wire rv32_op_imm   = opcode_6_5_00 & opcode_4_2_100 & opcode_1_0_11; 
  wire rv32_op       = opcode_6_5_01 & opcode_4_2_100 & opcode_1_0_11; 
  wire rv32_op_fp    = opcode_6_5_10 & opcode_4_2_100 & opcode_1_0_11; 
  wire rv32_system   = opcode_6_5_11 & opcode_4_2_100 & opcode_1_0_11; 

  wire rv32_auipc    = opcode_6_5_00 & opcode_4_2_101 & opcode_1_0_11; 
  wire rv32_lui      = opcode_6_5_01 & opcode_4_2_101 & opcode_1_0_11; 
  wire rv32_resved1  = opcode_6_5_10 & opcode_4_2_101 & opcode_1_0_11; 
  wire rv32_resved2  = opcode_6_5_11 & opcode_4_2_101 & opcode_1_0_11; 

  wire rv32_op_imm_32= opcode_6_5_00 & opcode_4_2_110 & opcode_1_0_11; 
  wire rv32_op_32    = opcode_6_5_01 & opcode_4_2_110 & opcode_1_0_11; 
  wire rv32_custom2  = opcode_6_5_10 & opcode_4_2_110 & opcode_1_0_11; 
```
基本就是按照上面的表格进行操作的，不断的重复利用刚刚的信号线，可以节省资源。如果在每一步都行比较操作 ，如下：
```verilog
//先比较再使用
wire rv32_op_imm   = opcode_6_5_00 & opcode_4_2_100 & opcode_1_0_11;
//使用时再比较
wire rv32_op_imm   = opcode[6:5]==2'b00 & opcode[4:2]==3'b100 & opcode[1:0]==2'b00;
```
按照道理来说，确实第一种方式可以节省一些比较器，但在综合时，两者是否相同就不清楚了。
#### 第二部：确定每条指令
`func7` `func3` 也和 `opcode` 同样被分解了，可以很方便的确认是哪一条指令。
```verilog
  wire rv32_add      = rv32_op     & rv32_func3_000 & rv32_func7_0000000;
  wire rv32_sub      = rv32_op     & rv32_func3_000 & rv32_func7_0100000;
  wire rv32_sll      = rv32_op     & rv32_func3_001 & rv32_func7_0000000;
  wire rv32_slt      = rv32_op     & rv32_func3_010 & rv32_func7_0000000;
  wire rv32_sltu     = rv32_op     & rv32_func3_011 & rv32_func7_0000000;
  wire rv32_xor      = rv32_op     & rv32_func3_100 & rv32_func7_0000000;
  wire rv32_srl      = rv32_op     & rv32_func3_101 & rv32_func7_0000000;
  wire rv32_sra      = rv32_op     & rv32_func3_101 & rv32_func7_0100000;
  wire rv32_or       = rv32_op     & rv32_func3_110 & rv32_func7_0000000;
  wire rv32_and      = rv32_op     & rv32_func3_111 & rv32_func7_0000000;
```
如上所示，每条指令都有一条单独的信号线 `wire` ，为 `1` 时，代表为当前指令。也都是按照手册一步步分析的。
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220707144203.png)
#### 第三步：生成控制信号
在第二步中确定了指令，接下来就是根据每条指令生成相对于的控制信号了，有些指令的控制信号可以合并。源码中的控制信号比较复杂，不太容易看懂。
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220707144522.png)
不过没关系，我们不需要深入理解，了解一下译码思路就行了，控制信号可以自己自定义。
#### 优点
初看 `decoder` 的代码其实会一脸茫然，不知道在干什么，没有上面两种好懂。但其实对着手册去理解，会发现这个译码思路基本是按照 `riscv手册` 来的。明白了流程以后十分清晰明了，添加指令也很简单。其实这种译码方式很像第一种行为建模方式，也是看 `opcode` `func3` `func7` 但更加的优雅简洁，代码易懂。只使用到了 `assign` 语句，利于了解综合后的电路结构。

## ALU设计
>[Index of /~teodorescu.1/download/teaching/cse675.au08 (ohio-state.edu)](https://web.cse.ohio-state.edu/~teodorescu.1/download/teaching/cse675.au08/)

#### 加法器与减法器
> verilog 中其实以及内置了 + - 等基础运算方式。

1. 内置的 + -  * / 运算
这个我查了一下资料，如果是部署在 `fpga` 上面，`fpga` 一般都会有 `加法器` `乘法器` 等资源，会直接使用这上面的资源。但如果没有呢？会被综合成上面样子呢？这个我找了找资料最后也不太清楚。
2. 串行进位加法器
这个学过数电的都做过实验
3. 超前进位加法
这个资料也很多，不过位数过高后，耗费的逻辑资源是很多的。
4. 减法器
在硬件层面，减法都是用加法来实现的，不多介绍

##### 本次目标
1. 只用一个加法器实现无符号数和有符号数的加减运算
2. 将比较运算用减法加标志位实现
3. 加法器用内置的 `+` ，乘法器用内置的 `*` ，除法器用内置的 `/` ，后续有时间可以在来研究和替换。

### 标志位生成
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220707222951.png)
### 条件转移指令
[条件转移指令 - 快懂百科 (baike.com)](https://www.baike.com/wikiid/6848039267466467421?from=wiki_content&prd=innerlink&view_id=2fvcjoeriysk00)
[x86 Control Flow (usc.edu)](https://ee.usc.edu/~redekopp/cs356/slides/CS356Unit5_x86_Control)
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220707223851.png)
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220707224732.png)

+ 加法器
+ 减法器，通过加法实现
+ 移位器
+ 比较运算，通过加法和比较实现
+ 异或
+ 与
+ 或
### 逻辑移位与算数移位
#### 位颠倒设计
+ 采用 `for` 循环参数化方式
```verilog
module test_invert (
    input  [16-1:0] in,
    output [16-1:0] out
);
  invertTem #(
      .DATA_LEN(16)
  ) u_invertTem (
      .in (in),
      .out(out)
  );
endmodule

module invertTem #(
    DATA_LEN = 1
) (
    input [DATA_LEN-1:0] in,
    input [DATA_LEN-1:0] out
);
  integer i;
  always @(*) begin
    for (i = 0; i < DATA_LEN; i = i + 1) begin
      out[i] = in[DATA_LEN-1-i];
    end
  end

endmodule
```
综合后电路图如下
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220708170711.png)

+ 采用拼接方式
```verilog
module test_invert (

    input  [16-1:0] in,

    output [16-1:0] out

);

  assign out = {
    in[00],
    in[01],
    in[02],
    in[03],
    in[04],
    in[05],
    in[06],
    in[07],
    in[08],
    in[09],
    in[10],
    in[11],
    in[12],
    in[13],
    in[14],
    in[15]
  };
endmodule
```
综合后电路图：
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220708170129.png)
总结
1. 第一种方式省资源，但操作复杂
2. 第二章方式多了一个 `i` ，但利用参数化操作，很方便。

#### 移位器设计
> 最近在看ALU的底层设计方式，详细看了龙芯出品的《计算机系统结构基础》，在这里面说在设计移位器的时候，可以用左移代替右移，以节省硬件资源。

![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220708215130.png)
看见上面那段话，我仔细思考了一下。将逻辑右移转换为逻辑左移需要两次位颠倒操作，这个可以解决。那么如何将算数右移转换为左移呢？这个我自己也想了一段时间，能力有限。就去研究其他开源处理器如何实现ALU操作的。
##### 简单粗暴的case语句
> 直接为每一种移位操作设计一个操作，`<<` `>>` ，实现移位操作

这个方法显然简单粗暴，可靠性高但不是我想要的。

##### 全部转换为左移操作
> 我在研究蜂鸟e203 alu 设计的时候，发现了这种巧妙的移位器设计方法。

`蜂鸟e203` 上面很多模块的设计方法都很厉害，它的 `ALU` 实现也和我平常学的不同，它设计了一个数据通路，所有的数据计算都是数据通路负责，`ALU` 只是向数据通路请求数据。这也增加了源码的理解难度，不过我不需要关系这么多，我只需要找他它具体计算的地方就行了。
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220708220339.png)
经过一段时间的理解，找了的源码中关于移位器的设计
```verilog
  // Impelment the Left-Shifter
  //
  // The Left-Shifter will be used to handle the shift op
  wire [`E203_XLEN-1:0] shifter_in1; //要移位的数
  wire [5-1:0] shifter_in2;          //移动次数
  wire [`E203_XLEN-1:0] shifter_res;

  // 是否移位
  wire op_shift = op_sra | op_sll | op_srl; 
  
     // Make sure to use logic-gating to gateoff the 
  
  /* 将右移动转换为左移 */
  assign shifter_in1 = {`E203_XLEN{op_shift}} &
          //   In order to save area and just use one left-shifter, we
          //   convert the right-shift op into left-shift operation
           (
               (op_sra | op_srl) ? 
                 {
    shifter_op1[00],shifter_op1[01],shifter_op1[02],shifter_op1[03],
    shifter_op1[04],shifter_op1[05],shifter_op1[06],shifter_op1[07],
    shifter_op1[08],shifter_op1[09],shifter_op1[10],shifter_op1[11],
    shifter_op1[12],shifter_op1[13],shifter_op1[14],shifter_op1[15],
    shifter_op1[16],shifter_op1[17],shifter_op1[18],shifter_op1[19],
    shifter_op1[20],shifter_op1[21],shifter_op1[22],shifter_op1[23],
    shifter_op1[24],shifter_op1[25],shifter_op1[26],shifter_op1[27],
    shifter_op1[28],shifter_op1[29],shifter_op1[30],shifter_op1[31]
                 } : shifter_op1
           );
  assign shifter_in2 = {5{op_shift}} & shifter_op2[4:0];

  /* 实际左移操作 */
  assign shifter_res = (shifter_in1 << shifter_in2);

  /* 逻辑移位结果 */
  wire [`E203_XLEN-1:0] sll_res = shifter_res;
  wire [`E203_XLEN-1:0] srl_res =  
                 {
    shifter_res[00],shifter_res[01],shifter_res[02],shifter_res[03],
    shifter_res[04],shifter_res[05],shifter_res[06],shifter_res[07],
    shifter_res[08],shifter_res[09],shifter_res[10],shifter_res[11],
    shifter_res[12],shifter_res[13],shifter_res[14],shifter_res[15],
    shifter_res[16],shifter_res[17],shifter_res[18],shifter_res[19],
    shifter_res[20],shifter_res[21],shifter_res[22],shifter_res[23],
    shifter_res[24],shifter_res[25],shifter_res[26],shifter_res[27],
    shifter_res[28],shifter_res[29],shifter_res[30],shifter_res[31]
                 };
  
  /* 算数右移结果 */
  wire [`E203_XLEN-1:0] eff_mask = (~(`E203_XLEN'b0)) >> shifter_in2;
  wire [`E203_XLEN-1:0] sra_res =
               (srl_res & eff_mask) | ({32{shifter_op1[31]}} & (~eff_mask));
```
+ 首先最基础的信号
```verilog
  wire [`E203_XLEN-1:0] shifter_in1; //要移位的操作数
  wire [5-1:0] shifter_in2;          //移动次数
  wire [`E203_XLEN-1:0] shifter_res; //移位后的结果
  // 是否进行移位操作
  wire op_shift = op_sra | op_sll | op_srl;
```
`op_sra` `op_sll` `op_srl` 分别对应着 `算数右移` `逻辑左移` `逻辑右移` ，因为符号位在最右边，所以没有算数左移。这三个信号由外部给出。
+ 接下来是将右移操作转换为左移。
```verilog
  /* 将右移动转换为左移 */
  assign shifter_in1 = {`E203_XLEN{op_shift}} &
          //   In order to save area and just use one left-shifter, we
          //   convert the right-shift op into left-shift operation
           (
               (op_sra | op_srl) ? 
                 {
    shifter_op1[00],shifter_op1[01],shifter_op1[02],shifter_op1[03],
    shifter_op1[04],shifter_op1[05],shifter_op1[06],shifter_op1[07],
    shifter_op1[08],shifter_op1[09],shifter_op1[10],shifter_op1[11],
    shifter_op1[12],shifter_op1[13],shifter_op1[14],shifter_op1[15],
    shifter_op1[16],shifter_op1[17],shifter_op1[18],shifter_op1[19],
    shifter_op1[20],shifter_op1[21],shifter_op1[22],shifter_op1[23],
    shifter_op1[24],shifter_op1[25],shifter_op1[26],shifter_op1[27],
    shifter_op1[28],shifter_op1[29],shifter_op1[30],shifter_op1[31]
                 } : shifter_op1
           );
```
代码的意思也很简单 `shifter_op1` 是原始要移位的数据，如果是 `op_sra` 或者 `op_srl` 就将 `shifter_op1` 颠倒给 `shifter_in1`
+ 实际的移位操作
```verilog
  /* 只有 op_shift 为 1 才移位 */
  assign shifter_in2 = {5{op_shift}} & shifter_op2[4:0];
  /* 实际左移操作 */
  assign shifter_res = (shifter_in1 << shifter_in2);
```
+ 移位结果转换
```verilog
  /* 逻辑左移移位结果 */
  wire [`E203_XLEN-1:0] sll_res = shifter_res;
  /* 逻辑右移结果需要再次颠倒一次，负负得正 */
  wire [`E203_XLEN-1:0] srl_res =  
                 {
    shifter_res[00],shifter_res[01],shifter_res[02],shifter_res[03],
    shifter_res[04],shifter_res[05],shifter_res[06],shifter_res[07],
    shifter_res[08],shifter_res[09],shifter_res[10],shifter_res[11],
    shifter_res[12],shifter_res[13],shifter_res[14],shifter_res[15],
    shifter_res[16],shifter_res[17],shifter_res[18],shifter_res[19],
    shifter_res[20],shifter_res[21],shifter_res[22],shifter_res[23],
    shifter_res[24],shifter_res[25],shifter_res[26],shifter_res[27],
    shifter_res[28],shifter_res[29],shifter_res[30],shifter_res[31]
                 };
  
  /* 算数右移结果 */
  wire [`E203_XLEN-1:0] eff_mask = (~(`E203_XLEN'b0)) >> shifter_in2;
  wire [`E203_XLEN-1:0] sra_res =
               (srl_res & eff_mask) | ({32{shifter_op1[31]}} & (~eff_mask));
```
+ 如何处理算数右移
这个理解起来还是有一点难度，算数右移是在逻辑右移的结果上进行处理的。
1. 制作移位掩码 `eff_mask` ，截取符号位实际个数。
假如原始操作为 `1111_0000` 往右移动 `2` 位，那么逻辑移位结果为 `0011_1100` ，算数移位结果为 `1111_1100` 。我们让 `1111_1111` 也向右逻辑移动两位作为我们的 `eff_mask` 掩码。`0011_1111`
2. 通过掩码补全符号位
`eff_mask & srl_res` ，`掩码` 与上 `逻辑右移结果` ，创造 `算数右移` 结果的后半部分 `0011_1100` ，`~eff_mask & 1111_1111` 创造符号位的掩码  
`mask2` `1100_0000` ，通过原始输入获得数据的符号位 `X` 。`mask2&XXXX_XXXX` 得到 `算数右移的前半部分` 。将前半部分与后半部分相 `与` 就得到算数右移最后的结果了。
整个计算过程还是非常巧妙的，利用移位操作创造掩码，通过掩码得到最后的结果。

> [!note]
> 11111

```ad-seealso
123123
123123123
```


## 执行模块
执行模块输入：
1. rs1_data
2. rs2_data
3. rd_data
4. imm_data
5. rs1_valid
6. rs2_valid
7. rd_valid
8. imm_valid
9. 访存读
10. 访存写
11. 


### ALU需要实现的操作
1. +
2. -
3. ^
4. |
5. &
6. 逻辑左移
7. 逻辑右移
8. 算数右移
9. <（有符号、无符号）
10. ==
11. ！=
12. >=(有符号无符号)
13. * （有符号、无符号）（高32、低32）
14. \\ （有符号、无符号）
15. 取余数 （有符号无符号）

### ALU 输出
1. 1bit比较输出
2. 64bit 运算输出

### ALU输入
1. 64bit A操作数
2. 64bit B操作数
3. ALU 操作码

### 回写模块
1. rd-idx
2. 数据输入
3. 

## 为NPC添加halt
### 定位 halt 函数
由于vscode跳转错误，找源码找了一段时间。最开始跳转到 `am.h`
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713104847.png)
找了大大概有十分钟才找到正真的 `halt` 函数，路径为 `abstract-machine/am/src/riscv/npc/trm.c` ，可以看到是一个 `while(1)` 死循环.
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713105116.png)
汇编代码也是一个死循环.
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713105200.png)
### 实现 halt 函数 ebreak
为了把 `halt` 中的 `while(1)` 改为 `ebreak` ,说实话确实不知道该怎么办,但 `nemu` 中已经实现了 `ebreak` 函数,当然是参考别人的了(`cv工程师`).经过寻找
`nemu` 中难度 `halt` 函数位于路径 `/home/leesum/ysyx-workbench/abstract-machine/am/src/platform/nemu/trm.c` .
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713110013.png)
全局搜索 `nemu_trap` ,在 `nemu.h` 下找到宏定义.
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713110105.png)
可以看到不同架构下的 `trap` 函数是不一样的, `riscv` 使用了内联汇编
```asm
asm volatile("mv a0, %0; ebreak" : :"r"(code))
```
我们直接复制粘贴就好了.
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713110318.png)
再次汇编查看结果
1. npc 汇编结果
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713110435.png)
2. nemu 汇编结果
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713110524.png)
一模一样,改造完成!!!
## 为NPC实现HIT GOOD/BAD TRAP
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713110714.png)
还是老方法,先参考别人的源码,再照着别人的思路实现自己的代码.
### nemu 中的 HIT GOOD/BAD TRAP
> 这个我之前做 pa 的时候研究了一下,好像是通过 a0 寄存器的值来判读的.
1. 找到 ebreak 指令实现细节
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713111032.png)
2. 跟随 `NEMUTRAP` 层层跳转
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713111312.png)

最总定位到 `set_nemu_state` 函数
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713111218.png)
其中 `state` 为 `NEMU_END` ; `pc` 为 `s->pc` ; `code` 为 `a0` 寄存器的值.
最终将 `a0` 寄存器的值赋给了 `nemu_state.halt_ret`.
3. 找到最终判断和输出的地方
上一步找到 `nemu_state.halt_ret` 后线索就断了,我们全局搜索 `HIT GOOD` ,最终在 `ysyx-workbench/nemu/src/cpu/cpu-exec.c` 中找到了相关函数 `cpu_exec` .其中
```c
  case NEMU_ABORT:
    Log("nemu: %s at pc = " FMT_WORD,
      (nemu_state.state == NEMU_ABORT ? ANSI_FMT("ABORT", ANSI_FG_RED) : (nemu_state.halt_ret == 0 ? ANSI_FMT("HIT GOOD TRAP", ANSI_FG_GREEN) : ANSI_FMT("HIT BAD TRAP", ANSI_FG_RED))),
      nemu_state.halt_pc);
```
说明了具体的逻辑 `nemu_state.halt_ret == 0` 则 `HIT GOOD` 否则 `HIT BAD TRAP` .

### 为 npc 添加HIT GOOD/BAD TRAP
> [!hint]
> 
具体思路参考上面的 nemu 方式就行了,在执行到 `ebreak` 指令时,读取一下 `a0` 寄存器器的值.

### 通过 ebreak 指令通知仿真结束
`verilog` 实现方法如下
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713112607.png)
在执行的时候,如果指令时 `ebreak` ,直接调用结束函数 `finish` .
`cpp` 实现方法.
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220713112718.png)
很简单,不多说.
### 添加 HIT GOOD/BAD TRAP
```c
  while (!top->contextp()->gotFinish()) {
    mysim.stepCycle();
  }
  // 仿真结束时,会跳到这里来,在这里读取 a0 寄存器的值来判断
 
```
具体实现如下
```cpp
void Simtop::npcTrap() {
    this->registerfile = cpu_gpr;
    this->pc = cpu_pc;
    uint64_t a0 = registerfile[10];
    cout << "a0:" << a0 << endl;
    if (a0 == 0) {
        cout << "PC:" << hex << pc << "\tHIT GOOD" << endl;
    }
    else {
        cout << "PC:" << hex << pc << "\tBAD TRAP" << endl;
    }
}
```

## 为 npc 添加类似 nemu 的命令行交互界面
> [!quote]
> 
阅读 `nemu` 的源码后,发现他使用的是 `readline` 来实现的命令行交互控制.而我使用的是一个 `github` 上开源库.(基于 `readline`)
[Svalorzen/cpp-readline: A very simple C++ wrapper for GNU readline. (github.com)](https://github.com/Svalorzen/cpp-readline)

使用很简单,看看里面的 `example` 就行. 需要注意的是,里面内置了退出命令 `quit` `exit` . 移植后主函数张这样
```c
int main() {
  /* 不知道为什么将 Simtop mysim 声明为全局变量会崩溃*/
  mysim_p = new Simtop;
  static Vtop* top = mysim_p->getTop();
  mysim_p->reset();
  /* 注册命令 */
  cr::Console c(">:");
  c.registerCommand("info", cmd_info);
  c.registerCommand("x", cmd_x);
  c.registerCommand("si", cmd_si);
  c.registerCommand("c", cmd_c);
  c.registerCommand("p", cmd_p);
  c.registerCommand("help", cmd_help);
  c.registerCommand("w", cmd_w);
  int retCode;
  do {
    retCode = c.readLine();
    // We can also change the prompt based on last return value:
    if (retCode == ret::Ok)
      c.setGreeting(">");
    else
      c.setGreeting("!>");
    if (retCode == 1) {
      std::cout << "Received error code 1\n";
    }
    else if (retCode == 2) {
      std::cout << "Received error code 2\n";
    }
  } while (retCode != ret::Quit);
  mysim_p->npcTrap();
  return 0;
}
```

## 为 npc 添加命令
> [!example]
> 
移植命令行交互界面后,添加命令就和 `nemu` 差不多了,可以将 `nemu` 上实现的命令搬过来,但是也需要一些修改.

## 为 npc 添加 difftest

> [!NOTE] 感悟
> 由于我 `pa` 没有做到 `difftest` 章节,初看源码是有点懵的,但参考了一下别人的实现方式,补充了一下动态链接库的知识后,还是可以理解的.

### Linux 动态链接库编程
> [!note]
> 
动态链接与静态链接在 408 中学过,但也仅仅是学过,一般在编程的时候就是编译的时候加上给 -l 参数,添加一些系统库,也没有深入了解.
1. 首先我们先将 `nemu` 编译成动态链接库文件.
在 `build` 目录下找到 `so` 文件.执行命令可以查看链接库里面的函数.
```bash
nm riscv64-nemu-interpreter-so
```
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220714223748.png)
可以看到确实有我们为 `difftest` 准备的函数.
2. 参考 `nemu` 动态链接库的使用方式
文件路径 `src/cpu/difftest/dut.c` 
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220714223926.png)
首先定义了函数指针,用于后面接收函数.
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220714224036.png)
然后用 `dlopen` 打开动态链接库 `so` 文件
其次用 `dlsym` 去寻找动态链接库中的函数地址,并用函数指针保存起来.
最后直接使用函数指针,就可以像普通函数一样使用了.
> [!info] 思考
> 
可以看到,定义函数指针的时候需要指定函数的参数,返回值等等,这个是和动态链接库中的函数一一对应的,但上面我们用 `nm` 命令查看函数时,只能看到名字,不能看到参数.一般动态链接库的作者会给你一个 `h` 文件方便使用.
### 完善 nemu 的 ref.c
```c
// 在DUT host memory的`buf`和REF guest memory的`dest`之间拷贝`n`字节,

// `direction`指定拷贝的方向, `DIFFTEST_TO_DUT`表示往DUT拷贝, `DIFFTEST_TO_REF`表示往REF拷贝

void difftest_memcpy(paddr_t addr, void* buf, size_t n, bool direction) {

  /* 一个一个字节拷贝,只需要实现 dut->ref 方向*/
  if (direction == DIFFTEST_TO_REF) {
    for (size_t i = 0; i < n; i++) {
      paddr_write(addr + i, 1, *((uint8_t*)buf + i));
    }
  }
  else {
    assert(0);
  }
}
// `direction`为`DIFFTEST_TO_DUT`时, 获取REF的寄存器状态到`dut`;
// `direction`为`DIFFTEST_TO_REF`时, 设置REF的寄存器状态为`dut`;
//riscv64_CPU_state
// dut 为一个指针
void difftest_regcpy(void* dut, bool direction) {
  CPU_state* reg_p = dut;
  if (DIFFTEST_TO_REF == direction) {
    for (int i = 0; i < 32; i++) {
      cpu.gpr[i] = reg_p->gpr[i];
    }
    cpu.pc = reg_p->pc;
  }
  else {
    for (int i = 0; i < 32; i++) {
      reg_p->gpr[i] = cpu.gpr[i];
    }
    reg_p->pc = cpu.pc;
  }
}
// 让REF执行`n`条指令
void difftest_exec(uint64_t n) {
  cpu_exec(n);
}
void difftest_raise_intr(word_t NO) {
  assert(0);
}
// 初始化REF的DiffTest功能
void difftest_init(int port) {
  /* Perform ISA dependent initialization. */
  init_isa();
}
```
其中 `CPU_state` 结构体为
```c
typedef struct
{
  word_t gpr[32]; // 64位
  vaddr_t pc;     // 64位
} riscv64_CPU_state;
```
### 完善 npc 仿真端的 difftest
> [!note]
> 
参考 nemu 中 `difftest` 的实现方式,我这里创建了一个 `difftest` 类方便管理

1. 声明函数指针
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220715175744.png)
2. 照葫芦画瓢实现 `init` 函数
```c
void Difftest::init(const char* ref_so_file, long img_size, int port) {
    assert(ref_so_file != NULL);
    void* handle;
    handle = dlopen(ref_so_file, RTLD_LAZY)`;
    assert(handle);
    diff_memcpy = (ref_Difftest_memcpy)dlsym(handle, "difftest_memcpy");
    assert(diff_memcpy);
    diff_regcpy = (ref_Difftest_regcpy)dlsym(handle, "difftest_regcpy");
    assert(diff_regcpy);
    diff_exec = (ref_Difftest_exec)dlsym(handle, "difftest_exec");
    assert(diff_exec);
    diff_raise_intr = (ref_Difftest_raise_intr)dlsym(handle, "difftest_raise_intr");
    assert(diff_raise_intr);
    diff_init = (ref_difftest_init)dlsym(handle, "difftest_init");
    assert(diff_init);

    diff_init(port);

    uint64_t membase = mysim_p->mem->getMEMBASE();
    /* 将程序镜像文件拷贝过去 */
    diff_memcpy(membase, mysim_p->mem->guest_to_host(membase), img_size, DIFFTEST_TO_REF);
    CPU_state regs = getDutregs();
    /* 让 dut 和 ref 寄存器初始值一样 */
    diff_regcpy(&regs, DIFFTEST_TO_REF);
}
```
 3. 实现 `difftest_step`
具体思路就是,让 `ref:nemu` 执行一次,然后对比 `dut` `ref` 的寄存器值,若不同停止执行,并报错.
```cpp
void Difftest::difftest_step() {
    /* 寄存器不一样 */
    diff_exec(1);
    if (!checkregs()) {
        mysim_p->top_status = mysim_p->TOP_STOP;
    }
}
```
4. 将 `difftest` 添加到原始仿真代码中
 > [!tip]
> 
在 `npc` 执行一条指令后(时钟变化一个周期),调用 `difftest_step` 让 `ref:nemu` 也执行一条指令.

只需要在具体的执行函数中加上 `difftst_step` 就行了.
### 移植中的注意事项
1. `nemu` 动态库的编译与链接
按照教程来,会在 `build` 目录下生成 `riscv64-nemu-interpreter-so` ,将其重命名为 `libnemu.so` 并移动到 `/lib` 目录便于编译器查找与链接.
2. 为 `makefile` 文件添加链接库 `-lasan -ldl -lnemu`
```makefile
GCC_LDFLAGS := -LDFLAGS "-lasan -lreadline -ldl -lnemu"
```
其中 `-lasan` 是为了解决报错 [ASan runtime does not come first in initial library list; you should either link runtime to your application or manually preload it with LD_PRELOAD. · Issue #796 · google/sanitizers (github.com)](https://github.com/google/sanitizers/issues/796)
3. 注意 `difftest` 的初始化顺序
`difftest` 必须要在 `npc` `reset` 后初始化,原因如下:
+ `npc` 在没有 `reset` 时, `pc` 的值为 0
![](https://cdn.jsdelivr.net/gh/zilongmix/doc/img/20220715182122.png)
+ `difftest` 在 `init` 时,会拷贝 `npc` 的寄存器组,而我们是通过指针传递的寄存器组文件,初始化的时候为 `NULL`.


