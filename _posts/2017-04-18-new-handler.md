
# C++ 内存分配(new，operator new)详解


本文主要讲述C++ new运算符、operator new、placement new 之间的种种关联，以及operaor new的重载。
> **提示：** STL容器所使用的heap内存是由容器所拥有的分配器对象（allocator objects)管理，不是被new和delete直接管理。
gcc 3.4.6 默认的allocator是不带cache的,只是对new/delete的简单封装。stl而出现内存没有回收的问题，那么一定是libc的cache没有释放，并不是stl的原因

-------------------

## 了解new-handler的行为

当operator new无法满足某一内存分配需求时，它会抛出异常，在抛出异常之前，它会先调用一个客户指定的错误处理函数，即new-handler。


```C++
namespace std {
    typedef void (*new_handler)();
    new_handler set_new_handler(new_handler p) throw();
}
```
set_new_handler是一个获取new_handler并返回一个旧的new_handler的函数。
当operator new无法满足内存申请时，它会不断调用new-handler函数，直到找到足够的内存。
一个设计良好的new-handler函数必须做以下事情：
<br />1. 让更多内存可被使用。
<br />2. 安装新的new-handler。
<br />3. 卸载旧的new-handler。
<br />4. 抛出bad_alloc（或派生自bad_alloc)的异常。该异常不会被operator new捕捉，因此会被传播到内存索求处。
<br />5. 不返回，通常调用abort或exit。
## new运算符和operator new()

>  operator new can be called explicitly as a regular function, but in C++, new is an operator with a very specific behavior: An expression with the new operator, first calls function operator new (i.e., this function) with the size of its type specifier as first argument, and if this is successful, it then automatically initializes or constructs the object (if needed). Finally, the expression evaluates as a pointer to the appropriate type. 
&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;;&emsp;&emsp;;&emsp;&emsp;;&emsp;&emsp;;&emsp;&emsp;;&emsp;&emsp;;&emsp;&emsp;;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;&emsp;—— [new运算符和operator new](https://www.cplusplus.com)

 比如我们写下代码：A* a = new A;
 实际上这里分为两步： 1. 分配内存   2. 构造对象
&emsp;&emsp;**分配内存**，该操作实际上就是调用operator new(size_t)完成。
&emsp;&emsp;如果类A重载了operator new，那么调用A::operator new(size_t)。
&emsp;&emsp;如果类A没有重载，那么调用::operator new(size_t).

## operator new的三种形式
```cpp
throwing (1):void* operator new(std::size_t size) throw (std::bad_alloc);
nothrow  (2):void* operator new(std::size_t size, const std::nothrow_t& nothorw_value) throw();
placement(3):void* operator new(std::size_t size, void* ptr) throw()
```


(1)和(2)的区别仅是是否抛出异常，当分配失败时，(1)会抛出bad_alloc，(2)返回null且不抛出异常。
用法实例：
``` C++
A* a = new A; //调用throwing(1)
A* a = new(std::nothrow) A; //调用nothrow(2)
```
(3)是placement new，它也是对operator new的一个重载，定义于&lt;new>中，它多接收一个prt参数，但它只是简单地返回ptr。其在new.h下的源代码如下:
``` C++
#ifndef __PLACEMENT_NEW_INLINE
#define __PLACEMENT_NEW_INLINE
inline void *__cdecl operator new(size_t, void *_P)
        {return (_P); }
#if     _MSC_VER >= 1200
inline void __cdecl operator delete(void *, void *)
    {return; }
#endif
#endif
```
那么placement new的作用是什么呢？
实际上，它可以实现在ptr所指向的地址上构建一个对象（通过调用构造函数），这在内存池技术上有广泛应用。
调用形式为：
``` C++
    new(p) A(); //也可以是A(4)等带参构造函数
```

## placement new存在的理由
1. 解决buffer的问题
&emsp; 若想在预分配的内存上创建对象，用缺省的new操作符是行不通的。
2. 时空效率问题
&emsp; 使用new操作符分配内存需要在堆中先查找足够大的内存，这个操作耗时较多，而且可能会出现内存无法分配的异常。placement new就可以解决这个问题，我们先提前分配好内存，在需要的时候直接构造对象。这种方式非常适用于对时间要求比较高的程序。

## operator new重载
operator new重载可以分为**类中重载**和**全局重载**。

> **提示：**placement new(3)是重载operator new的一个标准、全局的版本，它不能够被自定义的版本替代，而(1)、(2)可以被重载。

1. 类中重载
可以在类中直接重载(1)、(2)。
```
#include <iostream>
class A
{
public:
    A() {
        std::cout<<"call A constructor"<<std::endl;
    }

    ~A() {
        std::cout<<"call A destructor"<<std::endl;
    }
    void* operator new(size_t size) {
        std::cout<<"call A::operator new"<<std::endl;
        return malloc(size);
    }

    void* operator new(size_t size, const std::nothrow_t& nothrow_value) {
        std::cout<<"call A::operator new nothrow"<<std::endl;
        return malloc(size);
    }
};
int _tmain(int argc, _TCHAR* argv[])
{
    A* p1 = new A;
    delete p1;

    A* p2 = new(std::nothrow) A;
    delete p2;

    system("pause");
    return 0;
}
```

注意，这里的重载遵循作用域覆盖原则，即在里向外寻找operator new的重载时，只要找到operator new函数就不再向外查找了，如果参数符合则通过，如果参数不符合则报错，而**不管全局是否还有相匹配的函数原型**。比如，仅仅注释掉类A内部的operator new(size_t  size)函数，那么调用A* p2 = new(std::nothrow) A就会报错。
理所当然，operator new的重载还可以添加自定义的参数，如在类A中添加
``` 
void* operator new(size_t size, int x, int y, int z)
{
    std::cout << "X=" << x << "  Y=" << y << " Z=" << z << std::endl;
    return malloc(size);
}
```
这类重载被巧妙应用于动态内存分配调试和检测中。在本文后续的内容会进行展现。
2. 全局重载
略。
3. 全局::new
::new会直接调用全局operator new，并且也会调用构造函数。

## operator new运用技巧和一些实例探索
1. operator new重载运用于调试
我们可以利用自定义参数的operator new重载来获取更多的信息，比如一个很有用的做法就是给operator new添加两个参数：char * file, int line，这两个参数记录new运算符的位置，然后再在new时将文件名与行号导入，这样我们就能再分配内存失败时给出提示：输出文件名和行号。

```
//A.h
class A
{
public:
    A()
    {
        std::cout<<"call A constructor"<<std::endl;
    }

    ~A()
    {
        std::cout<<"call A destructor"<<std::endl;
    }

    void* operator new(size_t size, const char* file, int line)
    {
        std::cout<<"call A::operator new on file:"<<file<<"  line:"<<line<<std::endl;
        return malloc(size);
        return NULL;
    }

};
//Test.cpp
#include <iostream>
#include "A.h"
#define new new(__FILE__, __LINE__)

int _tmain(int argc, _TCHAR* argv[])
{
    A* p1 = new A;
    delete p1;

    A* p2 = new A;
    delete p2;

    system("pause");
    return 0;
}
```

> **提示：**需要将类的声明实现与new的使用分割开来，并且将类头文件放在宏定义之前。否则在类A中operator new重载中的new会被宏替换掉。

## delete的使用
TODO
## 关于new和内存分配的其他

---------
参考资料：
[1][C++ 内存分配(new，operator new)](http://blog.csdn.net/wudaijun/article/details/9273339 )
[2][C++中的new、operator new与placement new](http://www.cnblogs.com/luxiaoxun/archive/2012/08/10/2631812.html)
[3][C++ 工程实践(2)：不要重载全局 ::operator new()](http://blog.csdn.net/solstice/article/details/6198937)
[4][C++ In Action:Techniques](http://www.relisoft.com/book/tech/9new.html)

