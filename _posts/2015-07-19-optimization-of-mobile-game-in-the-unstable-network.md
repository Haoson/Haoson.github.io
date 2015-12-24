---
layout: post
title: "手游弱网络通信优化"
categories:
- 技术 
tags:
- optimization
- mobile game
---
##目录
* TOC
{:toc}

### 1. 移动网络的弱点
1. 网络制式多：2G/3G/4G/wifi； <br>
2. 各种制式之间网络速度差异大； <br>
3. 地理位置变动中网络切换（如发生制式之间的切换、基站之间的切换）等； <br>
4. 信号强度弱（建筑死角, 隧道, 深山老林, 无线信号会衰弱）；<br>
5. 信号不稳定（如在高速移动的火车上, 会有明显的多普勒效应, 网络延迟抖动）；<br>
6. 网络拥塞（人口密集的场所, 运营商降低人均通信带宽, 网络延迟增大）。

### 2. 弱网给游戏带来的影响 
　　简单来说, 弱网络环境会有更高的丢包率, 更大的延迟抖动, 不稳定的网络连接。具体到对游戏的影响，有三个方面：      

1. 网速慢导致请求响应时间变长甚至断线；<br>
2. 网络不稳定导致上行包/下行包丢失； <br>
3. 弱网络加剧异步设计系统的响应延迟。

### 3. 弱网处理方案 
　　针对上述提到的弱网给游戏带来的三点影响，下面分别给出方案处理。      

#### 3.1 如何解决网速慢导致响应延迟？
-  客户端保证一发一/多收模式
　　大部分协议都有一个定时器，在超时时间内，客户端期待服务端回包。根据协议的不同，服务端回包的个数也不同，例如对玩家购买金币的操作，服务端在回多个包，包括扣钻石的回包、反馈操作是否成功的回包等。不管是一发一收还是一发多收模式，一般情况下，每一个上行包都有一个对应的反馈下行包来表示操作结束，在这个反馈下行包之前，服务端也可能会发送别的回包给客户端。
　　补充一点的是，有一些协议不需要期待回包，所以客户端也会启动一个协议白名单机制，在白名单之中的协议，客户端发包之后不期待回包就可以继续操作。常见的有GM相关的协议、平台请求相关协议等都不需要期待回包。
　　在超时时间之内，客户端对UI交互进行锁定（模态菊花），也就是说，超时间时间之内，还没有期待到服务端的回包的情况下，玩家无法进行下一步的操作。为了保证玩家的体验，模态菊花会有一个淡入的效果，延迟越低模态菊花越不明显。
　　如果在超时时间之内客户端没有收到回包，此时客户端会启动超时探测。
-  客户端超时探测
　　对于在超时时间内没有期待到回包的情况，客户端会主动向服务器发送一个探测包，这个探测包只包含一个包头，不携带任何信息。在发送探测包的同时，客户端同样为这个探测包启动一个定时器，如果在超时时间内探测包有回包，我们认定上行包丢失（也有下行包丢失的可能，下面会详谈这个问题），直接忽略这种情况；如果探测包在超时时间内也没有回包，那么这时候我们断定是网络出现了问题，直接弹出提示框告知玩家请求失败，由玩家手动点击重连进入断线重连程序。
-  断线重连
　　以分区分服玩法的游戏为例，玩家完整登陆的过程是首先通过接入层的负载均衡机制连接到一个game zone上，然后以选中的zone为载体，通过DBProxy从DB中拉取角色数据。理想的状态下，想要优化玩家体验，断线重连过程就应该不需要重新走原来复杂的流程，这里又可以分几个小点优化：
1) 登陆简化
　　弱网环境下，玩家出现掉线的情况比较频繁，所以登陆流程需要细分，区分完整登陆和断线重连的情况，例如客户端在断线重连的时候可以简化鉴权。
2) 延迟下线机制
　　当玩家出现掉线后，服务端不马上清除玩家角色数据，而是为玩家角色数据设置一个延迟下线时间（如5分钟），然后在主程序中的时间事件里（tick定时回调）检查角色数据状态，如果玩家在时间未过期之前重新连接上，就可以直接将角色数据下发客户端，否则清除数据，下次玩家重新登陆上走完整登陆流程。
3) 会话机制
　　前面说到玩家登陆是通过接入层的负载均衡连接到一个game zone上的，如果玩家重连上的不是之前的那个zone，那么即使有延迟下线机制，还是得走完整登陆流程，所以这里还需要开启接入层的会话机制配合延迟下线机制。接入层的会话机制就相当于负载均衡的逻辑当中有一个保护时间，在这个保护时间之内玩家通过接入层连接到一个zone的时候将不再执行负载均衡，而是直连到我们的zone上。这里一个小细节是接入层的会话机制设置的时间最好与延迟下线的时间一致。

#### 3.2 如何解决丢包问题？
　　弱网络环境导致丢包具体来说有两种情况，一是上行包丢失，即服务器并未收到请求协议，客户端只需要重新请求即可；二是下行包丢失，服务器已经收到请求协议并处理，由于网络问题，客户端未收到回包，客户端不知道服务器是否已处理以及处理的返回数据是什么。我们这里主要讨论的是如何解决下行包丢失的问题。
　　下行包丢失又可以细分两种情况，一是对于客户端收不到回包不会影响玩家收益的请求，则不需要做特别处理，这类请求回包丢失时，玩家一般可以重新登陆或刷新界面可以恢复数据的请求，典型的有领取任务奖励；二是对于客户端收不到回包会影响玩家的请求，例如战斗过程中加血，如果这类关键请求在超时时间内没有收到回包，客户端需要自动重复发送上次请求，如果是网络问题，走断线重连流程，如果是下行包丢失，则需要相应的机制来处理。主要手段是消息会话机制。

-  消息会话机制
　　消息会话机制的实现思路是客户端和服务端配合工作，首先客户端为每个协议请求分配一个唯一的会话ID（序列号），当客户端进行重复操作的时候，如果上一个请求没有收到服务端的响应，这个时候就会分配相同的会话ID。服务端收到相同的会话ID则认为是重复的请求，重复的请求本质上是上一次请求的历史回放，应对回放的请求，最直接的解决方案就是返回回放的回复数据。因此只需要在逻辑处理完“非重复请求”后以及将回复数据提交网络层返回客户端之前，对回放数据进行缓存，在后续遇到这次回复的重复请求时，绕过逻辑处理，直接将之前缓存的回复数据返回客户端即可。
　　实现上，会话ID是在玩家初次完整登陆的时候服务端进行初始化（可以随机产生），并下发客户端，客户端上行消息中的ID需要跟当前的服务器一致（最近一次收到的下行包中的ID），服务端收到客户端注册的协议请求且检查ID合法后，将下行发送的ID加一。这里服务端在检查ID的时候，就可以发现哪些消息是重复消息，能够知道是否已经处理过该消息，更进一步，服务端可以进行下行包缓存处理，当处理重复请求时，从缓存中取出最旧的下行包ID与客户端重复上报的下行包ID比较，只要还在缓存中的，就可以依次下发满足条件的包，如果客户端上报的下行包ID过旧，下发可能失败，服务端回复错误包，客户端可以主动发起数据同步。

#### 3.3 如何解决异步设计系统在弱网下的表现？
　　公司目前游戏服务端大多采用多进程异步通信模型，这势必存在大量复杂的网络设计，很多功能存在着异步操作，例如访问公司关系链。这种系统架构下，消息经过的环节越多，相当于逻辑越复杂，中间流程耗时就越多，如果这时候碰上弱网络，相当于这么复杂的异步逻辑将会重复得到执行，这也就加剧了弱网络的负担。从上面分析可知，我们无法避免异步通信带来的开销，但是我们可以优化弱网络情况下异步通信的重复消耗，主要手段是数据缓存。

-  数据缓存
　　数据缓存机制其实已经在上面提到，例如下行包的缓存来优化消息丢包引起的重复请求问题，玩家角色数据的缓存来优化断线重连等。对于异步设计系统，我们可以缓存平台信息、用户关系链数据等等，以此来避免弱网络下重复的异步数据传输或者异步逻辑。

### 4. 结语 
　　文章笼统的给出了一些手游弱网络优化手段，这些手段对不同类型的手游可能并不都适用或者必要，在技术之外我们也要考虑实现这些手段的性价比。