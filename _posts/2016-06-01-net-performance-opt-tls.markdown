---
layout: post
title:  "网络性能优化：TLS优化"
date:   2016-06-01 21:35:31 +0800
categories: jekyll update
---
#### SSL/TLS
 

## SSL/TLS

SSL协议原型是由Netscape开发的，用于对网络上传输的数据加密保护。SSL运行在应用层，在传输层之上。后来IETF对SSL进行了标准化，即TLS。

最新的SSL版本是3.0，TLS1.2。

#### TLS的作用

TLS协议有三个最重要的作用：数据加密，认证和保证数据完整性。

#### TLS握手

客户端和服务端在TLS上传输数据之前，要进行一些准备工作：客户端和服务端要协议相同的TLS协议版本，选择加密套件，必要的情况下验证证书。要完成这些工作要在客户端和服务端交换这些信息，那么就会带来了RTT的增加的负面影响。

![拥塞控制公式]({{ site.url }}/assets/hpbn_0402.png)

0 ms

TLS运行在TCP之上，那么首先要完成TCP的三次握手，将要花费一个完整的RTT。

56 ms

TCP第三次握手的时候，客户端发送支持的TLS协议版本，支持的加密套件列表，以及一些其它的参数。也开始了TLS握手。

84 ms

服务端选择TLS的版本，选择一个两边都支持的加密套件，再加上服务端证书一起返回给客户端。当然，为了对客户端的认证可以附加一些参数，让客户端下次的发送数据中增加客户端的证书。

112 ms

假设双方已经完成了协商，选择好了TLS协议版本，加密套件，认可双方的证书。客户端使用服务端的公钥进行对称秘钥交换，发送使用服务端公钥加密的对称秘钥发送给服务端。

140 ms

服务端使用私钥解密客户端发送的对称秘钥。服务端处理完对称秘钥的交换，message authentication code (MAC)检查消息的完整性后，返回加密后的“Finished"消息客户端。

168 ms

客户用协商好的对称秘钥解密服务端的消息，验证MAC，所有的都没问题了，此时加密通道才真正的建立了起来，现在才可以传输加密的数据了。

从上面的流程可以看出，真正开始传输数据，使用了3个RTT的时间。三次TCP握手，四次TLS握手，其中一次同时进行。
168毫秒后才可以真正的传输数据。

在握手的过程中，需要进行交互对称密钥，对称密钥是用于对传输的数据加密的，因为所有数据使用非对称加密的话，计算量是非对称加密的好多倍。协商TSL的session也是使用非对称加密套机制进行的。大家想象一下这种情况，如果服务器端的私钥丢失了将会怎么样？也就是说中间的窃听者可以知道所有的通信数据。

#### Diffie–Hellman 秘钥交换

Diffie–Hellman 秘钥交换的一个关键点在于，两个端之间不需要交换加密数据的key。也就是说中间的监听者无法获取到解密的秘钥，相比就很安全了，即使服务端的私钥丢失了也没有关系。

![Diffie–Hellman 秘钥交换]({{ site.url }}/assets/500px-Diffie-Hellman_Key_Exchange.svg.png)

[维基百科](https://en.wikipedia.org/wiki/Diffie%E2%80%93Hellman_key_exchange)

解释一下这个过程：

Alice和Bob需要进行安全通行，首先要交换对称秘钥。

（1）Alice和Bob协商一个共同的颜色黄色，这个颜色是随机的，每次交互的时候可能是不同的。

（2）Alice和Bob再各自选择一个自己的颜色，他们各自都不知道对方选择的是什么颜色，Alice随机选择了红色，Bob随机选择了浅绿色，Alice和Bob分别混合他们手上的两种颜色，Alice的颜色混合成了橙色，Bob的颜色混合成了蓝色。

（3）Alice和Bob交换他们混合好的颜色。

（4）Alice把交换的蓝色和在（2）中的红色混合，Bob同样适用交换的橙色和（2）中的浅绿色混合，最后得到机密秘钥。

在秘钥交换的过程中，窃听者无法获知真正的秘钥是什么。

当然，在真正的网络环境下不会使用颜色，而会使用很大的数，颜色只是用来说明整个过程。理论上现在的超级计算机很难在合理的时间内破解它。


#### TLS Session Resumption

TLS一个完整的握手过程，需要很过个RT，如果每次建立连接都要这么走这个过程的话，将会引起很大的性能问题：延迟。好消息是，TLS提供了恢复和重用加密秘钥的机制。

回话恢复机制（RFC 5246）在SSL2.0中被第一次引进。它允许服务端生成一个32字节的回话标示（session identifier）做为“ServerHello”消息的一部分。服务端和客户端分别缓存这个标示。当客户端再次创建SSL连接的时候，在进行TLS握手的时候把回话的id直接带过去，从而减少一次RT。

![TLS session 恢复]({{ site.url }}/assets/hpbn_0403.png)

#### 信任链和CA（Certificate Authorities）

客户端和服务端的相互信任是TLS认证过程中很关键的部分，如果对方是不可信的，加密通道将是没有任何意义的。所以引进了证书的信任链和CA（Certificate Authorities）。CA作为受信的第三方，用于双方身份的确定和认证。

一般一个证书都是有期限的，或者遇到一些特殊情况，证书的私钥丢失或者被窃取，需要让当前的证书失效。就需要一个证书失效列表（Certificate Revocation List），操作系统或者浏览器都会定期的进行更新这个列表，缓存的失效列表也不能完全满足需求，又有了在线证书状态检查协议(Online Certificate Status Protocol)，用于时时的检查证书状态。一般情况下这两种机制混合使用。如果在线时时验证证书，会增加一个RT时间。

#### TLS Record Protocol

TLS Record 是TLS发送的最小单元。可以通过Type字段，区别发送的TLS数据内容的类型，数据类型包括，握手数据，警告数据，和真正的数据。结构如下：

![TLS Record Structure]({{ site.url }}/assets/hpbn_0408.png)

典型的发送数据流程如下：

* 协议处理模块接收应用层的数据

* 把接收到的数据切分成多个数据块：每个记录最大2的14次方字节，或者最大16KB。

* 应用层数据可以选择压缩。

* 添加MAC（Message authentication code）或者 HMAC

* 使用协商的秘钥对数据加密

上述步骤完成之后，加密的数据会向下传递给TCP层进行传输。在数据的接收端，流程和发送端类似，但是流程相反：使用协商的秘钥解密数据，验证MAC，把数据分发给上面的应用层。

上传的这些工作都是由TLS层自己处理的，不需要应用层关注，对其是透明的。不管怎么样还是要注意一下这些细节：

* 每个TLS记录最大16KB

* 每个记录包含5字节的头信息， 一个 MAC信息（SSL3 20字节，TLS1和TLS1.1 32字节）和秘钥偏移。

* 为了可以解密和验证记录，整个记录必须是完整的。

选择一个合适的TLS记录大小，也将是一个非常重要的优化。记录太小，TLS记录的结构信息会成为非常大的负载，如果太大可能会导致等待完整的数据增加了延迟。应用程序开发者一般没有机会调整这个大小。如果自己实现TLS的话，请注意这个优化。

#### 优化TLS

* 加密数据使用对称算法，节省加密时间。这是TLS自己标准的方式。

* 静态资源CDN。

* 服务器离用户更近。建立多个数据中心，或者服务器部署在多个地区。

* 建立代理服务器，提供提供连接池，减少握手时间等。

* TLS连接重用，Session恢复等。

* TLS False Start

对于第一次建立链接的握手，回话恢复机制是没有任何作用的。怎么去减少第一次的握手延迟呢？
TLS False Start机制是在TLS握手只完成一部分的时候，可以进行数据发送。TLS错误开始，意思就是不知道是否能够握手成功的情况下进行数据发送。TLS不改变握手机制，只影响协议发送数据的时间。ClientKeyExchange 记录完成之后，已经知道了加密秘钥，发送数据和握手并行进行。正常情况下减少一个RT。

如下图：

![TLS False Start]({{ site.url }}/assets/hpbn_0412.png)

* TLS记录的大小调整

一个TLS的记录最大16kb，在采用不同的加密算法的情况下，每个记录会包含20-40自己的头信息，M
AC和其他可选的占用空间。如果一个TLS记录和一个TCP数据包匹配的话，还需要考虑IP包的20字节和TCP最少20字节的头信息。结果是有潜在的60-100字节的协议负载，如果是最大传输数单元差不多1500字节，至少6%的结构负载。

记录太小，增加结构负载，如果记录太大，增加数据发送的延迟。

Google选择的方式是：发送第1M的数据，TLS记录大小和TCP segment匹配，直到调整到16KB。并且TLS记录的大小会根据网络环境时时调整。并且会调整OpenSSL的缓存大小。

* 不要使用TLS的压缩机制

  "CRIME"攻击，TLS压缩可以恢复秘钥认证的cookie，允许攻击者劫持session。

  TLS压缩不关系内容类型都进行压缩和解压缩，导致对已经压缩过的数据重复压缩，比如图片和适配。

  重复压缩浪费CPU时间。

  建议在HTTP层进行压缩。

 * 证书链长度和证书过期检查

  验证信任链需要浏览器递归的验证证书直到信任的跟证书。因此，第一项优化确保在握手的时候包含所有的中间证书，防止浏览器自行去查找验证中间的证书，查找的过程可能需要额外的DNS时间，TCP连接时间和HTTP请求时间。尽量使用在浏览器中存在的根证书。

  同时，也要确保不要包含不必要的证书。也就是说减少证书链的大小。证书链太大，TCP又有先慢机制，可能会增加TCP连接发送数据的次数，等待数据的完整性，在弱网环境下延迟会增加很明显。证书链的大小尽量小到2-3KB。
  
  每一个TLS连接还要求必须验证证书链的签名以及验证证书是否过期。这些都有可能增加延迟。一个优化是OCSP stapling：允许服务器在证书链中包含OCSP（CA返回的，经过签名的），让浏览器跳过在线验证。
  最新的服务器框架都已支持。


###### 参考

[1] [Diffie–Hellman key exchange](https://en.wikipedia.org/wiki/Diffie–Hellman_key_exchange)

[2] [High Performance Browser Networking](http://chimera.labs.oreilly.com/books/1230000000545)
