---
layout: post
title: "C++内存管理：原语篇"
categories:
- C++ 
tags:
- memory management  
- memory primitives
---
##目录
[TOC]
### 1. 说明
　　C++内存管理:原语篇是C++内存管理系列第一篇,主要讲C++中的new/delete expression、operator new/delete、array new/delete以及placement new/delete.文章中涉及到的部分的C++ 标准库源码都是来源于libstdc++-v3.　 
### 2. 应用程序和内存管理的关系
　　如下图，应用程序可以直接调用C++标准库提供的分配器分配内存（一般容器使用），可以直接使用new/array new等基本原语分配内存，也可以调用C Runtime Library提供的malloc分配内存,甚至可以直接调用操作系统提供的系统调用分配内存。图中的每一层都有一个指向下一层的指针，表示上层一般是调用下一层来实现。
![应用程序与内存管理的关系图](/assets/application_and_memory.jpg)

### 3. new/delete 底层实现
　　new/delete操作符属于语言内置，不能重载。当我们调用如下代码时：

`````
    Complex* cptr = new Complex(1,2);
`````
 　　编译器实际上将我们的代码转化为：

`````C++
    Complex* cptr;
    try{
        void* mem = ::operator new(sizeof(Complex));
        cptr = static_cast<Complex*>(mem);
        cptr->Complex::Complex(1,2);
    }catch(std::bad_alloc){
        //内存分配失败的时候将不执行构造函数，直接抛出异常
    }
`````
　　也就是说，调用new的时候，由三步组成，一是调用operator new分配内存，第二是编译器对指针做转型，第三是调用构造函数。

　　对于delete操作符，当我们调用如下代码时：

`````
    Complex* cptr  = new Complex(1,2);
    ...//do something
    delete cptr;
`````
　　编译器实际上将我们的代码转化为：

`````C++
    cptr->~Complex();
    ::operator delete(cptr);
`````

　　也就是说，调用delete的时候，由两步组成，先调用析构函数，然后释放内存。
　　按照上面的叙述，做如下实验验证。

`````C++
    #include<iostream>
    using namespace std;
    struct A{
        int a;
        A(int data):a(data){
            cout<<"A constructor.this pointer="<<this<<endl;
        }
        ~A(){
        cout<<"A destructor.this pointer="<<this<<endl;
        }
        void print(){
        cout<<"a="<<a<<endl;
        }
    };
    int main(int argc,char *argv[]){
        void* mem = ::operator new(sizeof(A));
        cout<<"mem address="<<mem<<endl;
        A* ptr = static_cast<A*>(mem);
        ptr->A(10);//在vs2013中可以通过编译，gcc中这一句不能通过编译
        A::A(9);//构造了一个临时变量
        ptr->print();
        ptr->~A();
        ::operator delete(ptr);
        return 0;
    }
`````
　　程序输出如下(vs2013环境下)：

``````
    mem address=0x8739008
    A constructor.this pointer=0x8739008
    A constructor.this pointer=0xbfda7850
    A destructor.this pointer=0xbfda7850
    a=10
    A destructor.this pointer=0x8739008
``````
　　可以看出，operator new分配的那一块内存的起始地址就是this指针的地址。A::A(9);是在栈上分配空间构造了一个临时变量，这个临时变量的生命周期存在于这一个语句，所以程序输出中的第三四句打印了构造和析构两条语句。
### 4. operator new/operator delete底层实现
　　new/delete expression分配/释放内存实际上调用operator new/operator delete完成，new/delete是一个表达式，显然不能重载，operator new/operator delete是函数，所以我们可以重载operator new/operator delete。对于operator new/delete来说，重载有两种形式，一是全局重载，二是类重载，一般来说，不会重载全局的operator new/delete,因为全局的operator new/delete只是简单的malloc/free，代码如下：

`````C++
　　operator new (std::size_t sz) _GLIBCXX_THROW (std::bad_alloc){
        void *p;
        /* malloc (0) is unpredictable; avoid it.  */
        if (sz == 0)//c++标准要求，即使在请求分配0字节内存时，operator new也要返回一个合法指针
            sz = 1;
  
        while (__builtin_expect ((p = malloc (sz)) == 0, false)){
            new_handler handler = std::get_new_handler ();//分配不成功，找出当前出错处理函数
            if (! handler)//如果用户没有定义内存分配错误处理函数，直接抛出bad_alloc异常，否则进入错误处理函数
                _GLIBCXX_THROW_OR_ABORT(bad_alloc());
            handler ();
        }
        return p;
    }　
`````

　　从上述代码中可以看到，operator new实际上会不只一次地尝试着去分配内存，它要在每次失败后调用出错处理函数(如果用户设置了错误处理函数)，还期望出错处理函数能想办法释放别处的内存。只有在指向出错处理函数的指针为空的情况下，operator new才抛出异常。那么首先看看这个handler怎么设置。直接上源码：

`````C++
    typedef void (*new_handler)();
    //Takes a replacement handler as the argument, returns the previous handler.
    new_handler set_new_handler(new_handler) throw();
    //Return the current new handler.
    new_handler get_new_handler() noexcept;
`````

　　set_new_handler函数接收一个参数为空返回值为空的函数指针为参数，返回一个参数为空返回值为空的函数指针，对于一个设计良好的new handler，最好做到下列事情之一[^1]：<br>
　　1. 让更多的memory可用<br>
　　2. 安装另一个new handler<br>
　　3. 丢出一个exception，类型为bad_alloc或者其衍生类型<br>
　　4. 直接调用abort或者exit


　　缺省的operator new和operator delete具有非常好的通用性，它的这种灵活性也使得在某些特定的场合下，可以进一步改善它的性能。尤其在那些需要动态分配大量的但很小的对象的应用程序里，情况更是如此。所以从效率上来讲,class可以接管内存管理责任，对operator new进行类重载。

　　如果对operator new进行了类重载，也要对operator delete也要进行重载，关于这点，可以参见侯捷老师翻译的《Effective C++》第十条。

　　关于如何对operator new/delete进行类重载，下一篇内存管理：自己动手写内存池会详细写到，这里暂时不详述。这里提一个小细节，类重载operator new/delete函数的时候，重载函数应该是static类型，因为当我们new一个对象的时候，根据new的底层实现可知，new第一步其实是调用了operator new，那么当我们重载了operator new，第一步就会调用我们重载的operator new函数，此时对象还没有分配内存以及初始化，也就是说对象此时还不存在，所以也不存在利用对象来调用函数，只能通过类调用，所以operator new重载函数应该是static类型。不过很多时候，我们知道static可以不写，这是因为编译器帮我们做了优化，你不写编译器就帮你加上，不过我们心里应该清楚这里是需要加上static的。


### 5. placement new/delete底层实现

### 6. array new/delete底层实现

[^1]: 侯捷-《Effective C++》
