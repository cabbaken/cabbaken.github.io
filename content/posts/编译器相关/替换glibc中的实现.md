---
title: 替换glibc中的实现
date: 2025-05-15T14:05:40Z
lastmod: 2025-05-15T14:11:07Z
categories: ["Compiler"]
tags: ["Compiler", "glibc"]
---

# 替换Glibc的实现

glibc often uses weak symbols internally. You can sometimes override them by defining a strong symbol.

```c
int printf(const char *format, ...) __attribute__((visibility("default")));
```

还可以通过`-Wl,--wrap=mcount`​来替换实现, 具体可见[-pg and -finstrument-functions]({{< ref "-pg and -finstrument-functions" >}}).

‍
