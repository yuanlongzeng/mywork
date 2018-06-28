### 一些概念：
微服务：一个完整的大型服务在架构上必须进行服务拆分，被打散成很多很多独立的小服务，每个小服务会由独立的进程去管理来对外提供服务
RPC (Remote Procedure Call)即远程过程调用，是分布式系统常见的一种通信方法  长连接数据交互  调用不同进程中的方法获取数据  
常见的多系统数据交互方案还有分布式消息队列、HTTP 请求调用、数据库和分布式缓存等。

Nginx与RPC:nginx可以作为反向代理服务器将各个子系统的服务统一，将后端多个服务地址聚合为单个地址来对外提供服务  
一般是走http协议（短连接）--一种特殊的 RPC  谷歌的gRPC就是建立在HTTP2.0之上  已没有明显界限  
但是Nginx 和后端服务之间还可以走其它的协议，  
比如 uwsgi 协议（gunicorn只能走http协议，uWSGI走uwsgi协议）、fastcgi 协议等，这两个协议都是采用了比 HTTP 协议更加节省流量的二进制协议  
关系：HTTP 与 RPC 的关系就好比普通话《官方方言》与方言的关系。  
要进行跨企业服务调用时，往往都是通过 HTTP API，也就是普通话，虽然效率不高，但是通用，没有太多沟通的学习成本。  
但是在企业内部还是 RPC 更加高效，同一个企业公用一套方言进行高效率的交流，要比通用的 HTTP 协议来交流更加节省资源  


如果两个子系统没有在网络上进行分离，而是运行在同一个操作系统实例之上的两个进程时，它们之间的通信手段还可以更加丰富。  
除了以上提到的几种分布式解决方案之外，还有共享内存、信号量、文件系统、内核消息队列、管道等，  
本质上都是通过操作系统内核机制来进行数据和消息的交互而无须经过网络协议栈。--已很少采用：单机应用意味着单点故障  

RPC 是两个子系统之间进行的直接消息交互，它使用操作系统提供的套接字（socket）来作为消息的载体，  
以特定的消息格式来定义消息内容和边界。

RPC消息协议：  
边界：  
文本-特殊分割符：\r\n  
二进制-长度前缀法：每个消息前加上该段消息的长度  
HTTP 协议是一种基于特殊分割符和长度前缀法的混合型协议  
结构：json可读性好但是冗余度大  
解决方案：  
消息的结构在同一条消息通道上是可以复用的，比如在建立链接的开始 RPC 客户端和服务器之间先交流协商一下消息的结构，  
后续发送消息时只需要发送一系列消息的 value 值，接收端会自动将 value 值和相应位置的 key 关联起来，形成一个完成的结构消息...  


如果消息的内容太大，就要考虑对消息进行压缩处理，这可以减轻网络带宽压力。但是这同时也会加重 CPU 的负担，
因为压缩算法是 CPU 计算密集型操作，会导致操作系统的负载加重。所以，最终是否进行消息压缩，一定要根据业务情况加以权衡  
务必挑选那些底层用 C 语言实现的算法库，因为 Python 的字节码执行起来太慢了。比较流行的消息压缩算法有 Google 的 snappy 算法  

流量的极致优化：变长整数varint  
负数编码：zigzag 将负数编码成正奇数，正数编码成偶数。解码的时候遇到偶数直接除 2 就是原值，遇到奇数就加 1 除 2 再取负就是原值  

Redis使用RESP（Redis Serialization Protocal）作为消息传输 文本协议  虽然性能不是最好的，但是实现简单，易于理解  
Protobuf：二进制协议  


重新连接：  
TCP 的超时重传策略要求必须收到读方的 ack （并不是服务器端的响应，是三次握手SYN_SEND、SYN_RECV、ESTABLISHED；握手过程中传送的包里不包含数据，三次握手完毕后，客户端与服务器才正式开始传送数据）  
之后才可以将数据从写缓存中移除，否则会继续留在写缓冲区以便后续可能的 TCP 重传  

网络通信的内容是【字节序列】，消息序列化（json将字典转为符合json数据格式要求的字符串数据），  
而用于界定消息边界的消息长度也是消息的一部分，它需要将 Python 的【整形转换成字节数组】（struct）  
 
python socket.send(str)  python字典使用json.dumps序列化成字符串    
socket.receiv(buffersize)得到的是bytes  使用json.loads()可以将各种对象（str,bytes）转换成python字典对象  


单线程模型：【代码】  
多线程模型：【代码】 
多进程模型：【代码】  
多进程preForking:预先fork多个子进程（进程池模型），会有惊群问题（2.6版本内核好像已经没有这个问题了？）但是使用epoll解决非阻塞socket时有这个问题
多进程多线程模型：  
单进程异步模型：异步IO--使用时间轮询（事件循环：select poll epoll）监听套接字的读写事件  
如果有的话该 API 会立即携带事件列表返回，否则会阻塞  
readfds：包含所有因为状态变为【可读】而触发select函数返回文件描述符  

惊群通常发生在server 上，当父进程绑定一个端口监听socket，然后fork出多个子进程，子进程们开始循环处理（比如accept）这个socket。每当用户发起一个TCP连接时，多个子进程同时被唤醒，然后其中一个子进程accept新连接成功，余者皆失败，重新休眠  

半包问题：本次读取的数据含有下一个数据包的内容  用户态的 ReadBuffer 是由用户代码来进行控制。
它的作用就是保留当前的半包消息为每个套接字维护一个写缓冲区