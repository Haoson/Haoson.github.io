---
layout: post
title: "C++内存管理：原语篇"
categories:
- C++ 
tags:
- memory management  
- memory primitives
---
- 目录
{:toc}

### 1. 说明
　　C++内存管理:原语篇是C++内存管理系列第一篇,主要讲C++中的new/delete expression、operator new/delete、array new/delete以及placement new/delete.文章中涉及到的部分的C++ 标准库源码都是来源于libstdc++-v3.　 

### 2. 应用程序和内存管理的关系
　　如下图，应用程序可以直接调用C++标准库提供的分配器分配内存（一般容器使用），可以直接使用new/array new等基本原语分配内存，也可以调用C Runtime Library提供的malloc分配内存,甚至可以直接调用操作系统提供的系统调用分配内存。图中的每一层都有一个指向下一层的指针，表示上层一般是调用下一层来实现。
![应用程序与内存管理的关系图](/assets/application_and_memory.jpg)

### 3. new/delete 底层实现
　　new/delete操作符属于语言内置，不能重载。当我们调用如下代码时：

        Complex* cptr = new Complex(1,2);

 　　编译器实际上将我们的代码转化为：

        Complex* cptr;
        try{
            void* mem = ::operator new(sizeof(Complex));
            cptr = static_cast<Complex*>(mem);
            cptr->Complex::Complex(1,2);
        }catch(std::bad_alloc){
            //内存分配失败的时候将不执行构造函数，直接抛出异常
        }

　　也就是说，调用new的时候，由三步组成，一是调用operator new分配内存，第二是编译器对指针做转型，第三是调用构造函数。<br>
　　对于delete操作符，当我们调用如下代码时：

        Complex* cptr  = new Complex(1,2);
        ...//do something
        delete cptr;

　　编译器实际上将我们的代码转化为：

        cptr->~Complex();
        ::operator delete(cptr);

　　也就是说，调用delete的时候，由两步组成，先调用析构函数，然后释放内存。<br>
　　按照上面的叙述，做如下实验验证。

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

　　程序输出如下(vs2013环境下)：

        mem address=0x8739008
        A constructor.this pointer=0x8739008
        A constructor.this pointer=0xbfda7850
        A destructor.this pointer=0xbfda7850
        a=10
        A destructor.this pointer=0x8739008

　　可以看出，operator new分配的那一块内存的起始地址就是this指针的地址。A::A(9);是在栈上分配空间构造了一个临时变量，这个临时变量的生命周期存在于这一个语句，所以程序输出中的第三四句打印了构造和析构两条语句。

### 4. operator new/operator delete底层实现
　　new/delete expression分配/释放内存实际上调用operator new/operator delete完成，new/delete是一个表达式，显然不能重载，operator new/operator delete是函数，所以我们可以重载operator new/operator delete。对于operator new/delete来说，重载有两种形式，一是全局重载，二是类重载，一般来说，不会重载全局的operator new/delete,因为全局的operator new/delete只是简单的malloc/free，代码如下：

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
        void operator delete(void* ptr) _NOEXCEPT{
            if (ptr)
                ::free(ptr);
        }

　　从上述代码中可以看到，operator delete实际上只是简单的调用CRT 提供的free来释放内存，operator new调用malloc来分配内存，但是operator new会不只一次地尝试着去分配内存，它要在每次失败后调用出错处理函数(如果用户设置了错误处理函数)，还期望出错处理函数能想办法释放别处的内存。只有在指向出错处理函数的指针为空的情况下，operator new才抛出异常。那么首先看看这个handler怎么设置。直接上源码：

        typedef void (*new_handler)();
        //Takes a replacement handler as the argument, returns the previous handler.
        new_handler set_new_handler(new_handler) throw();
        //Return the current new handler.
        new_handler get_new_handler() noexcept;

　　set_new_handler函数接收一个参数为空返回值为空的函数指针为参数，返回一个参数为空返回值为空的函数指针，对于一个设计良好的new handler，最好做到下列事情之一[^1]：

* 让更多的memory可用
* 安装另一个new handler
* 丢出一个exception，类型为bad_alloc或者其衍生类型
* 直接调用abort或者exit

　　缺省的operator new和operator delete具有非常好的通用性，它的这种灵活性也使得在某些特定的场合下，可以进一步改善它的性能。尤其在那些需要动态分配大量的但很小的对象的应用程序里，情况更是如此。所以从效率上来讲,class可以接管内存管理责任，对operator new进行类重载。<br>
　　如果对operator new进行了类重载，也要对operator delete也要进行重载，关于这点，可以参见侯捷老师翻译的《Effective C++》第十条。<br>
　　关于如何对operator new/delete进行类重载，下一篇内存管理：自己动手写内存池会详细写到，这里暂时不详述。这里提一个小细节，类重载operator new/delete函数的时候，重载函数应该是static类型，因为当我们new一个对象的时候，根据new的底层实现可知，new第一步其实是调用了operator new，那么当我们重载了operator new，第一步就会调用我们重载的operator new函数，此时对象还没有分配内存以及初始化，也就是说对象此时还不存在，所以也不存在利用对象来调用函数，只能通过类调用，所以operator new重载函数应该是static类型。不过很多时候，我们知道static可以不写，这是因为编译器帮我们做了优化，你不写编译器就帮你加上，不过我们心里应该清楚这里是需要加上static的。<br>


### 5. placement new/delete底层实现
　　placement new是重载operator new的一个标准、全局的版本，它不能被自定义的版本代替（不像普通的operator new和operator delete能够被替换成用户自定义的版本）。看一下placement new的源码如下：

        // Default placement versions of operator new.
        inline void* operator new(std::size_t, void* __p) _GLIBCXX_USE_NOEXCEPT{ return __p; }
        inline void* operator new[](std::size_t, void* __p) _GLIBCXX_USE_NOEXCEPT{ return __p; }
        // Default placement versions of operator delete.
        inline void operator delete  (void*, void*) _GLIBCXX_USE_NOEXCEPT { }
        inline void operator delete[](void*, void*) _GLIBCXX_USE_NOEXCEPT { } 

　　placement new可以实现在ptr所指地址上构建一个对象(通过调用其构造函数),这里的ptr可以指向动态分配的堆上内存，也可以指向栈上内存。一般而言placement new使用场景非常少，它的最简单应用就是可以将对象放置在内存中的特定位置，示例代码如下：

        #include <new>        // Must #include this to use "placement new"
        #include "Fred.h"     // Declaration of class Fred
        void someCode(){
            char memory[sizeof(Fred)];
            Fred* f = new(memory) Fred();
            //do something
            f->~Fred();   // 显示调用对象的析构函数
        } 

　　一般来说，我们应该尽可能不使用placement new，除非我们真的很关心创建的对象在内存中的位置，for example，when your hardware has a memory-mapped I/O timer device, and you want to place a Clock object at that memory location[^2]。还有，我们使用placement new构造对象，当对象调用结束后，我们需要显示的调用对象的析构函数。最最重要的是，一旦我们使用operator new之后，也就意味着我们接管了传给placement new的指针所指的内存区块的职责，这里有两个职责，一是内存区域是否足够容纳对象；二是如果区域是否满足字节对齐（如果对象需要字节对齐）。<br>
　　还要注意的是，在C++标准中，对于placement operator new []有如下的说明： placement operator new[] needs implementation-defined amount of additional storage to save a size of array. 所以我们必须申请比原始对象大小多出sizeof(int)个字节来存放对象的个数，或者说数组的大小。<br>

### 6. array new/delete底层实现
　　当使用new分配数组的时候，原理基本同new操作符，不过分配内存是调用operator new[]（即array new），array new能被重载，分配完内存之后，在数组里的每一个对象的构造函数都会被调用。这里唯一需要注意的是编译器只会调用无参的构造函数，这里无法通过参数给予对象初值。<br>
　　当delete操作符用于数组时，它为每个数组元素调用析构函数，然后调用operator delete[](即array delete)来释放内存。array delete同样能被重载。<br>
　　对于array new/delete，源码如下：

        void* operator new[] (std::size_t sz) _GLIBCXX_THROW (std::bad_alloc){
            return ::operator new(sz);
        }
        void operator delete[] (void* ptr) _NOEXCEPT{
            ::operator delete (ptr);
        }

　　可以看到，array new/delete底层实质上还是调用operator new/delete来分配空间。<br>
　　这里提一个小细节，如果我们new数组之后使用delete ptr;(没有那个中括号)会发生什么呢？从上面的源代码可以看到，array delete底层还是调用::operator delete来释放内存的，参数是指向将要释放的区块的开始地址，所以分配的内存一定是会正确释放的，不可能有内存泄露。前面提到delete数组时候，第一步是为每个数组元素调用析构函数，如果我们delete数组时候忘记写中括号，那么编译器认为这是一个对象，编译器就会调用第一个元素的析构元素，其余元素的析构函数将不会被调用。

### 7. 其他
　　第一点，前面提到重载placement new时可能遇到alignment问题，这里强调一下，C++标准里要求了所有的operator new（placement new是重载operator new的一个标准、全局的版本）返回的指针都有适当的对齐（取决于数据类型）。malloc就是在这样的要求下工作，所以令operator new返回一个得自malloc的指针是安全的，那么什么时候会遇到alignment问题呢？举个例子，直接上代码:

        static const int cooike = 0xABABABAB;
        void* operator new(size_t size)throw(bad_alloc){
            size_t real_size = size+2*sizeof(int);
            void* mem = malloc(real_size);
            if(!mem)
                throw bad_alloc();
            //将cookie写入到区块的最前面和最后面
            *(static_cast<int*>(mem)) = cooike;
            *(reinterpret_cast<int*>(static_cast<unsigned char*>(mem)+real_size-sizeof(int))) = cooike;
            //返回指针，指向位于第一个cookie之后的内存位置
            return static_cast<unsigned char*>(mem)+sizeof(int);
        }

可以看到上述代码返回的其实并不是一个得自malloc的指针，这个指针可能就是一个没有适当对齐的指针。所以重载operator new（包括placement new）都需要格外小心。<br>
　　第二点，从上面的介绍可知，new/delete,operator new/delete或者array new/delete等实质上都是调用malloc/free来分配/释放空间，对于malloc和free的底层实现，后续文章再做分析。<br>
　　第三点，一个小细节，释放空间的时候，我们注意到free只需要一个指针参数，指向要释放的区块的开始地址，那么释放空间的时候怎么知道释放的空间大小呢？这里简单说一下，分配内存的时候，会记录一个cooike，cooike记录了分配内存的大小，一般来说，这个cookie放在区块开始地址的"上面"，释放的时候只需要根据开始地址再向上找4个字节就能得到区块大小...<br>

### 8. 总结
　　这篇博客主要介绍了C++内存分配中的一些基本原语的底层实现和一些要注意的点，下一篇我将利用这些原语实现一个自己的内存池。

### 9. 链接
[^1]: 侯捷-《Effective C++》第三版
[^2]: [What is "placement new" and why would I use it?](http://www.parashift.com/c++-faq/placement-new.html)
