**关于inline对链接时的影响**

问题：
在分别编译以下代码成目标文件，然后链接报错undefined reference..

```shell
iusk@wgnmd:~/Documents/Bitusk/testing$ g++ -static testError.o ../src/Error.o
/usr/bin/ld: testError.o: in function `main':
testError.cpp:(.text+0x2b): undefined reference to `ErrorCatcher::invalidHashError[abi:cxx11]()'
collect2: error: ld returned 1 exit status

```

```C++
//File: include/Error.h
class ErrorCatcher{
public:
  static const std::string invalidFd();
};
```
```c++
//File: src/Error.cpp
inline const std::string ErrorCatcher::invalidFd() {
  return "INVALIDFD";
}
```

```c++
//File: testing/testError.cpp
#include "Error.h"
int main(int argc, char* argv[]) {
	std::cout<<ErrorCatcher::invalidFd()<<std::endl;
	return 0;
}
```



认真排除了常见的undefined ref的常见错误都没问题[^其余常见问题可见Ref.]，使用nm查看.o文件的函数符号发现并没有testError.o 所需要符号。如下所示：

```shell
iusk@wgnmd:~/Documents/Bitusk/testing$ nm -C testError.o 
//省略不必要的符号...
0000000000000280 t __static_initialization_and_destruction_0(int, int)
                 U ErrorCatcher::invalidFd[abi:cxx11]()
//......
```

*nm的详细使用可参照*[^2]

```shell
iusk@wgnmd:~/Documents/Bitusk/src$ nm -CA *.o
BtError.o:0000000000000000 r std::piecewise_construct
```

 由此可见编译器直接***inline***了所以定义的**static member func**，**symbol table**中没有所需要的符号，自然ld（链接器）不能找到所需要的符号，也就报错了。这是一个很微妙的问题。将.o文件归档为libmy.a链接时也会出现同样的报错。

解决办法：

在库文件中，对外提供链接符号的函数不能**inline**。在C++中放在声明里面的成员函数将会默认inline，若是对外提供链接符号，可能会有问题 ?? //TODO 实验？

//TODO 动态库尚未实验过



Q：为什么这儿编译会通过？？



Ref.

[1]  [htt"undefined reference to XXX"问题总结](https://zhuanlan.zhihu.com/p/81681440)

[2]  [linux下强大的文件分析工具 -- nm](https://www.cnblogs.com/downey-blog/p/10477835.html)

[3]  [Linux下Gcc编译链接静态库&动态库](https://www.cnblogs.com/thechosenone95/p/10605172.htm)

[4]  [Shared Libraries With GCC on linux](https://www.cprogramming.com/tutorial/shared-libraries-linux-gcc.html)

[5]  [Definitions And ODR(one definition rule)](https://en.cppreference.com/w/cpp/language/definition)

[6]  [static inline vs inline vs static in C++](https://gist.github.com/htfy96/50308afc11678d2e3766a36aa60d5f75#file-static_inline_example-md)



