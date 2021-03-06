# RPC

RPC(Remote Procedure Call):远程过程调用。



## 为什么要RPC？

两个不同的服务器上的服务提供的方法不在一个内存空间，所以需要通过网络编程才能传递方法调用所需要的参数。并且，方法调用的结果也需要通过网络编程来接收。但是，如果我们自己手动网络编程来实现这个调用过程的话工作量是非常大的，因为，我们需要考虑底层传输方式（TCP还是UDP）、序列化方式等等方面。

通过RPC可以帮助我们调用远程计算机上某个服务的方法，这个过程就像调用本地方法一样简单，并且不需要我们了解底层网络编程的具体细节。

举个例子：两个不同的服务 A、B 部署在两台不同的机器上，服务 A 如果想要调用服务 B 中的某个方法的话就可以通过 RPC 来做。

**RPC的出现就是为了让调用远程方法像调用本地方法一样简单。**



## 既然有了RPC，为什么还要HTTP？

HTTP协议定义了浏览器如何向服务器请求万维网文档的这样一个交互过程，这个协议是在全世界范围都要遵守的，要遵守一个大范围的一个通信协议，就要做到通用性，就比如我们的一些请求参数什么的，可能会有冗余的出现，产生了一些额外的开销。

而RPC 都是公司内部调用所以不需要太考虑通用性，只要公司内部保持格式统一即可。所以可以**自己定制化传输协议和序列化协议**来使得通信更高效。

>  就比如，RPC如果发生在中国内部的话，只需要普通话就可以交流了，但是RPC发生在全球范围内的话，可能就需要使用英语这样一个通用语言。

更重要的，**rpc框架功能更加齐全**，成熟的rpc相对http，更多的是封装了**“服务发现”，"负载均衡"，“熔断降级”**一类面向服务的高级特性。这些是http没有的。可以这么理解，[rpc框架](https://www.zhihu.com/search?q=rpc框架&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra={"sourceType"%3A"answer"%2C"sourceId"%3A191965937})是面向服务的更高级的封装。如果把一个http servlet容器上封装一层服务发现和函数代理调用，那它就已经可以做一个rpc框架了。



## RPC基本原理

1. **客户端（服务消费端）** ：调用远程方法的一端。
2. **客户端 Stub（桩）** ： 这其实就是一代理类。代理类主要做的事情很简单，就是把你调用方法、类、方法参数等信息传递到服务端。
3. **网络传输** ： 网络传输就是你要把你调用的方法的信息比如说参数啊这些东西传输到服务端，然后服务端执行完之后再把返回结果通过网络传输给你传输回来。网络传输的实现方式有很多种比如最基本的 Socket或者性能以及封装更加优秀的 Netty（推荐）。
4. **服务端 Stub（桩）** ：这个桩就不是代理类了。这里的服务端 Stub 实际指的就是接收到客户端执行方法的请求后，去指定对应的方法然后返回结果给客户端的类。
5. **服务端（服务提供端）** ：提供远程方法的一端。

![image-20220310185921494](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220310185921494.png)

1. 客户端（client）以本地调用的方式调用远程服务；
2. 客户端 Stub（client stub） 接收到调用后负责将方法、参数等组装成能够进行网络传输的消息体（序列化）：`RpcRequest`；
3. 客户端 Stub（client stub） 找到远程服务的地址，并将消息发送到服务提供端；
4. 服务端 Stub（桩）收到消息将消息反序列化为Java对象: `RpcRequest`；
5. 服务端 Stub（桩）根据`RpcRequest`中的类、方法、方法参数等信息调用本地的方法；**远程调用，如openFeign要指定对方的uri，根据这个uri，就知道找哪个Controller的哪个接口了。**
6. 服务端 Stub（桩）得到方法执行结果并将组装成能够进行网络传输的消息体：`RpcResponse`（序列化）发送至客户端；
7. 客户端 Stub（client stub）接收到消息并将消息反序列化为Java对象:`RpcResponse` ，这样也就得到了最终结果。

![image-20220310190127028](https://raw.githubusercontent.com/xuhaoyao/images/master/img/image-20220310190127028.png)