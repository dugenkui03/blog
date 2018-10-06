
本博文主要讲解Linux对硬件和软件资源的监控命令，包括：
1. 查看cpu、内存、上下文、vm状态的`vmstat`、top(类似msgtask)和简单的free；
2. IO相关信息`iostat -dx x y`;
2. 查看网络连接的`netstat`、网络IO流量概览的`nload`和每个套接字IO流量的`iftop`


##### 1. `vmstat`和其他服务器资源管理命令

`vmstat`是virtual memory status的缩写，即虚拟内存状态。可以用来监控CUP、虚拟内存、IO等多个服务器指标。

###### 1.1 基本使用方式

vm有两个参数：
```
vmstat x y
```
- x、y为两个整数，前者表示采样的时间间隔数，后者表示采样次数——省略一个参数表示一直采样直至手动停止，省略两个参数表示查看机器启动以来的各指标平均值(==第一行打印系统启动以来的平均值==)。

###### 1.2 参数解释

一次采样`vmstat`:

```
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 3  0      0 1502252  18464 261972    0    0    70    11  158  391  4  0 95  0  0
```
使用`man vmstat`可以查看vmstat参数说明和打印详解：

**进程相关**：
1. **r**：可运行的进程数目，包括running、或者等待时间片的进程；
2. **b**：处于阻塞状态（in uninterruptible sleep不可中断休眠）的进程数，通常指等待IO，比如磁盘、网络、输入。

**内存相关**：
1. swpd：使用的虚拟内存容量，单位字节；
2. **free**：空闲的内存容量；
3. buff：作为缓冲buffers的内存容量；
4. cache：作为操作系统缓存cache的内存容量；
5. inact/active(-a):活跃和不活跃的内存容量；

**页面调度相关swap(以下参数每秒不要超过10)**：
1. **si**(swap in):每秒内存从磁盘写入的块数；
2. **so**(swap out):每秒内存写出到磁盘的块数；

**IO相关**:
1. bi(block in):每秒从块设备(磁盘和其他)获取的块数；
2. bo:每秒从块设备获取的块数。
3. 主存和磁盘以块为单位传送数据。

**系统相关**：
1. in：The number of interrupts per second, including the clock；
2. <font color=red>**cs**</font>(context swiches)：每秒钟上下文切换的次数，**cs次数太多是需要考虑调整程序线程数量**；

**cup相关**：五种操作对CPU时间的占比
1. **us**(user time):cpu运行非内核代码的时间；
2. sy(system time):cpu运行内核代码的时间；
3. id(idle time):空闲时间，包括IO等待时间；
4. wa：等待IO的时间；
5. st:time stolen from a virtual machine.位置消耗时间。

###### 1.3 分析思路（注意事项、未实战，纯猜想：

以上重点参数已经加粗：

如果b一直不为0可以考虑是否存在死锁。

r表示使用和等待cup资源的进程个数，如果超过了cpu核数很多，就可能频繁的引起上下文切换，表现为cs很大。

如果si、so很大、free很小，可能主存性能满足不了现在工作，导致频繁的磁盘IO甚至抖动。

###### 1.4 实例解析


- 当上传文件并放入到本地磁盘时，bo突然很大：
![](https://wx2.sinaimg.cn/mw1024/006Xp67Kly1fqybdfgvw4j30pa02hwej.jpg)

- 当使用vim将本地磁盘数据读入到内存中时，bi突然变大：
```
1.procs ---------2.memory--------- ---3.swap-------4.io---- -5.system--------6.cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 1458020  22500 295004    0   0  1388     0  186  456  4  1 93  2  0

```
- 当开启爬虫程序，需要从网络读取数据，并写入磁盘数据库时，各参数变化如下：
![](https://wx2.sinaimg.cn/mw1024/006Xp67Kly1fqybkvqx4dj30c60dzq3u.jpg)
    1. 从网络设备读取数据时bi变大；
    2. 当写入数据到数据库时bo变大；
    3. 开启了十几个线程因此需要频繁的上下文切换；
    4. 用户程序使用cup时间us经常在90%以；
    5. 等待IO时间占比wa也变得不稳定。

**虽然没有换入换出si/so的例子，但是应当知道这是`vmstat`最终要的参数，当这两个参数太大时应该考虑优化程序的实现和升级内存容量。**

###### 1.5 <font color=red>**cup密集型机器和IO密集型机器**</font>

cup密集型服务器`vmstat`的`us`输出通常是一个很高的值，即cup花费在非内核代码上的cup时间占比应该很高。

cup密集型服务器上下文切换次数警告阈值是10万/s(具体情况看机器?)。

IO密集型服务器cup会花费大量时间等待IO请求完成，则意味着很多任务处于非中断休眠状态(`b`列)，并且`wa`数字也很高(等待IO时间)。

###### 1.6 其他

top命令可以查看动态刷新的各个进程的cup和内存使用率,以及**执行的命令和命令执行的用户和PID(进程ID)**，界面类似于windows的任务管理器：
![](https://wx3.sinaimg.cn/mw1024/006Xp67Kly1fqyc5zzn8qj30or088wfe.jpg)

free命令，界面如下：
```
root@iZwz94idfw2r7h2hnepjZ:~# free
              total        used        free      shared  buff/cache   available
Mem:        2048212      587876      981736        5264      478600     1301432
Swap:             0           0           0

```


#### 2.IO相关信息统计

###### 2.1 `iostat  -dx  a b`

设备和分区的IO统计信息和cup统计信息。参数`d x`分别表示显示设备使用状态和输出更多信息。a和b分别表示采样时间间隔和采样次数，同vmstat，第一次输出也是系统启动以来的平均值：`iostat -dx`:爬虫程序启动后的变化值：

```
root@iZwz94iww8uynepjZ:~# iostat -dx 3
Linux 4.4.0-63-generic (iZwz94iww8uynepjZ) 	05/04/2018 	_x86_64_	(1 CPU)

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00     0.00    0.00    0.33     0.00     5.33    32.00     0.00    0.00    0.00    0.00   0.00   0.00

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00     7.33    0.00    1.33     0.00    37.33    56.00     0.00    0.00    0.00    0.00   0.00   0.00

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00     4.00    0.00   10.00     0.00    57.33    11.47     0.01    0.80    0.00    0.80   0.80   0.80

Device:         rrqm/s   wrqm/s     r/s     w/s    rkB/s    wkB/s avgrq-sz avgqu-sz   await r_await w_await  svctm  %util
vda               0.00    28.52    0.00   87.25     0.00  1268.46    29.08     0.18    2.08    0.00    2.08   1.66  14.50

```
- 可以看到爬虫程序启动后因为要向数据库写入数据，相关指数明显变大；
- `r/s、w/s`：每秒钟发送到设备的读写请求；
- `avgqu-sz`:在设备队列中等待的请求数量；
- `await`:磁盘排队上花费的毫秒数，包括读和写；
- `svctm`:服务请求花费的毫秒数，不包括排队时间。

**重要概念：请求服务并发数**：
`concurrency=(r/s+w/s)*(svctm/1000)`
表示在采样周期内每秒设备处理的请求数。


##### 3. 网络资源
###### 3.1 `netstat`连接详情
>Print network connections, routing tables, interface statistics, masquerade connections, and multicast membership

打印网络连接、路由表、接口统计、伪装连接和多播membership。
- a、t、u表示罗列出所有的tcp和udp连接：
```
root@iZwz94i8afw2r7g62hnepjZ:~# netstat -at
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 localhost:32000         *:*                     LISTEN     
tcp        0      0 *:http-alt              *:*                     LISTEN     
tcp        0      0 *:ssh                   *:*                     LISTEN     
tcp        0      0 172.16.252.71:57346     211.151.27.128:http     TIME_WAIT  
tcp        0      0 172.16.252.71:57598     211.151.27.128:http     TIME_WAIT  
tcp        0      0 172.16.252.71:57306     211.151.27.128:http     TIME_WAIT  
tcp        0      0 172.16.252.71:39240     59.151.32.81:http       TIME_WAIT  
tcp        0      0 172.16.252.71:39274     59.151.32.81:http       TIME_WAIT  
tcp        0      0 172.16.252.71:39208     59.151.32.81:http       TIME_WAIT  
tcp        0      0 172.16.252.71:57480     211.151.27.128:http     TIME_WAIT  
tcp        0      0 172.16.252.71:39372     59.151.32.81:http       TIME_WAIT  
tcp        0      0 172.16.252.71:57450     211.151.27.128:http     TIME_WAIT  
tcp        0      0 172.16.252.71:57482     211.151.27.128:http     TIME_WAIT  

```

###### 3.2 输入输出流量:`nload`、`iftop`

`nload`查看总体的输入输出流量，并且可以查看峰值、谷值和平均值，太简单，如图开启爬虫：

![](https://wx4.sinaimg.cn/mw1024/006Xp67Kly1fqycx7zirvj30vd09ewf1.jpg)

`iftop`则可以查看每个套接字的输出输出流量：
![](https://wx2.sinaimg.cn/mw690/006Xp67Kly1fqyd0tb6xxj30m90a0js3.jpg)