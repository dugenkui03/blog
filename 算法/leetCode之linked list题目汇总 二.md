#### 总结要点：
1. 有些题如果想不出“最优解”，可以用O(n)的空间复杂度换区简单的解法，比如克隆链表；
2. 对链表操作时，不管是反转还是拼接，<font color=red>**注意将尾节点的后续节点设置为null，不要怕多余，因为可鞥有未考虑到的情况形成回环**</font>;
3. 对于代码`b=a;c=b;d=c...`等价于`...=d=c=b=a`，复制顺序是从右向左的；
4. `int a=1,b=a;`,顺序是从左到右;
5. <font color=red>**查找一个链表倒数第n个节点：让指向头结点的first节点先走n步，然后指向头结点的second节点从开始与first保持同步的单步速度向后移动(此时两者间隙为n)，当first移动到null时，second指向倒数第n个节点。**</font><font color=blue>**如果first只想指向队尾`first.next==null`，则second指向的是倒数第n个节点的前置节点，可用来删除倒数第n个节点;**</font>
6. 链表只有一个标记节点head，当我们的递归程序参数是链表的部分时，可以调用另一个函数使其参数为两个标记节点:
```
ListNode func(ListNode head){
    ...//找到队尾
    
    return twoParams(head,tail);
}

private static LidtNode twoParams(ListNode head,ListNode tail){
    ...
}
```
7. 递归三要素：停止条件；步进以使解决问题范围变小和减小重复的计算。其中**停止条件可以有多个，注意要使得“停止处理”处理到任何情况；**；
##### 328.[Odd Even Linked List](https://leetcode.com/problems/odd-even-linked-list/description/)

###### 问题
将链表奇数位置和偶数位置的节点分开，并保持各自的相对顺序；

###### 思路：指针
```
    public ListNode oddEvenList(ListNode head) {
        if(head==null||head.next==null||head.next.next==null) return head;
        
        ListNode odd=head,evenHead=head.next,even=evenHead;
        while(even!=null&&even.next!=null){
            odd.next=even.next;
            odd=odd.next;
            
            even.next=odd.next;
            even=even.next;
        }
        
        odd.next=evenHead;
        
        return head;
    }
```

##### [24. Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs/description/)

###### 问题
交换每对相邻的两个链表位置

###### 思路
指针、虚头部、travNode及其前后节点，以其是否为null（偶数个节点）和后续节点是否为null为结束条件。

```
    public ListNode swapPairs(ListNode head) {
        while(head==null||head.next==null) return head;
        
        ListNode dummy=new ListNode(0),pre=dummy;
        dummy.next=head;
        
        while(head!=null&&head.next!=null){
            ListNode next=head.next;
            
            head.next=next.next;
            next.next=pre.next;
            pre.next=next;
            
            pre=head;
            head=head.next;
        }
        
        return dummy.next;
    }
```

<font color=red>**重点题目**</font>:**[86. Partition List](https://leetcode.com/problems/partition-list/description/)**

###### 问题
给一个值x，将链表中小于x的节点和其他节点分开，并保留相对顺序（前小后大）；

###### 思路
<font color=red>**使用指针将链表拆解为两个链表，并且使用标记节点标记两条链表的头部，使用travNode节点对辅助两条链表各取所需要的节点.**</font>

```
    public ListNode partition(ListNode head, int x) {
        if(head==null||head.next==null) return head;
        
        ListNode smallHead=new ListNode(0),bigHead=new ListNode(0),sp=smallHead,bp=bigHead;
        
        while(head!=null){
            if(head.val<x){
                sp.next=head;
                sp=sp.next;
                //可以用一句代码代替 sp=sp.next=head;
            }else{
                bp.next=head;
                bp=bp.next;
            }
            head=head.next;
        }
        
        sp.next=bigHead.next;
        bp.next=null;//一定要注意队尾的后续节点为null
        return smallHead.next;
    }
```
- **一定要注意队尾的后续节点为null**；


##### X.X.X链表的插入排序

###### 问题

###### 思路
当我们对某一节点指针操作时，又不想丢失其后续节点的位置，简单直接的方法是<font color=red>**用临时指针指向这个节点**</font>。例如反转链表：
```
ListNode pre=null;

while(head!=null){
    ListNode next=head.next;//临时接单保存后续节点位置；
    
    head.next=pre;
    pre=head;
    head=next;
}

return pre;

```
**插入的位置是“后大前小”**
```
    public ListNode insertionSortList(ListNode head) {
        if(head==null||head.next==null) return head;
        
        ListNode dummy=new ListNode(0),pre=dummy;
        while(head!=null){
            //临时节点保存节点位置
            ListNode next=head.next;
            //查找插入位置（pre后续节点）
            while(pre.next!=null&&pre.next.val<head.val){
                pre=pre.next;
            }
            //插入
            head.next=pre.next;
            pre.next=head;
            //归位
            head=next;
            pre=dummy;
        }
        
        return dummy.next;
    }

```

##### [19. Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/description/)

###### 问题
删除倒数第n个节点；

###### 思路

 <font color=red>**查找一个链表倒数第n个节点：让指向头结点的first节点先走n步，然后指向头结点的second节点从开始与first保持同步的单步速度向后移动(此时两者间隙为n)，当first移动到null时，second指向倒数第n个节点。**</font><font color=blue>**如果first只想指向队尾`first.next==null`，则second指向的是倒数第n个节点的前置节点，可用来删除倒数第n个节点;**</font>

```
    public ListNode removeNthFromEnd(ListNode head, int n) {
        ListNode first=head,second=head;
        
        while(n--!=0) first=first.next;
        
        if(first==null) return head.next;//节点包括n个元素，因此把第一个删除即可；
        
        while(first.next!=null){//first!=null做停止条件时second指向的时要删除的节点，此条件停止时second指向的是要删除节点的前置节点
            second=second.next;
            first=first.next;
        }
        second.next=second.next.next;
        
        return head;
    }
```

##### <font color=red>**重点题目:**</font>[109. Convert Sorted List to Binary Search Tree](https://leetcode.com/problems/convert-sorted-list-to-binary-search-tree/description/)
###### 问题

将一个递增排序的链表转换为一个BST;

###### 思路
<font color=red>
1. BST的性质

    1. 节点左子树所有节点比他小，右子树所有节点值比他大；
    2. 每个节点两个子树高度相差不超过1；
    3. 中根遍历得出数值的顺序是递增的；
如上，可以先把节点排序；

2. 递归三要素：停止条件；步进以使解决问题范围变小和减小重复的计算。其中**停止条件可以有多个，注意要使得“停止处理”处理到任何情况；**；

```
public TreeNode sortedListToBST(ListNode head) {
        //fixme 停止条件可以不只是一种
        //如果没有第二个停止条件，那么最左侧将进入死循环；
        if(head==null) return null;
        if(head.next==null) return new TreeNode(head.val);
        
        ListNode fast=head,slow=head,pre=head;
        while(fast!=null&&fast.next!=null){
            pre=slow;//pre是slow的前置节点，用于取消指向slow的指针
            slow=slow.next;
            fast=fast.next.next;
        }
        //奇数个节点时slow指向中间，偶数个节点时指向后半段起始节点；
        
        TreeNode treeHead=new TreeNode(slow.val);
        pre.next=null;
        treeHead.left=sortedListToBST(head);
        treeHead.right=sortedListToBST(slow.next);
        
        return treeHead;
}
```

上述程序将链表裁断。也可以调用其他多个参数的函数，并用标记节点来标记链表段：

```
   public TreeNode sortedListToBST(ListNode head) {
        if(head==null) return null;
        
        return getTree(head,null);
    }
    
    private TreeNode getTree(ListNode head,ListNode tail){
        if(head==tail) return null;//注意停止条件
        
        ListNode walk=head,run=head;
        while(run!=tail&&run.next!=tail){//注意条件run.next!=null
            run=run.next.next;
            walk=walk.next;
        }
        TreeNode mid=new TreeNode(walk.val);
        mid.left=getTree(head,walk);
        mid.right=getTree(walk.next,tail);
        
        return mid;
    }
```
</font>

#### [92. Reverse Linked List II](https://leetcode.com/problems/reverse-linked-list-ii/description/)

###### 问题
给定一个链表和起始节点，反转起始节点。例如：
>Given 1->2->3->4->5->NULL, m = 2 and n = 4,
>
>return 1->4->3->2->5->NULL.

###### 思路
找到需要翻转的起始节点和反转次数，然后开始反转；

**x个需要翻转的节点，只需要反转x-1次**

```
/*
 * 思路：找到需要翻转的链表的前一个节点，然后反转之后的部分，并不断将反转之前的第一个节点指向后续节点
 */
class Solution {
    public ListNode reverseBetween(ListNode head, int m, int n) {
        if(head==null||head.next==null||m==n) return head;
        
        ListNode dummy=new ListNode(0),pre=dummy;
        dummy.next=head;
        
        n=n-m;
        while(--m!=0){
            pre=pre.next;
        }
        //dummy指向队头，pre指向要翻转的链表的前一个节点，n是要反转的次数
        
        //反转
        ListNode current=pre.next;   
        while(n--!=0){
             ListNode next=current.next;

             current.next=next.next;
             next.next=pre.next;
             pre.next=next;
        }
        
        return dummy.next;
    }
}

```
##### [148 Merge sort](https://leetcode.com/problems/sort-list/description/)

###### 问题
对链表进行排序，时间复杂度O(n.logn)，空间复杂度控制在常量级；
```
    static ListNode result=new ListNode(0);
    
    public ListNode sortList(ListNode head) {
        if(head==null||head.next==null) return head;
        
        //cut list into two
        ListNode pre=null,slow=head,fast=head;
        while(fast!=null&&fast.next!=null){
            pre=slow;
            slow=slow.next;
            fast=fast.next.next;
        }
        
        pre.next=null;
        
        //recurse sort
        ListNode l1=sortList(head);
        ListNode l2=sortList(slow);
        
        return merge(l1,l2);
    }
    
    static ListNode merge(ListNode l1,ListNode l2){
        
        ListNode p=result;
        
        while(l1!=null&&l2!=null){
            if(l1.val<l2.val){
                p.next=l1;
                l1=l1.next;
            }else{
                p.next=l2;
                l2=l2.next;
            }
            p=p.next;
        }
        
        if(l1==null){
            p.next=l2;
        }else{
            p.next=l1;
        }
        
        return result.next;
    }
```
##### [138. Copy List with Random Pointer](https://leetcode.com/problems/copy-list-with-random-pointer/description/)

###### 问题
>A linked list is given such that each node contains an additional random pointer which could point to any node in the list or null.
Return a deep copy of the list

###### 思路

见代码，分三步：
1. 首先复制每个节点及其next指针，使 1->2->3 变成 1->1'->2->2'->3->3`；
2. 然后复制random指针，画出图示即可知道 x'random指针指向的位置是x.random.next，即x.next.random=x.random.next;
3. 然后将两条连在一起且一样的链表分开即可；

```
/**
 * Definition for singly-linked list with a random pointer.
 * class RandomListNode {
 *     int label;
 *     RandomListNode next, random;
 *     RandomListNode(int x) { this.label = x; }
 * };
 */
 
    public RandomListNode copyRandomList(RandomListNode head) {
        if(head==null) return head;
        
        //首先复制每个节点及其next指针，使 1->2->3 变成 1->1'->2->2'->3->3`
        RandomListNode pre=head;
        while(pre!=null){
            RandomListNode next=pre.next;
            
            RandomListNode tmp=new RandomListNode(pre.label);
            pre.next=tmp;
            tmp.next=next;
            
            pre=next;
        }
        
        //然后复制random指针，画出图示即可知道 x'random指针指向的位置是x.random.next，即x.next.random=x.random.next;
        pre=head;
        while(pre!=null){
            if(pre.random!=null)
                pre.next.random=pre.random.next;
            pre=pre.next.next;
        }
        
        //然后将两条连在一起且一样的链表分开即可
        RandomListNode result=head.next,rp=result;
        pre=head;
        while(rp.next!=null){
            pre.next=rp.next;
            pre=pre.next;
            
            rp.next=pre.next;
            rp=rp.next;
        }
        //注意一定要处理尾节点
        rp.next=null;
        pre.next=null;
        
        return result;
    }
```

##### [25. Reverse Nodes in k-Group](https://leetcode.com/problems/reverse-nodes-in-k-group/description/)

###### 问题
>Given a linked list, reverse the nodes of a linked list k at a time and return its modified list.

>k is a positive integer and is less than or equal to the length of the linked list. If the number of nodes is not a multiple of k then left-out nodes in the end should remain as it is.

>You may not alter the values in the nodes, only nodes itself may be changed.

>Only constant memory is allowed.

>For example,
Given this linked list: 1->2->3->4->5
For k = 2,you should return: 2->1->4->3->5
For k = 3, you should return: 3->2->1->4->5

###### 思路

找到需要反转的起始节点，并且将其翻转后并与其他部分链接即可。注意有其实节点的“链表段反转方式(当然也可以在反转之后并设置连接,这样的话参数位置就不一样了)：
```
    private static ListNode reverse(ListNode head,ListNode tail){//这两个参数已经没有了前后顺序
        if(head==null||head.next==null) return head;

        tag=head;
        ListNode pre=null;
        while(pre!=tail){//注意这里的停止条件
            ListNode next=head.next;

            head.next=pre;
            pre=head;
            head=next;
        }

        return pre;
    }
```
代码
```
    public ListNode reverseKGroup(ListNode head, int k) {
        if(head==null||head.next==null||k==1) return head;

        ListNode dummy=new ListNode(0),dummy2=dummy,countIndex=dummy2;
        dummy.next=head;
        
        ListNode preCurrent=head,current;

        while(preCurrent!=null){
            //检查以后的长度是否足够反转用
            int count=k;
            countIndex=dummy2;
            while(countIndex!=null&&count!=0){
                countIndex=countIndex.next;
                count--;
            }
            if(countIndex!=null&&count==0){//此时的长度可供翻滚
                
                //对k-1个元素进行反转，preCUrrent是第一个且是翻转后位于最后一个的元素
                int cou=k;
                while(cou--!=1){
                    current=preCurrent.next;

                    preCurrent.next=current.next;
                    current.next=dummy2.next;
                    dummy2.next=current;
                }
                
                //归位
                dummy2=preCurrent;
                preCurrent=preCurrent.next;
            }else break;
        }
        return dummy.next;
    }
```

##### [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/description/)
###### 问题
给定一个链表数组（其中可能有null），其中的链表都是排好序的，返回一个包含所有节点的排好序的链表。并且分析其时间复杂度；

###### 思路
利用java的数据结构类**优先队列**,优先队列插入一个元素的时间复杂度是O(logn),删除一个元素的时间复杂度也是O(logg),因此操作n个元素的时间复杂度O(nlogn);

<font color=red>
1. 所有节点都要放进优先队列，所以插入次数是一定的；
2. 所有链表都是排好序的，因此如果只把每个链表的“遍历节点”放进优先队列，就可以保证最小节点一定在优先队列中；
3. 因此在优先队列里边只保存每个链表的“遍历节点”，比先把全部节点放进队列，在每次插入时队列里边有更少的元素，在删除节点时，也可以用更少的步骤进行调整；
</font>

```
    public static ListNode mergeKLists(ListNode[] lists) {
        //如果没有或者只有一个链表，则直接返回即可
        if(lists==null|| lists.length==0||lists.length==1){
            return (lists==null|| lists.length==0)?null:lists[0];
        }

        //设置优先队列：插入和删除的时间复杂度都是O(n)。注意其构造参数是队列容量和“比较类”；
        PriorityQueue<ListNode> queue=new PriorityQueue<>(lists.length,new Comparator<ListNode>() {
            @Override
            public int compare(ListNode o1, ListNode o2) {
                if(o1.val<o2.val)
                    return -1;
                return 1;
            }
        });

        // 将所有元素插入有限队列，时间复杂度是O(nlogn)；
        for (ListNode list:lists) {
            if(list!=null){
                queue.add(list);
            }
        }
        //取出所有元素并插入虚头部后边,插入元素的时间复杂度是O(logn);
        ListNode dummy=new ListNode(0),pre=dummy;
        while(!queue.isEmpty()){
            pre.next=queue.poll();
            pre=pre.next;
            
            if(pre.next!=null)
                queue.add(pre.next);
        }

        return dummy.next;
    }
```

##### []()

###### 问题
将链表向右旋转k个位置,比如：1->2->3向右旋转1个位置就是3->1->2，倒数第一个节点转到了前边，两个位置就是2->3->1。2+3*n个位置同2个位置一样；

###### 思路
1. 向右旋转k个位置等效于k%n个位置，并且新头部是倒数第n个节点；
2. 找到尾节点，如果k%size==0则直接返回head，否则将链表置为环装；
3. 利用“步数差”找到新头部的前置节点；
4. 尾节点设置为null，并返回新头部；

```
    //向右旋转k个位置等效于k%n个位置，并且新头部是倒数第n个节点；
    public ListNode rotateRight(ListNode head, int k) {
        if(k==0||head==null||head.next==null) return head;
        
        //找到尾节点，如果k%size==0则直接返回head，否则将链表置为环装
        ListNode tmp=new ListNode(0),pre=tmp;
        tmp.next=head;
        int size=0;
        while(tmp.next!=null){
            size++;
            tmp=tmp.next;
        }
        
        k=k%size;//更新k
        if(k==0) return head;
        
        tmp.next=head;//环装
        

        
        //利用“步数差”找到新头部的前置节点
        while(k--!=0){
            tmp=tmp.next;
        }
        while(tmp.next!=head){
            tmp=tmp.next;
            pre=pre.next;
        }
        
        //尾节点设置为null，并返回新头部
        ListNode newHead=pre.next;
        pre.next=null;
        
        return newHead;
    }
```