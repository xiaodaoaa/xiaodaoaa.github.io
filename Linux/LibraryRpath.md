---
title: linux指定当前路径为库搜索路径
date: 2022-03-04 14:45:50
tags: rpath
categories: Linux
description: linux系统中C/C++程序指定当前路径为库搜索路径。
---

众所周知， Linux 动态库的默认搜索路径是 `/lib` 和 `/usr/lib` 。动态库被创建后，一般都复制到这两个目录中。当程序执行时需要某动态库， 并且该动态库还未加载到内存中，则系统会自动到这两个默认搜索路径中去查找相应的动态库文件，然后加载该文件到内存中，这样程序就可以使用该动态库中的函 数，以及该动态库的其它资源了。

在 Linux 中，动态库的搜索路径除了默认的搜索路径外，还可以通过以下三种方法来指定。

-   在配置文件 `/etc/ld.so.conf` 中指定动态库搜索路径。每次编辑完该文件后，都必须运行命令 `ldconfig` 使修改后的配置生效 。
-   通过环境变量 `LD_LIBRARY_PATH` 指定动态库搜索路径。
-   在编译目标代码时指定该程序的动态库搜索路径。

下面我们重点讲一下第三种方法。

在C/C++程序里经常会调用到外部动态库文件中的库函数，运行程序时需要`export LD_LIBRARY_PATH`，不过使用它存在一些弊端，可能会影响到其它程序的运行。在经历的大项目中就遇到过，两个模块同时使用一外部动态库，而且版本还有差异，导致其中一模块出错，两模块是不同时期不同人员分别开发，修正起来费时费力。

对于上述问题，一个比较好的方法是在程序编译的时候加上参数`-Wl,-rpath`，指定编译好的程序在运行时动态库的目录。这种方法会将动态库路径写入到`elf`文件中去。 `-Wl` 的意思是将逗号后面的内容传递给 `linker`链接器。

`-Wa,<options>	Pass comma-separated <options> on to the assembler`

`-Wp,<options>	Pass comma-separated <options> on to the preprocessor`

`-Wl,<options>	Pass comma-separated <options> on to the linker` 

**示例**

-   CPrint.hpp

```c++
#pragma onece
#include <string>

void mPrint(std::string strMessage);
```



-   CPrint.cpp

```c++
#include <iostream>

void mPrint(std::string strMessage){
	std::cout << strMessage << std::endl;
}
```



-   main.cpp

```c++
#include "CPrint.hpp"

#include <iostream>

int main(int argc, char *argv[]){
	mPrint("Hello, world~");
	return 0;
}
```



生成`libcprint.so`:

```bash
g++ -fPIC -c CPrint.cpp
g++ -shared -fPIC CPrint.o -o libcprint.so
```

不使用参数`-Wl,-rpath`生成程序：

```bash
g++ -fPIC -o LibraryRpathTest main.cpp -L. -lcprint
```

运行程序时报找不到库文件的错误：

```bash
./LibraryRpathTest: error while loading shared libraries: libcprint.so: cannot open shared object file: No such file or directory
```



使用参数`-Wl,-rpath`生成程序：

```bash
g++ -fPIC -o LibraryRpathTest main.cpp -L. -lcprint -Wl,-rpath=. -Wl,-rpath='$ORIGIN' -Wl,-rpath='$ORIGIN/lib'
```

*注：*

```bash
Makefile工程添加示例：LD_FLAGS += -Wl,-rpath='$$ORIGIN' -Wl,-rpath='$$ORIGIN/lib'
Qt工程添加示例：QMAKE_LFLAGS += -Wl,-rpath="'\$\$ORIGIN'" -Wl,-rpath="'\$\$ORIGIN/lib'"
```

成功运行程序：

```bash
./LibraryRpathTest
Hello, world~
```



目录结构：

```bash
tree LibraryRpath 
LibraryRpath
├── CPrint.cpp
├── CPrint.hpp
├── CPrint.o
├── libcprint.so
├── LibraryRpathTest
└── main.cpp

```

