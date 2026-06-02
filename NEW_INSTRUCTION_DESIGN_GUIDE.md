# 新指令设计与修改指导

本文档用于 15 周断网设计考场速查。目标是在老师给出一条新指令后，能快速判断它属于哪一类、应该修改哪些 Verilog 文件、怎么写测试程序并验证结果。

项目根目录：

```text
D:\Computer_organization\project
```

## 1. 当前 CPU 的指令执行路径

本项目中，一条普通 RV32I 指令大致经过以下路径：

```text
Instruction Memory
-> decoder.v 拆 opcode/rd/funct3/rs1/rs2/funct7
-> ctrl_unit.v 判断指令类型
-> imm_gen.v 生成立即数
-> aluControl.v 选择 ALU 运算
-> alu.v 执行计算
-> datapath.v 选择操作数、处理跳转/访存/写回
-> regfile.v 写回 rd
```

最常修改的文件：

```text
rtl\rtl\core\ctrl_unit.v
rtl\rtl\core\aluControl.v
rtl\rtl\core\alu.v
rtl\rtl\core\datapath.v
rtl\rtl\core\imm_gen.v
rtl\rtl\core\branch_unit.v
rtl\rtl\mem\mem_ctrl.v
```

一般规律：

```text
aluControl.v：识别这条新指令应该触发哪种 ALU 运算
alu.v：实现真正的计算逻辑
ctrl_unit.v：决定这条指令是否写 rd、是否是 load/store/branch/jump
datapath.v：当新指令不是普通 ALU 指令时，修改数据通路
imm_gen.v：只有立即数格式特殊时才改
branch_unit.v：只有分支判断条件特殊时才改
mem_ctrl.v：只有 load/store 的字节选择或数据排列特殊时才改
```

## 2. 重要提醒：项目有两套 datapath

`rtl\rtl\core\datapath.v` 中有两个模块：

```verilog
module datapath
module datapath_singlecycle
```

`datapath` 是五级流水 CPU，`datapath_singlecycle` 是单周期 CPU。

`rtl\rtl\top\cpu_top.v` 会根据 `cpu_mode_pipeline` 选择当前使用哪一套：

```verilog
assign dbg_reg_data = use_pipeline ? pipe_dbg_reg_data : single_dbg_reg_data;
```

考场中如果老师用 GUI、EGO1 或随机测试，不一定只测某一种模式。最稳做法是：

```text
五级流水 datapath 改一遍
单周期 datapath_singlecycle 也改一遍
```

如果只改一套，另一种 CPU 模式下可能失败。

## 3. 拿到题后的第一步判断

老师一般会给：

```text
新指令助记符，例如 AAA s0,a0,a1
指令功能
指令编码格式
机器码或 opcode/funct3/funct7
测试样例：source1、source2、expected result
```

先判断它属于哪类：

```text
R-type ALU 指令：AAA rd, rs1, rs2
I-type ALU 指令：AAA rd, rs1, imm
U-type 指令：AAA rd, imm20
Load 指令：AAA rd, offset(rs1)
Store 指令：AAA rs2, offset(rs1)
Branch 指令：AAA rs1, rs2, label
Jump 指令：AAA rd, offset 或 AAA rd, rs1, imm
特殊复合指令：同时涉及访存、计算、跳转或特殊写回
```

最推荐的考场策略是优先把新指令归入普通 ALU 指令路径，因为这类修改最少、风险最低。

## 4. R-type ALU 新指令

形式：

```asm
AAA rd, rs1, rs2
```

常见例子：

```asm
MAX  rd, rs1, rs2
MIN  rd, rs1, rs2
NAND rd, rs1, rs2
NOR  rd, rs1, rs2
XNOR rd, rs1, rs2
AVG  rd, rs1, rs2
```

### 4.1 如果复用标准 R-type opcode

如果老师给：

```text
opcode = 0110011
funct3 = xxx
funct7 = xxxxxxx
```

通常改两个文件：

```text
rtl\rtl\core\aluControl.v
rtl\rtl\core\alu.v
```

### 4.2 修改 aluControl.v

先加新的 ALU 编号：

```verilog
localparam ALU_AAA = 4'd11;
```

然后在 R-type 的 `case (opcode)` 中识别新指令。

例如老师给：

```text
AAA rd, rs1, rs2
opcode = 0110011
funct3 = 000
funct7 = 0000001
```

可以写成：

```verilog
7'b0110011: begin
    case (funct3)
        3'b000: begin
            if (funct7 == 7'b0000001)
                alu_op = ALU_AAA;
            else
                alu_op = funct7[5] ? ALU_SUB : ALU_ADD;
        end
        3'b001: alu_op = ALU_SLL;
        3'b010: alu_op = ALU_SLT;
        3'b011: alu_op = ALU_SLTU;
        3'b100: alu_op = ALU_XOR;
        3'b101: alu_op = funct7[5] ? ALU_SRA : ALU_SRL;
        3'b110: alu_op = ALU_OR;
        3'b111: alu_op = ALU_AND;
    endcase
end
```

### 4.3 修改 alu.v

同样加：

```verilog
localparam ALU_AAA = 4'd11;
```

然后在 `case (alu_op)` 中加功能：

```verilog
ALU_AAA: y = ...;
```

常见功能写法：

```verilog
// 有符号最大值
ALU_AAA: y = ($signed(a) > $signed(b)) ? a : b;

// 有符号最小值
ALU_AAA: y = ($signed(a) < $signed(b)) ? a : b;

// NAND
ALU_AAA: y = ~(a & b);

// NOR
ALU_AAA: y = ~(a | b);

// XNOR
ALU_AAA: y = ~(a ^ b);

// 无符号平均值，避免 a+b 溢出
ALU_AAA: y = (a >> 1) + (b >> 1) + (a[0] & b[0]);
```

注意：`aluControl.v` 和 `alu.v` 里的 `ALU_AAA` 编号必须一致。

### 4.4 如果是 custom opcode

如果老师给的 opcode 不是 `0110011`，例如：

```text
opcode = 0001011
```

还要改 `ctrl_unit.v`，让它走普通 R-type ALU 路径：

```verilog
assign is_op = opcode == 7'b0110011 || opcode == 7'b0001011;
```

然后在 `aluControl.v` 中加：

```verilog
7'b0001011: begin
    alu_op = ALU_AAA;
end
```

## 5. I-type 立即数 ALU 新指令

形式：

```asm
AAA rd, rs1, imm
```

常见例子：

```asm
ADDI2 rd, rs1, imm     # rd = rs1 + imm * 2
ANDNI rd, rs1, imm     # rd = rs1 & ~imm
ORI2  rd, rs1, imm     # rd = rs1 | (imm << 1)
```

### 5.1 如果复用 OP-IMM opcode

如果老师给：

```text
opcode = 0010011
```

通常改：

```text
rtl\rtl\core\aluControl.v
rtl\rtl\core\alu.v
```

因为当前 datapath 已经会对 `is_opimm` 选择 I-type 立即数作为 ALU 第二操作数。

五级流水部分：

```verilog
wire [31:0] alu_b =
    id_ex_is_opimm ? id_ex_imm_i :
    id_ex_is_load ? id_ex_imm_i :
    id_ex_is_store ? id_ex_imm_s :
    id_ex_is_lui ? id_ex_imm_u :
    ex_rs2_data;
```

单周期部分也有类似逻辑：

```verilog
wire [31:0] alu_b =
    is_opimm ? imm_i :
    ...
```

因此只要新指令是 `rs1` 和 I-type 立即数做 ALU 运算，一般不用大改 `datapath.v`。

### 5.2 修改 aluControl.v 和 alu.v

在 `aluControl.v` 中：

```verilog
localparam ALU_AAA = 4'd11;
```

在 OP-IMM 分支中识别：

```verilog
7'b0010011: begin
    case (funct3)
        3'b000: alu_op = ALU_AAA;
        ...
    endcase
end
```

在 `alu.v` 中：

```verilog
localparam ALU_AAA = 4'd11;
...
ALU_AAA: y = ...;
```

### 5.3 如果是 custom opcode

如果老师给 custom opcode，例如：

```text
opcode = 0001011
```

可以在 `ctrl_unit.v` 中让它走 I-type ALU 路径：

```verilog
assign is_opimm = opcode == 7'b0010011 || opcode == 7'b0001011;
```

然后在 `aluControl.v` 中：

```verilog
7'b0001011: begin
    alu_op = ALU_AAA;
end
```

### 5.4 如果立即数格式特殊

当前 `imm_gen.v` 已有标准立即数：

```verilog
assign imm_i = {{20{inst[31]}}, inst[31:20]};
assign imm_s = {{20{inst[31]}}, inst[31:25], inst[11:7]};
assign imm_b = {{19{inst[31]}}, inst[31], inst[7], inst[30:25], inst[11:8], 1'b0};
assign imm_u = {inst[31:12], 12'b0};
assign imm_j = {{11{inst[31]}}, inst[31], inst[19:12], inst[20], inst[30:21], 1'b0};
```

如果老师给的立即数不是 I/S/B/U/J 格式，就要新增：

```verilog
output [31:0] imm_x
assign imm_x = ...;
```

然后同步修改：

```text
datapath 中 imm_gen 的实例化
datapath_singlecycle 中 imm_gen 的实例化
后续选择 alu_b 或跳转目标时使用 imm_x
```

考场中如果能复用已有格式，就尽量不要改 `imm_gen.v` 端口。

## 6. U-type 新指令

形式：

```asm
AAA rd, imm20
```

类似：

```asm
LUI
AUIPC
```

当前 `datapath.v` 中已经有：

```verilog
wire [31:0] ex_wb_data =
    id_ex_is_lui ? id_ex_imm_u :
    id_ex_is_auipc ? (id_ex_pc + id_ex_imm_u) :
    (id_ex_is_jal || id_ex_is_jalr) ? ex_pc_plus4 :
    alu_result;
```

如果新指令功能是：

```text
rd = imm_u
```

可以在 `ctrl_unit.v` 中归类到 `is_lui`：

```verilog
assign is_lui = opcode == 7'b0110111 || opcode == 7'bxxxxxxx;
```

如果功能是：

```text
rd = pc + imm_u
```

可以归类到 `is_auipc`：

```verilog
assign is_auipc = opcode == 7'b0010111 || opcode == 7'bxxxxxxx;
```

如果功能不是这两种，例如：

```text
rd = imm_u ^ rs1
```

就不能简单归类，需要改：

```text
ctrl_unit.v
aluControl.v
alu.v
datapath.v 中 ex_wb_data 或 alu_b
datapath_singlecycle 中 alu_wb_data
```

## 7. 特殊写回类指令

形式可能是：

```asm
ABS   rd, rs1
CLZ   rd, rs1
POPC  rd, rs1
SEXT  rd, rs1
```

这类指令一般还是尽量放进 `alu.v`。

例如 `ABS`：

```verilog
ALU_ABS: y = a[31] ? (~a + 32'd1) : a;
```

例如 `CLZ`，可以写组合逻辑：

```verilog
integer k;

always @(*) begin
    case (alu_op)
        ALU_CLZ: begin
            y = 32;
            for (k = 31; k >= 0; k = k - 1) begin
                if (a[k])
                    y = 31 - k;
            end
        end
        ...
    endcase
end
```

注意项目文件是 `.v`，尽量使用普通 Verilog 写法，不要依赖 SystemVerilog 特性。

## 8. Load 类新指令

形式：

```asm
AAA rd, offset(rs1)
```

如果新指令本质上是特殊 load，可能需要改：

```text
ctrl_unit.v
datapath.v 中 load_extend
datapath.v 中 load_extend_single
mem_ctrl.v
```

如果是 custom opcode load，要在 `ctrl_unit.v` 中：

```verilog
assign is_load = opcode == 7'b0000011 || opcode == 7'bxxxxxxx;
```

当前五级流水的 load 扩展函数在 `datapath.v`：

```verilog
function [31:0] load_extend;
    input [2:0]  funct3;
    input [1:0]  addr_lsb;
    input [31:0] load_word;
```

单周期还有：

```verilog
function [31:0] load_extend_single;
```

如果新增 load 类型，两个函数都要同步改。

例如新指令功能是 word 字节反转：

```text
rd = reverse_bytes(MEM[rs1 + imm])
```

可以加：

```verilog
3'b110: load_extend = {load_word[7:0], load_word[15:8], load_word[23:16], load_word[31:24]};
```

但必须确认该 `funct3` 没和已有 `LBU/LHU` 冲突。

如果只是地址计算仍为 `rs1 + imm`，`aluControl.v` 通常不用改，因为 load 默认走 ADD 地址计算。

## 9. Store 类新指令

形式：

```asm
AAA rs2, offset(rs1)
```

如果是 custom opcode store，在 `ctrl_unit.v` 中：

```verilog
assign is_store = opcode == 7'b0100011 || opcode == 7'bxxxxxxx;
```

store 不写 rd，所以 `writes_rd` 不要包含它。

如果 store 的字节选择或写数据排列特殊，修改：

```text
rtl\rtl\mem\mem_ctrl.v
```

`mem_ctrl.v` 负责：

```text
store_be：写哪些字节
store_wdata：写入数据如何排列
load_data：读出数据如何扩展
```

如果只是普通 `sw/sh/sb` 类型，尽量复用已有逻辑。

## 10. Branch 类新指令

形式：

```asm
AAA rs1, rs2, label
```

常见例子：

```asm
BGT  rs1, rs2, label
BLE  rs1, rs2, label
BEQZ rs1, label
BNEZ rs1, label
```

当前分支判断在 `branch_unit.v`：

```verilog
case (funct3)
    3'b000: branch_taken = rs1_data == rs2_data;
    3'b001: branch_taken = rs1_data != rs2_data;
    3'b100: branch_taken = $signed(rs1_data) < $signed(rs2_data);
    3'b101: branch_taken = $signed(rs1_data) >= $signed(rs2_data);
    3'b110: branch_taken = rs1_data < rs2_data;
    3'b111: branch_taken = rs1_data >= rs2_data;
    default: branch_taken = 1'b0;
endcase
```

如果新 branch 可以用已有 branch 等价替代，优先不用改硬件。

例如：

```asm
BGT rs1, rs2, label
```

等价于：

```asm
BLT rs2, rs1, label
```

如果必须硬件支持 custom branch：

1. `ctrl_unit.v` 中让它被识别为 branch：

```verilog
assign is_branch = opcode == 7'b1100011 || opcode == 7'bxxxxxxx;
```

2. 如果仅靠 `funct3` 能区分，在 `branch_unit.v` 加 case。

3. 如果需要 `opcode` 才能区分，需要修改 `branch_unit` 端口：

```verilog
module branch_unit (
    input  [6:0] opcode,
    input  [2:0] funct3,
    input  [31:0] rs1_data,
    input  [31:0] rs2_data,
    output reg    branch_taken
);
```

然后在 `datapath` 和 `datapath_singlecycle` 的实例化里都传入 opcode。

当前五级流水的跳转控制在 `datapath.v`：

```verilog
wire ex_ctrl_taken = id_ex_valid && (id_ex_is_jal || id_ex_is_jalr || (id_ex_is_branch && branch_taken));
wire [31:0] ex_target_pc =
    id_ex_is_jal ? (id_ex_pc + id_ex_imm_j) :
    id_ex_is_jalr ? ((ex_rs1_data + id_ex_imm_i) & 32'hffff_fffe) :
    (id_ex_pc + id_ex_imm_b);
```

如果新 branch 仍然使用 B-type 立即数，通常不用改 `ex_target_pc`。  
如果目标地址规则不同，要同时改五级流水和单周期的 next PC 逻辑。

## 11. Jump 类新指令

形式：

```asm
AAA rd, offset
AAA rd, rs1, imm
```

如果类似 `JAL`：

```verilog
assign is_jal = opcode == 7'b1101111 || opcode == 7'bxxxxxxx;
```

如果类似 `JALR`：

```verilog
assign is_jalr = opcode == 7'b1100111 || opcode == 7'bxxxxxxx;
```

如果新 jump 的目标地址规则不同，需要改 `datapath.v` 中：

```text
ex_ctrl_taken
ex_target_pc
ex_wb_data
```

因为 JAL/JALR 当前写回的是：

```verilog
pc + 4
```

如果新 jump 需要写回 `pc + 8`，或者不写 rd，就要额外处理。

单周期部分也要修改对应的：

```text
next_pc
alu_wb_data
wb_we
```

## 12. 复合类指令

复合指令是最危险的一类，比如：

```asm
AAA rd, rs1, rs2
功能：rd = MEM[rs1] + rs2
```

这种同时涉及：

```text
地址计算
BRAM 读等待
MEM 阶段数据
WB 阶段计算
流水暂停
forwarding/hazard
```

如果必须实现，可以考虑把它拆成 load-like 指令，在写回阶段做额外处理。

例如：

```text
rd = MEM[rs1 + imm] + rs2
```

可能需要新增流水寄存器：

```verilog
reg [31:0] mem_wb_rs2_data;
```

然后写回时：

```verilog
reg_wdata = is_new_load_calc ? (mem_wb_load_data + mem_wb_rs2_data) : 原逻辑;
```

这类改动风险较大，考场优先避免复杂改法。能把新指令归入 ALU 指令，就不要改成访存复合路径。

## 13. 新增控制信号的完整修改流程

如果必须新增 `is_aaa` 控制信号，修改顺序如下。

### 13.1 修改 ctrl_unit.v

端口加：

```verilog
output is_aaa
```

逻辑加：

```verilog
assign is_aaa = opcode == 7'bxxxxxxx;
```

如果写 rd：

```verilog
assign writes_rd = is_op || is_opimm || is_lui || is_auipc ||
                   is_jal || is_jalr || is_load || is_aaa;
```

### 13.2 修改五级流水 datapath

加 decode 阶段 wire：

```verilog
wire dec_is_aaa;
```

连接 `ctrl_unit`：

```verilog
.is_aaa(dec_is_aaa)
```

加 ID/EX 流水寄存器：

```verilog
reg id_ex_is_aaa;
```

reset 时清零：

```verilog
id_ex_is_aaa <= 1'b0;
```

decode 进入 EX 时赋值：

```verilog
id_ex_is_aaa <= dec_is_aaa;
```

EX 阶段使用：

```verilog
wire [31:0] ex_wb_data =
    id_ex_is_aaa ? aaa_result :
    id_ex_is_lui ? id_ex_imm_u :
    id_ex_is_auipc ? (id_ex_pc + id_ex_imm_u) :
    (id_ex_is_jal || id_ex_is_jalr) ? ex_pc_plus4 :
    alu_result;
```

如果该指令使用 rs1/rs2，还要检查 hazard 判断：

```verilog
wire dec_uses_rs1 = dec_is_op || dec_is_opimm || dec_is_load || dec_is_store ||
                    dec_is_branch || dec_is_jalr;
wire dec_uses_rs2 = dec_is_op || dec_is_store || dec_is_branch;
```

如果新指令用 rs1：

```verilog
dec_uses_rs1 要包含 dec_is_aaa
```

如果新指令用 rs2：

```verilog
dec_uses_rs2 要包含 dec_is_aaa
```

### 13.3 修改单周期 datapath_singlecycle

同步加：

```verilog
wire is_aaa;
```

连接 `ctrl_unit`：

```verilog
.is_aaa(is_aaa)
```

然后在对应位置使用：

```text
alu_b
alu_wb_data
wb_we
next_pc
cpu_dmem_we
```

具体改哪里取决于指令类型。

## 14. 测试程序模板

截图要求通常是：

```text
source1 从 0x4000 读
source2 从 0x4004 读
结果写到 0x400c
```

本项目已经把 `x3/gp` 固定为：

```text
0x00004000
```

因此：

```text
0(gp)  = 0x4000
4(gp)  = 0x4004
12(gp) = 0x400c
```

测试汇编模板：

```asm
lw   t0, 0(gp)       # t0 = mem[0x4000]
lw   t1, 4(gp)       # t1 = mem[0x4004]
.word 0x????????     # 新指令机器码，等价 AAA t2,t0,t1
sw   t2, 12(gp)      # mem[0x400c] = result
```

如果 RARS/LARS 不认识新指令，可以先用：

```asm
add t2, t0, t1
```

作为占位，生成机器码后，把占位机器码替换成老师给的新指令机器码。

## 15. 使用 board_cli.py 验证

连接真实 EGO1 时：

```powershell
cd "D:\Computer_organization\project"
python .\scripts\board_cli.py scan
python .\scripts\board_cli.py --port COM4 cli
```

COM 号按设备管理器实际显示修改。

进入 CLI 后可以手动写入测试数据和指令：

```text
reset
halt
wd 0x4000 0x00000005
wd 0x4004 0x00000003
wi 0x0  0x????????   # lw t0,0(gp)
wi 0x4  0x????????   # lw t1,4(gp)
wi 0x8  0x????????   # 新指令
wi 0xc  0x????????   # sw t2,12(gp)
run
halt
rd 0x400c
```

如果 `rd 0x400c` 读出的结果等于老师给的 expected result，就说明新指令功能通过。

## 16. 几类题目的最短修改路线

### 16.1 R-type，两个寄存器输入，一个 rd 输出

```text
ctrl_unit.v：custom opcode 时把它加入 is_op
aluControl.v：加 ALU_XXX 译码
alu.v：加 ALU_XXX 计算
datapath.v：通常不用改
```

### 16.2 I-type，一个寄存器加立即数，一个 rd 输出

```text
ctrl_unit.v：custom opcode 时把它加入 is_opimm
aluControl.v：加 ALU_XXX 译码
alu.v：加 ALU_XXX 计算
imm_gen.v：只有特殊立即数格式才改
datapath.v：通常不用改
```

### 16.3 Load 变体

```text
ctrl_unit.v：custom opcode 时把它加入 is_load
datapath.v：改 load_extend
datapath.v：改 load_extend_single
mem_ctrl.v：如果字节选择或数据排列特殊，再改
```

### 16.4 Store 变体

```text
ctrl_unit.v：custom opcode 时把它加入 is_store
mem_ctrl.v：改 store_be/store_wdata
datapath.v：通常不用大改，但要确认两套 datapath 都走到 mem_ctrl
```

### 16.5 Branch 变体

```text
ctrl_unit.v：custom opcode 时把它加入 is_branch
branch_unit.v：加判断条件
datapath.v：如果目标地址不是 pc + imm_b，改 ex_target_pc 和单周期 next_pc
```

### 16.6 Jump 变体

```text
ctrl_unit.v：加入 is_jal 或 is_jalr，或者新增控制信号
datapath.v：改目标 PC 和写回值
imm_gen.v：特殊立即数格式才改
```

### 16.7 特殊复合指令

```text
先判断能不能伪装成 ALU 指令
不能的话再新增控制信号
五级流水和单周期都要同步
额外检查 hazard、forwarding、load wait、写回路径
```

## 17. 常见错误清单

最容易出错的地方：

```text
只改 alu.v，忘了 aluControl.v
aluControl.v 和 alu.v 的 ALU 编号不一致
custom opcode 没加到 ctrl_unit.v
新指令写 rd，但 writes_rd 没包含它
新指令使用 rs1/rs2，但 hazard 判断 dec_uses_rs1/dec_uses_rs2 没包含它
只改五级流水，忘了 datapath_singlecycle
只改 load_extend，忘了 load_extend_single
RARS/LARS 不认识新指令，忘了用 .word 机器码
把 x3/gp 当普通寄存器写，导致结果不对
测试程序结果写错地址，老师要求通常是 0x400c
```

## 18. 考场优先级建议

拿到新指令后，按这个顺序判断：

```text
1. 它能不能当 R-type ALU 指令做？
2. 它能不能当 I-type ALU 指令做？
3. 它能不能复用已有 load/store/branch/jump 路径？
4. 是否必须新增控制信号？
5. 是否必须新增流水寄存器？
```

最稳方案：

```text
读 rs1/rs2 或 rs1/imm
ALU 算结果
写 rd
```

这种通常只需要改：

```text
aluControl.v
alu.v
ctrl_unit.v（仅 custom opcode 时需要）
```

如果老师给的是复杂访存或跳转指令，先复用现有路径，再做最小修改。不要一上来大改 datapath。

## 19. 20 条候选新指令补丁包

本节是一个考前候选补丁包。它假设老师可能给的指令主要集中在以下几类：

```text
R-type 自定义 ALU 指令
I-type 自定义立即数指令
Load 变体
Branch 变体
```

为了方便考场复制粘贴，本节统一设计 20 条候选指令，并给出一套可以同时支持这 20 条指令的代码修改方案。

如果老师给出的真实 opcode/funct3/funct7 和下面不同，只需要重点改 `aluControl.v` 中对应指令的译码条件；`alu.v` 中的计算逻辑通常可以直接复用。

### 19.1 候选指令总表

#### 1. MAX

类型：R-type ALU。

功能：

```text
MAX rd, rs1, rs2
rd = signed(rs1) > signed(rs2) ? rs1 : rs2
```

编码：

```text
opcode = 0001011
funct7 = 0000001
funct3 = 000
```

修改思路：把 custom opcode `0001011` 当作 R-type ALU 指令，在 `aluControl.v` 中译码为 `ALU_MAX`，在 `alu.v` 中执行有符号比较。

#### 2. MIN

类型：R-type ALU。

功能：

```text
MIN rd, rs1, rs2
rd = signed(rs1) < signed(rs2) ? rs1 : rs2
```

编码：

```text
opcode = 0001011
funct7 = 0000001
funct3 = 001
```

修改思路：与 `MAX` 类似，只是比较方向相反。

#### 3. MAXU

类型：R-type ALU。

功能：

```text
MAXU rd, rs1, rs2
rd = unsigned(rs1) > unsigned(rs2) ? rs1 : rs2
```

编码：

```text
opcode = 0001011
funct7 = 0000001
funct3 = 010
```

修改思路：使用无符号比较。

#### 4. MINU

类型：R-type ALU。

功能：

```text
MINU rd, rs1, rs2
rd = unsigned(rs1) < unsigned(rs2) ? rs1 : rs2
```

编码：

```text
opcode = 0001011
funct7 = 0000001
funct3 = 011
```

修改思路：使用无符号比较。

#### 5. NAND

类型：R-type ALU。

功能：

```text
NAND rd, rs1, rs2
rd = ~(rs1 & rs2)
```

编码：

```text
opcode = 0001011
funct7 = 0000001
funct3 = 100
```

修改思路：普通位运算，直接在 ALU 中实现。

#### 6. NOR

类型：R-type ALU。

功能：

```text
NOR rd, rs1, rs2
rd = ~(rs1 | rs2)
```

编码：

```text
opcode = 0001011
funct7 = 0000001
funct3 = 101
```

修改思路：普通位运算，直接在 ALU 中实现。

#### 7. XNOR

类型：R-type ALU。

功能：

```text
XNOR rd, rs1, rs2
rd = ~(rs1 ^ rs2)
```

编码：

```text
opcode = 0001011
funct7 = 0000001
funct3 = 110
```

修改思路：普通位运算，直接在 ALU 中实现。

#### 8. ANDN

类型：R-type ALU。

功能：

```text
ANDN rd, rs1, rs2
rd = rs1 & ~rs2
```

编码：

```text
opcode = 0001011
funct7 = 0000001
funct3 = 111
```

修改思路：常见扩展位运算，直接在 ALU 中实现。

#### 9. ORN

类型：R-type ALU。

功能：

```text
ORN rd, rs1, rs2
rd = rs1 | ~rs2
```

编码：

```text
opcode = 0001011
funct7 = 0000010
funct3 = 000
```

修改思路：常见扩展位运算，直接在 ALU 中实现。

#### 10. AVG

类型：R-type ALU。

功能：

```text
AVG rd, rs1, rs2
rd = unsigned_average(rs1, rs2)
```

编码：

```text
opcode = 0001011
funct7 = 0000010
funct3 = 001
```

修改思路：为了避免 `rs1 + rs2` 溢出，使用：

```verilog
(a >> 1) + (b >> 1) + (a[0] & b[0])
```

#### 11. ABS

类型：R-type unary ALU。

功能：

```text
ABS rd, rs1
rd = abs(signed(rs1))
```

编码：

```text
opcode = 0001011
funct7 = 0000010
funct3 = 010
```

修改思路：仍然使用 R-type 格式，但只使用 `rs1`，忽略 `rs2`。

#### 12. NEG

类型：R-type unary ALU。

功能：

```text
NEG rd, rs1
rd = -rs1
```

编码：

```text
opcode = 0001011
funct7 = 0000010
funct3 = 011
```

修改思路：在 ALU 中实现二进制补码取负。

#### 13. NOT

类型：R-type unary ALU。

功能：

```text
NOT rd, rs1
rd = ~rs1
```

编码：

```text
opcode = 0001011
funct7 = 0000010
funct3 = 100
```

修改思路：在 ALU 中实现按位取反。

#### 14. SEXTB

类型：R-type unary ALU。

功能：

```text
SEXTB rd, rs1
rd = sign_extend(rs1[7:0])
```

编码：

```text
opcode = 0001011
funct7 = 0000010
funct3 = 101
```

修改思路：取低 8 位，然后符号扩展到 32 位。

#### 15. ZEXTH

类型：R-type unary ALU。

功能：

```text
ZEXTH rd, rs1
rd = zero_extend(rs1[15:0])
```

编码：

```text
opcode = 0001011
funct7 = 0000010
funct3 = 110
```

修改思路：取低 16 位，然后零扩展到 32 位。

#### 16. CLZ

类型：R-type unary ALU。

功能：

```text
CLZ rd, rs1
rd = count_leading_zeros(rs1)
```

编码：

```text
opcode = 0001011
funct7 = 0000010
funct3 = 111
```

修改思路：在 ALU 中写组合逻辑函数，统计最高位开始连续 0 的个数。

#### 17. POPCNT

类型：R-type unary ALU。

功能：

```text
POPCNT rd, rs1
rd = number_of_1_bits(rs1)
```

编码：

```text
opcode = 0001011
funct7 = 0000011
funct3 = 000
```

修改思路：在 ALU 中写组合逻辑函数，统计 32 位中 1 的个数。

#### 18. ADDI2

类型：I-type ALU。

功能：

```text
ADDI2 rd, rs1, imm
rd = rs1 + (imm << 1)
```

编码：

```text
opcode = 0101011
funct3 = 000
```

修改思路：把 custom opcode `0101011` 当作 I-type ALU 指令，datapath 会自动把 `imm_i` 作为 ALU 第二操作数；ALU 中实现 `a + (b << 1)`。

#### 19. LREV

类型：Load 变体。

功能：

```text
LREV rd, offset(rs1)
word = MEM[rs1 + offset]
rd = reverse_bytes(word)
```

编码：

```text
opcode = 0000011
funct3 = 110
```

修改思路：复用标准 load opcode，不改 `ctrl_unit.v`；在 `datapath.v` 的 `load_extend` 和 `load_extend_single` 中增加 `funct3 == 110` 的字节反转逻辑。

#### 20. BGT

类型：Branch 变体。

功能：

```text
BGT rs1, rs2, label
if signed(rs1) > signed(rs2), pc = pc + imm_b
```

编码：

```text
opcode = 1100011
funct3 = 010
```

修改思路：复用标准 branch opcode，不改 `ctrl_unit.v`；在 `branch_unit.v` 中增加 `funct3 == 010` 的有符号大于判断。

## 20. 候选补丁包：具体代码修改

下面的代码修改可以同时支持第 19 节列出的 20 条候选指令。

### 20.1 修改 ctrl_unit.v

文件：

```text
rtl\rtl\core\ctrl_unit.v
```

只需要替换两行。

把原来的：

```verilog
assign is_opimm  = opcode == 7'b0010011;
assign is_op     = opcode == 7'b0110011;
```

替换成：

```verilog
assign is_opimm  = opcode == 7'b0010011 || opcode == 7'b0101011;
assign is_op     = opcode == 7'b0110011 || opcode == 7'b0001011;
```

说明：

```text
0001011 用作 R-type custom ALU 指令
0101011 用作 I-type custom ALU 指令
```

### 20.2 整体替换 aluControl.v

文件：

```text
rtl\rtl\core\aluControl.v
```

建议直接把整个文件替换为下面内容：

```verilog
`timescale 1ns / 1ps

module aluControl (
    input      [6:0] opcode,
    input      [2:0] funct3,
    input      [6:0] funct7,
    output reg [4:0] alu_op
);
    localparam ALU_ADD    = 5'd0;
    localparam ALU_SUB    = 5'd1;
    localparam ALU_SLL    = 5'd2;
    localparam ALU_SLT    = 5'd3;
    localparam ALU_SLTU   = 5'd4;
    localparam ALU_XOR    = 5'd5;
    localparam ALU_SRL    = 5'd6;
    localparam ALU_SRA    = 5'd7;
    localparam ALU_OR     = 5'd8;
    localparam ALU_AND    = 5'd9;
    localparam ALU_PASS   = 5'd10;

    localparam ALU_MAX    = 5'd11;
    localparam ALU_MIN    = 5'd12;
    localparam ALU_MAXU   = 5'd13;
    localparam ALU_MINU   = 5'd14;
    localparam ALU_NAND   = 5'd15;
    localparam ALU_NOR    = 5'd16;
    localparam ALU_XNOR   = 5'd17;
    localparam ALU_ANDN   = 5'd18;
    localparam ALU_ORN    = 5'd19;
    localparam ALU_AVG    = 5'd20;
    localparam ALU_ABS    = 5'd21;
    localparam ALU_NEG    = 5'd22;
    localparam ALU_NOT    = 5'd23;
    localparam ALU_SEXTB  = 5'd24;
    localparam ALU_ZEXTH  = 5'd25;
    localparam ALU_CLZ    = 5'd26;
    localparam ALU_POPCNT = 5'd27;
    localparam ALU_ADDI2  = 5'd28;

    always @(*) begin
        alu_op = ALU_ADD;
        case (opcode)
            7'b0110011: begin
                case (funct3)
                    3'b000: alu_op = funct7[5] ? ALU_SUB : ALU_ADD;
                    3'b001: alu_op = ALU_SLL;
                    3'b010: alu_op = ALU_SLT;
                    3'b011: alu_op = ALU_SLTU;
                    3'b100: alu_op = ALU_XOR;
                    3'b101: alu_op = funct7[5] ? ALU_SRA : ALU_SRL;
                    3'b110: alu_op = ALU_OR;
                    3'b111: alu_op = ALU_AND;
                endcase
            end

            7'b0010011: begin
                case (funct3)
                    3'b000: alu_op = ALU_ADD;
                    3'b001: alu_op = ALU_SLL;
                    3'b010: alu_op = ALU_SLT;
                    3'b011: alu_op = ALU_SLTU;
                    3'b100: alu_op = ALU_XOR;
                    3'b101: alu_op = funct7[5] ? ALU_SRA : ALU_SRL;
                    3'b110: alu_op = ALU_OR;
                    3'b111: alu_op = ALU_AND;
                endcase
            end

            7'b0110111: alu_op = ALU_PASS;

            7'b0001011: begin
                case ({funct7, funct3})
                    10'b0000001_000: alu_op = ALU_MAX;
                    10'b0000001_001: alu_op = ALU_MIN;
                    10'b0000001_010: alu_op = ALU_MAXU;
                    10'b0000001_011: alu_op = ALU_MINU;
                    10'b0000001_100: alu_op = ALU_NAND;
                    10'b0000001_101: alu_op = ALU_NOR;
                    10'b0000001_110: alu_op = ALU_XNOR;
                    10'b0000001_111: alu_op = ALU_ANDN;
                    10'b0000010_000: alu_op = ALU_ORN;
                    10'b0000010_001: alu_op = ALU_AVG;
                    10'b0000010_010: alu_op = ALU_ABS;
                    10'b0000010_011: alu_op = ALU_NEG;
                    10'b0000010_100: alu_op = ALU_NOT;
                    10'b0000010_101: alu_op = ALU_SEXTB;
                    10'b0000010_110: alu_op = ALU_ZEXTH;
                    10'b0000010_111: alu_op = ALU_CLZ;
                    10'b0000011_000: alu_op = ALU_POPCNT;
                    default:         alu_op = ALU_ADD;
                endcase
            end

            7'b0101011: begin
                case (funct3)
                    3'b000: alu_op = ALU_ADDI2;
                    default: alu_op = ALU_ADD;
                endcase
            end

            default: alu_op = ALU_ADD;
        endcase
    end
endmodule
```

### 20.3 整体替换 alu.v

文件：

```text
rtl\rtl\core\alu.v
```

建议直接把整个文件替换为下面内容：

```verilog
`timescale 1ns / 1ps

module alu (
    input  [31:0] a,
    input  [31:0] b,
    input  [4:0]  alu_op,
    output reg [31:0] y
);
    localparam ALU_ADD    = 5'd0;
    localparam ALU_SUB    = 5'd1;
    localparam ALU_SLL    = 5'd2;
    localparam ALU_SLT    = 5'd3;
    localparam ALU_SLTU   = 5'd4;
    localparam ALU_XOR    = 5'd5;
    localparam ALU_SRL    = 5'd6;
    localparam ALU_SRA    = 5'd7;
    localparam ALU_OR     = 5'd8;
    localparam ALU_AND    = 5'd9;
    localparam ALU_PASS   = 5'd10;

    localparam ALU_MAX    = 5'd11;
    localparam ALU_MIN    = 5'd12;
    localparam ALU_MAXU   = 5'd13;
    localparam ALU_MINU   = 5'd14;
    localparam ALU_NAND   = 5'd15;
    localparam ALU_NOR    = 5'd16;
    localparam ALU_XNOR   = 5'd17;
    localparam ALU_ANDN   = 5'd18;
    localparam ALU_ORN    = 5'd19;
    localparam ALU_AVG    = 5'd20;
    localparam ALU_ABS    = 5'd21;
    localparam ALU_NEG    = 5'd22;
    localparam ALU_NOT    = 5'd23;
    localparam ALU_SEXTB  = 5'd24;
    localparam ALU_ZEXTH  = 5'd25;
    localparam ALU_CLZ    = 5'd26;
    localparam ALU_POPCNT = 5'd27;
    localparam ALU_ADDI2  = 5'd28;

    function [31:0] sra32;
        input [31:0] value;
        input [4:0]  shamt;
        reg [31:0] shifted;
        begin
            shifted = value >> shamt;
            if (value[31] && shamt != 5'd0)
                sra32 = shifted | (32'hffff_ffff << (6'd32 - {1'b0, shamt}));
            else
                sra32 = shifted;
        end
    endfunction

    function [31:0] clz32;
        input [31:0] value;
        integer i;
        reg found_one;
        begin
            clz32 = 32'd0;
            found_one = 1'b0;
            for (i = 31; i >= 0; i = i - 1) begin
                if (!found_one) begin
                    if (value[i])
                        found_one = 1'b1;
                    else
                        clz32 = clz32 + 32'd1;
                end
            end
        end
    endfunction

    function [31:0] popcnt32;
        input [31:0] value;
        integer i;
        begin
            popcnt32 = 32'd0;
            for (i = 0; i < 32; i = i + 1)
                popcnt32 = popcnt32 + value[i];
        end
    endfunction

    always @(*) begin
        case (alu_op)
            ALU_ADD:    y = a + b;
            ALU_SUB:    y = a - b;
            ALU_SLL:    y = a << b[4:0];
            ALU_SLT:    y = ($signed(a) < $signed(b)) ? 32'd1 : 32'd0;
            ALU_SLTU:   y = (a < b) ? 32'd1 : 32'd0;
            ALU_XOR:    y = a ^ b;
            ALU_SRL:    y = a >> b[4:0];
            ALU_SRA:    y = sra32(a, b[4:0]);
            ALU_OR:     y = a | b;
            ALU_AND:    y = a & b;
            ALU_PASS:   y = b;

            ALU_MAX:    y = ($signed(a) > $signed(b)) ? a : b;
            ALU_MIN:    y = ($signed(a) < $signed(b)) ? a : b;
            ALU_MAXU:   y = (a > b) ? a : b;
            ALU_MINU:   y = (a < b) ? a : b;
            ALU_NAND:   y = ~(a & b);
            ALU_NOR:    y = ~(a | b);
            ALU_XNOR:   y = ~(a ^ b);
            ALU_ANDN:   y = a & ~b;
            ALU_ORN:    y = a | ~b;
            ALU_AVG:    y = (a >> 1) + (b >> 1) + {31'd0, (a[0] & b[0])};
            ALU_ABS:    y = a[31] ? (~a + 32'd1) : a;
            ALU_NEG:    y = ~a + 32'd1;
            ALU_NOT:    y = ~a;
            ALU_SEXTB:  y = {{24{a[7]}}, a[7:0]};
            ALU_ZEXTH:  y = {16'd0, a[15:0]};
            ALU_CLZ:    y = clz32(a);
            ALU_POPCNT: y = popcnt32(a);
            ALU_ADDI2:  y = a + (b << 1);

            default:    y = 32'd0;
        endcase
    end
endmodule
```

### 20.4 修改 datapath.v 的 ALU 控制信号宽度

文件：

```text
rtl\rtl\core\datapath.v
```

因为上面把 `alu_op` 从 4 位扩展到了 5 位，所以要改三处。

把：

```verilog
reg [3:0]  id_ex_alu_op;
```

替换成：

```verilog
reg [4:0]  id_ex_alu_op;
```

把：

```verilog
wire [3:0] dec_alu_op;
```

替换成：

```verilog
wire [4:0] dec_alu_op;
```

把单周期部分的：

```verilog
wire [3:0] alu_op;
```

替换成：

```verilog
wire [4:0] alu_op;
```

说明：`datapath.v` 中原来的 `ALU_ADD/ALU_SUB/...` localparam 仍然可以保留，不强制改成 5 位，因为它们只用于旧指令和复位默认值。

### 20.5 修改 datapath.v 的 load_extend

文件：

```text
rtl\rtl\core\datapath.v
```

为了支持候选指令 `LREV`，要改两处：五级流水的 `load_extend` 和单周期的 `load_extend_single`。

#### 20.5.1 五级流水 load_extend

找到五级流水中的：

```verilog
                3'b101: load_extend = addr_lsb[1] ? {16'd0, load_word[31:16]} :
                                                     {16'd0, load_word[15:0]};
                default: load_extend = load_word;
```

替换成：

```verilog
                3'b101: load_extend = addr_lsb[1] ? {16'd0, load_word[31:16]} :
                                                     {16'd0, load_word[15:0]};
                3'b110: load_extend = {load_word[7:0], load_word[15:8],
                                        load_word[23:16], load_word[31:24]};
                default: load_extend = load_word;
```

#### 20.5.2 单周期 load_extend_single

找到单周期中的：

```verilog
                3'b101: load_extend_single = load_addr_lsb[1] ? {16'd0, load_word[31:16]} :
                                                               {16'd0, load_word[15:0]};
                default: load_extend_single = load_word;
```

替换成：

```verilog
                3'b101: load_extend_single = load_addr_lsb[1] ? {16'd0, load_word[31:16]} :
                                                               {16'd0, load_word[15:0]};
                3'b110: load_extend_single = {load_word[7:0], load_word[15:8],
                                               load_word[23:16], load_word[31:24]};
                default: load_extend_single = load_word;
```

### 20.6 整体替换 branch_unit.v

文件：

```text
rtl\rtl\core\branch_unit.v
```

为了支持候选指令 `BGT`，建议直接替换为：

```verilog
`timescale 1ns / 1ps

module branch_unit (
    input  [2:0]  funct3,
    input  [31:0] rs1_data,
    input  [31:0] rs2_data,
    output reg    branch_taken
);
    always @(*) begin
        case (funct3)
            3'b000: branch_taken = rs1_data == rs2_data;
            3'b001: branch_taken = rs1_data != rs2_data;
            3'b010: branch_taken = $signed(rs1_data) > $signed(rs2_data);
            3'b100: branch_taken = $signed(rs1_data) < $signed(rs2_data);
            3'b101: branch_taken = $signed(rs1_data) >= $signed(rs2_data);
            3'b110: branch_taken = rs1_data < rs2_data;
            3'b111: branch_taken = rs1_data >= rs2_data;
            default: branch_taken = 1'b0;
        endcase
    end
endmodule
```

## 21. 候选指令机器码生成方法

RARS/LARS 默认不认识自定义指令，所以建议用 `.word 0x????????` 写入机器码。

### 21.1 R-type custom 指令编码

适用于：

```text
MAX/MIN/MAXU/MINU/NAND/NOR/XNOR/ANDN/ORN/AVG/ABS/NEG/NOT/SEXTB/ZEXTH/CLZ/POPCNT
```

编码公式：

```text
inst = (funct7 << 25) | (rs2 << 20) | (rs1 << 15) |
       (funct3 << 12) | (rd << 7) | 0x0B
```

常用寄存器编号：

```text
t0 = x5
t1 = x6
t2 = x7
gp = x3
```

如果要生成：

```asm
MAX t2, t0, t1
```

即：

```text
rd  = x7
rs1 = x5
rs2 = x6
funct7 = 0000001
funct3 = 000
opcode = 0001011
```

可以用 PowerShell：

```powershell
python -c "funct7=0b0000001; funct3=0b000; rd=7; rs1=5; rs2=6; inst=(funct7<<25)|(rs2<<20)|(rs1<<15)|(funct3<<12)|(rd<<7)|0x0B; print(f'0x{inst:08x}')"
```

### 21.2 R-type 候选指令 funct 表

```text
MAX    funct7=0000001 funct3=000
MIN    funct7=0000001 funct3=001
MAXU   funct7=0000001 funct3=010
MINU   funct7=0000001 funct3=011
NAND   funct7=0000001 funct3=100
NOR    funct7=0000001 funct3=101
XNOR   funct7=0000001 funct3=110
ANDN   funct7=0000001 funct3=111
ORN    funct7=0000010 funct3=000
AVG    funct7=0000010 funct3=001
ABS    funct7=0000010 funct3=010
NEG    funct7=0000010 funct3=011
NOT    funct7=0000010 funct3=100
SEXTB  funct7=0000010 funct3=101
ZEXTH  funct7=0000010 funct3=110
CLZ    funct7=0000010 funct3=111
POPCNT funct7=0000011 funct3=000
```

### 21.3 ADDI2 编码

适用于：

```asm
ADDI2 rd, rs1, imm
```

编码公式：

```text
inst = ((imm & 0xfff) << 20) | (rs1 << 15) |
       (0 << 12) | (rd << 7) | 0x2B
```

例如：

```asm
ADDI2 t2, t0, 4
```

PowerShell：

```powershell
python -c "imm=4; rd=7; rs1=5; inst=((imm&0xfff)<<20)|(rs1<<15)|(0<<12)|(rd<<7)|0x2B; print(f'0x{inst:08x}')"
```

### 21.4 LREV 编码

适用于：

```asm
LREV rd, offset(rs1)
```

编码公式：

```text
inst = ((imm & 0xfff) << 20) | (rs1 << 15) |
       (6 << 12) | (rd << 7) | 0x03
```

例如：

```asm
LREV t2, 0(gp)
```

PowerShell：

```powershell
python -c "imm=0; rd=7; rs1=3; inst=((imm&0xfff)<<20)|(rs1<<15)|(6<<12)|(rd<<7)|0x03; print(f'0x{inst:08x}')"
```

### 21.5 BGT 编码

适用于：

```asm
BGT rs1, rs2, offset
```

其中 offset 是相对当前 PC 的字节偏移，必须是偶数。

编码公式比较复杂，建议用下面 Python：

```powershell
python -c "offset=8; rs1=5; rs2=6; funct3=2; opcode=0x63; imm=offset&0x1fff; inst=(((imm>>12)&1)<<31)|(((imm>>5)&0x3f)<<25)|(rs2<<20)|(rs1<<15)|(funct3<<12)|(((imm>>1)&0xf)<<8)|(((imm>>11)&1)<<7)|opcode; print(f'0x{inst:08x}')"
```

## 22. 候选指令通用测试模板

老师通常要求：

```text
source1 从 0x4000 读
source2 从 0x4004 读
结果写到 0x400c
```

本项目中：

```text
x3/gp = 0x00004000
```

因此测试程序基本模板是：

```asm
lw   t0, 0(gp)
lw   t1, 4(gp)
.word 0x????????     # 候选新指令，例如 MAX t2,t0,t1
sw   t2, 12(gp)
```

如果使用 `ABS/NEG/NOT/SEXTB/ZEXTH/CLZ/POPCNT` 这类 unary 指令，仍然可以写成：

```asm
lw   t0, 0(gp)
.word 0x????????     # 候选新指令，例如 ABS t2,t0
sw   t2, 12(gp)
```

如果使用 `ADDI2`：

```asm
lw   t0, 0(gp)
.word 0x????????     # ADDI2 t2,t0,imm
sw   t2, 12(gp)
```

如果使用 `LREV`：

```asm
.word 0x????????     # LREV t2,0(gp)
sw   t2, 12(gp)
```

如果使用 `BGT`，可以设计：

```asm
lw   t0, 0(gp)
lw   t1, 4(gp)
.word 0x????????     # BGT t0,t1,+8，成立则跳过下一条 addi
addi t2, zero, 0
addi t2, zero, 1
sw   t2, 12(gp)
```

当 `t0 > t1` 时，最终 `0x400c` 应为 `1`。

## 23. 用 board_cli.py 快速验证候选指令

进入硬件 CLI：

```powershell
cd "D:\Computer_organization\project"
python .\scripts\board_cli.py --port COM4 cli
```

COM 号按实际情况修改。

测试普通 R-type 候选指令时：

```text
reset
halt
wd 0x4000 0x00000005
wd 0x4004 0x00000003
wi 0x0  0x0001a283
wi 0x4  0x0041a303
wi 0x8  0x????????   # 候选新指令机器码
wi 0xc  0x0071a623
run
halt
rd 0x400c
```

说明：

```text
0x0001a283 = lw t0,0(gp)
0x0041a303 = lw t1,4(gp)
0x0071a623 = sw t2,12(gp)
```

如果 `rd 0x400c` 的结果等于 expected result，说明通过。

