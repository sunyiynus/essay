#### nvim coc-clangd补全代码#include*"my.h"*解析



###### ***ENV***

> OS： Ubuntu 20.04 LTS
>
> 编辑器: nvim
>
> 代码补全工具: coc-clangd



```c++
//myproject/src/myhead.h
class myTest {
public:
	void f();
};
```

```c++
//myproject/src/myhead.cpp

#include "myTest.h"
void myTest::f() { std::cout<< "ufo-42-mouse"<<std::endl; }
```

###### 问题

在这种情况下clangd并不能解析出header在那儿，于是乎就是满屏红的未申明。

***第一种办法***

```c++
//myproject/src/myhead.cpp
#include "../src/myhead.h"
//......
```

这种办法相当丑陋不推荐。



***第二种***

还是直接引用头文件，不带相对路径。但是需要配置clangd识别项目所需要的*compile_commands.json* 文件，这个包含了很多编译所需要的各种信息。这些编译指令是clangd理解你的源代码所需要知道的。

```json
blili@nmd:~/Documents/Bitusk/testing$ cat compile_commands.json 
[
  {
    "arguments": [
      "/usr/bin/x86_64-linux-gnu-g++-9",
      "-c",
      "-I",
      "../include/", #头文件所在目录
      "-Wall",
      "-g",
      "-o",
      "testError.o",
      "testError.cpp"
    ],
    "directory": "/home/iusk/Documents/Bitusk/testing", //当前目录
    "file": "/home/iusk/Documents/Bitusk/testing/testError.cpp", //输入的文件
    "output": "/home/iusk/Documents/Bitusk/testing/testError.o" //目标文件
  }
]

```

这个文件为项目每个源代码文件提供编译指令，它通常是由其他工具产生。

<img src="/home/iusk/Documents/essay/picture/image-20210509154954440.png" alt="image-20210509154954440" style="zoom: 50%;" />

###### CMake-based projects

如果你是使用cmake构建你的项目，那么只需要跑一下这个命令：

```shell
$ cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=1
```

compile_commands.json会写到你的build目录。如果你的build目录不同，那么你可以软链接或者直接复制过去。

```shell
$ ln -s ~/myproject/compile_commands.json ~/myproject-build/
```

###### Other build systems, using [Bear](https://github.com/rizsotto/Bear) 

Bear 是一个通过记录这个build过程来生成compile_commands.json的工具。对于make-based build，只需要跑一下：

```shell
$ make clean; bear -- make
```





###### ref.

https://clangd.llvm.org/installation.html#project-setup

[compile_commands.json详解](https://clang.llvm.org/docs/JSONCompilationDatabase.html)

https://github.com/rizsotto/Bear