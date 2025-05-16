---
title: Compiler Flags for Stack Protection
date: 2025-03-11T09:46:48Z
lastmod: 2025-05-12T17:52:07Z
categories: ["Compiler"]
tags: ["Compiler", "C/C++", "OS"]
---

https://developers.redhat.com/articles/2022/06/02/use-compiler-flags-stack-protection-gcc-and-clang#

## Stack Canary

Clang和gcc使用许多机制来保护栈, 最典型的就是stack canary.
Stack canary是最初级的侦察栈溢出的方法. 其原理是在运行时的栈帧中插入额外的内存并设置为特定值, 这个值不能被攻击者得知. 在函数跳转的时候会检查这个值是否被改变并以此判断栈是否被破坏.
Stack canary(在gcc和clang中)可以通过以下编译选项控制

- `-fstack-protector`
- `-fstack-protector-strong`
- `-fstack-protector-all`
- `-fstack-protector-explicit`
  Inserts a guard variable onto the stack frame for each vulnerable function or for all functions.
  以下是一个对比示例:

```c
int f1(int a)
{
	return a;
}

void f2(int a)
{
	return ;
}

int main()
{
	f1(597);

	return 0;
}
```

编译:

```
gcc t.c -o unknow -O0
gcc t.c -o protection -fstack-protector-all -O0
gcc t.c -o no_protection -fno-stack-protector -O0
```

最后对比可以发现默认状态下是没有开启stack canary的.
x86-64架构下开启后对比:
![](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/Pasted%20image%2020241223150716-20250311094648-ln8eicx.png)
可以看到每次的函数调用都会在进入函数的时候多使用一块内存, 并在函数返回前调用`__stack_chk_fail`检查这块内存.
内核中的使用: https://lwn.net/Articles/584225/
更多信息可见[arm doc](https://developer.arm.com/documentation/dui0774/l/Compiler-Command-line-Options/-fstack-protector---fstack-protector-all---fstack-protector-strong---fno-stack-protector) 和 https://chatgpt.com/share/677202d2-4d58-8009-b1cf-c6eafb3d4b08

## shadow stack and safestack

此技术将stack分为两个区域, 用不连续的内存区域存储用户变量和过去的变量.
GCC和Clang使用:

- `-mshstk`
  增加shadow stack
  Clang使用
- `-fsanitize=safe-stack`
  增加safestack
  [gcc doc](https://gcc.gnu.org/onlinedocs/gcc/x86-Options.html#index-mshstk)
  [llvm doc](https://clang.llvm.org/docs/SafeStack.html)

## fortified source

https://www.redhat.com/en/blog/security-technologies-fortifysource
`FORTIFY_SOURCE` 提供轻量的编译和运行时内存保护, 可以适用于操作系统中的所有库和应用.
在gcc中, FORTIFY_SOURCE一般是通过用内置的`*_chk`替换一些字符串和内存函数来检测溢出(如将`memcpy`替换为`__memcpy_chk`), 这些函数会通过做一些必要的计算来检测溢出. 如果发现溢出, 程序将会终止. `FORTIFY_SOURCE`一般在汇编层面增加指令, 因此开销很小.
gcc和clang都使用

- `-D_FORTIFY_SOURCE=<n>`
  来使用fortified source. `<n>`越高, 越能提供更强的保护, 但是相应的性能也会下降, `<n>`最高为3.

## Control flow integrity

https://www.intel.com/content/www/us/en/developer/articles/technical/technical-look-control-flow-enforcement-technology.html
https://techcommunity.microsoft.com/blog/windowsosplatform/understanding-hardware-enforced-stack-protection/1247815
函数返回时可能会出现破坏栈来控制程序的跳转(这里是指执行非正常情况的指令, 如使用`jmp`指令去到意想不到的地方). 一个应对方法是使用硬件/软件支持保证跳转/返回地址正确.
GCC和clang可以使用:

- `-fcf-protection=[full]`
  生成Intel CET([Control-flow Enforcement Technology](https://www.intel.com/content/www/us/en/developer/articles/technical/technical-look-control-flow-enforcement-technology.html))的机器码.
  Gcc和clang也可以使用:
- `-mbranch-protection=none|bti|pac-ret+leaf|pac-ret[+leaf+b-key]|standard`
  生成支持aarch64 BTI([Branch Target Identification](https://developer.arm.com/documentation/ddi0596/2020-12/Base-Instructions/BTI--Branch-Target-Identification-))的机器码.
  以上的两个参数都需要硬件支持, clang提供了纯软件的实现, 可以使用:
- `-fsanitize=cfi`

## Stack allocation control

以下选项是一些对stack allocation有影响的选项(这不一定会提供额外的安全性保证, 当其副作用有可能对栈安全有影响).
在x86平台, gcc和clang可以使用

- `-fsplit-stack`
  在stack内存用完后, 自动分配一个不连续的栈.
  在使用GNU linker(`ld`)时, 可以将stack limit通过symbol或register告诉程序.
- `-fstack-limit-register`
- `-fstack-limit-symbol`
  Clang可以使用:
- -fno-stack-array
  禁止在stack中使用array.

## Stack usage and statistics

栈的大多数操作是存储调用信息和固定大小的本地变量, 但也会出现使用VLA或使用`alloca()`分配动态大小的对象的情况. 这经常是栈溢出或是与其他部分内存冲突的来源.
以下选项可以对`alloca()`和VLA的使用给出warning:

- 通用:
  - `-Wframe-larger-than`
  - `-Walloca`
  - `-Walloca-larger-than`
  - `-Wvla -Wvla-larger-than`
- GCC:
  - `-Wstack-usage`
  - `-fstack-usage`
  - `-Walloca`
  - `-Walloca-larger-than`
  - `-Wvla`
  - `-Wvla-larger-than`
  - `-Wframe-larger-than`
  - `-mwarn-dynamicstack` (s390x)
  - `-Wstack-protector`
  - `-Wtrampolines`
  - `-Winfinite-recursion`
- Clang:
  - `-fstack-usage`
  - `-Walloca`
  - `-Wframe-larger-than`
  - `-Wstack-usage`

## 扩展

https://www.redhat.com/en/blog/security-technologies-stack-smashing-protection-stackguard
https://www.redhat.com/en/blog/security-technologies-execshield

## 参考

https://developer.arm.com/documentation/dui0774/l/Compiler-Command-line-Options/-fstack-protector---fstack-protector-all---fstack-protector-strong---fno-stack-protector
https://developers.redhat.com/articles/2022/06/02/use-compiler-flags-stack-protection-gcc-and-clang#
https://lwn.net/Articles/584225/
https://www.redhat.com/en/blog/security-technologies-fortifysource

‍
