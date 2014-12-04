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

### 3.new/delete 底层实现
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

