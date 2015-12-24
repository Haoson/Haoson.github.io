---
layout: post
title: "从一行代码引发的bug说起"
categories:
- 技术 
tags:
- libcurl
- epoll
---
- 目录
{:toc}

### 1. bug描述
　　最近发现我们游戏的cardinfo进程的client会出现不够用（目前上限500）的情况。<br> 
1) cardinfo进程是一个与外部开放平台交互的进程，例如调用QQ开放平台实现游戏内一键加群绑群。 <br>
2) cardinfo进程底层使用libcurl库+epoll的事件模式与外部进行通信。这个外部包含两个，一是开放平台，二是cardzone（gamesvr）进程。 <br>
3) cardinfo进程的client概念是一个请求就产生一个client，这个请求是由cardzone发出的。一般流程是玩家点击进入帮派界面的时候就会向cardzone发        送一条查询帮派群的请求，cardzone转发请求到cardinfo进程，cardinfo进程向开放平台发出请求，收到异步消息之后向cardzone <br>
4) 正常情况下，请求结束之后，client马上释放，释放后的client可以供下一个请求使用。 <br>

### 2. libcurl概述
　　libcurl是一个开源的网络传输库，cardinfo进程使用libcurl实现http服务。这里简单介绍一下libcurl库，方便后续的bug分析。      
　　libcurl主要提供了两种接口：easy interface和multi interface。两者之间的区别主要是：前者只是最简单的阻塞式服务，后者可以实现非阻塞式服务。Cardinfo进程使用的是multi interface。multi interface是建立在easy interface基础之上的，它只是简单的将多个easy handler添加到一个multi stack，而后同时传输而已。      

#### 2.1 使用easy interface 
1) 创建一个easy handle，easy handle用于执行每次操作（数据传输）。 <br>
2) 在easy handle上设置属性和操作。使用curl_easy_setopt函数可以设置easy handle的属性和操作，设置这些属性和操作控制libcurl如何与远程主机进行数据通信。<br>
    　　对于easy interface的使用，关键在于curl_easy_setopt函数，通过这个函数，我们可以设置相关的属性，包括设置url，timeout，http协议（究竟是post，get还是其他），设置这些属性的同时还可以设置相应的回调函数，例如如果我们想对接收到的数据进行保存，可以设置CURLOPT_WRITEFUNCTION属性并注册回调函数：

        curl_easy_setopt(easy_handle, CURLOPT_WRITEFUNCTION, write_data);

3) 调用curl_easy_perform，将执行真正的数据通信。curl_easy_perfrom将连接到远程主机，执行必要的命令，并接收数据。当接收到数据时，先前设置的回调函数将被调用。<br>

#### 2.2 使用multi interface
1) 使用curl_multi_init()函数创建一个multi handler <br>
2) 使用curl_easy_init()创建一个或多个easy handler，按照上述介绍的使用easy interface来对每个easy handler设置相关的属性<br>
3) 通过curl_multi_add_handler将这些easy handler添加到multi handler <br>
4) 调用curl_multi_perform进行数据传输<br>

#### 2.3 使用epoll+multi interface（重点）
　　libcurl对大量请求连接提供了管理socket的方法，用户可使用select/poll/epoll事件管理器监控socket事件。Cardinfo就是通过epoll来操作multi interface的。其实每个easy handler在低层就是一个socket，通过epoll来管理这些socket，在有数据可读/可写/异常的时候，通知libcurl读写数据，libcurl读写完成后再通知用户程序改变监听socket状态。这里要重点关注两个属性的设置：

        curl_multi_setopt(multi, CURLMOPT_SOCKETFUNCTION, mycurl_socket_callback); 
        curl_multi_setopt(multi, CURLMOPT_TIMERFUNCTION, mycurl_timeout_callback);

详细的见下代码分析。

### 3. bug分析 

#### 3.1 第一阶段：分配释放有问题？
![bug日志记录](/assets/1.png)
从日志文件来看，分配client失败，找到执行失败的函数new_client，定义如下：
![new_client函数定义](/assets/2.png)
　　new_client逻辑很简单，从池子中取free block，成功就返回client指针，否则返回NULL。如果这里出错，那就是池子中的free block被耗尽。gdb断点确认下：
![gdb确认](/assets/3.png)
　　果然，所有block已经全部分配出去。get_freeblock函数是blocklist提供的接口，blocklist作为基础数据结构被很多项目使用，这里面出现bug的可能性微乎及微，这里不提供源码，实际上在排查过程中，也进入get_freeblock中断点查看分配过程，确实没有问题。<br>
那么可能出现问题的是没有正确释放，先看一下free_client函数的定义：
![gdb确认](/assets/4.png)
　　free_client的逻辑也是十分简单，free_block也是blocklist底层数据结构提供的接口，出现bug的可能性也很小。<br>
综上述，排除了分配释放函数出错的可能性。

#### 3.2 第二阶段： 请求太频繁，client上限不够？
　　目前client上限是500，一个请求等同于一个client，如果请求太频繁，是有可能导致分配失败。从两个方面验证这个假设：<br>
1) 从之前正常分配释放的日志来看：<br>
![分配client日志](/assets/5.png)
　　new一个client之后，处理请求，请求结束之后立马free client。对于这种短链接，即使某一时刻请求数飙升超过上限，之后的时候也应该有请求能够得到响应并打印日志。实际上在错误日志中我们发现一旦client满了之后，再也不能够处理任何请求，也就说，所有block一旦分配完毕，再也没有被释放出来。从这个角度来看，应该可以排除请求太频繁导致client超上限处理不过来的猜测。<br>
2) 分析分配频率和释放频率<br>
　　实际排查过程中我试图通过定量分析new client的频率和free client的频率来验证是否是请求太频繁的原因。对于new client频率的计算主要手段是统计日志，统计每分钟new 的数量，随机抽样十分钟，然后求平均数，基本可以得出平均每分钟new client的数量；对于free client的频率统计相对较麻烦，主要是通过分析调用方的调用频率，这里暂且不详细描述，可以参见第三阶段分析。<br>
　　这种分析方法非常麻烦，我也在这里浪费了不少时间，其实通过第一种的逻辑分析就可以排除请求太频繁的猜测，但是当时总觉得定量比定性分析准确。<br>

#### 3.3 第三阶段： 调用方出现了问题？
分配和释放的代码如下（做了简化处理）：
![new_client调用函数定义](/assets/6.png)
![free_client调用函数定义](/assets/7.png)
　　可以看出，new client调用时机 请求到达cardinfo进程的时候，free client有两处，一处是curl_multi_add_handle调用失败的时候，第二处是curl_multi_info_read从multi handler成功读取完数据的时候，也就是请求完成的时候释放。<br>
　　对于new client，可以看到没有并无任何令人迷惑的地方，断点调试之后也没发现异常，这里暂且不表，重点分析free client的两处场景，分析之前先把整个流程和重要函数调用介绍一遍：<br>
　　curl_multi_add_handle作用在libcurl概述章节有简单介绍。而且在libcurl概述章节中的使用用epoll+multi interface小节中，提到了两个重要属性的设置：

        curl_multi_setopt(multi, CURLMOPT_TIMERFUNCTION, mycurl_timeout_callback);
        curl_multi_setopt(multi, CURLMOPT_SOCKETFUNCTION, mycurl_socket_callback);

这里再次提到是因为调用curl_multi_add_handle的时候，最后就会调用mycurl_timeout_callback回调函数。先看一下这个回调函数的定义：
![mycurl_timeout_callback回调函数定义](/assets/8.png)
　　该回调函数的原型见官网[^1]，该函数的调用时机是当函数中的第二个参数（left）改变的时候就会被调用，第二个参数的改变由libcurl完成，先看着curl_multi_add_handle调用关系图是：

        curl_multi_add_handle-->update_timer-->mycurl_timeout_callback。

这里的update_timer就会改变left值，最终也就是的回调被调用。

再分析一下mycurl_timeout_callback：<br>
　　回调中先调用curl_multi_socket_action[^2]，然后调用mycurl_readinfo（见上定义）。curl_multi_socket_action将初始化请求并得到一个socket(fd)，然后调用上述提到的第二个重要参数设置的mycurl_socket_callback回调。先看看这个回调的函数定义（代码有简化）：
![mycurl_socket_callback回调函数定义-1](/assets/9.png)
![mycurl_socket_callback回调函数定义-2](/assets/10.png)
　　可以看到mycurl_socket_callback[^2]主要是为epoll注册事件，epoll wait函数在proc程序中，程序片段如下：
![epoll_wait使用](/assets/11.png)
　　可以看出，由于回调的存在，libcurl调用关系比较复杂，这里我把重要调用过程通过图的方式展示出来：
![libcurl调用关系](/assets/12.jpg)
　　分析完流程，回到bug分析，也就是free client到底有没有成功释放，可以看到两处free动作：

- 第一处是add handle未成功的时候，也就是数据传输之前，handle获取失败，直接free掉，这里相当于检查，通过错误日志来看，client满之前数据传输日志已经打印出来，所以排除掉这里出问题的可能。
- 第二处是在请求处理结束之后正常释放，如果这里出问题，那也就是handle状态不是CURLMSG_DONE,所以代码没有执行到这一步。继续看日志，由于mycurl_readinfo（定义见上）中打印了日志，
![日志打印](/assets/13.png)
且日志文件中也出现了要打印的内容，这里让人无比疑惑，到底是谁出错导致没释放？这个地方组内同事帮我做了两件很重要的工作：<br>
一是发现了日志文件中出现的一两次的错误（40多万条日志数据中出现了两次）：
![errno9错误](/assets/14.png)
二是为线上cardinfo进程的错误日志添加了更多输出信息：打印每次分配释放的序列号。<br>
　　对于错误日志中出现的errno=9，也就是bad file fd，文件描述符出现了问题！查看该进程的fd使用情况：
![socket fd状态](/assets/15.png)
Socket没有释放，而且随着时间会不断增多。为什么socket没有释放？程序中全部采用短链接，应该不会出现不释放的情况，继续分析。

　　通过分析新添加的带有分配释放序列号的日志文件，我分别统计了new client[2]和free client[2]出现的次数，new了332次，free了331次，果然没有释放。找到最后一次new client[2]打印的地方，可以看到：
![new_client详细日志](/assets/16.png)
　　仔细看这一段日志，可以发现new client[2]之后，epoll_ctl add fd=20成功，随后马上epoll_ctl del sock fd=20，但是这个删除fd的操作对象竟然是client[0]！<br>
　　程序中唯一一处操作socket fd的地方就是mycurl_readinfo（定义见上）中的close函数，这里逻辑是协议接收完毕之后，立马显示删除socket，然后给cardzone进程回包，free client等。


#### 3.4 bug总结
　　通过查看libcurl官方文档，可以知道默认情况下libcurl完成一个任务以后，出于重用连接的考虑不会马上关闭（每一个easy handle底层是一个socket），如果没有新的TCP请求来重用这个连接，那么只能等到CLOSE_WAIT超时。这种重用包含两层含义：一是对于同一个请求的重用，给定时间内新的请求过来可以直接连接；二是如果要将该handle对不同的请求进行重用，那么只需要curl_easy_reset[^3]来将该handle重新初始化到创建时的状态即可。两种情况下连接都会保持。如果我们手动close socket，虽然该handle已经完成了任务，但在高并发的情况下，很有可能close掉的是一个已经给其他请求重用的handle。这样问题就出现了，一旦close掉了被分配出去的socket，那么在该socket上将不会有数据传输，如果不会有数据传输，那么curl_multi_info_read执行的时候该CURL的状态就不会走到CURLMSG_DONE态，如果走不到该状态，那么free client将不会被调用。

### 4. 困难&经验 
* 调试的时候遇到的最大的困难是无法重现，通过上述分析可知bug重现的条件是在高并发下有概率的出现，所以在线下重现该bug的时候浪费了相当多的时间；
* Libcurl库有非常多的坑，各种异步、回调很容易让人踩坑，这里还是不涉及多线程的情况，如果一旦有多线程，问题会更复杂；
* 最大的收获是错误日志是调试时候的好东西，特别是对于线上问题，没法直接在线上debug，只有靠日志。

### 5. 链接 
[^1]: [http://curl.haxx.se/libcurl/c/CURLMOPT_TIMERFUNCTION.html](http://curl.haxx.se/libcurl/c/CURLMOPT_TIMERFUNCTION.html)
[^2]: [http://curl.haxx.se/libcurl/c/curl_multi_socket_action.html](http://curl.haxx.se/libcurl/c/curl_multi_socket_action.html)
[^3]: [http://curl.haxx.se/libcurl/c/curl_easy_reset.html](http://curl.haxx.se/libcurl/c/curl_easy_reset.html)
* [How to reuse connection using curl multi handle?](http://curl.haxx.se/mail/lib-2014-12/0003.html)
* [High Performance Libcurl Tips](https://moz.com/devblog/high-performance-libcurl-tips/)
* [CURL异步调用和遇到的CURL内部问题](http://blog.chinaunix.net/uid-20384806-id-1954347.html)
