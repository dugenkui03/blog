##### 1.引言

本文是对使用java收发TCP报文的讲解，主要有：
1. 实现多客户端—服务器Socket通信；
2. 连接超时、通信超时、半连接等参数设置；
3. 可中断套接字方面的讲解。

在网络编程中(此例)，使用的输入输出流对象主要是：
```
Scanner/InputStream、PrintStream/OutputStream
```

[聊天室小程序](https://github.com/dugenkui03/tcp2talk)展示了怎么通过Socket让连接到同一个服务器的多个用户进行通信，但是没有使用半连接和可中断套接字——游戏中某个区的所有用户可以再公共区域讲话大抵就是这个原理吧。

##### 2.连接和传输数据

###### 建立连接

通信之前首先要建立连接，并创建IO流对象。TCP链接需要知道server的域名和服务端口，即套接字内容：
```
//无超时
Socket soc=new Socket("www.XXX.com",80);
```

此行代码就是建立TCP连接的过程，但是如果**连接不成功会无限期的阻塞下去**，因此可以使用无连接`Socket`对象的`connect(new InetSocketAddress(host,port),timeout)`方法建立设置超时时间的链接：

```
Socket soc=new Socket();
//设置超时时间
soc.connect(new InetSocketAddress(host,port),timeout);
```

###### 创建IO流

创建网络IO的IO流对象可以用以下方法：

```
//从套接字(服务器)读取数据;
InputStream in=soc.getInputStream();
//向套接字写数据
OutputStream out=soc.getOutputStream();
```
如果没有数据可以从套接字读取，读操作将会被**阻塞，最终受操作系统限制而超时停止**，可以对<font color=red>读写操作设置超时时间</font>，**如果读写操作在完成之前就超过了时间限制，此方法在超时时会抛异常**:
```
Socket soc=new Socket("www.xxx.com",80);
//毫秒为单位
//数据传输超时时间，服务器数据传输时间过长是会导致超时，抛异常
//fixme 客户端向服务器请求相应到服务端响应完毕的时间，如果已经相应但是没有读取则不抛异常
soc.setSoTimeOut(1000);
```

###### InetAddress类获取域名对应IP地址

```
//获取域名对应的网络地址(ip)
InetAddress address=InetAddress.getByName("www.XX.com");

byte[] addByteCode=address.getAddress();//返回地址对应的字节数组；

String host_ip=address.getHostAddress();//获取十进制IP地址

//获取域名对应的所有网络地址
InetAddress []allAdds=InetAddress.getAllByName("www.xxx.com");
```

**实现服务器**

服务器的功能应该如下四个步骤：
1. 监听某个端口；
2. 简历Socket链接；
3. 创建一个线程为打开的Socket连接服务，一般也需要输入输出流对象；
4. **关闭服务器套接字**。

实现参见[局域网聊天小程序](https://github.com/dugenkui03/tcp2talk)

###### 半连接 half-close

半连接即通过关闭套接字的输入或输出流来告知对方输入或输出以产能结束。相关函数如下：

```
void shutdownOutput();

void shutdownInput();

//查看输入输出流是否关闭
boolean isOutputShutdown();
boolean isInputShutdown();
```

##### 3.可中断套接字

当建立套接字或者读取数据时，当前线程可能会发生阻塞，这种情况无法通过`interrupt`解除阻塞，此时可以使用 **`SocketChannel`** 类。其主要流程如下如下：

```
//建立可中断的套接字
SocketChannel channel=SocketChannel.open(new InetSocketAddress(host,port);

//一般还要用 boolean isUnresolved()判定InetSocketAddress是否解析成功

//建立输入流
Scanner in=new Scanner(channel);

//调用静态方法建立输出流
OutputStream outStream=Channels.newOutputStream(channel);
//同样的方式可以建立输出流

```

