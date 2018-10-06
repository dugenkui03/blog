##### 1.controller

`controller`默认单利模式，可以在类定义上使用参数将其设置为多例模式：
```
@Scope(value="prototype")

//默认单例
@Scope("singleton")
```
[博客讲解](https://blog.csdn.net/qq_27026603/article/details/67953879)