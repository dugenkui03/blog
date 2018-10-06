

##### 引言

堆转储快照用于分析heap堆信息。

dump文件(堆转储快照)生成方式：

1. jmap命令：`jmap -dump:format=b,file=fileName.hprof pid`.
2. ，**`-XX:HeapDumpOnOutOfMemoryError`** 参数作用时在虚拟机发生OOM异常时自动生成dump文件。


