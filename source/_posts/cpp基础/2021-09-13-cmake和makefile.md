---
title: cmake 和makefile
date: 2021-09-13
updated: 2021-09-13
tags: 
  - cpp
  - compile
categories:
  - cpp base
aplayer: true
---

> makefile关系到了整个工程的编译规则, cmake是一个脚本语言,可以更方便的生成makefile

### makefile

基础写法
```
target ... : prerequisites(依赖的文件) ...
    command(执行的命令)
    ...
    ...


edit : main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
    cc -o edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o

main.o : main.c defs.h
    cc -c main.c
kbd.o : kbd.c defs.h command.h
    cc -c kbd.c
command.o : command.c defs.h command.h
    cc -c command.c
display.o : display.c defs.h buffer.h
    cc -c display.c
insert.o : insert.c defs.h buffer.h
    cc -c insert.c
search.o : search.c defs.h buffer.h
    cc -c search.c
files.o : files.c defs.h buffer.h command.h
    cc -c files.c
utils.o : utils.c defs.h
    cc -c utils.c
clean :
    rm edit main.o kbd.o command.o display.o \
        insert.o search.o files.o utils.o
```

当引用头文件和链接库文件时'

`$@` 表示目标文件, `$^`表示依赖文件, `$<`  表示第一个依赖文件, 
```cpp
INCLUDE = -I./include
LIB_PATH = -L./lib -lf1 -lf2 -lf3

TARGET1 = test1
TARGET2 = test2
TARGET3 = test3

.PHONY: all 
all: $(TARGET1) $(TARGET2) $(TARGET3) # 这是链接
$(TARGET1) : $(TARGET1).o
     $(GCC) -o $@ $< $(LIB_PATH)
$(TARGET2) : $(TARGET2).o
     $(GCC) -o $@ $< $(LIB_PATH)
 $(TARGET3) : $(TARGET3).o         
     $(GCC) -o $@ $< $(LIB_PATH)

#build object 这是编译
%.o : %.c
    $(GCC) -c $< -o $(INC_CLUDE)

.PHONY: clean
clean:
    rm -f $(TARGET1) $(TARGET1).o
    rm -f $(TARGET2) $(TARGET2).o
    rm -f $(TARGET3) $(TARGET3).o

#定义变量，使用变量:$(变量名)
CC=g++
#定义变量srcs，表示需要编译的源文件，需要表明路径，如果直接写表示这些cpp文件和makefile在同一个目录下，如果有多个源文件，每行以\结尾
SRCS=main.cpp\
        udp.cpp
#定义变量OBJS,表示将原文件中所有以.cpp结尾的文件替换成以.o结尾，即将.cpp源文件编译成.o文件
OBJS=$(SRCS:.cpp=.o)
#定义变量，表示最终生成的可执行文件名
EXEC=maincpp
#start，表示开始执行，冒号后面的$(OBJS)表示要生成最终可执行文件，需要依赖那些.o文件的
start:$(OBJS)
        #相当于执行：g++ -o maincpp .o文件列表，-o表示生成的目标文件
        $(CC) -o $(EXEC) $(OBJS)
#表示我的.o文件来自于.cpp文件
.cpp.o:
        #如果在依赖关系中，有多个需要编译的.cpp文件，那么这个语句就需要执行多次。-c $<指的是需要编译的.cpp文件,-o $@指这个cpp文件编译后的中间文件名。比如在依赖关系中，有a.cpp和b.cpp，即$(OBJS)的值为a.cpp b.cpp，那么这条语句需要执行2次，第一次的$@为a.o,$<为a.cpp，第二次的$@为b.o,$<为b.cpp。-c表示只编译不链接，-o表示生成的目标文件
        #-DMYLINUX:-D为参数，MYLINUX为在cpp源文件中定义的宏名称，如#ifdef MYLINUX。注意-D和宏名称中间没有空格
        $(CC) -o $@ -c $< -DMYLINUX

#执行make clean指令
clean:
        #执行make clean指令时，需要执行的操作，比如下面的指令时指删除所有.o文件
        rm -rf $(OBJS)

```

### cmake基础功能

* project项目信息
* add_subdirectory 添加子目录
* 指定生成目标(相当于-o)add_executable
* 添加链接库target_link_libraries
* 生成链接库add_library

<!-- more -->

```
# 根目录的CMakeLists.txt
# CMake 最低版本号要求
cmake_minimum_required (VERSION 2.8)

# 项目信息
project (Demo3 C CXX)

# 查找当前目录下的所有源文件
# 并将名称保存到 DIR_SRCS 变量
aux_source_directory(. DIR_SRCS)

# 添加 math 子目录
add_subdirectory(math)

# 指定生成目标(可执行文件) 
add_executable(Demo main.cc)

# 添加链接库
target_link_libraries(Demo MathFunctions)

# 子文件夹的CMakeLists.txt
# 查找当前目录下的所有源文件
# 并将名称保存到 DIR_LIB_SRCS 变量
aux_source_directory(. DIR_LIB_SRCS)

# 生成链接库
add_library (MathFunctions ${DIR_LIB_SRCS})
```

注意
* target_link_libraries里库文件的顺序符合gcc链接顺序的规则，即被依赖的库放在依赖它的库的后面

```
target_link_libraries(hello A B.a C.so)
在上面的命令中，libA.so可能依赖于libB.a和libC.so，如果顺序有错，链接时会报错
```

###  头文件目录，库文件目录，库文件三点

* 类似于g++的`-I`, `-L`, `-l`三点, cmake也有三点

```
#添加头文件目录, 相当于g++ -I
include_directories(/home/larry/myproject/myc++proj/muduostd)
# 添加库文件目录, 相当于g++ -L
link_directories(/home/larry/myproject/myc++proj/muduostd/build1/lib)

# 添加库链接
link_libraries(pthread)
#或在目标文件中链接
target_link_libraries(muduo_http muduo_net muduo_base pthread)
```
### 变量常量


`cmake`提供一些变量方便使用，例如指定当前目录等等

```
PROJECT_BINARY_DIR， 如果in source 编译(也就是项目根目录编译),指得就是工程顶层目录,如果是 out-of-source(一般使用, 就是建立Build文件夹在文件夹中) 编译,指的是工程编译发生的目录。

PROJECT_SOURCE_DIR, 表示项目所在的文件夹目录
CMAKE_CURRRENT_BINARY_DIR, CMakeList.txt所在的目录
CMAKE_BUILD_TYPE, 编译类型, 可以设置`Debug`, `Release`
CMAKE_PROJECT_NAME, 返回 PROJECT 指令定义的项目名称
CMAKE_CXX_FLAGS, 设置C++的flags,

set(CMAKE_CXX_COMPILER      "clang++" )         # 显示指定使用的C++编译器
set(CMAKE_CXX_FLAGS   "-std=c++11")             # c++11
set(CMAKE_CXX_FLAGS   "-g")                     # 调试信息
set(CMAKE_CXX_FLAGS   "-Wall")                  # 开启所有警告

set(CMAKE_CXX_FLAGS_DEBUG   "-O0" )             # 调试包不优化
set(CMAKE_CXX_FLAGS_RELEASE "-O2 -DNDEBUG " )   # release包优化
EXECUTABLE_OUTPUT_PATH 执行文件的输出目录
LIBRARY_OUTPUT_PATH 库文件的输出目录

set(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)
set(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
```


#### 支持gdb的调试

```
SET(CMAKE_BUILD_TYPE "Debug")
SET(CMAKE_CXX_FLAGS_DEBUG "$ENV{CXXFLAGS} -O0 -Wall -g2 -ggdb")
SET(CMAKE_CXX_FLAGS_RELEASE "$ENV{CXXFLAGS} -O3 -Wall")
```

#### set

赋值给一般变量(以后方便引用)

```
set(HEADERS
  HttpContext.h
  HttpRequest.h
  HttpResponse.h
  HttpServer.h
  )
# 安装头文件目录
install(FILES ${HEADERS} DESTINATION include/muduo/net)

设置安装路径
SET(CMAKE_INSTALL_PREFIX < install_path >)
```

#### if和options

* options可以给变量赋值, 从而被if条件语句所引用

```
if(address)
    message("defined address!!!!!!!!!!")
else()
    message("NOT defined address!!!!!!!!!")
endif()

# options赋值可以被if引用(set不行)
option(address "hello world" ON)
message("option is ${address}")

if(address)
    message("defined address!!!!!!!!!!")
else()
    message("NOT defined address!!!!!!!!!")
endif()

# 输出结果
NOT defined address!!!!!!!!!!
option is a
defined address!!!!!!!!!!
```

#### MESSAGE

* 为用户显示一条消息

```
STATUS = 非重要消息；
WARNING = CMake 警告, 会继续执行；
AUTHOR_WARNING = CMake 警告 (dev), 会继续执行；
SEND_ERROR = CMake 错误, 继续执行，但是会跳过生成的步骤；
FATAL_ERROR = CMake 错误, 终止所有处理过程；


if(PROTOBUF_FOUND)
message(STATUS "found protobuf")
endif()
```

### find_package

引如外部包

```
find_package(CURL)
find_package(ZLIB)
find_path(CARES_INCLUDE_DIR ares.h)
find_library(CARES_LIBRARY NAMES cares)
find_path(MHD_INCLUDE_DIR microhttpd.h)
```

* cmake在安装路径(比如`/usr/local/share/cmake-3.16/Modules$`)已经提供了一些官方依赖包, 以.cmake结尾可以直接用find_pakcage进行引用。

* 然而我们一般使用`find_pakcage`得到非官方库, 这时候需要对该库进行一些处理

```
# clone该项目
git clone https://github.com/google/glog.git 
# 切换到需要的版本 
cd glog

# 根据官网的指南进行安装
cmake -H. -Bbuild -G "Unix Makefiles"
cmake --build build
cmake --build build --target install
```
之后就可以通过`find_pakcage`引用`glog`库了。

* 精细的寻找，可以指定版本等细节`find_package(<PackageName> [version] [EXACT] [QUIET] [MODULE]`

```
# 在指定目录下寻找头文件和动态库文件的位置，可以指定多个目标路径
find_path(ADD_INCLUDE_DIR libadd.h /usr/include/ /usr/local/include ${CMAKE_SOURCE_DIR}/ModuleMode)
find_library(ADD_LIBRARY NAMES add PATHS /usr/lib/add /usr/local/lib/add ${CMAKE_SOURCE_DIR}/ModuleMode)

if (ADD_INCLUDE_DIR AND ADD_LIBRARY)
    set(ADD_FOUND TRUE)
endif (ADD_INCLUDE_DIR AND ADD_LIBRARY)
```

#### ADD_DEPENDENCIES

假设需要生成一个可执行文件,该文件生成需要链接a.so b.so c.so d.so四个动态库正常来讲,我们一把只需要以下两条指令即可:
```
ADD_EXECUTABLE(main main.cpp)
TARGET_LINK_LIBRARIES(main a.so b.so c.so d.so)
```
但是编译的时候报错,一些符号的定义找不到,而这些符号恰恰就在这几个库中,假设在a.so 和 b.so中,在上述两条指令之间加上一条指令即可编译通过:
```
ADD_DEPENDENCIES(main a.so b.so)
```
原因比较简单,生成main需要依赖a.so和b.so中的符号定义,然而a.so和b.so库的生成是在main编译生产之后的,添加这条语句就是提醒编译器需要先生成main的依赖(a.so,b.so),然后再去生成main.


