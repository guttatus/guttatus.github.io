+++
author = "guttatus"
title = "格雷码"
date = "2023-08-20"
description = "数字电路中的编码"
tags = [
    "Encoding",
]
categories = [
    "IC",
]
toc = true
+++

## 格雷码

格雷码（Gray code）是一个数列集合，相邻两数间只有一个位元改变，为无权数码，且格雷码的顺序不是唯一的。

传统的二进制系统例如数字3的表示法为011，要切换为邻近的数字4，也就是100时，装置中的三个位元都得要转换，因此于未完全转换的过程时装置会经历短暂的，010,001,101,110,111等其中数种状态，也就是代表着2、1、5、6、7，因此此种数字编码方法于邻近数字转换时有比较大的误差可能范围。格雷码的发明即是用来将误差之可能性缩减至最小，编码的方式定义为**每个邻近数字都只相差一个位元**，因此也称为**最小差异码**，可以使装置做数字步进时只更动最少的位元数以提高稳定性。

数字0～15的格雷码编码如下：

| Decimal | Binary | Gray |
| :-----: | :----: | :--: |
|    0    |  0000  | 0000 |
|    1    |  0001  | 0001 |
|    2    |  0010  | 0011 |
|    3    |  0011  | 0010 |
|    4    |  0100  | 0110 |
|    5    |  0101  | 0111 |
|    6    |  0110  | 0101 |
|    7    |  0111  | 0100 |
|    8    |  1000  | 1100 |
|    9    |  1001  | 1101 |
|   10    |  1010  | 1111 |
|   11    |  1011  | 1110 |
|   12    |  1100  | 1010 |
|   13    |  1101  | 1011 |
|   14    |  1110  | 1001 |
|   15    |  1111  | 1000 |

### 二进制数据转格雷码公式  
$G：$格雷码   
$B：$二进制码   
$n：$正在计算的位  

$$ G(n) = B(n+1) \oplus B(n) $$

使用verilog描述如下：
``` verilog
module bin2gray #(
   parameter WIDTH=10
) (
  input  [WIDTH-1:0] bin_i,
  output [WIDTH-1:0] gray_o
);
  assign gray_o = bin_i ^ (bin_i>>1);
 
endmodule
```

### 格雷码转二进制数据公式  
$G：$格雷码   
$B：$二进制码   
$n：$正在计算的位  

$$ B(n) = B(n+1) \oplus G(n) $$

使用verilog描述如下：
``` verilog
module gray2bin #(
  parameter WIDTH=10
) (
  input  [WIDTH-1:0] gray_i,
  output [WIDTH-1:0] bin_o
);
  /*
  	assign bin_o[0] = gray_i[3] ^ gray_i[2] ^ gray_i[1] ^ gray_i[0];
  	assign bin_o[1] = gray_i[3] ^ gray_i[2] ^ gray_i[1];
  	assign bin_o[2] = gray_i[3] ^ gray_i[2];
  	assign bin_o[3] = gray_i[3];
  */
  genvar i;
  generate
    for(i=0;i<WIDTH;i++) begin
      assign bin_o[i] = ^(gray_i >> i);
    end
  endgenerate
endmodule
```