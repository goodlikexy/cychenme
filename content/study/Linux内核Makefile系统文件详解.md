---
title: "Linux 内核 Makefile 系统文件详解"
slug: "linux-kernel-makefile"
date: 2026-07-19
description: "从 C 工程构建流程、目标依赖关系和常见 make 语法入手，整理理解 Linux Kbuild 前需要掌握的 Makefile 基础。"
tags:
  - "Linux"
  - "Makefile"
  - "Kbuild"
  - "构建系统"
managedBy: publish-openclaw-notes
---

## 元信息

- 类型：技术博客 / 教程
- 作者/机构：知乎「玩转Linux内核」
- 原文链接：https://zhuanlan.zhihu.com/p/437667448
- 阅读日期：2026-07-19
- 关联主题：GNU Make、Linux Kbuild 前置知识
- 阅读状态：粗读

## 一句话结论

这篇文章主要讲 GNU Makefile 基础，不是 Linux 内核 Kbuild 的完整解析。读它之前要先理解 C 程序从 `.c` 到 `.o` 再到可执行文件的过程；读它时要抓住一条主线：Makefile 用“目标、依赖、命令”描述构建关系，`make` 根据文件变化只执行必要命令。

## 先建立整体图

一个 C 工程的构建过程可以先理解成：

```text
源代码 .c
  -> 编译 compile
目标文件 .o
  -> 链接 link
最终可执行文件
```

Makefile 做的事情不是替你写 C 代码，而是告诉 `make`：

```text
哪些 .c 要编译成哪些 .o
哪些 .o 要链接成最终程序
哪些头文件 .h 变化后需要重新编译
哪些清理、测试、安装命令可以用 make xxx 触发
```

## 文件类型速查

| 文件或后缀 | 是什么 | 例子 |
| --- | --- | --- |
| `.c` | C 源代码文件，通常放函数实现 | `main.c`、`add.c` |
| `.h` | C 头文件，通常放函数声明、结构体声明、宏定义 | `add.h` |
| `.o` | object file，目标文件；由 `.c` 编译得到，通常不能直接运行 | `main.o` |
| `.a` | archive，静态库；本质上是一组 `.o` 文件的打包 | `libfoo.a` |
| `.d` | dependency file，依赖文件；记录 `.o` 依赖哪些 `.c` / `.h` | `main.d` |
| `.mk` | Makefile 片段，常被主 Makefile 用 `include` 引入 | `config.mk` |
| `.make` | 普通文件名后缀，可作为 Makefile 使用，但不是默认名 | `foo.make` |
| `.g` | 原文例子里的普通输入文件后缀，不是 Makefile 固定概念 | `text.g` |
| 无后缀文件 | Linux/Unix 下常见可执行文件名 | `hello`、`vmlinux` |
| `Makefile` | 构建规则文件，给 `make` 读取 | `Makefile` |

`.d` 需要特别记住：它不是源代码，也不是编译产物本身，而是“依赖关系记录”。例如 `main.d` 里面可能写着：

```make
main.o: main.c defs.h config.h
```

意思是：`main.o` 依赖 `main.c`、`defs.h`、`config.h`。

`.g` 不需要特别记。原文的：

```make
bigoutput littleoutput : text.g
```

只是举例说 `bigoutput` 和 `littleoutput` 依赖一个叫 `text.g` 的文件。这里 `.g` 可以换成 `.txt`、`.in`、`.data`，它不是 GNU make 的固定后缀。

## gcc、cc 和常见参数

`gcc` 是常用 C 编译器命令。

`cc` 是 Unix 传统 C 编译器命令名。很多系统里 `cc` 会指向 gcc 或 clang。文章里写 `cc` 时，可以先理解成“C 编译器”。

| 参数 | 含义 | 例子 |
| --- | --- | --- |
| `-c` | 只编译，不链接；把 `.c` 变成 `.o` | `gcc -c main.c` |
| `-o` | 指定输出文件名（做链接并把名字指定） | `gcc main.o add.o -o hello` |
| `-g` | 生成调试信息，方便 gdb 调试 | `gcc -g main.c -o hello` |

-o:output；-c:complie；-g:debug;

如果不指定名字，编译或者链接出来默认名字是a.out

例子：

```bash
gcc -c main.c
```

生成：

```text
main.o
```

再执行：

```bash
gcc main.o add.o -o hello
```

生成最终程序：

```text
hello
```

## 编译和链接

编译解决的是“单个源文件能不能翻译成机器能理解的中间文件”。

例如：

```bash
gcc -c add.c
```

得到：

```text
add.o
```

链接解决的是“多个中间文件能不能合成最终程序”。

例如：

```bash
gcc main.o add.o -o hello
```

如果 `main.o` 里调用了 `add()`，链接器会去 `add.o` 或库文件里找 `add()` 的实现。找不到就会报链接错误。

## 一个完整小例子

目录里有：

```text
main.c
add.c
add.h
Makefile
```

`add.h`：

```c
int add(int a, int b);
```

`add.c`：

```c
#include "add.h"

int add(int a, int b) {
    return a + b;
}
```

`main.c`：

```c
#include <stdio.h>
#include "add.h"

int main(void) {
    printf("%d\n", add(1, 2));
    return 0;
}
```

`Makefile`：

```make
hello: main.o add.o
	gcc main.o add.o -o hello

main.o: main.c add.h
	gcc -c main.c

add.o: add.c add.h
	gcc -c add.c

.PHONY: clean
clean:
	rm -f hello main.o add.o
```

执行：

```bash
make
```

会构建默认目标 `hello`。

执行：

```bash
make clean
```

会执行 `clean` 目标，删除构建产物。

## Makefile 是什么

Makefile 是一个文本文件，默认文件名通常就叫：

```text
Makefile
```

它没有后缀。

`make` 默认会在当前目录按常见名字查找：

```text
GNUmakefile
makefile
Makefile
```

所以一般直接执行：

```bash
make
```

不需要指定文件名。

如果构建文件不叫默认名字，就用 `-f`：

```bash
make -f build.mk
make -f foo.make
```

## make 命令是什么

`make` 是读取 Makefile 并执行构建规则的命令行工具。

它不是简单地从第一行执行到最后一行，而是：

1. 读取 Makefile。
2. 找到默认目标。
3. 建立目标和依赖之间的关系。
4. 比较文件修改时间。
5. 只重新生成过期的目标。

例如只输入：

```bash
make
```

它执行第一条规则的第一个目标。

输入：

```bash
make clean
```

它执行名为 `clean` 的目标。

输入：

```bash
make hello
```

它执行名为 `hello` 的目标。

## Makefile 规则

Makefile 的基本格式是：

```make
target: prerequisites
	command
```

中文解释：

```text
目标: 依赖
	生成目标的命令
```

例子：

```make
main.o: main.c add.h
	gcc -c main.c
```

含义：

- 目标是 `main.o`。
- 依赖是 `main.c` 和 `add.h`。
- 如果 `main.o` 不存在，或者 `main.c` / `add.h` 比 `main.o` 新，就执行 `gcc -c main.c`。

命令前面必须是 Tab，不是普通空格。

## 最终目标和多个目标

Makefile 里可以有很多目标。

例如：

```make
all: hello test

hello: main.o add.o
	gcc main.o add.o -o hello

test: test.o add.o
	gcc test.o add.o -o test

clean:
	rm -f hello test *.o
```

这里有 4 个目标：

- `all`
- `hello`
- `test`
- `clean`

只输入 `make` 时，默认执行第一条规则的第一个目标，也就是 `all`。这个默认目标也叫最终目标。

`clean` 通常写在后面，否则如果它成为第一条规则，只输入 `make` 时就会默认清理文件。

## 伪目标 `.PHONY`

有些目标不是为了生成同名文件，而是表示一个动作。

例如：

```make
clean:
	rm -f hello *.o
```

`clean` 不是要生成一个叫 `clean` 的文件，而是一个清理动作。

这种目标应该声明为伪目标：

```make
.PHONY: clean
clean:
	rm -f hello *.o
```

意思是：`clean` 永远按动作目标处理，不要把它当成真实文件。

## 变量

Makefile 变量就是文本替换。

```make
objects = main.o add.o

hello: $(objects)
	gcc $(objects) -o hello
```

等价于：

```make
hello: main.o add.o
	gcc main.o add.o -o hello
```

常见赋值方式：

| 写法 | 含义 |
| --- | --- |
| `=` | 使用时再展开 |
| `:=` | 定义时立即展开 |
| `+=` | 追加内容 |
| `?=` | 变量未定义时才赋值 |

初学先记：大多数简单场景用 `=` 能看懂；工程里为了避免展开时机混乱，经常用 `:=`。

## 反斜杠 `\`

Makefile 里的 `\` 常表示续行。

```make
objects = main.o add.o foo.o \
          bar.o utils.o
```

等价于：

```make
objects = main.o add.o foo.o bar.o utils.o
```

它只是为了让长行更好读。

## 通配符

通配符用来匹配一批文件。

| 通配符 | 含义 | 例子 |
| --- | --- | --- |
| `*` | 匹配任意长度字符 | `*.o` 匹配所有 `.o` 文件 |
| `?` | 匹配一个字符 | `file?.c` 匹配 `file1.c` |
| `[12]` | 匹配集合中的一个字符 | `file[12].c` 匹配 `file1.c`、`file2.c` |

例子：

```make
clean:
	rm -f *.o
```

表示删除当前目录下所有 `.o` 文件。

## 引用其它 Makefile

一个 Makefile 太长时，可以拆成多个文件，再用 `include` 引入。

```make
include config.mk
include rules.mk
```

可以理解成：

```text
把 config.mk 的内容插入到这里
把 rules.mk 的内容插入到这里
```

如果文件可能不存在，可以写：

```make
-include main.d
```

前面的 `-` 表示：找不到也不要报错，继续执行。

这在自动依赖里很常见，因为第一次构建时 `.d` 文件还没生成。

## 文件搜寻

默认情况下，make 只在当前目录找依赖文件。

如果源文件、头文件放在别的目录，可以用 `VPATH` 或 `vpath`。

全局搜索路径：

```make
VPATH = src:include
```

意思是：当前目录找不到时，再去 `src` 和 `include` 找。

按文件类型指定路径：

```make
vpath %.c src
vpath %.h include
```

意思是：

- `.c` 文件去 `src` 找。
- `.h` 文件去 `include` 找。

Linux 内核源码有很多目录，例如 `kernel/`、`mm/`、`fs/`、`drivers/`，所以构建系统必须处理跨目录找文件。

## 多目标

多个目标有共同依赖，可以写在一条规则里。

```make
a.o b.o: common.h
```

等价于：

```make
a.o: common.h
b.o: common.h
```

意思是：`common.h` 改了，`a.o` 和 `b.o` 都要重新检查是否需要构建。

多目标规则里常见自动变量 `$@`：

```make
bigoutput littleoutput: text.g
	generate text.g -$(subst output,,$@) > $@
```

这里：

- `text.g` 是普通输入文件名，不是固定概念。
- `$@` 表示当前正在生成的目标。

当生成 `bigoutput` 时，`$@` 是 `bigoutput`。

当生成 `littleoutput` 时，`$@` 是 `littleoutput`。

## 静态模式

静态模式用于批量表达“同名 `.c` 生成同名 `.o`”这类规则。

重复写法：

```make
foo.o: foo.c
	gcc -c foo.c -o foo.o

bar.o: bar.c
	gcc -c bar.c -o bar.o
```

静态模式写法：

```make
objects = foo.o bar.o

$(objects): %.o: %.c
	gcc -c $< -o $@
```

解释：

- `$(objects)` 是目标集合：`foo.o bar.o`。
- `%.o` 表示目标模式。
- `%.c` 表示依赖模式。
- `%` 表示相同的文件名前缀，例如 `foo`、`bar`。
- `$@` 表示当前目标。
- `$<` 表示第一个依赖。

所以构建 `foo.o` 时：

```make
gcc -c $< -o $@
```

等价于：

```bash
gcc -c foo.c -o foo.o
```

## 自动生成依赖性

自动生成依赖性解决的是：头文件 `.h` 改了以后，哪些 `.o` 需要重新编译。

假设 `main.c` 里有：

```c
#include "defs.h"
#include "config.h"
```

那么 `main.o` 实际依赖：

```make
main.o: main.c defs.h config.h
```

如果手工维护，文件一多很容易漏。

所以让 gcc 自动生成依赖：

```bash
gcc -MM main.c
```

输出类似：

```make
main.o: main.c defs.h config.h
```

常见做法是每个 `.c` 生成一个 `.d` 文件：

```text
main.c -> main.d
add.c  -> add.d
```

`main.d` 里保存：

```make
main.o: main.c defs.h config.h
```

主 Makefile 再包含这些 `.d` 文件：

```make
sources = main.c add.c
-include $(sources:.c=.d)
```

其中：

```make
$(sources:.c=.d)
```

表示把：

```text
main.c add.c
```

替换成：

```text
main.d add.d
```

这样 make 就能自动知道：改了某个 `.h` 文件后，哪些 `.o` 需要重新编译。

## make 的工作方式

make 大体按这个流程工作：

1. 在当前目录找 `Makefile`。
2. 读取 `include` 引入的其他 Makefile。
3. 展开变量和规则。
4. 找到默认最终目标。
5. 从最终目标出发建立依赖链。
6. 比较目标和依赖的修改时间。
7. 对过期目标执行命令。

关键点：make 不是脚本顺序执行器，而是依赖图构建器。

## Linux 内核里的 Makefile

Linux 内核不是一个 Makefile，而是一套 Makefile / Kbuild 文件。

大体包括：

- 顶层 `Makefile`：整个内核构建入口。
- 各目录里的 `Makefile`：描述本目录构建哪些对象。
- `Kbuild` 文件：部分目录使用的 Kbuild 规则文件。
- `scripts/Makefile.*`：通用构建规则。
- `.config`：内核配置，决定哪些功能编进内核、哪些编成模块、哪些不编。

这篇文章只讲 GNU Make 基础，还没有真正讲 Linux 内核 Kbuild 的核心变量，例如 `obj-y`、`obj-m`、`CONFIG_*`。

## 读这篇文章时的正确顺序

建议按这个顺序读，而不是按原文顺序硬啃：

1. 先懂 `.c`、`.h`、`.o`、链接、可执行文件。
2. 再懂 Makefile 的 `target: prerequisites`。
3. 再懂 `make` 如何找默认目标。
4. 再懂变量、伪目标、通配符。
5. 再看 `include`、文件搜寻、多目标、静态模式。
6. 最后看 `.d` 自动依赖。
7. 之后再进入 Linux 内核 Kbuild。

## 核心概念表

| 概念 | 中文解释 |
| --- | --- |
| target | 目标，可以是文件，也可以是动作名 |
| prerequisite | 依赖，目标生成前需要检查的文件或目标 |
| command | 命令，真正执行的 shell 命令 |
| 默认目标 | 只输入 `make` 时执行的目标，通常是第一条规则的第一个目标 |
| 伪目标 | 不对应真实文件的动作目标，例如 `clean` |
| 自动变量 `$@` | 当前目标 |
| 自动变量 `$<` | 第一个依赖 |
| 自动变量 `$^` | 所有依赖 |
| `include` | 引入其他 Makefile 文件 |
| `VPATH` / `vpath` | 指定文件搜索路径 |
| `.d` 文件 | 自动生成的依赖关系文件 |

## 后续问题

| 问题 | 为什么重要 | 下一步 |
| --- | --- | --- |
| Linux Kbuild 和普通 GNU Makefile 有哪些差异？ | 本文只讲 Make 基础，尚未进入内核构建系统核心 | 阅读内核官方 `Documentation/kbuild/` |
| `obj-y`、`obj-m`、`CONFIG_*` 如何决定编译进内核还是编译成模块？ | 这是理解驱动和模块构建的关键 | 拆真实内核目录 Makefile |
| 顶层 `Makefile` 如何递归进入子目录？ | 关系到内核源码目录和构建流程的对应 | 阅读顶层 Makefile 与 `scripts/Makefile.*` |

## 来源摘记

- 原文强调 Makefile 的核心是文件依赖性：当依赖文件比目标文件更新，或目标不存在时，执行对应命令。
- 原文说明 GNU make 的工作方式是先读入 Makefile 和 include 文件，初始化变量，推导规则，创建依赖链，再决定并执行需要更新的目标。
