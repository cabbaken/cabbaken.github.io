---
title: Introduce to C Function Call (Calling Convention)
date: 2025-05-06T14:59:09Z
lastmod: 2025-05-16T17:38:26Z
tags: ["OS", "C/C++", "ABI"]
categories: ["Operating System"]
---

# Introduce to C Function Call

## Vocabulary Interpretion

### ABI (Application Binary Interface)

[ABI](https://en.wikipedia.org/wiki/Application_binary_interface) defines how data structures and computational routines are accessed in machine code.  
It basically includes:

1. ISA (Instruction Set Architecture), with details like registers, stack organization, memory access type, etc.
2. Size, layout, and alignments of basic data types(like int, double, char, etc.).
3. Calling conventions (interpreted below)
4. System call
5. Binary format of object file, excutable file,and libraries.

The compiler, OS and library work with a user program with [ABI compatibility](https://stackoverflow.com/questions/72074771/do-static-libraries-behave-like-dynamic-libraries-in-terms-of-abi-compatibility).

### Calling Convention

Calling conventions define how function calls are made at assembly level.
It includes:

1. How parameters are passed to functions
2. How the stack work when functions are called
3. How registers are used
4. How the return value retrieved

## Basic function call

If you want to call a function, you should define function, optionally declare function, and then make function call. It's pretty simple even for a beginner of C language. Let's take an example:

```c++
int f1(int a,int b)
{
	int c=a+b;
	return c;
}
int main()
{
	int a=3,b=2,c;
	c=f1(a,b);

	return 0;
}
```

## Detail

You can make function calls with just one line code `c=f1(a,b)`​. But what happen behind this line? How the parameters transfer? How return value transfer?
I want to dive into the details. So I decide to read the assembly code. Let's read it with `objdump`​.

```assembly
0000000000401110 <f1>:
  401110:       55                      push   %rbp
  401111:       48 89 e5                mov    %rsp,%rbp
  401114:       89 7d fc                mov    %edi,-0x4(%rbp)          ; save the first parameter to stack
  401117:       89 75 f8                mov    %esi,-0x8(%rbp)          ; save the second ...
  40111a:       8b 45 fc                mov    -0x4(%rbp),%eax          ; do the calculate
  40111d:       03 45 f8                add    -0x8(%rbp),%eax
  401120:       89 45 f4                mov    %eax,-0xc(%rbp)          ; save calculate result to stack
  401123:       8b 45 f4                mov    -0xc(%rbp),%eax          ; return value stored in eax
  401126:       5d                      pop    %rbp
  401127:       c3                      ret    
  401128:       0f 1f 84 00 00 00 00    nopl   0x0(%rax,%rax,1)
  40112f:       00 

0000000000401130 <main>:
  401130:       55                      push   %rbp
  401131:       48 89 e5                mov    %rsp,%rbp
  401134:       48 83 ec 10             sub    $0x10,%rsp
  401138:       c7 45 fc 00 00 00 00    movl   $0x0,-0x4(%rbp)          ; 0x4:c
  40113f:       c7 45 f8 03 00 00 00    movl   $0x3,-0x8(%rbp)          ; 0x8:a
  401146:       c7 45 f4 02 00 00 00    movl   $0x2,-0xc(%rbp)          ; 0xc:b
  40114d:       8b 7d f8                mov    -0x8(%rbp),%edi          ; first parameter in edi
  401150:       8b 75 f4                mov    -0xc(%rbp),%esi          ; second parameter in esi
  401153:       e8 b8 ff ff ff          call   401110 <f1>              ; call function
  401158:       89 45 f0                mov    %eax,-0x10(%rbp)
  40115b:       31 c0                   xor    %eax,%eax
  40115d:       48 83 c4 10             add    $0x10,%rsp
  401161:       5d                      pop    %rbp
  401162:       c3                      ret    
```

We can analyse the assembly code line by line. We can easily find out the assignment of a, b, c happens at 401146, so, theoretically, the next line is the beginning of the function call.
I believe you already know some basic assembly code. So I will explain these code briefly.

* ​`mov	-0x8(%rbp),%edi`​: move value in `0x8($rbp)`​ to `$edi`​
* ​`mov	-0xc(%rbp),%esi`​: move value in `0xc($rbp)`​ to `$esi`​

I guess this is the code about parameter transfering because the next line is obvilously calling `f1`​.  So, I got some questions.

1. Why use `edi`​ and `esi`​ to transfer parameters?
2. Can I use the stack or use other registers?
3. Who defined it?

It's obvious that the compiler generate the code, so we need to ask the compiler why it do like this. We can easily find the notion of "[Calling Convention](https://wiki.osdev.org/Calling_Conventions)", you can read the simple introduction [here](#calling-convention). We know why the compiler choose to do like above by read the [docs](https://gcc.gnu.org/onlinedocs/gcc-11.2.0/gcc/x86-Function-Attributes.html#x86-Function-Attributes):

> On 32-bit and 64-bit x86 targets, you can use an ABI attribute to indicate which calling convention should be used for a function. The `ms_abi`​ attribute tells the compiler to use the Microsoft ABI, while the `sysv_abi`​ attribute tells the compiler to use the System V ELF ABI, which is used on GNU/Linux and other systems. The default is to use the Microsoft ABI when targeting Windows. On all other systems, the default is the System V ELF ABI.

Now, we know the reason of the compiler's logic. And it's obviously that we can transfer parameters by other way. For example, you can edit the assembly code and then compile it to achieve that. We can get the relative code by

```bash
gcc -S [filename]
```

then we can edit the code like this. It can pass the compiler and run nomorally.

```
14,15c14,15
<       movl    %edi, -20(%rbp)
<       movl    %esi, -24(%rbp)
---
>       movl    %r8d, -20(%rbp)
>       movl    %r9d, -24(%rbp)
18c18
<       addl    %edx, %eax
---
>       addl    %edx,%eax
43,44c43,44
<       movl    %edx, %esi
<       movl    %eax, %edi
---
>       movl    %edx, %r8d
>       movl    %eax, %r9d
```

I just change some registers and the code runs successful. Is this proving that I can casually use the stack and registers to pass parameter or retrive return value?
The answer is NO! Imagine a scenario like "I want to call a function from a library or system call", you will find that the program may not work properly.
But why? we can see that if we change the registers that carry the parameters, the function won't know which register it should read if we did't edit the function! That is definitely a disaster.
To avoid this disaster, the [ABI](#abi-application-binary-interface) was born. The compiler should implement the standard that ABI provides to achieve the "program compatibility" which makes the libraries, system call, and user-programs work together properly.
I won't introduce the details of ABI to you here. You can read the docs of the ABI. The mainly used ABI are [Microsoft x64 ABI](https://learn.microsoft.com/en-us/cpp/build/x64-software-conventions) and [System V ABI](https://wiki.osdev.org/System_V_ABI). the former mainly used in Windows and the latter is used on GNU/Linux and other systems. You can also find some detail about ABI [here](https://pub-33412179390046d2b4017e671ebbd429.r2.dev/calling_conventions.pdf)
But if we can switch the function we define with different ABI? Yes! You can read [this](https://stackoverflow.com/questions/66756382/how-to-make-my-c-program-compiled-with-sysv-calling-convention-run-under-mingw) for more info.

## Reference

* [https://learn.microsoft.com/en-us/cpp/build/x64-software-conventions?view=msvc-170](https://learn.microsoft.com/en-us/cpp/build/x64-software-conventions?view=msvc-170)
* [https://gcc.gnu.org/onlinedocs/gcc-11.2.0/gcc/x86-Function-Attributes.html#x86-Function-Attributes](https://gcc.gnu.org/onlinedocs/gcc-11.2.0/gcc/x86-Function-Attributes.html#x86-Function-Attributes)
* [https://en.wikipedia.org/wiki/Application_binary_interface](https://en.wikipedia.org/wiki/Application_binary_interface)
* [https://en.wikipedia.org/wiki/Calling_convention](https://en.wikipedia.org/wiki/Calling_convention)
* [https://osdev.org/System_V_ABI](https://osdev.org/System_V_ABI)
* [calling_conventions.pdf](assets/calling_conventions-20250506150106-g9upgy3.pdf)

‍
