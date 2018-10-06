https://music.163.com/#/playlist?id=123216450
##### [31. Next Permutation](https://leetcode.com/problems/next-permutation/description/)

###### 问题
>Implement next permutation排列, which rearranges重排列 numbers into the lexicographically next greater(下一个比较大的) permutation of numbers.

>If such arrangement is not possible, it must rearrange it as the lowest possible order (ie, sorted in ascending order).

>The replacement must be in-place(原地算法), do not allocate extra memory.

>Here are some examples. Inputs are in the left-hand column and its corresponding outputs are in the right-hand column.

>1,2,3 → 1,3,2  
3,2,1 → 1,2,3  
1,1,5 → 1,5,1

意思是几个数字排列的所有可能总有大小顺序，找到所给的比所给的排列大“一个”的排列，如果所给的排列是最大的，就返回最小的排列。

###### 思路

分三步：
1. 从尾部想前找到第一个打破非严格递减排列的数字下标i，即允许有重复的数存在；
2. 如果存在这样的数字且下标为i，则找到递减数列中比这个数大的数中最小的一个，然后交换这个两个数；
3. 逆转逆转后的递减数列；

```
    //1. 从尾部想前找到第一个打破递减排列的数字下标i；todo 注意是>=符号
    //2. 找到递减尾部中大于小标为i的数字中最小的一个，将其与i交换；
    //3. 逆序交换后的递增尾部
    public void nextPermutation(int[] nums) {
        //1. 找到打破递减的数字下标,即i.todo 注意是>=符号
        int i=nums.length-2;
        for(;i>=0&&nums[i]>=nums[i+1];i--);
        
        //第二步，当i不小于0时，才可进行交换
        if(i>=0){
            int j=i+1;
            for(;j<=nums.length-1&&nums[j]>nums[i];j++);        
            swap(nums,i,j-1);
        }

        
        //第三步
        i=i+1;
        int j=nums.length-1;
        while(i<j){
            swap(nums,i++,j--);
        }
        
    }
    
    //交换nums中第i个和第j个数字
    private void swap(int nums[],int i,int j){
        nums[i]=nums[i]^nums[j];
        nums[j]=nums[i]^nums[j];
        nums[i]=nums[i]^nums[j];
    }
```

##### [34. Search for a Range](https://leetcode.com/problems/search-for-a-range/description/)

###### 问题

>Given an array of integers sorted in ascending升序 order, find the starting and ending position of a given target value.

>Your algorithm's runtime complexity must be in the order of O(log n).

>If the target is not found in the array, return [-1, -1].

>For example,
Given [5, 7, 7, 8, 8, 10] and target value 8,
return [3, 4].

###### [思路.链接](https://leetcode.com/problems/search-for-a-range/discuss/14699/Clean-iterative-solution-with-two-binary-searches-(with-explanation))


分别使用二分查找找到左边界和右边界，代码如下：

```
    public int[] searchRange(int[] nums, int target) {
        int res[]=new int[2];
        res[0]=-1;
        res[1]=-1;
        
        if(nums==null||nums.length==0) return res;
        
        int i=0,j=nums.length-1;//左右边界
        
        //找到左边界
        while(i<j){
            int mid=(i+j)/2;
            if(nums[mid]<target) i=mid+1;
            else
                j=mid;
        }
        if(nums[i]!=target) return res;
        else res[0]=i;//找到的左边界；
        
        //找到右边界:不用重置左边界，因为左边界此时已经是等于target的数字下标
        j=nums.length-1;
        while(i<j){
            int mid=(i+j)/2+1;//使其向右倾斜
            if(nums[mid]>target) j=mid-1;
            else i=mid;
        }
        res[1]=j;
        
        return res;
    }
```

##### []

