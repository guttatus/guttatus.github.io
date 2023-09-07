+++
author = "guttatus"
title = "异步FIFO"
date = "2023-09-06"
description = "浅谈异步FIFO的原理和实现"
tags = [
    "CDC",
    "FIFO",
]
categories = [
    "IC",
]
toc = true
autonumbering = true
+++

## 异步FIFO

异步FIFO是指：对于一个FIFO,其数据从一个时钟域写入FIFO缓冲区，而数据在另一个时钟域从FIFO缓冲区读出，其中这两个时钟是异步的。异步FIFO被用于安全地把数据从一个时钟域同步到另一个时钟域。

异步FIFO设计的困难之处在于生成FIFO指针并找到可靠的方式来确定FIFO的空满状态。

为了理解异步FIFO的设计，我们需要清楚FIFO的指针是如何工作的。FIFO的写指针（write pointer）总是指向下一个字被写入的的地址。当复位时，读写指针都被设为0,这也是FIFO下一个字被写入的地址。在FIFO写入操作中，数据被写入指针指向的内存位置，然后写入指针递增以指向下一个要写入的位置。相似的，FIFO读指针总是指向当前读的位置。在复位时，指针被置0，此时FIFO为空且读指针指向无效数据（因为FIFO为空且空标志已置位）。一旦第一个数据字写入 FIFO，写指针就会递增，空标志被清除，读指针仍在寻址第一个FIFO存储器字的内容，并立即将第一个有效字驱动到FIFO数据输出端口，由接收器逻辑读取。如果接收器不得不在读FIFO之前增加读指针，接收器将要等待一个时钟周期以从FIFO输出数据字，并等待第二个时钟周期以将数据字捕获到接收器中。这将造成不必要的低效。

当读和写指针相等时，FIFO为空。当两个指针在复位操作期间都复位为0时，或者当读指针赶上写指针并从FIFO读取最后一个字时，就会发生这种情况。

当指针再次相等时，即当写指针回绕并追上读指针时，FIFO已满。问题在于，当指针相等时，FIFO要么为空，要么为满，但是其中哪个呢?

用于区分空满的一种方法是，对读写只能都增加一位额外的高bit。当写指针跨过FIFO的最高位地址，写指针的最高bit（MSB）加一，其他bit置0。读指针也是采用同样的方式。如果两个指针的MSB不同，则意味着写指针比读指针多绕行一次。如果两个指针的MSB相同，则意味着两个指针绕行的次数相同。使用n位指针，其中n-1是访问整个FIFO内存缓冲区所需的地址位数，当两个指针（包括 MSB）相等时，FIFO 为空。当两个指针（除了MSB）相等（MSB不同）时，FIFO 已满。

### 关键技术 - 读写信号跨时钟域同步
首先，FIFO的关键是需要判断读空和写满，而这两个信号的产生依赖读地址和写地址。在异步FIFO中，读和写是分在两个时钟域中的，在写时钟域，需要得到读地址的信息进而判断是否写满（写指针是否追上读指针），同理，在读时钟域，也需要写地址的信息。我们知道跨时钟域的单比特数据一般可以用双寄存器法进行同步，但读写地址通常都是多比特的信号，此时如何进行同步呢？

当然，多比特的同步肯定可以通过增加握手信号（[多周期路径规划](https://guttatus.github.io/post/cdc-2/#%E5%A4%9A%E5%91%A8%E6%9C%9F%E8%B7%AF%E5%BE%84%E8%A7%84%E5%88%92)）来解决。但实际上对于数值上连续增加的信号，可以采用[格雷码](https://guttatus.github.io/post/gray-code/)进行多比特到单比特的传输。有了格雷码，就可以将读写地址简单地通过[双触发同步器](https://guttatus.github.io/post/cdc-1/#%E5%8F%8C%E8%A7%A6%E5%8F%91%E5%90%8C%E6%AD%A5%E5%99%A8)同步到对应的时钟域了。

### 异步FIFO原理图（样式1）
下图展示了样式1的异步FIFO的<cite>原理图[^1]</cite>。

![async_fifo_style_1](/img/posts/async-fifo/async_fifo_style_1.png)

上述设计可以分为六个具有以下功能和时钟域的模块：  
- `async_fifo.v` : 顶层模块包含所有时钟域。仅用作包装来实例化设计中使用的所有其他FIFO模块。
- `fifomem.v` : 由写入和读取时钟域访问的FIFO内存缓冲区。该缓冲区较为常见的是实例化的同步双端口RAM。其他存储器类型也可以用作FIFO缓冲区。
- `sync_r2w.v` : 同步器模块，用于将读指针同步到写时钟域。`wptr_full`模块将使用同步的读指针来生成FIFO写满状态。该模块仅包含与写入时钟同步的触发器。该模块中不包含其他逻辑。
- `sync_w2r.v` : 同步器模块，用于将写指针同步到读时钟域。`rptr_empty`模块将使用同步写指针来生成FIFO读空条件。该模块仅包含与读时钟同步的触发器。该模块中不包含其他逻辑。
- `rptr_empty.v` : 该模块与读时钟域完全同步，包含产生FIFO读指针和空标志的逻辑。
- `wptr_full.v` : 该模块与写时钟域完全同步，包含产生FIFO写指针和满标志的逻辑。

#### RTL 实现
异步FIFO（样式1）的RTL实现在[guttatus/async_fifo/style1](https://github.com/guttatus/verilog-examples/tree/main/FIFO/async_fifo/style1)中给出，这里只对代码中的关键部分进行阐述。

##### 读空信号的产生
读空信号的产生比较简单，需要比读指针`rgraynext`和同步到读时钟域的写指针`rq2_wptr`的值是否一致即可。下面给出Verilog描述。
``` Verilogtu
  
always @(posedge rclk or negedge rrst_n) begin
  if(!rrst_n) begin
    rempty <= 1'b1;
  end
  else begin
    rempty <= rempty_val;
  end
end
```

##### 写满信号的产生
写满信号相较于读空信号的复杂之处在于写慢信号的读写指针绕行了不同的圈数。从上文我们可以知道，在二进制编码的情况下，我们只需要比较读写指针的额外的最高位(MSB),即可确定读写指针绕行圈数是否一致z。但是，在格雷码编码下，情况略有不同。

下图给出了地址空间为3bits,对应8深度FIFO的读写指针的4bits格雷码编码。如果FIFO写入前七个位置（0-6），然后通过读相同的七个位置来清空FIFO，则两个指针将相等并指向地址`Gray-7`（FIFO 为空）。在下一次写操作中，写指针将递增4位格雷码指针，从而使写指针的MSB与读指针的MSB不同，但是写指针的其余部分不与读指针的其余位置完全匹配，因此如果按照上文中二进制编码的比较方式，FIFO满标志将被置位。这是错误的！

![gray8](/img/posts/async-fifo/gray8.png)

对于格雷码编码的读写指针，对于写满的判断条件应为：

1. `wptr`和同步的`wq2_rptr`的MSB 不相等。
2. `wptr`和同步的`wq2_rptr`的 <u>2nd</u> MSB 不相等。
3. `wptr` 和同步 `wq2_rptr` 其他位必须相同。

下面给出Verilog描述。

``` Verilog
/*************************************************
 * assign wfull_val = (wgraynext[ASIZE]     != wq2_rptr[ASIZE]   &&
 *                     wgraynext[ASIZE-1]   != wq2_rptr[ASIZE-1] &&
 *                     wgraynext[ASIZE-2:0] != wq2_rptr[ASIZE-2:0]);
 ************************************************/
assign wfull_val = (wgraynext == {~wq2_rptr[ASIZE:ASIZE-1],wq2_rptr[ASIZE-2:0]});

always @(posedge wclk or negedge wrst_n) begin
  if(!wrst_n) begin
    wfull <= 0;
  end
  else begin
    wfull <= wfull_val;
  end
end
```

#### 仿真结果

- 读慢写快
![readslowwritefast](/img/posts/async-fifo/readslowwritefast.png)

- 读快写慢
![readfastwriteslow](/img/posts/async-fifo/readfastwriteslow.png)

### 异步FIFO原理图（样式2）
下图展示了样式2的异步FIFO<cite>原理图[^2]</cite>。与样式1不同之处在于样式2在格雷码指针之间进行异步比较，以生成异步控制信号来设置和重置空触发器和满触发器。

![async_fifo_style_2](/img/posts/async-fifo/async_fifo_style_2.png)

上述设计可以分为五个具有以下功能和时钟域的模块：
- `async_fifo.v` : 顶层模块包含所有时钟域。仅用作包装来实例化设计中使用的所有其他FIFO模块。
- `fifomem.v` : 由写入和读取时钟域访问的FIFO内存缓冲区。该缓冲区较为常见的是实例化的同步双端口RAM。其他存储器类型也可以用作FIFO缓冲区。
- `async_cmp.v` : 异步指针比较模块，用于生成控制异步"满"和"空"状态位的断言的信号。该模块仅包含组合比较逻辑。该模块中不包含时序逻辑。
- `rptr_empty.v` : 该模块与读时钟域完全同步，包含产生FIFO读指针和空标志的逻辑。
- `wptr_full.v` : 该模块与写时钟域完全同步，包含产生FIFO写指针和满标志的逻辑。

#### 异步比较器实现细节
异步比较器电路原理图如下图所示。

![direction](/img/posts/async-fifo/direction.png)

在用异步比较器产生FIFO空满信号时，无需添加额外的最高比特。而是将地址空间分为四个象限，并对两个计数器的两个MSB进行解码，以确定在两个指针相等时FIFO是已满还是已空。

如果写指针位于读指针后面一个象限，则表示"可能已满"的情况，如下图所示。当这种情况发生时，`direction`锁存器被置位。

![full](/img/posts/async-fifo/full.png)

如果写指针比读指针领先一个象限，则表示"可能变空"的情况，如下图所示。当这种情况发生时，`direction`锁存器被清除。

![empty](/img/posts/async-fifo/empty.png)

当FIFO复位时，`direction`锁存器也会被清除，以指示FIFO"即将变空"（实际上，当两个指针都复位时，它是空的）。设置和重置`direction`锁存器不是时序关键的，并且方向锁存器消除了地址一样时解码器的歧义。

更困难的问题源于写入和读取时钟的异步特性。当其中一个或两个计数器或多或少同时改变多个位时，比较两个异步计时的计数器可能会导致不可靠的解码尖峰。这里的解决方案是使用格雷码编码，其中从任何计数到下一个计数只有一位发生变化。然后，任何解码器或比较器将仅从一个有效输出切换到下一个有效输出，而不存在虚假解码毛刺的危险。

#### RTL实现
异步 FIFO（样式 2）的RTL实现在[guttatus/async_fifo/style2](https://github.com/guttatus/verilog-examples/tree/main/FIFO/async_fifo/style2) 中给出。

[^1]: Clifford E. Cummings, "Simulation and synthesis techniques for asynchronous FIFO design." SNUG 2002 (Synopsys Users Group Conference, San Jose, CA, 2002) User Papers. Vol. 281. 2002.
[^2]: Clifford E. Cummings and Peter Alfke, "Simulation and Synthesis Techniques for Asynchronous FIFO Design with Asynchronous Pointer Comparisons." SNUG 2002 (Synopsys Users Group Conference, San Jose, CA, 2002) User Papers,
March 2002, Section TB2, 3rd paper.
