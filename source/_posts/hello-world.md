---
title: 基于logisim的mips单周期处理器设计
tags: 
  - co
  - Logisim
  - MIPS
catagories:
  - co
  - Logisim
  - MIPS
---
# 基于logisim的mips单周期处理器设计

## 设计思路

#### 一、CPU功能

cpu功能即为控制指令执行，包含对指令的四种操作：

* 取数:从主存中取出数据并送至某个寄存器中
* 存数：将某个寄存器中的值存入主存
* 传送：将某个寄存器中的数据传送至ALU或另一个寄存器
* 运算：ALU进行运算，结果保存至某个寄存器中

cpu设计一般思路：
1. 分析指令系统需求
2. 分类型依次构建数据通路（R型，I型，J型）
3. 设计控制器  
4. 合并完成
   
***
#### 二、单周期cpu设计

**1. 本次设计需支持的指令：**

* R型：
  add, sub
* I型：
   ori, lw, sw, lui, beq,
* 空指令：
  nop
  ***

**2. 模块需求：**
* IFU（取指令单元）包含：
  * PC（程序计数器）：寄存器实现，起始地址：0x00003000。
  * IM（指令存储器）：ROM实现，12位地址，32位位宽，内部起始地址从零开始
* GRF（寄存器堆）包含：
  * 32个32位寄存器
* ALU（算术逻辑单元）：
  * 进行各种运算
* DM（数据存储器）：
  * RAM实现：32位位宽
* Controller（控制器）：
  * 输入指令的OP（指令前六位）以及Func（R型指令后六位）
  * 输出各个其他的模块的使能信号
***
**3. 模块设计：**
* **IFU:**
  <img src="cpuimage/IFU.png" alt="IFU" style="zoom: 60%;" />
  功能描述：
  
  * PC每次右移两位后使用Bit Extender变为12位从IM(ROM)中取出指令
  * 取指令：instr = IM[PC]
  * PC每次取指令完成后自增：PC <= PC + 4  
  
* **GRF:**
  * 在P0课下的GRF上稍作调整，外观及引脚调整为
  <img src="cpuimage/GRF1.png" alt="IFU" style="zoom:50%;" />
  * GRF内部基本由三部分组成：
  1. 通过Rs,Rt地址利用多路选择器找到对应寄存器并输出其中存的数据R1,R2
  <img src="cpuimage/GRF2.png" alt="IFU" style="zoom: 33%;" />
  2. 通过RegAddr利用解码器找到需要修改的寄存器
  <img src="cpuimage/GRF3.png" alt="IFU" style="zoom: 33%;" />
  3. 通过各个使能信号决定是否修改RegAddr对应的寄存器的值（并将32个标签与对应寄存器相连）
  <img src="image-4.png" alt="Alt text" style="zoom: 40%;" />
  
* **ALU:**
  * 根据本次设计需支持的8个指令，进行ALU各分支设计：
    * add, lw, sw : 加法
    * sub : 减法
    * ori : 或运算
    * lui : 移位
    * beq : 判断Rs与Rt是否相等
  * 设计图如下：
  <img src="image.png" alt="Alt text" style="zoom: 40%;" />
  * 通过Controller确定的ALUOp决定输出分支
  
* **Controller:**
  controller 由两部分组成：
  1. 通过Op确定是哪条指令，并连接对应标签（其中R型指令Op相同，需根据Func进一步判断）
      <img src="image-3.png" alt="Alt text" style="zoom: 50%;" />
      该部分我没有采用P3教程中的方法，而是利用Comparater直接将Op与常量比较看属于哪条指令
  1. 通过各指令的数据通路，决定Controller的输出使能信号的高低电平
      本次设计的Controller一共有8个输出使能信号，分别是：
      <img src="image-5.png" alt="Alt text" style="zoom: 50%;" />
  * branch : beq 时置1，其余均置0
  * MemtoReg : 选择 ALU运算结果或从DM中读出数据 存至GRF
  * MemWrite : DM读数据（lw）
  * MemRead : DM写数据（sw）
  * ALUOp : 决定ALU模块输出的分支
  * ALUSrc : 决定ALU输入是 Rt 对应数据还是 imm 立即数（R型指令置1，I型指令置1）
  * RegWrite : 是否改变GRF中寄存器的值
  * RegDst : 选择进入GRF的RegAddr是 Rt 还是 Rd（R型指令有Rd，I型指令无Rd）
  <img src="image-7.png" alt="Alt text" style="zoom:40%;" />
  
* **DM:**
  * RAM实现，具有读写功能  
    <img src="image-8.png" alt="Alt text" style="zoom:50%;" />    
    **4. 数据通路：**
    将上述各模块相连，并将各个指令依次构建数据通路，详见PPT
***
​	**5. 数据通路合并，设计完成：**

![Alt text](image-6.png)

***

## 思考题：

  **1. 上面我们介绍了通过 FSM 理解单周期 CPU 的基本方法。请大家指出单周期 CPU 所用到的模块中，哪些发挥状态存储功能，哪些发挥状态转移功能。**
   答：状态存储功能模块：GRF、DM；状态转移功能模块：IFU、ALU、Controller
  **2. 现在我们的模块中 IM 使用 ROM， DM 使用 RAM， GRF 使用 Register，这种做法合理吗？ 请给出分析，若有改进意见也请一并给出。**
   答：合理
   IM 使用 ROM：IM 存储指令代码和常量数据，ROM 能够提供非易失性存储和快速的读取速度，因此适合用于存储指令和常量数据但是，如果需要修改指令或者增加新的指令，需要重新烧录或更换 ROM 芯片，不够灵活。
   DM 使用 RAM：DM 存储程序运行时的数据，RAM 提供了易写易读的特性，适合作为数据存储器。但是，RAM 是易失性存储，断电后数据会丢失。此外，RAM 的速度相对较慢，访问时间比寄存器长。改进的方法可以考虑使用带备份电池的非易失性存储器，比如 SRAM 或者 FRAM，能够保留数据并提供较快的访问速度。
   GRF 使用 Register：GRF 存储寄存器的值，这些寄存器直接用于 CPU 运算操作。寄存器提供了最快的访问速度和多端口的并行读写能力。
  **3. 在上述提示的模块之外，你是否在实际实现时设计了其他的模块？如果是的话，请给出介绍和设计的思路。**
  答：暂时没有设计其他的模块
  **4. 事实上，实现 nop 空指令，我们并不需要将它加入控制信号真值表，为什么？**
  答：nop在CPU运行周期内不做任何行为，因此将所有控制信号置低电平即可确保CPU的各个部件不受任何影响，相当于执行了一个没有行为的周期，也即执行了nop空指令。
  **5. 阅读 Pre 的 “MIPS 指令集及汇编语言” 一节中给出的测试样例，评价其强度（可从各个指令的覆盖情况，单一指令各种行为的覆盖情况等方面分析），并指出具体的不足之处。**
  <img src="image-9.png" alt="Alt text" style="zoom:50%;" />
  答： 该测试指令强度较弱
   1.同一种指令位置过于集中，应该将不同指令穿插运行
   2.从单一指令的覆盖情况来看，指令行为比较一致，同时也没有进行不同类型数据的测试以及边缘数据的测试。
   3.同时指令总数较少，不一定可以反应出cpu的问题所在



  





  

  

