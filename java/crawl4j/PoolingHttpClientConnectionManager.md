[博客](https://kmg343.gitbooks.io/httpcl-ient4-4-no2/content/233_lian_jie_chi_guan_li_qi.html)

PoolingHttpClientConnectionManager是HttpClient中的连接池管理类。每个Http请求的连接和断开都要经过TCP的三次握手和四次挥手，如果会产生大量请求的程序每个请求都经过这个过程将会耗费大量的资源。连接池可以使得请求结束后链接返回连接池供下次调用使用。

基本使用示例如下：
```
//crawl4j项目中实现了其导出类，我们用项目相关代码做示例
PoolingHttpClientConnectionManager connectionManager=new SniPollingHttpClientConnectionManager(socketFactoryRegist,dnsResolver);
connectionManager.setMaxTotal(20);//设置最大连接数
connectionManager.setDefaultMaxPerRoute(10);//设置每个主机的最大连接数；

connectionManager.closeExpiredConnections();//关闭过期的链接
// Optionally, close connections that have been idle longer than 30 sec
//关闭空闲的链接（空闲的时间长度，空闲的时间单位）
connectionManager.closeIdleConnections(30, TimeUnit.SECONDS);

connectionManager.shutdown();//关闭连接管理器，释放其所有资源，包括其所有的链接
```