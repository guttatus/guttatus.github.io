+++
author = "guttatus"
title = "跨过那个时钟域(2)"
date = "2023-08-23"
description = "浅谈数字电路设计中多比特信号跨时钟域的同步处理方法"
tags = [
    "CDC",
]
categories = [
    "IC",
]
toc = true
autonumbering = true
+++

## 多比特信号跨时钟域的同步处理方法
当在时钟域间传递多比特信号时，简单的同步器无法保证数据被安全地传递。多比特跨时钟域的主要问题在与：在将多比特信号从一个时钟域同步到另一个时钟域时，可能出现偶发的数据变化偏斜（data changing skew），这种偏斜最终在第二个时钟域的不同上升沿被采样。即使能完美地控制和匹配这些多比特信号的走线长度，上升和下降时间的差异以及整个模具的工艺变化都可能会引入足够的偏斜，从而导致在精心匹配的多条信号上采样失败。

为了避免多比特信号跨时钟域采样偏斜，主要有三类同步策略：
1. 多比特信号融合策略：尽可能地将多比特信号跨时钟域融合为单比特信号跨时钟域。
2. 多周期路径规划策略：时钟同步加载信号（synchronized load signal）安全地传递多比特跨时钟域信号。
3. 使用[格雷码](https://guttatus.github.io/post/gray-code/)传递多比特跨时钟域信号。

   
### 多比特融合策略
尽可能地多比特信号跨时钟域融合为单比特信号跨时钟域。当在设计中遇到多比特跨时钟域时，我们应该考虑清楚是否真的需要多比特信号来控制逻辑跨过时钟域边界。

下面将通过一些简单的实例来说明为什么简单的使用同步器来同步多比特跨时钟域信号不是好的解决方案。如果控制信号的顺序或排列对设计来说非常重要，那么设计者必须小心地将这些信号同步到接收时钟域。

#### 实例1 -- 两个同时需要的控制信号
如下图所示，在接收时钟域中有一个寄存器同时需要*load*信号和*enable*信号来将数据写入寄存器。如果*load*信号和*enable*信号在发送时钟域的同一时钟上升沿被驱动有效（即两个信号需要同时有效），那么这两个控制信号之间如果存在一点微小的偏斜，就有可能在接收时钟域中被同步到不同的时钟周期。在这种情况下，数据将不会被写入寄存器。

![Passing multiple control signals between clock domains](/img/posts/cdc/enableload.png)

针对这一问题，解决方案是将这两个控制信号合并为一个控制信号。如下图所示，在接收时钟域仅由一个使能信号驱动*load*和*enable*输入信号。将两个信号合并将消除两个控制信号到达时间发生偏移的可能性。

![Consolidating control signals before passing between clock domains](/img/posts/cdc/onesingle.png)

#### 实例2 -- 两个相移定序控制信号
如下图所示，有两个使能信号*ld1*和*ld2*，它们依次从发送时钟域驱动到接收时钟域，以控制两级流水寄存器寄存数据。问题在于，在发送时钟域中，b_ld1信号正好稍微在b_ld2信号有效前结束有效，这就导致在接收时钟域上升沿采集b_ld1和b_ld2脉冲信号时产生一个细微的间隙。在同步后，两个使能控制信号间隔了两个时钟周期，而不是一个时钟周期，导致在接收时钟域的使能控制信号链中形成一个单周期间隙。这样就导致a2没有被及时加载到第二个寄存器。这里需要注意的是，如果a2能保持一直有效，在经过一个时钟周期间隔后，a2仍能被加载到第二个寄存器中，但是这里要求严格的流失操作，所以问题就出现了。如果a2的数据不能保持一直有效，那么就造成a2的数据丢失。

![Passing sequential control signals between clock domains](/img/posts/cdc/phaseshift.png)

针对这一问题，解决方案是，只发送一个控制信号到接收时钟域，在接收时钟域生成第二个寄存器的使能信号。

![Logic to generate proper sequencing signals in the new clock domains](/img/posts/cdc/generatesignal.png)

### 实例3 -- 两个无法合并的多比特跨时钟域信号
如下图所示，两个编码控制信号在时钟域间被传递。如果两个编码控制信号在采样时有细微的偏斜，在接收时钟域的一个时钟周期将产生错误的解码输出。

![Encoded control signals passed between clock domains](/img/posts/cdc/encode.png)

多周期路径（Multi-Cycle Path,MCP）规划和FIFO技术可以用来解决多比特信号跨时钟域问题。

这里有至少两种多周期路径规划方法可以用于解决这一问题：

1. 闭环 -- 带反馈的多周期路径规划
2. 闭环 -- 带确认反馈的多周期路径规划

这里同样至少有两种FIFO策略可以用于解决这一问题：

1. 异步FIFO
2. 二深度FIFO

### 多周期路径规划
多周期路径规划是指在**传输非同步数据到接收时钟域时配上一个同步的控制信号**，数据和控制信号被同时发送到接收时钟域，同时控制信号在接收时钟域使用两级寄存器同步到接收时钟域，使用此同步后的控制信号来加载数据。使用这种技术有以下两个好处：

- 在时钟域之间发送数据时在发送时钟域无需计算对应脉冲宽度。
- 发送时钟域只需要将使能切换到接收时钟域，以表明数据已经通过并准备加载。使能信号不需要返回到其初始逻辑电平。

使用这种策略在时钟域间传递多比特信号时，数据无需同步，只需同步使能信号到接收时钟域中。接收时钟域不允许采样多比特数据直到同步使能信号经过同步器同步到达接收时钟域。

这种策略被称为多周期路径规划的原因是：非同步数据直接被传递到接收时钟域并保持多个接收时钟周期，只有一个使能信号被同步到接收时钟域，并且只有在使能信号被识别后才允许非同步数据发生变化。

由于非同步数据在采样前经过多个时钟周期的传递并保持稳定，因此不存在采样值亚稳态的风险。

下图展示了一种常见的跨时钟域使能控制信号处理方法。将使能信号经过同步器同步后传递给一个同步脉冲发生器，然后产生一个指示信号用于指示非同步多周期数据，即可在下一个接收时钟上升沿进行采集。使用这种方法的关键之处在于，产生同步脉冲时，输入的控制信号的极性并不重要。从图中我们可以发现，只要输入信号d的极性发生改变，都能产生同步使能脉冲信号。所以**输入信号的每次切换对应一次数据的变化**。

![Synchronized pulse generation logic](/img/posts/cdc/genpulse.png)

值得一提的是，在单时钟域应用设计时，也经常用到类似的技术来产生数据有效信号，下图给出了同步使能脉冲产生电路的等效符号。

![Synchronized enable pulse generation logic and equivalent symbol](/img/posts/cdc/equsym.png)

除了产生脉冲p外，同步使能脉冲产生电路还产生了比d输入延迟三个时钟周期的q输出。q输出经常被用作反馈信号，并通过发送时钟域中的另一个同步使能脉冲产生电路作为确认信号传递。

#### 脉冲同步法（开环的结绳法）
下图给出了一种典型的脉冲同步法（开环的结绳法）设计。

![Multi-Cycle Path (MCP ) formulation toggle-pulse generation](/img/posts/cdc/togpls.png)

脉冲同步器的工作方式如下：
- 在发送时钟域产生一组单周期脉冲，其间隔至少需要比接收时钟域时钟域的**两个时钟周期**大，否则接收时钟域中无法进行边沿检测。同时这组脉冲也表示了发送时钟域中数据传送的开始和结束。
- 在发送时钟域中，两个脉冲被电平翻转器（由异或门构成）将脉冲之间的区域变为一段高电平（结绳toggle)
- 发送时钟域中的使能信号在接收时钟域中通过同步使能脉冲产生电路产生两个单周期脉冲
- 接收时钟域中需要监测单周期脉冲并进行信号的采样

[https://github.com/guttatus/verilog-examples/tree/main/CDC/toggle-pulse](https://github.com/guttatus/verilog-examples/tree/main/CDC/toggle-pulse)给出了脉冲同步法的一种实现。其仿真结果如下图所示。

![togpls_wave](/img/posts/cdc/togpls_wave.png)
