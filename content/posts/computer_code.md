+++
title = '《原码,反码,补码》笔记'
date = '2015-03-15T13=47:58+08:00'
tags = ["Coding", "IT"]
+++


大一第一学期学的《C 语言程序设计》的时候就接触到原码，反码和补码了，当时也没有认真听，只是稍微了解了一下，并没有留下太多记忆深刻的东西，这学期的专业课《数字电子技术基础》又讲到了，便把内容整理了出来，方便自己和大家理解

#### 机器数和真值

1、机器数

一个数在计算机中的二进制表示形式, 叫做这个数的机器数。

机器数是带符号的，在计算机用一个数的最高位存放符号, 正数为 0, 负数为 1.

> 比如，十进制中的数 +3 ，计算机字长为 8 位，转换成二进制就是 00000011。如果是 -3 ，就是 10000011 。

> 那么，这里的 00000011 和 10000011 就是机器数。

2、真值

因为第一位是符号位，所以机器数的形式值就不等于真正的数值。例如上面的有符号数 10000011，其最高位 1 代表负，其真正数值是 -3 而不是形式值 131（10000011 转换成十进制等于 131）。所以，为区别起见，将带符号位的机器数对应的真正数值称为机器数的真值。

> 例：0000 0001 的真值 = +000 0001 = +1，1000 0001 的真值 = –000 0001 = –1

I

#### 原码, 反码, 补码的基础概念和计算方法.

1. 原码

原码就是符号位加上真值的绝对值, 即用第一位表示符号, 其余位表示值. 比如如果是 8 位二进制:

> [+1]原 = 0000 0001

> [-1]原 = 1000 0001

第一位是符号位. 因为第一位是符号位, 所以 8 位二进制数的取值范围就是:

[1111 1111 , 0111 1111]

即

[-127 , 127]

原码是人脑最容易理解和计算的表示方式.

2. 反码

反码的表示方法是:

正数的反码是其本身

负数的反码是在其原码的基础上, 符号位不变，其余各个位取反.

> [+1] = [00000001]原 = [00000001]反

> [-1] = [10000001]原 = [11111110]反

可见如果一个反码表示的是负数, 人脑无法直观的看出来它的数值. 通常要将其转换成原码再计算.

3. 补码

补码的表示方法是:

正数的补码就是其本身

负数的补码是在其原码的基础上, 符号位不变, 其余各位取反, 最后+1. (即在反码的基础上+1)

> [+1] = [00000001]原 = [00000001]反 = [00000001]补

> [-1] = [10000001]原 = [11111110]反 = [11111111]补

对于负数, 补码表示方式也是人脑无法直观看出其数值的. 通常也需要转换成原码在计算其数值.

updated @3.15
