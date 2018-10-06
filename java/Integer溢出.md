java的Integer类型溢出即，Integer类型最大值+1编程最小值，或者Integer类型最小值-1变成Integer类型最大值。类似于一个首尾相连的环状。由此带来的表现是：
- 溢出时造成加减法不准确,Integer范围`[-2147483648,2147483647]`
- a-b<0与a<b判断不一致，前者可能由于溢出造成结果是错的；

示例如下：

```
//第一个判断是正确的
System.out.println(Integer.MAX_VALUE<-1);//false
System.out.println(Integer.MAX_VALUE+1<0);//true
```
因此整型判断大小是应该用如下代码：

```
a<b?-1:(a==b?0:1);返回整型结果的判断
//或者直接
a<b;
```
