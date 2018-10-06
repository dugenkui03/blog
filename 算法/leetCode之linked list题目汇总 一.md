
#### 总结

**对链表的操作主要分为两种**：
1. 多个指针配合操作节点，一般空间复杂度低；
2. 递归操作；
3. 还可以使用java内置的数据结构，比如`Stack、PriorityQueue、List、Set、Map`等等；

**其他要点**：
1. 对一个位置的删除和插入，都需要知道前一个节点；
    1. 虚头部用于保留“前一个节点”；
    2. 节点赋值可以转移“前一个节点位置”； 
    3. 删除元素很多时候用自底向上的递归方法很方便；
    4. <font color=red>**修改一个类对象，务必用函数返回修改后的对象给引用**</font>，如果是修改一个对象里边的基础变量应该不用返回值复制；



###### [237. Delete Node in a Linked List](https://leetcode.com/problems/delete-node-in-a-linked-list/description/)

###### 题目
删除链表中的节点，并且函数参数就是指向节点的指针：
`public void deleteNode(ListNode node)`
###### 解题思路:<font color=red>通过对指针节点后一个节点对其赋值来方便链表删除操作</font>
1. 删除某个节点需要知道其前一个节点，此题可以将所需删除节点的下一个节点值赋予此节点，那么所需删除节点就是所给指针的下一个节点，下图所示比如A是需要删除的节点：
```
graph LR
A((A)) --> B 
B --> C

```
B的值赋值给A，此时将B删除即可完成要求，而我们已经有了A的指针；

```
graph LR
A --> B((B)) 
B --> C

```

代码：
```
    public void deleteNode(ListNode node) {
       node.val=node.next.val;
       node.next=node.next.next;
    }
```

##### [206.reverse linke list](https://leetcode.com/problems/reverse-linked-list/description/)

###### 指针,<font color=red>虚头部</font>

“前后指针”的方式反转链表，<font color=red>貌似这个速度更快（在判定回文一题中用此方法明显较快）</font>：
```
    public ListNode reverseList(ListNode head) {
        if(head==null||head.next==null) return head;

        ListNode prev=null;
        
        while(head!=null){         
            ListNode next=head.next;
            
            head.next=prev;
            prev=head;
            head=next;
        }
        
        return prev;//fixme 这里返回的是pre
    }
```
明白这种“反转”的思想即可，以后会用到：

虚头部保留了节点插入位置，而指针遍历用于不断将节点翻转到最前边：
```
    public ListNode reverseListPoint(ListNode head) {
        if(head==null||head.next==null) return head;

        ListNode dummy=new ListNode(0),current=dummy;
        dummy.next=head;

        while(head.next!=null){
            current=head.next;

            head.next=current.next;
            current.next=dummy.next;
            dummy.next=current;
        }

        return dummy.next;
    }
```
###### 递归方式

递归改变指针方向，并返回最后一个节点:
```
    public ListNode reverseListRecursive(ListNode head) {
        if(head==null||head.next==null) return head;

        ListNode next=head.next;
        ListNode newHead=reverseListRecursive(next);
        next.next=head;
        head.next=null;//fixme 不可缺少,否则最后一个节点即原head会进入死循环

        return newHead;
    }

```

##### [21.merge two sorted list](https://leetcode.com/problems/merge-two-sorted-lists/description/)

###### 问题.如题

###### 使用指针进行归并:<font color=red>虚头部、bravNode、边界条件</font>

```java
    public ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        if(l1==null||l2==null)
            return l1==null?l2:l1;
        
        ListNode dummy=new ListNode(0),smaller=dummy;
        
        while(l1!=null&&l2!=null){
            if(l1.val<l2.val){
                smaller.next=l1;
                l1=l1.next;
            }else{
                smaller.next=l2;
                l2=l2.next;
            }
            smaller=smaller.next;
        }
        
        if(l1==null)
            smaller.next=l2;
        else
            smaller.next=l1;
        
        return dummy.next;
    }
```

##### [83. Remove Duplicates from Sorted List：保留一个重复的节点](https://leetcode.com/problems/remove-duplicates-from-sorted-list/description/)

- 使用指针:空间复杂度低，时间复杂度高。<font color=red>当两个节点一直存在前后关系或者相差一定数量的节点时，就可以用一只指针完成操作</font>
```
    public ListNode deleteDuplicates(ListNode head) {
        if(head==null||head.next==null) return head;
        
        ListNode pre=head,current=pre.next;
        
        while(current!=null){
            if(current.val==pre.val){
                current=current.next;
                pre.next=current;
            }else{
                pre=pre.next;
                current=current.next;
            }
        }
        
        return head;
    }
    

 因为以上代码中pre和current永远有前后关系，因此可以改进为

     public ListNode deleteDuplicates(ListNode head) {
        if(head==null||head.next==null) return head;
        
        ListNode pre=head;
        
        while(pre.next!=null){
            if(pre.next.val==pre.val){
                pre.next=pre.next.next;
            }else{
                pre=pre.next;
            }
        }
        return head;
    }

```

- <font color=red>自底向上递归删除元素：空间复杂度高，时间复杂度低</font>
```
    public ListNode deleteDuplicates(ListNode head) {
        if(head==null||head.next==null) return head;
        
        head.next=deleteDuplicates(head.next);
        
        return head.val==head.next.val?head.next:head;//head值与head.next值相等的话，返回head.next作为head前一个节点的后续节点，head被删除
    }

```
##### <font color=red>**重点问题:**</font>[82 Remove Duplicates from Sorted List II:是重复的就删除掉](https://leetcode.com/problems/remove-duplicates-from-sorted-list-ii/description/)

###### 问题
删除重复的节点，一个也不保留，节点值递增；

###### 思路
1. 使用flag标记链表的某一段是否要删除：
2. 第二种解法，遍历节点，将所有重复的节点节点值放入List，然后遍历链表，其值在list中则删除。时间复杂度和空间复杂度更高；

```
class Solution {
    public ListNode deleteDuplicates(ListNode head) {
        if(head==null||head.next==null) return head;
        
        ListNode dummy=new ListNode(0),prev=dummy;
        dummy.next=head;
        boolean flag=false;//是否进行删除操作
        
        while(head!=null){
            while(head.next!=null&&head.val==head.next.val){//所指向的节点与后一个节点值相同，则前进一步在进行比较，同时将flag置为true
                head=head.next;
                flag=true;
            }
            
            if(flag){//head及之前的都应该删除；
                prev.next=head.next;
                flag=false;
            }else{
                prev.next=head;
                prev=head;
            }
            head=head.next;
        }
        
        return dummy.next;
    }
}

```


##### [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/description/)

###### 问题
查找链表是否有环;

衍生问题，是否有环，环的起始节点在哪儿；

###### 解答
思路：
- <font color=red>两个遍历节点，速度不一致；
- 初次相遇时慢的节点走了步长等于环状长度；
- 如果快的和慢的在初次相遇后，另设一个慢节点指针从起点开始行走，则两个慢节点指针第一次在环装起始节点相遇，因为一个节点第二次走到起始节点的长度等于`起点距离环起点+环装长度`</font>

```

    public boolean hasCycle(ListNode head) {
        if(head==null||head.next==null) return false;
        
        ListNode fast=head,slow=head;
        do{
            slow=slow.next;
            fast=fast.next.next;
        }while(fast!=slow&&fast!=null&&fast.next!=null);
        
        return fast==slow;
    }
    
```

###### [衍生问题解答](https://leetcode.com/problems/linked-list-cycle-ii/description/)

思路见以上汇总，解答
```
    public ListNode detectCycle(ListNode head) {
        if(head==null||head.next==null) return null;
        
        ListNode fast=head,slow=head;
        
        do{
            fast=fast.next.next;
            slow=slow.next;
        }while(slow!=fast&&fast!=null&&fast.next!=null);
        
        if(slow!=fast) return null;
        
        ListNode slow2=head;
        
        while(slow!=slow2){
            slow=slow.next;
            slow2=slow2.next;
        }
        
        return slow;
    }
```

##### [234. Palindrome Linked List](https://leetcode.com/problems/palindrome-linked-list/description/)

###### 问题
判断一个链表是否是回文：121 和 1221 都是回文

###### 思路
<font color=red> **回文链表后半段翻转后和等长的前半段一样** </font>，步骤：
1. 找到中间节点（偶数则分界点找前半段）；
2. 反转中间节点以后的部分；
3. 对比两半段的节点，知道指针指向null;

```
    public boolean isPalindrome(ListNode head) {
        if(head==null||head.next==null) return true;
        
        ListNode fast=head,slow=head;
        while(fast!=null&&fast.next!=null){
            fast=fast.next.next;
            slow=slow.next;
        }
        //此时slow的位置：偶数节点是处于后半段分解，奇数节点时处于中间节点
        //fixme：以上情况，此时只要把slow及以后的反转，不论奇偶，只要两端前半部分相等就符合回文；

        slow=reverse(slow);//fixme： 如果仅仅是反转而不进行复制操作，那么slow指的就是最后一个节点，无法从开始遍历

        while(head!=null&&slow!=null){
            if(head.val==slow.val){
                head=head.next;
                slow=slow.next;
            }else return false;
        }
        
        return true;
    }

//这种逆转链表方法比之前提到的第一种快一毫秒
    public ListNode reverse(ListNode head) {
        ListNode prev = null;
        while (head != null) {
            ListNode next = head.next;
            head.next = prev;
            prev = head;
            head = next;
        }
        return prev;
    }
```

##### [203 remove list element](https://leetcode.com/problems/remove-linked-list-elements/description/)

###### 问题
去掉链表中值等于val的所有节点；

###### 思路

链表中去掉节点（可能包括头部在内），只要<font color=red>设置虚头部及其作为起始位置的“串联节点”即可，并且两者下一个节点都指向head，操作如下：
1. “遍历节点”指向的节点如果不删除，则`pre=pre.next;或 pre=travNode`;
2. 对于要删除的节点，直接用`pre.next=travNode.next`跳过即可；</font>

```
    public ListNode removeElements(ListNode head, int val) {
        if(head==null) return null;
        
        ListNode dummy=new ListNode(0),pre=dummy;
        dummy.next=head;
        
        while(head!=null){
            if(head.val==val){
                pre.next=head.next;
            }else{
                pre=pre.next;
            }
            head=head.next;
        }
        
        return dummy.next;
    }
```

**自底向上的递归删除**
```
    public ListNode removeElements(ListNode head, int val) {
        if(head==null) return null;
        
        head.next=removeElements(head.next,val);
        
        return head.val==val?head.next:head;
    }
```
##### [160. Intersection of Two Linked Lists](https://leetcode.com/problems/intersection-of-two-linked-lists/description/)

###### 问题
两个链表可能相交也可能不相交，求其交点或者返回null;

###### 思路

两个节点指针沿着A、B链表的头指针开始遍历，走到链尾时则在从另一个链表头部开始遍历，则第一次相交或者说相同节点为交点或者不是交点的null（当两者都是从第二条链表遍历结束时）:
<font color=red>n+m步都为null或者汇聚后才会跳出循环</font>
```
    //只会用n+m时间，而非O(n+m)
    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        if(headA==null||headB==null) return null;
        
        ListNode a=headA,b=headB;
        while(a!=b){//n+m步都为null或者汇聚后才会跳出循环
            a=a==null?headB:a.next;
            b=b==null?headA:b.next;
        }
        return a;
    }
```
##### [725. Split Linked List in Parts](https://leetcode.com/problems/split-linked-list-in-parts/description/)

>Input: 
root = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10], k = 3
Output: [[1, 2, 3, 4], [5, 6, 7], [8, 9, 10]]

将一个链表分为k段，而且每段包含元素值相差不超过1。这个题直接遍历求长度，然后求解即可；

##### [2. Add Two Numbers](https://leetcode.com/problems/add-two-numbers/description/)

###### 问题

两个非空链表代表两个正数，链表方向为低位指向高位，返回其求和后的链表；

###### 思路


```
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        if(l1==null||l2==null) return l1==null?l2:l1;
        
        ListNode dummy=new ListNode(0),pre=dummy;
        int sum=0;
        while(l1!=null||l2!=null){
            if(l1!=null){
                sum+=l1.val;
                l1=l1.next;
            }
            if(l2!=null){
                sum+=l2.val;
                l2=l2.next;
            }
            
            ListNode tmp=new ListNode(sum%10);
            pre.next=tmp;
            pre=tmp;
            
            sum/=10;
        }
        
        if(sum==1){
            pre.next=new ListNode(1);
        }
        
        return dummy.next;
    }
```



##### [445. Add Two Numbers II](https://leetcode.com/problems/add-two-numbers-ii/description/)

###### 问题
两个非空链表代表两个正数，链表方向为高位指向低位，返回其求和后的链表，示例：

`Input: (7 -> 2 -> 4 -> 3) + (5 -> 6 -> 4)
Output: 7 -> 8 -> 0 -> 7`

不能修改两个链表

###### 思路

<font color=red>**使用stack保存两个链表的数据，而且其特性可以保证暴露的部分正好是从低位到高位**</font>；

如果可以修改链表，则逆转链表，求和，然后在相加即可（见解法二）.

- 解法1
```
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        if(l1==null||l2==null) return l1==null?l2:l1;

        ListNode ta=l1,tb=l2;
        Stack<Integer> sa=new Stack<>();
        Stack<Integer> sb=new Stack<>();
        while(ta!=null){
            sa.push(ta.val);
            ta=ta.next;
        }
        while(tb!=null){
            sb.push(tb.val);
            tb=tb.next;
        }

        ListNode result=new ListNode(0),pre=result;
        int sum=0;
        while(!sa.empty()||!sb.empty()){
            if(!sa.empty()) sum+=sa.pop();
            if(!sb.empty()) sum+=sb.pop();

            result.val=sum%10;
            ListNode tmp=new ListNode(sum/10);
            tmp.next=result;
            result=tmp;

            sum/=10;
        }
        
        return result.val==0?result.next:result;
    }
```
- 解法2：逆转链表，相对较快
```
    public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
        l1=reverse(l1);
        l2=reverse(l2);
        
        ListNode result=new ListNode(0);
        int sum=0;
        while(l1!=null||l2!=null){
            if(l1!=null) {
                sum+=l1.val;
                l1=l1.next;
            }
            if(l2!=null) {
                sum+=l2.val;
                l2=l2.next;
            }
            
            result.val=sum%10;
            ListNode tmp=new ListNode(sum/10);
            tmp.next=result;
            result=tmp;
            
            sum/=10;
        }
        
        return sum==1?result:result.next;
    }
    
    private static ListNode reverse(ListNode head){
        ListNode pre=null;
        
        while(head!=null){
            ListNode next=head.next;
            
            head.next=pre;
            pre=head;
            head=next;
        }
        
        return pre;
    }
```

###### [143. Reorder List](https://leetcode.com/problems/reorder-list/description/)

###### 问题
以链表中间为轴，将对称的点放在一起：
>Given {1,2,3,4}, reorder it to {1,4,2,3}.

###### 思路
同判断一条链表是否回文：
1. 找到重点或者前半部分最后一个节点
2. 逆转后半部分；
2. 依次插入

```
    public void reorderList(ListNode head) {
        if(head==null||head.next==null) return;
        
        ListNode fast=head.next,slow=head;
        while(fast!=null&&fast.next!=null){
            fast=fast.next.next;
            slow=slow.next;
        }
        //此时slow指向中间或者前半段
        
        //对后一部分进行翻转
        ListNode cur=slow.next;
        while(cur.next!=null){
            ListNode next=cur.next;
            
            cur.next=next.next;
            next.next=slow.next;
            slow.next=next;
        }
        
        //此时将slow以后的元素一次插入前半段每个间隔即可；
        cur=slow.next;
        ListNode pre=head;
        while(pre!=slow){//slow是最后一个奇数，pre到这里之后所有元素都在相应的位置了
            slow.next=cur.next;
            
            cur.next=pre.next;
            pre.next=cur;
            pre=cur.next;
            
            cur=slow.next;
        }
    }
```

