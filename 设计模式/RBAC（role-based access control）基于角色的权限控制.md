
- 权限对web应用来说就是对url的控制，还有页面的展示，还有。。。

##### 1.传统权限模型

简单的权限控制模型，如**用户集合和权限集合**中的元素多对多对应。


##### 2. 引入==角色==概念：<font color=red>RBAC0</font>

但如上模型的缺点是，如果拥有相同权限某类人——比如同一个部门的人，需要增加权限时，每个人都需要在自己对应的权限集合中添加一个元素，这样**操作繁琐并且容易出错**，可行的做法是**从权限集合中抽象抽一个子集作为角色**，比如*x部门成员*，将x部门成员与角色对应即可，这样x部门全部成员的权限即可通过角色来控制。图示如下：

![](https://wx1.sinaimg.cn/mw690/006Xp67Kly1fsx2io1l46j30gp097q6q.jpg)

##### 3. 引入用户组

同理，除了引入**角色**的概念来简化赋权外，用户集合也可以抽象出类似于 权限集合/角色 的用户组，将角色与用户组关联来进行赋权，用户拥有权限的集合是**用户关联的角色和用户所在用户组关联的角色的并集**。图示如下(**注意括号中角色与权限是关联的)：
![](https://wx1.sinaimg.cn/mw690/006Xp67Kly1fsx2njv9umj30b40eq0x6.jpg)


##### 4. 权限设计：将权限与资源关联

资源只有一种，即**数据**，但是具体的应用系统中需要更细致的分类来方便权限与资源的关联。

我们对数据的操作有**增删改查四种**。权限类型也可以按照这四种操作做几个基本的划分。比如：

1. 增删改——只有管理员——拥有此三种权限的人，对应的页面才能够渲染出相应的创建和编辑的按钮；
2. 查——不同权限的人渲染页面数据不同，比如管理员可以渲染记录的更多列来查看更详细的数据。

以上都是基于**页面**渲染来说的。现在公司将权限基本分为三类：**页面模板、接口和数据模板**。我的理解是这种方法和**增删改查**的分别是横切和纵切，相交又包含。一种权限分类图示如下：

![](https://wx2.sinaimg.cn/mw690/006Xp67Kly1fsx3dyw3b6j30h20f110a.jpg)

- 可以根据“权限类型”的取值来区分是哪一类权限，如“MENU”表示菜单的访问权限、“OPERATION”表示功能模块的操作权限、“FILE”表示文件的修改权限、“ELEMENT”表示页面元素的可见性控制等。

- 注意没添加一个页面元素或者文件表，在“文件表”、“权限文件关联表”和“权限表”中会相应的添加一条记录。

综上，成型的RBAC模型如下：
![](https://wx3.sinaimg.cn/mw1024/006Xp67Kly1fsx3hzecx1j30rc0fgwr4.jpg)

##### 5.角色分层模型RBAC1（Hierarchal RBAC）

以上，3、4小节可以看作是实践拓展，RBAC模型分为RBAC0/1/2/3三种，RBAC0是RBAC1和RBAC2的基础，RBAC3是RBAC1和RBAC2的合并。

**RBAC1将角色做了分层**，引入了等级的概念，相同角色的低等级可以**继承**高等级的权限。图示如下：

![](https://wx2.sinaimg.cn/mw1024/006Xp67Kly1fsx3trkfvxj30hs08tq5z.jpg)

##### 6.角色限制模型RBAC2（Constraint RBAC）

RBAC2是在RBAC0的基础上加上了**约束用户能够拥有的角色**的思想，主要引入了
- **静态职责分离SSD(Static Separation of Duty)**：在用户和角色的指派阶段加入的，主要约束有互斥角色、基数约束和先决条件约束等；
- **动态职责分离DSD(Dynamic Separation of Duty)**：是会话和角色之间的约束，可以动态的约束用户拥有的角色。<font color=red>**会话是动态的概念，用户必须通过会话才可以设置角色，是用户与激活的角色之间的映射关系**</font>。

两者对比图示如下：
![](https://wx2.sinaimg.cn/mw690/006Xp67Kly1fsx49xqwyqj30hs08ujux.jpg)

##### 7.RBAC3是RBAC1与RBAC2的合并，既有角色分级，又有用户赋予角色是的约束。


综上，一个完善的权限框架有：
1. 用户-角色-权限(RBAC0)；
2. 用户分组；
3. 角色分级且可继承(RBAC1)；
4. 用户赋予角色时的限制(RBAC2)
5. 完善的权限设计；


上述将角色赋予用户时，要灵活的使用用户的信息，**使用表达式来进行灵活的绑定**，比如性别为男的用户自动赋权可以上男厕所。
