---
title: -pg and -finstrument-functions
date: 2025-03-19T19:10:17Z
lastmod: 2025-05-15T14:11:31Z
tags: ["Compiler", "C/C++"]
categories: ["Compiler"]
---

# -pg and -finstrument-functions

两者皆可用于debug/profiling, 其原理:

* -pg: 在函数开头插入`_mcount()`​.
* -finstrument-functions: 在函数开头插入`__cyg_profile_func_enter()`​, 结尾插入`__cyg_profile_func_exit()`​.

‍

使用`-pg`​的例子:

a.cpp:

```c++
#include <iostream>

// use extern "C" to create the symbol in C style, or linker cannot find it
// later
// use __attribute__((no_instrument_function)) tell compiler not to insert 
// mcount here, or it will lead to infinite recursion.
extern "C" __attribute__((no_instrument_function)) void __wrap_mcount()
{
        std::cout<<"Jump!\n";
}
```

t.cpp:

```c++
int b=2;

int main()
{
  int a=1;
  int c=a+b;

  return 0;
}
```

编译:

```bash
# insert mcount() with -pg
g++ a.cpp -pg -c -o a
g++ t.cpp -pg -c -o t
# replace mcount() with __wrap_mcount()
# please read more info about `--wrap`
# for the replacement.
g++ a t -Wl,--wrap=mcount -o a.out
```

这里的`-Wl`​是指后续使用','分隔的option会直接传给linker.

这里提供另一种替换glibc实现的方法[替换glibc中的实现]({{< ref "替换glibc中的实现" >}}).
