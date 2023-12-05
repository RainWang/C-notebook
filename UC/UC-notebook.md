<!-- TOC -->

- [1. 程序构建过程](#1-程序构建过程)
- [2. 多模块开发](#2-多模块开发)
    - [2.1 编写头文件](#21-编写头文件)
    - [2.2 库文件基础](#22-库文件基础)
        - [2.2.1 静态库](#221-静态库)
            - [2.2.1.1 静态库制作过程](#2211-静态库制作过程)
            - [2.2.1.2 静态库缺点](#2212-静态库缺点)
        - [2.2.2 动态库](#222-动态库)
            - [2.2.2.1 动态库制作过程](#2221-动态库制作过程)
            - [2.2.2.2 动态库优点](#2222-动态库优点)
    - [2.3 动态加载](#23-动态加载)
        - [2.3.1 自动加载](#231-自动加载)
        - [2.3.2 动态加载](#232-动态加载)
- [3. 程序中的错误处理](#3-程序中的错误处理)
- [4. 内存管理](#4-内存管理)
- [5. 文件管理](#5-文件管理)

<!-- /TOC -->
# 1. 程序构建过程
1. 预处理过程：预处理指令执行，注释消除
>gcc -E hello.c -o hello.i<br>
2. 编译：将C语言编译为汇编语言，检查程序中的语法错误
>gcc -S hello.i -o hello.s<br>
3. 汇编：将汇编语言翻译为机器指令
>gcc -c hello.s<br>
4. 连接：将目标文件和运行时文件，库函数合成可执行文件
>gcc hello.o<br>
5. 程序的执行
>./a.out<br>

# 2. 多模块开发
## 2.1 编写头文件
>#ifndef MATH_H<br>
>#define MATH_H<br>
>int t_add(int,int);<br>
>#endif<br>

## 2.2 库文件基础
### 2.2.1 静态库
库函数命名规则：.a
#### 2.2.1.1 静态库制作过程
1. 将所有要加入库中的源文件编译为目标文件。
>// 生成mul.o和add.o文件<br>
>gcc -c mul.c add.c<br>
2. 将第一步生成的所有目标文件打包成静态库文件。
>// 生成静态库文件<br>
>ar -r libpmath.a *.o<br>
>// 查看静态库包含哪些目标文件<br>
>ar -t libpmath.a<br>
3. 使用静态库文件链接生成可执行文件。
>// 生成main.o文件<br>
>gcc main.c -c<br>
>// 生成a.out可执行文件<br>
>gcc main.o -L. -lpmath<br>

#### 2.2.1.2 静态库缺点
1. 库函数被包含在每个进程的代码段中，对于很多个并发进程的系统，主存资源浪费严重。
2. 库函数被包含在可执行文件中，如果多个程序链接可静态库文件，静态库文件的内容被重复包含在可执行文件中，造成磁盘空间的浪费。
3. 当有库文件更新的时候，需要重新链接。更新困难，维护成本提高。

### 2.2.2 动态库
库函数命名规则：.so
#### 2.2.2.1 动态库制作过程
1. 将所有要加入库的源文件编译为与位置无关的目标文件。
>// 生成与位置无关的mul.o和add.o目标文件<br>
>gcc -c -fPIC mul.c add.c<br>
2. 将第一步生成的目标文件打包到动态库文件中。
>// 生成动态库文件<br>
>gcc -shared -o libpmath.so *.o<br>

>// 把动态库路径加到动态库环境变量中，或者放到系统指定的库路径下（/lib或者/usr/lib）<br>
>export LD_LIBARY_PATH=$LD_LIBARY_PATH:t_math<br>
3. 使用动态链接库生成可执行文件。
>// 生成a.out可执行文件<br>
>gcc main.c -lpmath<br>
4. 执行可执行文件。
>// 查看可执行文件依赖的动态库<br>
>ldd a.out<br>

#### 2.2.2.2 动态库优点
1. 在内存中只有一份拷贝，被所有进程共享，节省内存空间。
2. 库函数和程序独立存在，被所有程序动态链接，节省磁盘空间。
3. 当有库文件更新的时候，由于链接，更新容易，维护成本较低。

## 2.3 动态加载
### 2.3.1 自动加载
程序在开始执行的时候，将依赖的动态库文件加载到内存，再进行函数的链接，称为自动加载。

### 2.3.2 动态加载
程序在执行期间，需要使用某个动态库中的文件的时候，可以向动态链接器发出请求，请求将动态库文件加载到内存中，这种行为称为动态加载。<br>
动态链接器为动态库加载提供了相应的API：<br>
1. 将动态库加载到内存
>#include<dlfcn.h> <br>
>void* dlopen(const char* filename,int flags);

- 参数： <br>
    - filename：<br>
    共享库路径。如果只给定名字，则按照动态链接库的搜索路径找动态库文件（LD_LIBARY_PATH指定的路径下或者默认路径下找）。<br>
    - flags：<br>
    加载方式，可取以下值：<br>
    RTLD_LAZY-延迟加载，使用共享库中的符号（如调用库中的函数）时才加载。<br>
    RTLD_NOW-立即加载，函数返回的时候，已经加载到内存。
- 返回值： <br>
    - 成功：返回动态库加载到内存的地址。<br>
    - 失败：返回NULL，可以使用dlerror(3)函数诊断错误的原因。<br>
- 注意： <br>
    - 动态链接器的API都需要使用到dl动态库文件，所以链接的时候要使用-ldl。<br>

2. 关闭动态库
>#include<dlfcn.h> <br>
>int dlclose(void* handle);
- 参数： <br>
    - handle：<br>
    动态库加载到内存的地址，dlopen(3)的返回值。
- 返回值： <br>
    - 成功：0。<br>
    - 失败：非0，可以使用dlerror(3)函数诊断错误的原因。<br>
- 注意： <br>
    - 仅仅使动态库的引用记数减一，并不一定从内存移除，只有引用计数为0时，才从内存移除。<br>

3. 获取dlopen API的错误信息
>#include<dlfcn.h> <br>
>char* dlerror(void);
- 返回值： <br>
返回错误原因字符串的首地址，NULL代表没有错误产生。<br>

4. 从动态库中找符号的地址
>#include<dlfcn.h> <br>
>void* dlsym(void* handle,const char* symbol);
- 参数： <br>
    - handle：<br>
    动态库加载到内存的地址，dlopen(3)的返回值。
    - symbol：<br>
    符号的名字，包括函数的名字，全局变量的名字，静态局部变量的名字。
- 返回值： <br>
    - 成功：返回符号的地址。<br>
    - 失败：返回NULL，可以使用dlerror(3)函数诊断错误的原因。<br>

# 3. 程序中的错误处理


# 4. 内存管理


# 5. 文件管理