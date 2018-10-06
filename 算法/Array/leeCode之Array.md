
##### [1. Two Sum](https://leetcode.com/problems/two-sum/description/)

###### 问题
一个数组中一定有两个数字和`为target`，求出这两个数字的下标。

###### 思路

将数字一次存储在`HashMap<数值，下标>`中，然后每加入一个数字就检查是否`Map`中是否有与其和为`target`的数字，有则返回。

<font color=red>**注意`key`在`HashMap`中的存储方式是哈希表，所以查找速度会很快。**</font>

```
    /**
     *  返回数组后欧能和为target的两个数的下标
     */
    public int[] twoSum(int[] nums, int target) {
        //key是数值大小，value是数组下标
        Map<Integer,Integer> result=new HashMap<>();

        for (int i = 0; i < nums.length; i++) {
            //如果数组中已经有值为 target-nums[i] 的数，即两者之和为target，则返回
            if(result.containsKey(target-nums[i])){
                return new int[]{i,result.get(target-nums[i])};
            }
            result.put(nums[i],i);
        }
        return null;
    }
```

##### [15. 3Sum](https://leetcode.com/problems/3sum/description/)

##### 问题
给定一个数组，求出所有和为0的唯一三元组；
>Given an array S of n integers, are there elements a, b, c in S such that a + b + c = 0? Find all unique triplets in the array which gives the sum of zero.

##### [思路](https://leetcode.com/problems/3sum/discuss/7380/Concise-O(N2)-Java-solution)

将数组排序，然后从第一个开始依次向后遍历，并且“被遍历节点”后的数据段用两个界限游标指向，并不断聚合；

```
    public List<List<Integer>> threeSum(int[] nums) {
        //同Collectoins和Executors有很多静态工具方法，数组也有对应的Arrays
        Arrays.sort(nums);
        List<List<Integer>> res=new ArrayList<>();
        
        for(int i=0;i<nums.length-2;i++){
            if(i>0&&nums[i]==nums[i-1]) continue;
            
            //i后续数据段的上下边界
            int low=i+1;
            int high=nums.length-1;
            
            while(low<high){
                if(nums[i]+nums[low]+nums[high]<0){//1. while(low<high){ if(XX) ... }应该等价于while(YY&&XX){...}
                    low++;
                }else if(nums[i]+nums[low]+nums[high]>0){
                    high--;
                }else{
                    res.add(Arrays.asList(nums[i],nums[low],nums[high]));
                    
                    //这里会知道找到一个与nums[low]不同的数才停止自增
                    //注意：low和high开始时已经变了一次了，因此nums[low-1]是旧的位置，nums[low]是新的位置
                    do{ low++; }while(low<high&&nums[low]==nums[low-1]);
                    do{ high--;}while(low<high&&nums[high]==nums[high+1]);
                }
            }
        }
        return res;
    }
```


##### [16. 3Sum Closest](https://leetcode.com/problems/3sum-closest/description/)

###### 问题
>Given an array S of n integers, find three integers in S such that the sum is closest to a given number, target. Return the sum of the three integers. You may assume that each input would have exactly one solution.

>For example, given array S = {-1 2 1 -4}, and target = 1.
The sum that is closest to the target is 2. (-1 + 2 + 1 = 2).

###### 思路
同上一题，但是这里每遍历一个位置则记录一次最小值

```
    public int threeSumClosest(int[] nums, int target) {
        Arrays.sort(nums);//排序
        
        int closetNum=0;
        boolean flag=false;//是否是第一次进入
        for(int i=0;i<nums.length-2;i++){
            if(i>0&&nums[i]==nums[i-1]) continue;//与前一个数字重复，则不用在进行计算
            
            int left=i+1;
            int right=nums.length-1;
            
            int nowSum=0;
            while(left!=right){
                nowSum=nums[i]+nums[left]+nums[right];
                if(target>nowSum){
                    left++;
                }else if(target<nowSum){
                    right--;
                }else{//找到三者之和与target相等的，直接返回target即可
                    return target;
                }
                
                if(flag==true&&Math.abs(closetNum-target)>Math.abs(nowSum-target)){
                    closetNum=nowSum;
                }else if(flag==false){
                    closetNum=nowSum;
                    flag=true;
                }
            }
        }
        
        return closetNum;
    }
```




[11. Container With Most Water](https://leetcode.com/problems/container-with-most-water/description/)

###### 问题

>Given n non-negative integers a1, a2, ..., an, where each represents a point at coordinate(坐标) (i, ai). n vertical(垂直的) lines are drawn such that the two endpoints(端点) of line i is at (i, ai) and (i, 0)（两个垂直的线段通过端点ai和0画出来）. Find two lines, which together with x-axis forms a container（连着x轴形成一个容器）, such that the container contains the most water.

>Note: You may not slant(倾斜) the container and n is at least 2.

###### 思路

在两个端点设置两个指针，然后比较短的那个指针向中间靠拢，并且计算每个当前位置容器容量大小，详情参见[leecode_discuss](https://leetcode.com/problems/container-with-most-water/discuss/)

- `!= or ==`快于`< or >`;

```
    public int maxArea(int[] height) {
        //两个指针指向数组边界
        int l=0,r=height.length-1;
        //当前“最大面积”
        int maxArea=0;
        
        //两个指针向中间移动
        while(l!=r){
            //如果左侧指针指向垂线高度较小，则向右步进
            if(height[l]<=height[r]){
                maxArea=(height[l]*(r-l)>maxArea?height[l]*(r-l):maxArea);
                l++;
            }
            //注意是可以连着用两个if而非if-else的，因为刚进入while时两个指针肯定指向不同，
            //而其一步进后如果指向相同，则高度一定不同，不会满足if判断
            if(height[l]>height[r]){
                maxArea=(height[r]*(r-l)>maxArea?height[r]*(r-l):maxArea);
                r--;
            }
        }
        
        return maxArea;
    }
```


##### [26. Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array/description/)

###### 问题
>Given a sorted array, remove the duplicates in-place such that each element appear only once and return the new length.

>Do not allocate extra space for another array, you must do this by modifying the input array in-place with O(1) extra memory.

移除一个排序数组中的重复元素（保留一个），并且返回处理后的数组长度。

###### 思路

使用一个指针指向第一个元素，然后不断将不重复的元素放进指针指向的下一个位置。



```
    public int removeDuplicates(int [] nums){
        if(nums==null||nums.length==1) return nums==null?0:1;
        int diffIndex=0;

        for (int i = 1; i < nums.length; i++) {
            //如果与后一个数不相等
            if(nums[diffIndex]!=nums[i]){
                diffIndex++;
                nums[diffIndex]=nums[i];
            }
        }

        return diffIndex+1;
    }
```

##### [27. Remove Element](https://leetcode.com/problems/remove-element/description/)

###### 问题
>Given an array and a value, remove all instances of that value in-place and return the new length.
Do not allocate extra space for another array, you must do this by modifying the input array in-place with O(1) extra memory.
The order of elements can be changed. It doesn't matter what you leave beyond the new length.

###### 思路
同上一题

```
    public int removeElement(int[] nums, int val) {
        
        int validValueIndex=0;
        for(int i=0;i<nums.length;i++){
            if(nums[i]!=val){
                nums[validValueIndex]=nums[i];
                validValueIndex++;
            }
        }
        
        return validValueIndex;
    }
```

##### 二分查找

```
    //排序数组：找到元素所在位置下标，不存在则返回插入位置下标
    public int searchInsert(int[] nums, int target) {
        if(nums==null||nums.length==0) return 0;

        int low=0,heigh=nums.length-1;
        //比较条件时low<=heigh，没有=的话只有一个元素会报错
        while(low<=heigh){
            int mid=(low+heigh)/2;
            if(nums[mid]==target) return mid;
            else if(nums[mid]>target) heigh=mid-1;
            else low=mid+1;
        }
        
        return low;//注意是返回low
    }
```