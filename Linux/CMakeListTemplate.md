---
title: CMakeLists.txt基本通用模板
date: 2022-01-13 17:44:38
tags: CMake
categroies: Linux
description: CMakeLists.txt基本通用模板。
---

```bash
cmake_minimum_required(VERSION 3.9)
project(LevealDBTry)
 
 
#设定编译参数
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_BUILD_TYPE "Debug")
 
#设定源码列表.cpp
set(SOURCE_FILES ./main.cc)
#设定所有源码列表 ：aux_source_directory(<dir> <variable>)
#比如:aux_source_directory(${CMAKE_SOURCE_DIR} DIR)  将${CMAKE_SOURCE_DIR}目录下，也就是最顶级目录下所有的.cpp文件放入DIR变量中，后面的add_executable就可以很简化
#    add_executable(hello_world ${DIR})
 
 
#设定头文件路径
include_directories(../include/)
#include_directories("路径1"  “路径2”...)
 
 
#设定链接库的路径（一般使用第三方非系统目录下的库）
link_directories(../build/)
#link_directories("路径1"  “路径2”...)
 
 
#添加子目录,作用相当于进入子目录里面，展开子目录的CMakeLists.txt
#同时执行，子目录中的CMakeLists.txt一般是编译成一个库，作为一个模块
#在父目录中可以直接引用子目录生成的库
#add_subdirectory(math)
 
 
#生成动/静态库
#add_library(动/静态链接库名称  SHARED/STATIC(可选，默认STATIC)  源码列表)
#可以单独生成多个模块
 
 
#生成可执行文件
add_executable(myLevealDB   ${SOURCE_FILES} )
#比如：add_executable(hello_world    ${SOURCE_FILES})
 
 
target_link_libraries(myLevealDB  pthred glog)#就是g++ 编译选项中-l后的内容，不要有多余空格
 
ADD_CUSTOM_COMMAND( #执行shell命令
          TARGET myLevelDB 
          POST_BUILD #在目标文件myLevelDBbuild之后，执行下面的拷贝命令，还可以选择PRE_BUILD命令将会在其他依赖项执行前执行  PRE_LINK命令将会在其他依赖项执行完后执行  POST_BUILD命令将会在目标构建完后执行。
          COMMAND cp ./myLevelDB  ../
) 
```



-   cmake 常用变量

1、**CMAKE_BINARY_DIR** ，**PROJECT_BINARY_DIR _BINARY_DIR**

这三个变量指代的内容是一致的，如果是in source 编译，指得就是工程顶层目录，如果是out-of-source 编译，指的是工程编译发生的目录。PROJECT_BINARY_DIR 跟其他指令稍有区别，现在，你可以理解为他们是一致的。

２、**CMAKE_SOURCE_DIR**，**PROJECT_SOURCE_DIR _SOURCE_DIR**

这三个变量指代的内容是一致的，不论采用何种编译方式，都是工程顶层目录。

也就是在in source 编译时，他跟CMAKE_BINARY_DIR 等变量一致。PROJECT_SOURCE_DIR 跟其他指令稍有区别，现在，你可以理解为他们是一致的。

３、**CMAKE_CURRENT_SOURCE_DIR** 

指的是当前处理的CMakeLists.txt 所在的路径，比如上面我们提到的src子目录。

４、**CMAKE_CURRRENT_BINARY_DIR**

如果是in-source 编译，它跟CMAKE_CURRENT_SOURCE_DIR 一致，如果是out-of-source 编译，他指的是target 编译目录。使用我们上面提到的ADD_SUBDIRECTORY(src bin) 可以更改这个变量的值。使用SET(EXECUTABLE_OUTPUT_PATH < 新路径>)并不会对这个变量造成影响，它仅仅修改了最终目标文件存放的路径。

５、**CMAKE_CURRENT_LIST_FILE** 

输出调用这个变量的CMakeLists.txt 的完整路径。

６、**CMAKE_CURRENT_LIST_LINE** 

输出这个变量所在的行。

 7、**CMAKE_MODULE_PATH**

这个变量用来定义自己的cmake 模块所在的路径。如果你的工程比较复杂，有可能会自己编写一些cmake 模块，这些cmake 模块是随你的工程发布的，为了让cmake 在处理CMakeLists.txt 时找到这些模块，你需要通过SET指令，将自己的cmake 模块路径设置一下。

比如：

`SET(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)` 这时候你就可以通过INCLUDE 指令来调用自己的模块了。

８、**EXECUTABLE_OUTPUT_PATH**，**LIBRARY_OUTPUT_PATH** 

分别用来重新定义最终结果的存放目录，前面我们已经提到了这两个变量。

 9、**PROJECT_NAME** 

返回通过PROJECT 指令定义的项目名称。



-   cmake添加宏定义

CMakeLists.txt文件：

```bash
cmake_minimum_required(VERSION 3.9)
project(myProject)
IF(GPU)
#在命令行中使用cmake -DGPU，会进入这一行，C++代码中自动会有#define GPU
  ADD_DEFINITIONS(-DGPU) #注意一定要有-D
ENDIF(GPU)
 
add_executable(a.out testCmake.cc)
```



testCmake.cc文件：

```c++
#include <iostream>
using namespace std;

int main()
{
#ifdef GPU
	cout<<"GPU"<<endl;//定义了GPU输出这一行
#else
	cout<<"CPU"<<endl;//没有定义输出这一行
#endif
	cout<<"hello world"<<endl;//无论定义与否都会执行
	
	return 0;
}
```



执行CMake的命令：

```bash
mkdir build
cd build
cmake -DGPU=on ..
```



注意：一定是在GPU命令前加上D，不然无法识别。



>   [CMakeLists.txt基本通用模板_Likes的博客](https://blog.csdn.net/songchuwang1868/article/details/84774844)
