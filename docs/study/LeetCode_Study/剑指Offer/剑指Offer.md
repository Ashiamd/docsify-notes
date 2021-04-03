# 剑指Offer

> [leetcode-LCOF](https://leetcode-cn.com/problemset/lcof/)

### 面试题57 - II. 和为s的连续正数序列

> [面试题57 - II. 和为s的连续正数序列](https://leetcode-cn.com/problems/he-wei-sde-lian-xu-zheng-shu-xu-lie-lcof/)

语言：java

思路：数学的方法就不去细推了。这里用滑动窗口解决问题(这个比较好理解)，如果当前和小于目标值，就右边界向右滑动；如果当前和大于目标值，就左边界向右滑动；如果当前和等于目标值，则记录下来。

代码：（3 ms，82.69%）

```java
class Solution { 
  public int[][] findContinuousSequence(int target) { 
    List<int[]> resList = new LinkedList<>(); 
    // 滑动窗口
    for(int l = 1,r = 1,sum=0;r<=(target/2+1);++r){
      sum+=r; 
      while(sum>target){
        sum-=l; 
        ++l;
      }
      if(sum==target)  {
        int[] resTmp = new int[r-l+1];
        for(int i =0;i<resTmp.length;++i){
          resTmp[i] = l+i;
        }
        resList.add(resTmp);
      }
      int[][] res = new int[resList.size()][];
      for(int i =0;i<res.length;++i){ 
        res[i] = resList.get(i); 
      }
      return res; 
    }
  }
}
```

参考代码(0ms):这个直接看应该是数学方法，看着就很抽象

```java
class Solution {
    public int[][] findContinuousSequence(int target) {
        List<int[]> result = new ArrayList<>();
        int i = 1;

        while(target>0)
        {
            target -= i++;
            if(target>0 && target%i == 0)
            {
                int[] array = new int[i];
                for(int k = target/i, j = 0; k < target/i+i; k++,j++)
                {
                    array[j] = k;
                }
                result.add(array);
            }
        }
        Collections.reverse(result);
        return result.toArray(new int[0][]); 
    }
}
```

### 面试题68 - I. 二叉搜索树的最近公共祖先

> [面试题68 - I. 二叉搜索树的最近公共祖先](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-de-zui-jin-gong-gong-zu-xian-lcof/)

语言：java

思路：题目关键就是“**二叉搜索树**”。搜索树即左子树小于根节点，右子树大于根节点的树。如果找到了公共祖先，这个公共祖先要么正好大小夹在两个节点之间，要么就正好是其中一个节点。

代码（7ms，55.9%）：

```java
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if(root==null)
            return null;
        if(p.val<root.val&&q.val<root.val){
            return lowestCommonAncestor(root.left,p,q);
        }
        if(p.val>root.val&&q.val>root.val){
            return lowestCommonAncestor(root.right,p,q);
        }
        return root;
    }
}
```

参考代码(6ms,44.91%)：其实就是递归换成非递归。

```java
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if (root == null){
            return root;
        }
        while (root != null){
            if (root.val < p.val && root.val < q.val){
                root = root.right;
                continue;
            }
            if ( root.val > p.val && root.val > q.val){
                root = root.left;
                continue;
            }
            break;
        }
        return root;
    }
}
```

### 面试题03. 数组中重复的数字

> [面试题03. 数组中重复的数字](https://leetcode-cn.com/problems/shu-zu-zhong-zhong-fu-de-shu-zi-lcof/)

语言：java

思路：抽屉原理。如果把数组看作抽屉，那么原本抽屉里面存放的内容不一定正好是"下标"，我们把当前拿到的抽屉里面的内容和其"下标"对应的抽屉里的内容置换。如果当前的抽屉装的内容不是其"下标"，而内容对应的数字"下标"的抽屉已经存放了现在这个内容，说明这个内容已经重复过了。假设每个抽屉都没有重复内容，那么顶多就是每个抽屉的内容最后都和其"下标"一致，而不会出现前面那种冲突的情况。

代码（1ms，92.39%）：

```java
class Solution {
    public int findRepeatNumber(int[] nums) {
        for(int i = 0,len = nums.length;i<len;++i){
            if(nums[i]!=i&&nums[i]==nums[nums[i]])
                return nums[i];
            int tmp = nums[nums[i]];
            nums[nums[i]] = nums[i];
            nums[i] = tmp;
        }
        return -1;
    }
}
```

### 面试题39. 数组中出现次数超过一半的数字

> [面试题39. 数组中出现次数超过一半的数字](https://leetcode-cn.com/problems/shu-zu-zhong-chu-xian-ci-shu-chao-guo-yi-ban-de-shu-zi-lcof/)

语言：java

思路：排序后找中间那个。因为出现次数超过一半，所以中间那个肯定会是答案。

代码(2ms,76.6%)：

```java
class Solution {
    public int majorityElement(int[] nums) {
        Arrays.sort(nums);
        return nums[nums.length/2];
    }
}
```

参考代码（下述文字、代码摘自下述文章中）：

**摩尔投票法：** 核心理念为 **“正负抵消”** ；时间和空间复杂度分别为 O(N) 和 O(1)；是本题的最佳解法。

**算法流程**:

1. 初始化： 票数统计 votes = 0， 众数 x；

2. 循环抵消： 遍历数组 nums 中的每个数字 num；

+ 当 票数 votes 等于 0 ，则假设 当前数字 num为 众数 x ；

+ 当 num = x时，票数 votesvotes 自增 1 ；否则，票数 votes 自减 1 。

3. 返回值： 返回 众数x即可。

> [面试题39. 数组中出现次数超过一半的数字（摩尔投票法，清晰图解](https://leetcode-cn.com/problems/shu-zu-zhong-chu-xian-ci-shu-chao-guo-yi-ban-de-shu-zi-lcof/solution/mian-shi-ti-39-shu-zu-zhong-chu-xian-ci-shu-chao-3/)

```java
class Solution {
    public int majorityElement(int[] nums) {
        int x = 0, votes = 0;
        for(int num : nums){
            if(votes == 0) x = num;
            votes += num == x ? 1 : -1;
        }
        return x;
    }
}
```

参考代码2(1ms): 同样是摩尔投票法。

```java
class Solution {
    public int majorityElement(int[] nums) {
        if (nums == null || nums.length == 0) {
            throw new RuntimeException("参数错误");
        }
        int cnt = 1;
        int preNum = nums[0];
        for (int i = 1; i < nums.length; i++) {
            if (nums[i] == preNum) {
                cnt++;
            } else {
                cnt--;
                if (cnt == 0) {
                    preNum = nums[i];
                    cnt++;
                }
            }

        }
        return preNum;
    }
}
```

### 面试题68 - II. 二叉树的最近公共祖先

> [面试题68 - II. 二叉树的最近公共祖先](https://leetcode-cn.com/problems/er-cha-shu-de-zui-jin-gong-gong-zu-xian-lcof/)

语言：java

思路：很明显需要递归。边界条件需要想一下，就是如果当前节点是需要找的节点或者null，就返回。而对于左右子树，这里如果左子树没找到对应的节点会返回null，说明p和q都在右子树，反之亦然。如果左右子树都不为null，则说明当前根节点就是最近的公共祖先。

代码（8ms,86.17%）：

```java
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if(root==null||root==p||root==q)
            return root;
        TreeNode left = lowestCommonAncestor(root.left, p, q);
        TreeNode right = lowestCommonAncestor(root.right, p, q);
        if(left==null)
            return right;
        if(right==null)
            return left;
        return root;
    }
}
```

参考代码（7ms）：思想一致，就是可以少往下走一层，因为如果左右节点为null就没必要往下判断了。

```java
class Solution {
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        if(root==null||root==p||root==q){
            return root;
        }
        TreeNode left=lowestCommonAncestor(root.left,p,q);
        TreeNode right=lowestCommonAncestor(root.right,p,q);
        if(left==null&&right==null){
            return null;
        }
        if(left==null) return right;
        if(right==null) return left;
        return root;
    }
}
```

### 面试题57. 和为s的两个数字

> [面试题57. 和为s的两个数字](https://leetcode-cn.com/problems/he-wei-sde-liang-ge-shu-zi-lcof/)

语言：java

思路：x和y分别表示两个数字，一个从左往右，一个从右往左。

代码（2ms，98.84%）：

```java
class Solution {
    public int[] twoSum(int[] nums, int target){
        for(int len=nums.length,x=0,y=len-1,sum=0;x<y;){
            sum = nums[x]+nums[y];
            if(sum==target)
                return new int[]{nums[x],nums[y]};
            else{
                if(sum<target)
                    ++x;
                else
                    --y;
            }
        }
        return null;
    }
}
```

参考代码1（0ms）：**试了一下，这个其实不完全正确.**

如果输入的是

[2，2，3，3，3，3，3，6]

6

并不能得到正确答案[3,3],会返回-1

```java
class Solution { 
    public int[] twoSum(int[] nums, int target) {
        int[] result = {-1, -1};
        int numMid = getNumMid(target);
        int indexNumMid = findMidIndex(nums, numMid);
        result = findLeftAndRight(nums, indexNumMid, target);   
        return result;
    }

    private int getNumMid(int target) {
        int numMid = -1;
        if(target % 2 == 1) {
            numMid = target/2;
        }
        else {
            numMid = target/2 - 1;
        }
        return numMid;
    }

    private int findMidIndex(int[] nums, int target) {
        int indexLeft = 0;
        int indexRight = nums.length - 1;
        while(indexLeft <= indexRight) {
            int numMid = (indexLeft + indexRight) / 2;
            if(nums[numMid] == target) {//
                return numMid;
            }
            else if(nums[numMid] < target) {
                indexLeft = numMid + 1;
            }
            else if(nums[numMid] > target) {
                indexRight = numMid - 1;
            }
        }
        return indexRight;
    }

    private int[] findLeftAndRight(int[] nums, int indexNumMid, int target) {
        int indexCur = indexNumMid;
        int[] result = {-1, -1};
        while(indexCur >= 0) {
            int numAnother = target - nums[indexCur];
            for (int i = indexNumMid; i < nums.length; i++) {
                if(nums[i] > numAnother) {
                    break;
                }
                if(nums[i] == numAnother) {
                    result[0] = nums[indexCur];
                    result[1] = numAnother;
                    return result;
                }
            }
            indexCur --;
        }
        return result;
    }
}
```

参考代码2（1ms）：单次二分查找，**同样不完全正确**，遇到上面参考代码1提到的输入时，返回空数组[]

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        int[] arr=new int[2];
        int[] arr1={};
        if(nums.length<=1){
            return arr1;
        }
        int m=target/2;
        int i=0;
        int j=0;
        for(int x=0;x<nums.length;x++) {
            if(nums[x]>m) {
                i=x-1;
                j=x;
                break;
            }

        }
        while(nums[i]+nums[j]!=target) {
            if(nums[i]+nums[j]<target) {
                j++;
            }
            if(nums[i]+nums[j]>target) {
                i--;
            }
            if(i<0||(j>=nums.length)){
                break;
            }
        }
        if(i<0||(j>=nums.length)){
            return arr1;
        }else{
            arr[0]=nums[i];
            arr[1]=nums[j];
            return arr;

        }

    }
}
```

### 面试题21. 调整数组顺序使奇数位于偶数前面

> [面试题21. 调整数组顺序使奇数位于偶数前面](https://leetcode-cn.com/problems/diao-zheng-shu-zu-shun-xu-shi-qi-shu-wei-yu-ou-shu-qian-mian-lcof/)

语言：java

思路：双指针，一个往后，一个往前，类似一次快排的过程。左边找到不是奇数的位置，右边找到不是偶数的位置，然后调换

代码（2ms，99.92%）：

```java
class Solution {
    public int[] exchange(int[] nums) {
        // 0001 1110 0011 1100 0111 1111 0010 1101 1001 1110 1000 1101 0010
        // 双指针
        int i = 0,j=nums.length-1;
        while(i<j){
            while(i<j&&(nums[j] & 1) == 0)
                --j;
            while(i<j&&(nums[i] & 1) == 1)
                ++i;
            if(i<j){
                int tmp = nums[i];
                nums[i] = nums[j];
                nums[j] = tmp;
                ++i;
                --j;
            }
        }
        return nums;
    }
}
```

参考代码（52ms,16.51%）：

**快慢双指针**

+ 定义快慢双指针fast和low，fast 在前，low 在后 .

+ fast的作用是向前搜索奇数位置，low的作用是指向下一个奇数应当存放的位置

+ fast向前移动，当它搜索到奇数时，将它和 nums[low] 交换，此时 low 向前移动一个位置 

+ 重复上述操作，直到 fast 指向数组末尾 .

> [快慢指针法](https://leetcode-cn.com/problems/diao-zheng-shu-zu-shun-xu-shi-qi-shu-wei-yu-ou-shu-qian-mian-lcof/solution/ti-jie-shou-wei-shuang-zhi-zhen-kuai-man-shuang-zh/)

```c++
class Solution {
public:
    vector<int> exchange(vector<int>& nums) {
        int low = 0, fast = 0;
        while (fast < nums.size()) {
            if (nums[fast] & 1) {
                swap(nums[low], nums[fast]);
                low ++;
            }
            fast ++;
        }
        return nums;
    }
};
```

参考快慢指针后，写个java版的：(2ms,99.92%)。这里需要注意的是fast必须从0开始判断，不然low可能少走一次

```java
class Solution {
    public int[] exchange(int[] nums) {
        int low = 0,len = nums.length,fast = 0;
        while(fast<len){
            if(1==(nums[fast]&1)){
                if(low!=fast){
                    int tmp = nums[low];
                    nums[low] = nums[fast];
                    nums[fast] = tmp;
                }
                ++low;
            }
            ++fast;
        }
        return nums;
    }
}
```

### 面试题52. 两个链表的第一个公共节点

> [面试题52. 两个链表的第一个公共节点](https://leetcode-cn.com/problems/liang-ge-lian-biao-de-di-yi-ge-gong-gong-jie-dian-lcof/)

语言：java

思路：假设A和B的相交部分长度c，各自非相交的部分长度为a和b；也就是A自己走完就走B，而B自己走完就走A，如果是相交的，那么相同的时候，节点不为null，否则相遇的点就是null。

+ 相交：c = c -> a+b+c = b+a+c
+ 不相交：c=0 -> a+b = b+a

代码（1ms，100%）：

```java
public class Solution {
    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        ListNode a = headA,b = headB;
        while(a!=b){
            a = a==null?headB:a.next;
            b = b==null?headA:b.next;
        }
        return a;
    }
}
```

### 面试题62. 圆圈中最后剩下的数字

> [面试题62. 圆圈中最后剩下的数字](https://leetcode-cn.com/problems/yuan-quan-zhong-zui-hou-sheng-xia-de-shu-zi-lcof/)

语言：java

思路：约瑟夫环（其实已经算数学题了）[约瑟夫环——公式法（递推公式）](https://blog.csdn.net/u011500062/article/details/72855826)

代码：(7ms,99.82%)

```java
class Solution {
    public int lastRemaining(int n, int m) {
        int res = 0;
        for(int i =1;i<=n;++i){
            res = (res+m)%i;
        }
        return res;
    }
}
```

### 面试题50. 第一个只出现一次的字符

> [面试题50. 第一个只出现一次的字符](https://leetcode-cn.com/problems/di-yi-ge-zhi-chu-xian-yi-ci-de-zi-fu-lcof/)

语言：java

思路：用LinkedHashMap来记录遍历过的字符，这里不能用单纯只HashMap->不保证插入顺序。

代码 (40ms，34.56%)：

```java
class Solution {
    public char firstUniqChar(String s) {
        char res = ' ';
        HashMap<Character,Integer> maps = new LinkedHashMap<>();
        for(char c:s.toCharArray()){
            Integer count = maps.get(c);
            if(count==null){
                maps.put(c, 1);
            }else{
                maps.put(c,c+1);
            }
        }
        for(Map.Entry<Character,Integer> entry:maps.entrySet()){
            if(entry.getValue()==1)
                return entry.getKey();
        }
        return res;
    }
}
```

参考代码1（2ms）：从前往后和从后往前，查找24个字母的下标，进行比较。

```java
class Solution {
    public char firstUniqChar(String s) {
        int r = Integer.MAX_VALUE;
        for (char c = 'a'; c <= 'z'; c++) {
            int index = s.indexOf(c);
            if (index != -1 && index == s.lastIndexOf(c)) {
                r = Math.min(r, index);
            }
        }
        return r == Integer.MAX_VALUE ? ' ' : s.charAt(r);
    }
}
```

参考代码2（4ms）：抽屉原理。

```java
class Solution {
    public char firstUniqChar(String s) {
        if (s == null || s.length() == 0) {
            return ' ';
        }
        int[] re = new int[26];
        char[] ch = s.toCharArray();
        for (int i=0; i<ch.length; i++) {
            re[ch[i]-'a']++;
        }
        for (int i=0; i<ch.length; i++) {
            if (re[ch[i]-'a'] == 1) {
                return ch[i];
            }
        }
        return ' ';
    }
}
```

参考代码3-1：LinkedHashMap的优化使用方法，毕竟只要知道是否重复，不需要知道具体数量

> [面试题50. 第一个只出现一次的字符（哈希表 / 有序哈希表，清晰图解）](https://leetcode-cn.com/problems/di-yi-ge-zhi-chu-xian-yi-ci-de-zi-fu-lcof/solution/mian-shi-ti-50-di-yi-ge-zhi-chu-xian-yi-ci-de-zi-3/)

```java
class Solution {
    public char firstUniqChar(String s) {
        Map<Character, Boolean> dic = new LinkedHashMap<>();
        char[] sc = s.toCharArray();
        for(char c : sc)
            dic.put(c, !dic.containsKey(c));
        for(Map.Entry<Character, Boolean> d : dic.entrySet()){
           if(d.getValue()) return d.getKey();
        }
        return ' ';
    }
}
```

参考代码3-2：使用HashMap存储，但是最后根据原字符串String遍历。最后遍历还是会遍历到重复的字母去判断

```java
class Solution {
    public char firstUniqChar(String s) {
        HashMap<Character, Boolean> dic = new HashMap<>();
        char[] sc = s.toCharArray();
        for(char c : sc)
            dic.put(c, !dic.containsKey(c));
        for(char c : sc)
            if(dic.get(c)) return c;
        return ' ';
    }
}
```

参考代码4：多一个数组，用于顺序判断下标对应的字符是否重复出现。其实就是把最后的map遍历改成了数组遍历。确实运行后会比LinkedHashMap、HashMap的接替方式快。算空间换时间。

```java
class Solution {
    public char firstUniqChar(String s) {
        int [] b = new int[s.length()];
        Arrays.fill(b,0);
        Map<Character,Integer> map = new HashMap<>();
        char [] chars = s.toCharArray();
        for(int i=0;i<chars.length;++i){
            Integer index = map.get(chars[i]);
            if (index != null){
                b[index] = 0;
                b[i] = 0;
            }else{
                map.put(chars[i],i);
                b[i] =1;
            }
        }
        //遍历获取第一个出现一次的字符
        for(int i = 0;i<chars.length;++i){
            if(b[i] == 1){
                return chars[i];
            }
        }
        return ' ';
    }
}
```

### 面试题42. 连续子数组的最大和

> [面试题42. 连续子数组的最大和](https://leetcode-cn.com/problems/lian-xu-zi-shu-zu-de-zui-da-he-lcof/)

语言：java

思路：因为要求连续，所以计算的和的时候，要么重新计算，要么累加。再者，取最大值。当下一个数小于0的时候，越加越小，那不如重新计算和；而下一个数大于0的时候，则累加。每次计算后，取最大值保存起来。

代码（1ms，98.78%）：

```java
class Solution {
    public int maxSubArray(int[] nums) {
        int max = nums[0];
        int sum =nums[0];
        for(int i = 1,len = nums.length;i<len;++i){
            if(sum<0){
                sum=nums[i];
            }
            else{
                sum+=nums[i];
            }
            max = Math.max(sum,max);
        }
        return max;
    }
}
```

参考代码：思路和我是一样的，这个用了更少的变量，不过需要修改原数组。这个看着更精简，意味着需要一定的理解能力。

> [面试题42. 连续子数组的最大和（动态规划，清晰图解）](https://leetcode-cn.com/problems/lian-xu-zi-shu-zu-de-zui-da-he-lcof/solution/mian-shi-ti-42-lian-xu-zi-shu-zu-de-zui-da-he-do-2/)

```java
class Solution {
    public int maxSubArray(int[] nums) {
        int res = nums[0];
        for(int i = 1; i < nums.length; i++) {
            nums[i] += Math.max(nums[i - 1], 0);
            res = Math.max(res, nums[i]);
        }
        return res;
    }
}
```

### 面试题40. 最小的k个数

> [面试题40. 最小的k个数](https://leetcode-cn.com/problems/zui-xiao-de-kge-shu-lcof/)

语言：java

思路：快排

代码1(8ms,64.36%)：

```java
class Solution {
    public int[] getLeastNumbers(int[] arr, int k) {
        Arrays.sort(arr);
        return Arrays.copyOf(arr,k);
    }
}
```

代码2：（快速选择，2 ms,99.49%）：

```java
class Solution {
    public int[] getLeastNumbers(int[] arr, int k) {
        if(k==0||arr.length==0)
            return new int[0];
        return quickTopN(arr,0,arr.length-1,k-1);
    }


    public int[] quickTopN(int[] arr,int start,int end,int k){
        int midIndex = quickSelect(arr,start,end);
        if(midIndex == k) return Arrays.copyOf(arr,k+1);
        return midIndex>k? quickTopN(arr,start,midIndex-1,k):quickTopN(arr,midIndex+1,end,k);
    }

    public int quickSelect(int[] arr,int start,int end){
        int base = arr[start];
        int l = start,r = end;
        while(l<r){
            while(l<r&&arr[r]>=base)
                --r;
            while(l<r&&arr[l]<=base)
                ++l;
            if(l<r){
                int tmp = arr[l];
                arr[l] = arr[r];
                arr[r] =tmp;
            }
        }
        arr[start] = arr[l];
        arr[l] = badse;
        return l;
    }
}
```

代码3（最大堆，14ms，40.7%,巨慢）：

```java
class Solution {
    public int[] getLeastNumbers(int[] arr, int k) {
        if(k==0||arr.length==0)
            return new int[0];
        PriorityQueue<Integer> maxStack = new PriorityQueue<>(k,(v1, v2) -> v2 - v1);
        for(int i:arr){
            if(maxStack.size()<k){
                maxStack.add(i);
            }else{
                if(i<maxStack.peek()){
                    maxStack.poll();
                    maxStack.add(i);
                }
            }
        }
        int[] res = new int[k];
        int j = 0;
        for(int i : maxStack){
            res[j++] = i;
        }
        return res;
    }
}
```

参考代码1（1ms）：毕竟只要最小的k个数字，所以没必要进行完整的快排。只要某次快排的基准点移动到所需的前k个的位置时，就可以直接停止快排了。

```java
class Solution {
    public int[] getLeastNumbers(int[] arr, int k) {
        if (k == 0 || arr.length == 0) {
            return new int[0];
        }
        // 最后一个参数表示我们要找的是下标为k-1的数
        return quickSearch(arr, 0, arr.length - 1, k - 1);
    }

    private int[] quickSearch(int[] nums, int l, int r, int k) {
        int ans=partition(nums,l,r);
        if(ans==k) return Arrays.copyOf(nums,k+1);
        else if(ans>k) return quickSearch(nums,l,ans-1,k);
        return quickSearch(nums,ans+1,r,k);
    }

    // 快排切分，返回下标j，使得比nums[j]小的数都在j的左边，比nums[j]大的数都在j的右边。
    private int partition(int[] nums, int l, int r) {
        int temp=nums[l];
        int i=l,j=r+1;
        while(true){
            while(++i<=r&&nums[i]<temp);
            while(--j>=l&&nums[j]>temp);
            if(i>=j) break;
            int x=nums[i];
            nums[i]=nums[j];
            nums[j]=x;
        }
        nums[l]=nums[j];
        nums[j]=temp;
        return j;
    }
}
```

参考代码2（2ms）：抽屉思想。用tong[]记录每个数字出现的次数；然后遍历tong，小的数字肯定在前面，会被最先装填到需要返回的数组中。

```java
class Solution {
    public int[] getLeastNumbers(int[] arr, int k) {
        if (k == 0 || arr.length == 0) {
            return new int[0];
        }
        if (k >= arr.length) return arr;
        int tong[] = new int[10000];
        for (int num : arr) {
            tong[num]++;
        }
        int[] res = new int[k];
        int i = 0;
        for (int j = 0; j < tong.length; j++) {
            while (tong[j]>0) {
                res[i++] = j;
                tong[j] --;
                if (i==k)break;
            }
            if (i==k)break;
        }
        return res;
    }
}
```

参考代码3（二叉搜索树，这里同时是红黑树，34ms,18.85%）：同样巨慢

> [4种解法秒杀TopK（快排/堆/二叉搜索树/计数排序）]([https://leetcode-cn.com/problems/zui-xiao-de-kge-shu-lcof/solution/3chong-jie-fa-miao-sha-topkkuai-pai-dui-er-cha-sou/#%E4%B8%89%E3%80%81%E4%BA%8C%E5%8F%89%E6%90%9C%E7%B4%A2%E6%A0%91%E4%B9%9F%E5%8F%AF%E4%BB%A5-%E8%A7%A3%E5%86%B3-topk-%E9%97%AE%E9%A2%98%E5%93%A6](https://leetcode-cn.com/problems/zui-xiao-de-kge-shu-lcof/solution/3chong-jie-fa-miao-sha-topkkuai-pai-dui-er-cha-sou/#三、二叉搜索树也可以-解决-topk-问题哦))

```java
class Solution {
    public int[] getLeastNumbers(int[] arr, int k) {
        if (k == 0 || arr.length == 0) {
            return new int[0];
        }
        // TreeMap的key是数字, value是该数字的个数。
        // cnt表示当前map总共存了多少个数字。
        TreeMap<Integer, Integer> map = new TreeMap<>();
        int cnt = 0;
        for (int num: arr) {
            // 1. 遍历数组，若当前map中的数字个数小于k，则map中当前数字对应个数+1
            if (cnt < k) {
                map.put(num, map.getOrDefault(num, 0) + 1);
                cnt++;
                continue;
            } 
            // 2. 否则，取出map中最大的Key（即最大的数字), 判断当前数字与map中最大数字的大小关系：
            //    若当前数字比map中最大的数字还大，就直接忽略；
            //    若当前数字比map中最大的数字小，则将当前数字加入map中，并将map中的最大数字的个数-1。
            Map.Entry<Integer, Integer> entry = map.lastEntry();
            if (entry.getKey() > num) {
                map.put(num, map.getOrDefault(num, 0) + 1);
                if (entry.getValue() == 1) {
                    map.pollLastEntry();
                } else {
                    map.put(entry.getKey(), entry.getValue() - 1);
                }
            }

        }

        // 最后返回map中的元素
        int[] res = new int[k];
        int idx = 0;
        for (Map.Entry<Integer, Integer> entry: map.entrySet()) {
            int freq = entry.getValue();
            while (freq-- > 0) {
                res[idx++] = entry.getKey();
            }
        }
        return res;
    }
}
```

### 面试题30. 包含min函数的栈

> [面试题30. 包含min函数的栈](https://leetcode-cn.com/problems/bao-han-minhan-shu-de-zhan-lcof/)

语言：java

思路：据说可以每个节点都存min来实现，就试着整了个

代码（19ms,90.51%）：

```java
class MinStack {

    class MyNode01{
        public int val;
        public int min;
        public MyNode01 next;

        public MyNode01(int val, int min,MyNode01 next) {
            this.val = val;
            this.min = min;
            this.next = next;
        }
    }

    MyNode01 head;

    /**
     * initialize your data structure here.
     */
    public MinStack() {
        head = null;
    }

    public void push(int x) {
        if(head==null){
            head = new MyNode01(x, x, null);
        }else{
            head = new MyNode01(x,Math.min(x, head.min),head);
        }
    }

    public void pop() {
        head = head.next;
    }

    public int top() {
        return head.val;
    }

    public int min() {
        return head.min;
    }
}

/**
 * Your MinStack object will be instantiated and called as such:
 * MinStack obj = new MinStack();
 * obj.push(x);
 * obj.pop();
 * int param_3 = obj.top();
 * int param_4 = obj.min();
 */
```

参考代码1（17ms）: 也是每次都放入当前的min。不过这里不需要另外实现节点，通过2次pop、push代替。

```java
class MinStack {

    private int min;
    private Stack<Integer> stack;

    /** initialize your data structure here. */
    public MinStack() {
        min = Integer.MAX_VALUE;
        stack = new Stack<>();
    }

    public void push(int x) {
        min = Math.min(min,x);
        stack.push(x);
        stack.push(min);
    }

    public void pop() {
        stack.pop();
        stack.pop();
        if(stack.empty())
            min = Integer.MAX_VALUE;
        else 
            min = stack.peek();
    }

    public int top() {
        stack.pop();
        int t = stack.peek();
        stack.push(min);
        return t;
    }

    public int min() {
        return min;
    }
}
```

### 面试题55 - II. 平衡二叉树

> [面试题55 - II. 平衡二叉树](https://leetcode-cn.com/problems/ping-heng-er-cha-shu-lcof/)

语言：java

思路：DFS，递归判断每个子树是否是平衡二叉树。先判断当前的节点，如果符合左右子树深度相差不大于1，就递归判断其左右子树。（会重复多次重复的递归过程）

代码（1ms，99.95%）：

```java
class Solution {
    public boolean isBalanced(TreeNode root) {
        if(root==null)
            return true;
        if(Math.abs(depth(root.left)-depth(root.right))<=1)
            return isBalanced(root.left)&&isBalanced(root.right);
        return false;
    }

    public int depth(TreeNode root){
        if(root==null)
            return 0;
        return Math.max(depth(root.left),depth(root.right))+1;
    }
}
```

参考代码1（0ms）：后序遍历+剪枝操作。如果不是平衡二叉树，就直接返回。

```java
class Solution {
    public boolean isBalanced(TreeNode root) {
        return depth(root) != -1;
    }

    private int depth(TreeNode root) {
        if (root == null) {
            return 0;
        }
        int left = depth(root.left);
        if (left == -1) {
            return -1;
        }
        int right = depth(root.right);
        if (right == -1) {
            return -1;
        }
        if (left == right + 1 || left == right) {
            return left + 1;
        } else if (right == left + 1) {
            return right + 1;
        } else {
            return -1;
        }
    }
}
```

> [面试题55 - II. 平衡二叉树（从底至顶、从顶至底，清晰图解）](https://leetcode-cn.com/problems/ping-heng-er-cha-shu-lcof/solution/mian-shi-ti-55-ii-ping-heng-er-cha-shu-cong-di-zhi/)

参考剪枝操作后，重写（1ms,99.95%,果然LeetCode的Java测试用例少，判断时间机制也比较迷，哈哈）：

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
    public boolean isBalanced(TreeNode root) {
        return recur(root)!=-1;
    }

    public int recur(TreeNode root){
        if(root==null)
            return 0;
        int left = recur(root.left);
        if(left == -1)
            return -1;
        int right = recur(root.right);
        if(right == -1)
            return -1;
        if(Math.abs(left-right)>1)
            return -1;
        else
            return Math.max(left,right)+1;
    }
}
```

### 面试题66. 构建乘积数组

> [面试题66. 构建乘积数组](https://leetcode-cn.com/problems/gou-jian-cheng-ji-shu-zu-lcof/)
>
> [面试题66. 构建乘积数组（表格分区，清晰图解）](https://leetcode-cn.com/problems/gou-jian-cheng-ji-shu-zu-lcof/solution/mian-shi-ti-66-gou-jian-cheng-ji-shu-zu-biao-ge-fe/)

语言：java

思路：原本直接暴力计算，双层for，果不其然，超时了。后面看了网上的解法，所谓对称遍历。

代码（2ms，78.87%）：

```java
class Solution {
    public int[] constructArr(int[] a) {
        if(a.length==0)
            return new int[0];
        int[] res = new int[a.length];
        int tmp = 1;
        res[0] = 1;
        for(int i = 1;i<a.length;++i){
            res[i] = res[i-1] * a[i-1];
        }
        for(int i = a.length-2; i>=0 ;--i){
            tmp *= a[i+1];
            res[i]*=tmp;
        }
        return res;
    }
}
```

参考代码（1ms）：同样对称数组，更加简洁-->双100%

```java
class Solution {
    public int[] constructArr(int[] a) {
        int n = a.length;
        int[] B = new int[n];
        for (int i = 0, product = 1; i < n; product *= a[i], i++)       /* 从左往右累乘 */
            B[i] = product;
        for (int i = n - 1, product = 1; i >= 0; product *= a[i], i--)  /* 从右往左累乘 */
            B[i] *= product;
        return B;
    }
}
```

###  面试题28. 对称的二叉树

> [面试题28. 对称的二叉树](https://leetcode-cn.com/problems/dui-cheng-de-er-cha-shu-lcof/)
>
> [面试题28. 对称的二叉树（递归，清晰图解）](https://leetcode-cn.com/problems/dui-cheng-de-er-cha-shu-lcof/solution/mian-shi-ti-28-dui-cheng-de-er-cha-shu-di-gui-qing/)

语言：java

思路：只想到了非递归的做法，根据题解，算是了解递归的思路了。这个题以前作过了，但是递归的操作还是记不清了，额。

代码（0ms）：判断根节点是否对称，即往下的子树是否对称。分为left和right两路，如果left和right按照对称的路径去行走时，要是同时为null，就是到头了，返回true；要是只有其中一个为null或者两者值不同，那么就是非对称，返回false；否则继续往下判断两路对称走向时，是否对称。

```java
class Solution {
    public boolean isSymmetric(TreeNode root) {
        return root==null||recur(root.left,root.right);
    }

    public boolean recur(TreeNode left,TreeNode right){
        if(left==null&&right==null)
            return true;
        if(left==null||right==null||left.val!=right.val)
            return false;
        return recur(left.left,right.right)&&recur(left.right,right.left);
    }
} 
```

这里直接复制网友的答题评论了，感觉说的很有道理

> [面试题28. 对称的二叉树--评论区--Tai_Park网友评论](https://leetcode-cn.com/problems/dui-cheng-de-er-cha-shu-lcof/comments/)

```none
做递归思考三步：

1. 递归的函数要干什么？
	函数的作用是判断传入的两个树是否镜像。
	输入：TreeNode left, TreeNode right
	输出：是：true，不是：false
2. 递归停止的条件是什么？
	左节点和右节点都为空 -> 倒底了都长得一样 ->true
	左节点为空的时候右节点不为空，或反之 -> 长得不一样-> false
	左右节点值不相等 -> 长得不一样 -> false
3. 从某层到下一层的关系是什么？
	要想两棵树镜像，那么一棵树左边的左边要和二棵树右边的右边镜像，一棵树左边的右边要和二棵树右边的左边镜像
	调用递归函数传入左左和右右
	调用递归函数传入左右和右左
	只有左左和右右镜像且左右和右左镜像的时候，我们才能说这两棵树是镜像的
	调用递归函数，我们想知道它的左右孩子是否镜像，传入的值是root的左孩子和右孩子。这之前记得判个root==null。
```

### 面试题65. 不用加减乘除做加法

> [面试题65. 不用加减乘除做加法](https://leetcode-cn.com/problems/bu-yong-jia-jian-cheng-chu-zuo-jia-fa-lcof/)

语言：java

思路：不能用加减乘除，那就只能想到用位运算了。

1. 不需要进位的部分之和为 a^b

2. 进位的部分：(a&b) << 1

3. 不需要进位的部分和进位的部分再次求和：(a^b) ^ [(a&b)<<1]

4. ....

   把数字都视为32位二进制数字时，那么相加就只有01 10 00 11 四种情况，前三种不需要进位，a^b的结果都是0；最后11这种，需要进位变成100，即(a&b)<<1。重复操作，将未进位部分和进位部分相加，直到没有进位后，就结束加法运算了。这个算操作系统原理学过的知识吧。

代码（0ms）：

```java
class Solution {
    public int add(int a, int b) {
        int noCarry = a;
        int carry = b;
        while(carry!=0){
            a = noCarry;
            b = carry;
            noCarry = a^b;
            carry = (a&b)<<1;
        }
        return noCarry;
    }
}
```

参考代码：更加精简的版本

> [面试题65. 不用加减乘除做加法（位运算，清晰图解）](https://leetcode-cn.com/problems/bu-yong-jia-jian-cheng-chu-zuo-jia-fa-lcof/solution/mian-shi-ti-65-bu-yong-jia-jian-cheng-chu-zuo-ji-7/)

```java
class Solution {
    public int add(int a, int b) {
        while(b != 0) { // 当进位为 0 时跳出
            int c = (a & b) << 1;  // c = 进位
            a ^= b; // a = 非进位和
            b = c; // b = 进位
        }
        return a;
    }
}
```

### 面试题60. n个骰子的点数

> [面试题60. n个骰子的点数](https://leetcode-cn.com/problems/nge-tou-zi-de-dian-shu-lcof/)
>
> [【n个骰子的点数】：详解动态规划及其优化方式](https://leetcode-cn.com/problems/nge-tou-zi-de-dian-shu-lcof/solution/nge-tou-zi-de-dian-shu-dong-tai-gui-hua-ji-qi-yo-3/)

语言：java

思路：没想到比较好的方法，看了下网上的动态规划讲解。还是需要多培养一些动态规划的思想，哈哈。

代码(0ms)：

```java
class Solution {
    public double[] twoSum(int n) {
        int[] points = new int[n*6+1];
        for(int i = 1;i<=6;++i){
            points[i] = 1;
        }
        for(int i = 2;i<=n;++i){
            for(int j = 6*i;j>=i;--j){
                points[j] = 0;
                for(int cur = 1;cur<=6;++cur){
                    if(j-cur<i-1){
                        break;
                    }
                    points[j] += points[j-cur]; 
                }
            }
        }
        double total = Math.pow(6,n);
        double[] res = new double[n*6-n+1];
        for(int i = n; i <= 6*n; ++i){
            res[i-n] = points[i] / total;
        }
        return res;
    }
}
```

参考代码(0ms)：

```java
class Solution {
    public double[] twoSum(int n) {
       int [][]dp = new int[n+1][6*n+1];
        //边界条件
        for(int s=1;s<=6;s++)dp[1][s]=1;
        for(int i=2;i<=n;i++){
            for(int s=i;s<=6*i;s++){
                //求dp[i][s]
                for(int d=1;d<=6;d++){
                    if(s-d<i-1)break;//为0了
                    dp[i][s]+=dp[i-1][s-d];
                }
            }
        }
        double total = Math.pow((double)6,(double)n);
        double[] ans = new double[5*n+1];
        for(int i=n;i<=6*n;i++){
            ans[i-n]=((double)dp[n][i])/total;
        }
        return ans; 
    }
}
```

### 面试题53 - I. 在排序数组中查找数字 I

> [面试题53 - I. 在排序数组中查找数字 I](https://leetcode-cn.com/problems/zai-pai-xu-shu-zu-zhong-cha-zhao-shu-zi-lcof/)
>
> [面试题53 - I. 在排序数组中查找数字 I（二分法，清晰图解）](https://leetcode-cn.com/problems/zai-pai-xu-shu-zu-zhong-cha-zhao-shu-zi-lcof/solution/mian-shi-ti-53-i-zai-pai-xu-shu-zu-zhong-cha-zha-5/)

语言：java

思路：明显要二分查找。就是中间的<=、>=、<、>的选择，我没能把握好，可以说边界条件没把握好吧，多提交了几次，比较蛋疼。先一个循环找出目标数字群的右边第一个位置；然后第二个循环找出目标数字群的左边第一个位置。

代码（0ms）：

```java
class Solution {
    public int search(int[] nums, int target) {
        int left = 0,right = nums.length-1,mid;
        while(left<=right){
            mid = (left+right) / 2;
            if(nums[mid]<=target)
                left = mid+1;
            else
                right = mid-1;
        } 
        int end = left;

        left = 0;
        right = nums.length-1;
        while(left<=right){
            mid = (left+right) / 2;
            if(nums[mid]>=target)
                right = mid-1;
            else
                left = mid+1;
        }
        int start = right;
        return end-start-1;
    }
}
```

参考代码（0ms）：同样是二分查找，感受到了边界条件选取的重要性。和"查找"相关的算法，主要都是边界条件的选取可能疏忽大意了。

```java
class Solution {
    public int search(int[] nums, int target) {
        int l = 0;
        int r = nums.length;
        int start = leftBorder(l,r,nums,target);
        int end = rightBorder(l,r,nums,target);
        if(start == -1 && end == -1){
            return 0;
        }else {
            return end - start +1 ;
        }
    }

    private int rightBorder(int l,int r,int[] nums,int target) {
        while (l < r){
            int mid = l + (r-l)/2;
            if(nums[mid] == target){
                l = mid+1;
            }else if(nums[mid] > target){
                r = mid;
            }else if(nums[mid] <target){
                l = mid+1;
            }
        }
        if(l == 0){
            return -1;
        }
        return nums[l-1] == target?l-1:-1;
    }

    private int leftBorder(int l,int r,int[] nums,int target ){
        while (l < r){
            int mid = l + (r-l)/2;
            if(nums[mid] == target){
                r = mid;
            }else if(nums[mid] > target){
                r = mid ;
            }else if(nums[mid] < target){
                l = mid + 1;
            }
        }
        if(l == nums.length){
            return -1;
        }
        return nums[l] == target? l : -1;
    }
}
```

### 面试题11. 旋转数组的最小数字

> [面试题11. 旋转数组的最小数字](https://leetcode-cn.com/problems/xuan-zhuan-shu-zu-de-zui-xiao-shu-zi-lcof/)

语言：java

思路：看着就是需要二分查找的样子。

+ 右边界数值 > 中间；有边界修改成mid
+ 右边界数值 < 中间；左边界++(这个其实应该改成左边界 = mid + 1，减少计算)
+ 右边界 = 中间； 右边界--

代码（1ms,50.41%）：

```java
class Solution {
    public int minArray(int[] numbers) {
        int i = 0,j = numbers.length-1,mid; // 左 >= 右，旋转后，找出最靠近左半部的那个边界。
        while(i<j){
            mid = (i+j)/2;
            if(numbers[mid]<numbers[j]) // 1 1 1 2 3 4 0 0 0 1 1 1 1 1 1 1
                // 0 0 0 0 1 1 1 2 2 2 3 3 3 3 3 3
                // 1 1 1 1 1 1 1 1 1 1 1 1 1 1 0 1
                j = mid;
            else if(numbers[mid]>numbers[j])
                ++i;
            else
                --j;
        }
        return numbers[i];
    }
}
```

参考代码1（0ms）：同样是二分查找，但是`numbers[m] > numbers[j]`的处理更好

> [面试题11. 旋转数组的最小数字（二分法，清晰图解）](https://leetcode-cn.com/problems/xuan-zhuan-shu-zu-de-zui-xiao-shu-zi-lcof/solution/mian-shi-ti-11-xuan-zhuan-shu-zu-de-zui-xiao-shu-3/)

```java
class Solution {
    public int minArray(int[] numbers) {
        int i = 0, j = numbers.length - 1;
        while (i < j) {
            int m = (i + j) / 2;
            if (numbers[m] > numbers[j]) i = m + 1;
            else if (numbers[m] < numbers[j]) j = m;
            else j--;
        }
        return numbers[i];
    }
}
```

### 面试题53 - II. 0～n-1中缺失的数字

> [面试题53 - II. 0～n-1中缺失的数字](https://leetcode-cn.com/problems/que-shi-de-shu-zi-lcof/)

语言：java

思路：同样还是二分查找，就是要对边界条件判断清楚。这里有个情况是没踩过坑的可能不一定直到的，就是[0]要输出1，而[1]要输出0；

因为题目数字都是从0开始的，而且明确只会少1个数字，那么假如什么都没有少，那么应该正好数字对应其在数组中的下标；如果中间少了一个，那么前面的还是数字对应下标，后面的则是数字大于下标。

+ 中间mid的数组数字大于下标，说明mid处于缺失数字的右边区域，可以缩小右边界
+ 中间mid的数组数字等于下标，说明mid处于缺失数字的左边区域，可以缩小左边界

这里while要让左边界可以和右边界相等，比如[0]需要返回1，表示缺失数字1，这时候需要执行一次i = mid+1，才能得到正确答案。

最后返回的是数字i，也就是第一个下标和数字不相符合的数字，正好是缺失数字。

代码（0ms）：

```java
class Solution {
    public int missingNumber(int[] nums) {
        int i = 0,j = nums.length-1,mid;
        while(i<=j){
            mid = (i+j) / 2;
            if(nums[mid]>mid){
                j = mid-1;
            }else if(nums[mid]==mid){
                i = mid+1;
            }// 0 1 2 3 4 (5) 6 7 8 9 10 11 12
        }
        return i;
    }
}
```

参考代码：更加简洁了。同样也是二分法。

> [面试题53 - II. 0～n-1 中缺失的数字（二分法，清晰图解）](https://leetcode-cn.com/problems/que-shi-de-shu-zi-lcof/solution/mian-shi-ti-53-ii-0n-1zhong-que-shi-de-shu-zi-er-f/)

```java
class Solution {
    public int missingNumber(int[] nums) {
        int i = 0, j = nums.length - 1;
        while(i <= j) {
            int m = (i + j) / 2;
            if(nums[m] == m) i = m + 1;
            else j = m - 1;
        }
        return i;
    }
}
```

### 面试题59 - I. 滑动窗口的最大值

> [面试题59 - I. 滑动窗口的最大值](https://leetcode-cn.com/problems/hua-dong-chuang-kou-de-zui-da-zhi-lcof/)
>
> [面试题59 - I. 滑动窗口的最大值（单调栈，清晰图解）](https://leetcode-cn.com/problems/hua-dong-chuang-kou-de-zui-da-zhi-lcof/solution/mian-shi-ti-59-i-hua-dong-chuang-kou-de-zui-da-1-6/)
>
> [Java-单调双向队列-画图详解](https://leetcode-cn.com/problems/hua-dong-chuang-kou-de-zui-da-zhi-lcof/solution/java-dan-diao-shuang-xiang-lian-biao-hua-tu-xiang-/)

语言：java

思路：没想到比较好的方法，学习了下所谓的单调栈方法解题。

代码（17ms，48.97%）：

```java
class Solution {
    public int[] maxSlidingWindow(int[] nums, int k) {
        if(nums.length == 0 || k == 0)
            return new int[0];
        Deque<Integer> deque = new LinkedList<>();
        int index = 0,tmp;
        int[] res = new int[nums.length-k+1];
        for(int i = 0;i< k;++i){
            tmp = nums[i];
            while(!deque.isEmpty()&&deque.peekLast()<tmp){
                deque.pollLast();
            }
            deque.addLast(tmp);
        }
        if(!deque.isEmpty())
            res[index++] = deque.peek();
        for(int i = k;i<nums.length;++i){
            tmp = nums[i];
            if(!deque.isEmpty()&&nums[i-k]==deque.peek())
                deque.poll();
            while(!deque.isEmpty()&&deque.peekLast()<tmp){
                deque.pollLast();
            }
            deque.addLast(tmp);
            res[index++] = deque.peek();
        }
        return res;
    }
}
```

参考代码1（19ms，44.51%）：

> [面试题59 - I. 滑动窗口的最大值（单调栈，清晰图解）](https://leetcode-cn.com/problems/hua-dong-chuang-kou-de-zui-da-zhi-lcof/solution/mian-shi-ti-59-i-hua-dong-chuang-kou-de-zui-da-1-6/)

```java
class Solution {
    public int[] maxSlidingWindow(int[] nums, int k) {
        if(nums.length == 0 || k == 0) return new int[0];
        Deque<Integer> deque = new LinkedList<>();
        int[] res = new int[nums.length - k + 1];
        for(int i = 0; i < k; i++) { // 未形成窗口
            while(!deque.isEmpty() && deque.peekLast() < nums[i]) deque.removeLast();
            deque.addLast(nums[i]);
        }
        res[0] = deque.peekFirst();
        for(int i = k; i < nums.length; i++) { // 形成窗口后
            if(deque.peekFirst() == nums[i - k]) deque.removeFirst();
            while(!deque.isEmpty() && deque.peekLast() < nums[i]) deque.removeLast();
            deque.addLast(nums[i]);
            res[i - k + 1] = deque.peekFirst();
        }
        return res;
    }
}
```

参考代码2（1ms）：

这个看样子是把情况分成了3种：

1. 数组为空或者不存在滑动窗口，直接返回空数组
2. 滑动窗口能包括整个数组，那就直接找最大值，返回就好了
3. 滑动窗口是数组的一小部分
   + 在数组完全容纳整个滑动窗口之前，先找出最大值和其下标，然后记录
   + 开始移动滑动窗口，如果正好移动时最大值的所属下标被移除了，那就重新计算最大值。没想到这个还挺快的，估计是因为用deque频繁插入、删除反而耗时了。

```java
class Solution {
    public int[] maxSlidingWindow(int[] nums, int k) {
        if (nums == null || nums.length <= 0) return new int[] {};
        if (k >= nums.length) {
            int max = nums[0];
            for (int i = 0; i < nums.length; i++) {
                max = max > nums[i] ? max : nums[i];
            }
            return new int[]{max};
        }

        int max = nums[0], max_tag = 0;
        int[] ans = new int[nums.length - k + 1];
        for (int i = 0; i < k; i++) {
            if (nums[i] >= max) {max = nums[i]; max_tag = i;}
        }
        ans[0] = max;
        for (int i = k; i < nums.length; i++) {
            if (nums[i] >= max) {max = nums[i]; max_tag = i; }
            if (max_tag == i - k) {
                max = nums[i - k + 1]; max_tag = i - k + 1;
                for (int j = max_tag; j <= i; j++) {
                    if (nums[j] >= max) {max = nums[j]; max_tag = j;}
                }

            }
            ans[i - k + 1] = max;
        }   
        return ans;
    }
}
```

参考代码3（12ms）：

deque内存下标而不是数值，更方便判断（不需要用2个for循环）

```java
class Solution {
  public int[] maxSlidingWindow(int[] nums, int k) {
    if(nums == null || nums.length < 2) {
      return nums;
    }
    LinkedList<Integer> queue = new LinkedList<>();
    int [] result = new int[nums.length - k + 1];
    for (int i = 0; i < nums.length ; i++ ){
      while (!queue.isEmpty() && nums[queue.peekLast()] <= nums[i]){
        queue.pollLast();
      }
      queue.addLast(i);
      if(queue.peek() <= i-k){
        //看一下队首还在不在滑动窗口里面，不在就扔了
        queue.poll();
      }
      if (i+1 >= k){
        result[i+1-k] = nums[queue.peek()];
      }

    }

    return result;
  }
}
```

### 面试题61. 扑克牌中的顺子

> [面试题61. 扑克牌中的顺子](https://leetcode-cn.com/problems/bu-ke-pai-zhong-de-shun-zi-lcof/)

语言：java

思路：因为鬼牌是特殊情况，所以需要先统计。

0. 排序

1. 统计鬼牌数量
2. 判断是否顺子
   + 除去鬼牌后，剩余的牌，两辆之间相差的数值超过1的部分需要鬼牌去弥补。
   + 如果鬼牌数量拿去弥补后，不够用了（<0）或者出现前后两张牌相同，则不是顺子

代码（1ms，84.82%）：

```java
class Solution {
    public boolean isStraight(int[] nums) {
        Arrays.sort(nums);
        int ghost = 0;
        int i = 0;
        int tmp;
        while(nums[i]==0){
            ++i;
            ++ghost;
        }
        if(ghost>0||i==0)
            ++i;
        for(;i<5;++i){
            tmp = nums[i]-nums[i-1];
            if(tmp>1)
                ghost-=(tmp-1);
            if(ghost<0||0==tmp)//0 0 1 4 5 
                return false;
        }
        return true;
    }
}
```

参考代码1（1ms，84.82%）:把问题转换成5张牌的最大值和最小值相差不能超过4且不能出现重复。而是否重复可以通过set来判断；最大最小值则是手动统计

> [面试题61. 扑克牌中的顺子（集合 Set / 排序，清晰图解）](https://leetcode-cn.com/problems/bu-ke-pai-zhong-de-shun-zi-lcof/solution/mian-shi-ti-61-bu-ke-pai-zhong-de-shun-zi-ji-he-se/)

```java
class Solution {
    public boolean isStraight(int[] nums) {
        Set<Integer> repeat = new HashSet<>();
        int max = 0, min = 14;
        for(int num : nums) {
            if(num == 0) continue; // 跳过大小王
            max = Math.max(max, num); // 最大牌
            min = Math.min(min, num); // 最小牌
            if(repeat.contains(num)) return false; // 若有重复，提前返回 false
            repeat.add(num); // 添加此牌至 Set
        }
        return max - min < 5; // 最大牌 - 最小牌 < 5 则可构成顺子
    }
}
```

参考代码2（1ms,84.82%）：先排序再判断除了鬼牌外是否出现重复，是否最大最小值之差小于5

> [面试题61. 扑克牌中的顺子（集合 Set / 排序，清晰图解）](https://leetcode-cn.com/problems/bu-ke-pai-zhong-de-shun-zi-lcof/solution/mian-shi-ti-61-bu-ke-pai-zhong-de-shun-zi-ji-he-se/)

```java
class Solution {
    public boolean isStraight(int[] nums) {
        int joker = 0;
        Arrays.sort(nums); // 数组排序
        for(int i = 0; i < 4; i++) {
            if(nums[i] == 0) joker++; // 统计大小王数量
            else if(nums[i] == nums[i + 1]) return false; // 若有重复，提前返回 false
        }
        return nums[4] - nums[joker] < 5; // 最大牌 - 最小牌 < 5 则可构成顺子
    }
}
```

### 面试题29. 顺时针打印矩阵

> [面试题29. 顺时针打印矩阵](https://leetcode-cn.com/problems/shun-shi-zhen-da-yin-ju-zhen-lcof/)

语言：java

思路：设置上下左右4个边界，每个走向用一个for循环解决。每次走到尽头，改变一下对应的边界值。这个想起来大一好像还是学校的作业题。

代码（1ms，97.41%）：

```java
class Solution {
    public int[] spiralOrder(int[][] matrix) {
        if(matrix.length==0)
            return new int[]{};
        int x_max = matrix.length-1, x_min = 0;
        int y_max = matrix[0].length-1, y_min = 0;
        int[] res = new int[(x_max+1)*(y_max+1)];
        int index=0;
        while(true){
            for(int j = y_min;j<=y_max;++index,++j) res[index] = matrix[x_min][j]; // 右
            if(++x_min>x_max) break;
            for(int i = x_min;i<=x_max;++index,++i) res[index] = matrix[i][y_max]; // 下
            if(--y_max<y_min) break;
            for(int j = y_max;j>=y_min;++index,--j) res[index] = matrix[x_max][j]; // 左
            if(--x_max<x_min) break;
            for(int i = x_max;i>=x_min;++index,--i) res[index] = matrix[i][y_min]; // 上
            if(++y_min>y_max) break;
        }
        return res;
    }
}
```

参考代码（0ms）：思路上没有什么本质的区别。看着还更复杂了，额。

```java
class Solution {

    int[] res;
    int x=0;
    public int[] spiralOrder(int[][] matrix) {
        if(matrix.length<=0 || matrix[0].length<=0)
            return new int[0];
        res = new int[matrix.length*matrix[0].length];
        int start = 0;
        int rows = matrix.length;
        int cols = matrix[0].length;
        while( matrix[0].length>2*start &&  matrix.length>2*start){
            print(matrix,rows,cols,start);
            start++;
        }
        return res;
    }

    public void print(int[][] matrix, int rows, int cols, int start) {

        int endX = cols-start-1;
        int endY = rows-start-1;

        for (int i=start;i<=endX;i++){
            res[x++]=(matrix[start][i]);
        }
        if (start<endY){
            for (int i=start+1;i<=endY;i++){
                res[x++]=matrix[i][endX];
            }
        }
        if (start<endX && start<endY){
            for (int i=endX-1;i>=start;i--){
                res[x++]=(matrix[endY][i]);
            }
        }
        if(start<endY-1 && start<endX){
            for (int i=endY-1;i>start;i--){
                res[x++]=(matrix[i][start]);
            }
        }

    }
}
```

### 面试题10- II. 青蛙跳台阶问题

> [面试题10- II. 青蛙跳台阶问题](https://leetcode-cn.com/problems/qing-wa-tiao-tai-jie-wen-ti-lcof/)

语言：java

思路：经典的动态规划-斐波那契（相信这个题目大家都做烂了）。从结果往回推敲。最后一次，只可能是跳1步or跳两步，所以f(n)= f(n-1)+f(n-2)；然后换成代码，可以把f(n)用数组来表示。一开始跳1是只有一种，跳2则两种。

代码(0ms，100%)：

```java
class Solution {
    public int numWays(int n) {
        if(n<=1)
            return 1;
        if(n==2)
            return 2;
        int[] f = new int[n+1];
        f[1] = 1;
        f[2] = 2;
        for(int i=3;i<=n;++i){
            f[i] = (f[i-1]+f[i-2])%1000000007;
        }
        return f[n];
    }
}
```

参加代码：直接用3个值来表示f(n)、f(n-1)、f(n-2)。常量级的空间。

> [面试题10- II. 青蛙跳台阶问题（动态规划，清晰图解）](https://leetcode-cn.com/problems/qing-wa-tiao-tai-jie-wen-ti-lcof/solution/mian-shi-ti-10-ii-qing-wa-tiao-tai-jie-wen-ti-dong/)

```java
class Solution {
    public int numWays(int n) {
        int a = 1, b = 1, sum;
        for(int i = 0; i < n; i++){
            sum = (a + b) % 1000000007;
            a = b;
            b = sum;
        }
        return a;
    }
}
```

### 面试题58 - I. 翻转单词顺序

> [面试题58 - I. 翻转单词顺序](https://leetcode-cn.com/problems/fan-zhuan-dan-ci-shun-xu-lcof/)

语言：java

思路：就从后面往前遍历单词，没啥好说的。

代码（3ms，63.75%）：

```java
class Solution {
    public String reverseWords(String s) {
        s = s.trim();
        StringBuilder sb = new StringBuilder();
        for(int i = s.length()-1,j=i;i>=0;){
            while(i>=0&&s.charAt(i)!=' ') --i;
            sb.append(s.substring(i+1, j+1)).append(' ');
            while(i>=0&&s.charAt(i)==' ') --i;
            j = i;
        }
        return sb.toString().trim();
    }
}
```

参考代码（1ms）：没有什么本质的区别，不同划分单词直接用了java的split方法。

```java
public class Solution {
    public String reverseWords(String s) {
        String[] str = s.split(" ");
        StringBuilder stringBuilder = new StringBuilder();
        for (int i = str.length - 1; i >= 0; i--) {
            stringBuilder.append(str[i]);
            if (i != 0 && !"".equals(str[i - 1])) {
                stringBuilder.append(" ");
            }
        }

        return stringBuilder.toString();
    }
}
```

### 面试题04. 二维数组中的查找

> [面试题04. 二维数组中的查找](https://leetcode-cn.com/problems/er-wei-shu-zu-zhong-de-cha-zhao-lcof/)

语言：java

思路：暴力查找不可取。本来想从左上到右下那条线作为判断标准，后面发现好像不大行。后面思考题目，上到下，左到右递增。那么其实左下角和右上角也可以当作特殊点去考虑。

这里取左下角作为起点查找目标值：

+ 当前值大于目标值，那么坐标往上移动；因为坐标是当前列最大的了，其上面才可能存在target
+ 当前值小于目标值，那么坐标往右移动；因为坐标是当前行最小的了，其右边才可能存在target

重复上面操作，直到找到target或者走到边界，返回false 

代码（0ms）：

```java
class Solution {
    public boolean findNumberIn2DArray(int[][] matrix, int target) {
        for(int i = matrix.length-1,k = 0;i>=0;--i){
            for(int j = k;j<=matrix[0].length-1;++j){
                if (matrix[i][j]==target)
                    return true;
                else if(matrix[i][j]>target){
                    break;
                }else{
                    ++k;
                }
            }
        }
        return false;
    }
}
```

参考代码：这个也是取左下角作为起点，然后查找target。同样是改变边界去查找

>[面试题04. 二维数组中的查找（左下角标志数法，清晰图解）](https://leetcode-cn.com/problems/er-wei-shu-zu-zhong-de-cha-zhao-lcof/solution/mian-shi-ti-04-er-wei-shu-zu-zhong-de-cha-zhao-zuo/)

```java
class Solution {
    public boolean findNumberIn2DArray(int[][] matrix, int target) {
        int i = matrix.length - 1, j = 0;
        while(i >= 0 && j < matrix[0].length)
        {
            if(matrix[i][j] > target) i--;
            else if(matrix[i][j] < target) j++;
            else return true;
        }
        return false;
    }
}
```

### 面试题10- I. 斐波那契数列

> [面试题10- I. 斐波那契数列](https://leetcode-cn.com/problems/fei-bo-na-qi-shu-lie-lcof/)

语言：java

思路：没啥好说的。因为这题就完全是斐波那契，没有什么变换啥的。如果不懂，就需要去百度下斐波那契，了解一下概念什么的。

代码（0ms）：

```java
class Solution {
    public int fib(int n) {
        if(n<=1)
            return n;
        int res = 0,a=0,b=1;
        for(int i = 2;i<=n;++i){
            res = (a+b)%1000000007;
            a = b;
            b = res;
        }
        return res;
    }
}
```

参考代码：用的retun a，更加精简了。

> [面试题10- I. 斐波那契数列（动态规划，清晰图解）](https://leetcode-cn.com/problems/fei-bo-na-qi-shu-lie-lcof/solution/mian-shi-ti-10-i-fei-bo-na-qi-shu-lie-dong-tai-gui/)

```java
class Solution {
    public int fib(int n) {
        int a = 0, b = 1, sum;
        for(int i = 0; i < n; i++){
            sum = (a + b) % 1000000007;
            a = b;
            b = sum;
        }
        return a;
    }
}
```

### 面试题64. 求1+2+…+n

> [面试题64. 求1+2+…+n](https://leetcode-cn.com/problems/qiu-12n-lcof/)

语言：java

思路：看到题目的要求，很严苛。感觉主要是要锻炼查看复杂代码的思维？不然想不到平时谁会有这种奇怪的需求。一开始没什么想法，看到评论区提示可以用&&，||短路+递归。于是试一试。

代码（1ms，61.83%）：

```java
class Solution {
    public int sumNums(int n) {
        int res = 0;
        boolean tmp = n > 1 && (res = sumNums(n-1))>0;
        return n+res;
    }
}
```

参考代码1（0ms）：总感觉这个有点投机取巧了吧... 等差数列 [n * (n+1)] / 2 

```java
class Solution {
    public int sumNums(int n) {
        return  ((int)Math.pow(n, 2) + n) >> 1;
    }
}
```

参考代码2：(1ms，61.83%)

> [面试题64. 求 1 + 2 + … + n（逻辑符短路，清晰图解）](https://leetcode-cn.com/problems/qiu-12n-lcof/solution/mian-shi-ti-64-qiu-1-2-nluo-ji-fu-duan-lu-qing-xi-/)

```java
class Solution {
    public int sumNums(int n) {
        boolean x = n > 1 && (n += sumNums(n - 1)) > 0;
        return n;
    }
}
```

参考代码3：(1ms，61.83%)

> [面试题64. 求 1 + 2 + … + n（逻辑符短路，清晰图解）](https://leetcode-cn.com/problems/qiu-12n-lcof/solution/mian-shi-ti-64-qiu-1-2-nluo-ji-fu-duan-lu-qing-xi-/)

```java
class Solution {
    int res = 0;
    public int sumNums(int n) {
        boolean x = n > 1 && sumNums(n - 1) > 0;
        res += n;
        return res;
    }
}
```

### 面试题56 - II. 数组中数字出现的次数 II

> [面试题56 - II. 数组中数字出现的次数 II](https://leetcode-cn.com/problems/shu-zu-zhong-shu-zi-chu-xian-de-ci-shu-ii-lcof/)
>
> [面试题56 - II. 数组中数字出现的次数 II（位运算 + 有限状态自动机，清晰图解）](https://leetcode-cn.com/problems/shu-zu-zhong-shu-zi-chu-xian-de-ci-shu-ii-lcof/solution/mian-shi-ti-56-ii-shu-zu-zhong-shu-zi-chu-xian-d-4/)
>
> [状态机解决此问题详解 （数字电路）](https://leetcode-cn.com/problems/shu-zu-zhong-shu-zi-chu-xian-de-ci-shu-ii-lcof/solution/zhuang-tai-ji-jie-jue-ci-wen-ti-xiang-jie-shu-zi-d/)

语言：java

思路：除了map以外，没想到比较好的思路。后面查看评论区，了解到可以用状态机和位运算组合解题。记录int32位数字每一二进制位出现的次数，最后每个二进制位%3，组装到int结果值中。

代码(5ms，87.22%)：

```java
class Solution {
   public int singleNumber(int[] nums) {
        int[] binary32 = new int[32];
        for(int i :nums){
            for(int j = 0;j<32;++j){
                binary32[j] += i&1;
                i>>=1;
            }
        }
        int res = 0,m = 3;
        for(int i = 0;i<32;++i){
            res |= binary32[i] % m << i;
        }
        return res;
    }
}
```

参考代码1（1ms）：有限状态机和位运算的超结合。这个比较烧脑，适合经常和数学打交道的高手。

```java
class Solution {
    public int singleNumber(int[] nums) {
        int low = 0, high = 0;
        for (int num : nums) {
            low = low ^ num & ~high;
            high = high ^ num & ~low;
        }

        return low;
    }
}
```

### 面试题56 - I. 数组中数字出现的次数

> [面试题56 - I. 数组中数字出现的次数](https://leetcode-cn.com/problems/shu-zu-zhong-shu-zi-chu-xian-de-ci-shu-lcof/)
>
> [数组中数字出现的次数-官方解题](https://leetcode-cn.com/problems/shu-zu-zhong-shu-zi-chu-xian-de-ci-shu-lcof/solution/shu-zu-zhong-shu-zi-chu-xian-de-ci-shu-by-leetcode/)

语言：java

思路：自愧不如，位运算的题目还是做得不够多。看讨论区说可以用异或+分组实现。

代码（2ms，95.49%）：

```java
class Solution {
    public int[] singleNumbers(int[] nums) {
        int a=0,b=0,res=0;
        for(int i : nums){
            res ^= i;
        }
        int tmp = 1;
        while(0 == (res & 1)){
            res >>>=1;
            tmp <<= 1;
        }
        for(int i : nums){
            if(0==(i&tmp))
                a^=i;
            else
                b^=i;
        }
        return new int[]{a,b};
    }
}
```

参考代码1（1ms）：看了老久，算是大致弄懂里面的位运算了，把注解加上去了

```java
class Solution {
    public int[] singleNumbers(int[] nums) {
        int eor = 0;
        for(int i:nums) eor ^= i;
        int rightOne = eor&(~eor+1);// 只保留32位中第一个1，其他位全0
        int eors = 0;
        for(int i:nums){
            if((i&rightOne) != 0) eors ^= i;
        }
        return new int[]{eors,eor^eors};//任何一个数a异或两次b得到a
    }
}
```

### 138. 复制带随机指针的链表

> [138. 复制带随机指针的链表](https://leetcode-cn.com/problems/copy-list-with-random-pointer/)
>
> [复制带随机指针的链表--官网解答](https://leetcode-cn.com/problems/copy-list-with-random-pointer/solution/fu-zhi-dai-sui-ji-zhi-zhen-de-lian-biao-by-leetcod/)

语言：java

思路：官方解答有3种。很详细。这里挑战一下第三种

代码：（0ms，100%；39.5MB，50%）

```java
class Solution {
    public Node copyRandomList(Node head) {
            
            if(head==null)
                return null;
            
            Node newHead = head;
            
            while(newHead!=null){

                Node newNode = new Node(newHead.val);

                newNode.next = newHead.next;
                newHead.next = newNode;

                newHead = newNode.next;
            }
            
            newHead = head;
            
            while(newHead!=null){
                if(newHead.random!=null)
                    newHead.next.random = newHead.random.next;
                newHead = newHead.next.next;
            }
            
            newHead = head;
            
            Node res = newHead.next;
            Node tmpNext;
            while(newHead!=null){
                tmpNext = newHead.next;
                newHead.next = tmpNext.next;
                tmpNext.next = tmpNext.next==null?null:tmpNext.next.next;
                newHead = newHead.next;
            }
            
            return res;
        }
}
```

### 面试题07. 重建二叉树

> [面试题07. 重建二叉树](https://leetcode-cn.com/problems/zhong-jian-er-cha-shu-lcof/)
>
> [面试题07. 重建二叉树--官方讲解](https://leetcode-cn.com/problems/zhong-jian-er-cha-shu-lcof/solution/mian-shi-ti-07-zhong-jian-er-cha-shu-by-leetcode-s/)

语言：java

思路：官方有提供递归和迭代两种版本的讲解。个人感觉迭代的那个确实不是很好理解，就先挑战了一下迭代版本。

代码（迭代，2ms，99.35%） :

```java
public TreeNode buildTree(int[] preorder, int[] inorder) {
  if(( preorder.length & inorderÅ..length )== 0)
    return null;
  TreeNode root = new TreeNode(preorder[0]);
  int preLen = preorder.length;
  LinkedList<TreeNode> stack = new LinkedList<>();
  stack.addFirst(root);
  for(int preIndex = 1,inIndex = 0;preIndex<preLen;++preIndex){
    int curVal = preorder[preIndex];
    TreeNode preNode = stack.peekFirst();
    if(preNode.val!=inorder[inIndex]){
      preNode.left = new TreeNode(curVal);
      stack.addFirst(preNode.left);
    }else{
      while(!stack.isEmpty() && stack.peekFirst().val == inorder[inIndex]){
        preNode = stack.pop();
        ++inIndex;
      }
      preNode.right = new TreeNode(curVal);
      stack.push(preNode.right);
    }
  }
  return root;
}
```

代码（递归，3ms，81.20%）：

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {

  public TreeNode buildTree(int[] preorder, int[] inorder) {
    if(( preorder.length & inorder.length )== 0)
      return null;
    HashMap<Integer,Integer> map = new HashMap<>();
    int len = inorder.length;
    for(int i = 0;i<len;++i){
      map.put(inorder[i],i);
    }
    TreeNode root = buildTree(preorder, inorder, 0, len-1, 0, len-1, map);
    return root;
  }

  public TreeNode buildTree(int[] preorder, int[] inorder,int preL,int preR,int inL,int inR,HashMap<Integer,Integer> map) {
    if(preL > preR)
      return null;
    int rootVal = preorder[preL];
    TreeNode root = new TreeNode(rootVal);

    if(preL != preR){
      int inRootIndex = map.get(rootVal);
      root.left = buildTree(preorder,inorder,preL+1, preL+inRootIndex-inL, inL,inRootIndex-1 , map);
      root.right = buildTree(preorder,inorder,preR-inR+inRootIndex+1, preR, inRootIndex+1,inR , map);
    }

    return root;
  }
}
```

### 面试题47. 礼物的最大价值

> [面试题47. 礼物的最大价值](https://leetcode-cn.com/problems/li-wu-de-zui-da-jie-zhi-lcof/)

语言：java

思路：一开始用的递归，然后超时了。就改用动态规划了。要获取到终点的最大值，就是慢慢往回退直到起点。那么写起来就是从起点到终点，每次走过的地方，都直接记录成当前最大值就好了。

+ 先把最上面一行和最左边一列计算了。因为这两条路，都是当前值 加 上一步的值，比较简单
+ 从（1，1）坐标开始，每一次计算当前值 + Math.max(向左or向上)，这样每次走到哪，就知道当前位置的最大值了。

代码（2ms，98.41%）：

```java
class Solution {
   public int maxValue(int[][] grid) {
        int lenX = grid.length;
        if(lenX!=0) {
            int lenY = grid[0].length;
            if (lenY != 0) {
                for(int i = 1;i<lenX;++i){
                    grid[i][0] += grid[i-1][0];
                }
                for(int j = 1;j<lenY;++j){
                    grid[0][j] += grid[0][j-1];
                }
                for (int i = 1;i<lenX;++i){
                    for(int j = 1;j<lenY;++j){
                        grid[i][j] += Math.max(grid[i-1][j],grid[i][j-1]);
                    }
                }
                return grid[lenX-1][lenY-1];
            }
        }
        return 0;
    }
}
```

参考代码（1ms）：其实和迭代写法差不多，这里写成了递归的形式。

```java
class Solution {
    public int maxValue(int[][] grid) {
        if (grid == null || grid.length == 0 || grid[0].length == 0) {
            return 0;
        }

        int x = grid.length - 1;
        int y = grid[0].length - 1;

        return maxValueHelper(grid, x, y,new int[grid.length][grid[0].length]);
    }

    public int maxValueHelper(int[][] grid, int x, int y,int[][] cache) {

        if (cache[x][y]!=0){
            return cache[x][y];
        }

        //x , y 只有两种移动方向，要不x-1,要不 y-1, 每一个点的maxValue要不是 x-1的值加上这个值就是 y-1的值加上这个值
        if (x == 0 && y == 0) {
            cache[x][y]=grid[0][0];
            return grid[0][0];
        }
        if (x == 0) {
            int leftSum= 0;
            for (int i = 0; i <= y; i++) {
                leftSum +=grid[0][i];
                cache[0][i]=leftSum;
            }

            return leftSum;
        }
        if (y == 0) {
            int leftSum= 0;
            for (int i = 0; i <= x; i++) {
                leftSum +=grid[i][0];
                cache[i][0]=leftSum;

            }
            return leftSum;
        }
        cache[x][y]=grid[x][y] + Math.max(maxValueHelper(grid, x - 1, y,cache), maxValueHelper(grid, x, y - 1,cache));

        return cache[x][y];
    }
}
```

### 面试题32 - I. 从上到下打印二叉树

> [面试题32 - I. 从上到下打印二叉树](https://leetcode-cn.com/problems/cong-shang-dao-xia-da-yin-er-cha-shu-lcof/)

语言：java

思路：就BFS。每一层遍历就好了。

代码（4ms，15.11%）：不知道为啥会那么慢，额。

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
class Solution {
        public int[] levelOrder(TreeNode root) {
            List<Integer> list = new LinkedList<>();

            if (root != null) {
                Queue<TreeNode> queue = new LinkedList<>();
                queue.add(root);
                TreeNode cur;
                while (!queue.isEmpty()) {
                    cur = queue.poll();
                    list.add(cur.val);
                    if (cur.left != null)
                        queue.add(cur.left);
                    if (cur.right != null)
                        queue.add(cur.right);
                }
            }
            return list.stream().mapToInt(Integer::intValue).toArray();
        }
    }
```

参考代码(1ms)：思路上没什么区别，就是最后int【】转化不一样了

```java
class Solution {
//用队列实现
    public int[] levelOrder(TreeNode root) {
        if (root==null)
            return new int[0];
        LinkedList<TreeNode> queue = new LinkedList<>();
        ArrayList<Integer> ans = new ArrayList<>();
        queue.push(root);
        while (!queue.isEmpty()){
            TreeNode node=queue.peek();
            if (node.left!=null)
                queue.add(node.left);
            if (node.right!=null)
                queue.add(node.right);
            ans.add(queue.poll().val);
        }
        int[] data=new int[ans.size()];
        for (int i = 0; i < ans.size(); i++) {
            data[i]=ans.get(i);
        }

        return data;
    }
}
```

参考代码2：

> [面试题32 - I. 从上到下打印二叉树（层序遍历 BFS ，清晰图解）](https://leetcode-cn.com/problems/cong-shang-dao-xia-da-yin-er-cha-shu-lcof/solution/mian-shi-ti-32-i-cong-shang-dao-xia-da-yin-er-ch-4/)

```java
class Solution {
    public int[] levelOrder(TreeNode root) {
        if(root == null) return new int[0];
        Queue<TreeNode> queue = new LinkedList<>(){{ add(root); }};
        ArrayList<Integer> ans = new ArrayList<>();
        while(!queue.isEmpty()) {
            TreeNode node = queue.poll();
            ans.add(node.val);
            if(node.left != null) queue.add(node.left);
            if(node.right != null) queue.add(node.right);
        }
        int[] res = new int[ans.size()];
        for(int i = 0; i < ans.size(); i++)
            res[i] = ans.get(i);
        return res;
    }
}
```

### 面试题63. 股票的最大利润

> [面试题63. 股票的最大利润](https://leetcode-cn.com/problems/gu-piao-de-zui-da-li-run-lcof/)

语言：java

思路：直接每次计算当前值与上一个的差值，然后累加到一个暂存值，要是 < 0就是亏了，重新用0计数，不然就累加。然后每次都计算当前累加值和最后返回的结果值，哪个大就用哪个

代码(2ms,63.77%)：

```java
class Solution {
    
    public int maxProfit(int[] prices) {
        int res = 0;
        for(int i = 1,len = prices.length,tmp=0;i<len;++i){
            tmp = tmp + prices[i] - prices[i-1];
            tmp = Math.max(tmp, 0);
            res = Math.max(tmp,res);
        }
        return res;
    }
}
```

参考代码1（2ms，63.77%）：思路差不多。不过这里用的统计最大值和最小值。

> [面试题63. 股票的最大利润（动态规划，清晰图解）](https://leetcode-cn.com/problems/gu-piao-de-zui-da-li-run-lcof/solution/mian-shi-ti-63-gu-piao-de-zui-da-li-run-dong-tai-2/)

```java
class Solution {
    public int maxProfit(int[] prices) {
        int cost = Integer.MAX_VALUE, profit = 0;
        for(int price : prices) {
            cost = Math.min(cost, price);
            profit = Math.max(profit, price - cost);
        }
        return profit;
    }
}
```

参考代码2（0ms）：

```java
class Solution {
    public int maxProfit(int[] prices) {
        int minPrice=Integer.MAX_VALUE;
        int maxProfit=0;
        for(int i=0;i<prices.length;i++){
            minPrice=Math.min(minPrice,prices[i]);
            maxProfit=Math.max(maxProfit,prices[i]-minPrice);
        }
        return maxProfit;
    }
}
```

### 面试题49. 丑数

> [面试题49. 丑数](https://leetcode-cn.com/problems/chou-shu-lcof/)

语言：java

思路：2,3,5三个数字都要计算的话，那就分别用3个值来表示。但是这样每次计算完都需要比较一下有没有重复计算的值被记录了。

首先，三个值肯定是要分别计算的。为了避免每次计算后都看暂存的结果集有没有重复的元素，就需要方法确保下次用来计算的值不管怎么算都不会重复。

这里用动态规划，反正确定了要第几个，可以直接new一个对应大小的数组来存储结果。

然后每次计算3个值，只有其中最小的一个才能更新基数值（数组对应的下标值），3个数3个基数值；

因为只有计算出来的3个数中最小的一个才能更新下标，等于每轮计算，都是用最新、最小的数字作为基数来计算的，保证不会漏数。

代码（2ms，98.95%）：

```java
class Solution {
   public int nthUglyNumber(int n) {
        int[] tmp = new int[n];
        tmp[0] = 1;
        for(int i = 1,a=0,b=0,c=0,valA,valB,valC;i<n;++i){
            valA = tmp[a] * 2;
            valB = tmp[b] * 3;
            valC = tmp[c] * 5;
            tmp[i] = Math.min(valA,Math.min(valB,valC));
            if(tmp[i]==valA) ++a;
            if(tmp[i]==valB) ++b;
            if(tmp[i]==valC) ++c;
        }
        return tmp[n-1];
    }
}
```

参考代码（1ms）：

```java
class Solution {
    public static Ugly u = new Ugly();
    public int nthUglyNumber(int n) {
        return u.nums[n - 1];
    }
}

class Ugly {
    public int[] nums = new int[1690];
    Ugly() {
        nums[0] = 1;
        int min, i2 = 0, i3 = 0, i5 = 0;
        int n2, n3, n5;
        for(int i = 1; i < 1690; ++i) {
            n2 = nums[i2] * 2;
            n3 = nums[i3] * 3;
            n5 = nums[i5] * 5;
            min = Math.min(Math.min(n2, n3), n5);
            nums[i] = min;
            if (min == n2) ++i2;
            if (min == n3) ++i3;
            if (min == n5) ++i5;
        }
    }
}
```

### 面试题36. 二叉搜索树与双向链表

> [面试题36. 二叉搜索树与双向链表](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-yu-shuang-xiang-lian-biao-lcof/)

语言：java

思路：中序遍历，可以非递归也可以递归。需要注意的是记得用一个变量存储上一个节点。

代码1（1ms，25.56%）：非递归

```java
class Solution {
    public Node treeToDoublyList(Node root) {
        if (root == null)
            return null;
        Deque<Node> deque = new LinkedList<>();
        // resHead就是答案的头节点;resTail就是答案的尾节点;
        // cur是当前遍历的节点;pre是上一个cur节点
        Node resHead = null, resTail = null, cur = root, pre = null;
        while (!deque.isEmpty() || cur != null) {
            // (1) 往左下角走,直到左下角的尽头
            while (cur != null) {
                deque.addFirst(cur);
                cur = cur.left;
            }
            // (2) 由于上次cur = null
            //     => cur到了某节点左下角 or cur到了某节点右子树，但是右子树是null
            //     => 从 记录栈 里面取之前遍历过但没用到的节点
            cur = deque.pop();
            resTail = cur; // 每个节点都可能作为最后的答案尾节点
            if (pre == null) { // 说明第一次到达左下角，那肯定是root根节点的左下角，作为答案头节点
                resHead = cur;
            } else { // pre上一个节点(左中右的 左or中节点)
                pre.right = cur;
                cur.left = pre;
            }
            pre = cur;
            cur = cur.right; // 中序遍历 => 左中右(前面while往左，而cur可以视为就是中,cur.right就是往右)
        }
        resTail.right = resHead;
        resHead.left = resTail;
        return resHead;
    }
}
```

代码2（0ms，100%）：递归

```java
class Solution {
    Node pre,head;

    public Node treeToDoublyList(Node root) {
        if(root==null)
            return null;
        recur(root);
        head.left = pre;
        pre.right = head;
        return head;
    }

    public void recur(Node root){
        if(root==null)
            return ;
        recur(root.left);
        if(pre!=null)
            pre.right = root;
        else
            head = root;
        root.left = pre;
        pre = root;
        recur(root.right);
    }
}
```

参考代码：递归的方法。

> [面试题36. 二叉搜索树与双向链表（中序遍历，清晰图解）](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-yu-shuang-xiang-lian-biao-lcof/solution/mian-shi-ti-36-er-cha-sou-suo-shu-yu-shuang-xian-5/)

```java
class Solution {
    Node pre, head;
    public Node treeToDoublyList(Node root) {
        if(root == null) return null;
        dfs(root);
        head.left = pre;
        pre.right = head;
        return head;
    }
    void dfs(Node cur) {
        if(cur == null) return;
        dfs(cur.left);
        if(pre != null) pre.right = cur;
        else head = cur;
        cur.left = pre;
        pre = cur;
        dfs(cur.right);
    }
}
```

### 面试题32 - III. 从上到下打印二叉树 III

> [面试题32 - III. 从上到下打印二叉树 III](https://leetcode-cn.com/problems/cong-shang-dao-xia-da-yin-er-cha-shu-iii-lcof/)

语言：java

思路：每层遍历的顺便要求变化，奇数回->从左到右，偶数回->从右到左；如果和以往层次遍历那样只用一个队列queue不太行。如果用一个dequeue也是可以的。

1. dequeue（奇数回的"放"推理出偶数回的"取"；偶数回的"放"推理出奇数回的"取"）

   ​	这里假设从1开始算回合数，根节点就是第1回合(奇数回)，根节点的左右节点就是第二回合(偶数回)。

   ​	下面的“先后”，谁先就谁在前，谁后就谁在后，比如1-5都是先小后大，那么顺序就是 1 2 3 4 5 ，从后往前取就是5 4 3 2 1。

   + 奇数回，放节点到dequeue->先left后right；取节点出dequeue->从前往后;
   + 偶数回，取节点出dequeue->从后往前；放节点到dequeue->先right后left;

2. 双栈法，思路和上面单个dequeue其实没区别，就是改用两个Stack或者两个Dequeue。

下面用的双栈法。

代码（1ms，99.86%）：

```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> res = new LinkedList<>();
        if(root!=null){
            Deque<TreeNode> stackLtoR = new LinkedList<>(); // 从左到右入栈;   取出来遍历便是 右到左
            Deque<TreeNode> stackRtoL = new LinkedList<>(); // 从右到左入栈;   取出来遍历便是 左到右
            stackRtoL.addFirst(root);
            boolean LtoRisEmpty = true,RtoLisEmpty = false;
            while(true){
                LtoRisEmpty = stackLtoR.isEmpty();
                RtoLisEmpty = stackRtoL.isEmpty();

                if(LtoRisEmpty&&RtoLisEmpty)
                    break;

                if (!LtoRisEmpty){
                    int size = stackLtoR.size();
                    List<Integer> list = new LinkedList<>();
                    TreeNode tmp;
                    for(int i = 0;i<size;++i){
                        tmp = stackLtoR.pollFirst();
                        list.add(tmp.val);
                        if(tmp.right!=null)
                            stackRtoL.addFirst(tmp.right);
                        if(tmp.left!=null)
                            stackRtoL.addFirst(tmp.left);
                    }
                    res.add(list);
                }
                if (!RtoLisEmpty){
                    int size = stackRtoL.size();
                    List<Integer> list = new LinkedList<>();
                    TreeNode tmp;
                    for(int i = 0;i<size;++i){
                        tmp = stackRtoL.pollFirst();
                        list.add(tmp.val);
                        if(tmp.left!=null)
                            stackLtoR.addFirst(tmp.left);
                        if(tmp.right!=null)
                            stackLtoR.addFirst(tmp.right);
                    }
                    res.add(list);
                }
            }
        }
        return res;
    }
}
```

参考代码1（0ms）：其实和Deque的一个思路，就是没有借助Dequeue，而是直接在结果集的List上操作而已。

试着在这个原参考代码上加了一些注释，好理解一点

```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> ans = new ArrayList<>();
        helper(root, ans, 1);
        return ans;
    }

    public void helper(TreeNode root, List<List<Integer>> ans, int level){
        if(root == null) return;
        LinkedList<Integer> levelList;
        if(level > ans.size()) { // 如果第X层还没有对应的List用来存储节点，就新建一个
            levelList = new LinkedList<>();
            ans.add(levelList);
        } else {
            levelList = (LinkedList<Integer>)ans.get(level - 1);
        }
        if((level & 1) == 1) { // 如果是奇数回，按正常顺序尾插法(队列)，之后list就是正序了
            levelList.addLast(root.val);
        } else {// 偶数回，正常顺序头插法(栈)，之后list就是倒序了
            levelList.addFirst(root.val);
        }
        helper(root.left, ans, level + 1); //先左后右，那么只需改变每次list放入节点值的方式
        helper(root.right, ans, level + 1); 

    }
}
```

### 面试题31. 栈的压入、弹出序列

> [面试题31. 栈的压入、弹出序列](https://leetcode-cn.com/problems/zhan-de-ya-ru-dan-chu-xu-lie-lcof/)

语言：java

思路：每次都入栈当前数字，如果和出栈队列的当前数字相同，就while循环出栈直到为空。如果不相同就不弹出。那么最后要是没有空，那么肯定就是出栈的顺序有问题了。

代码（2ms，96.06%）：

```java
class Solution {
   public boolean validateStackSequences(int[] pushed, int[] popped) {
        Deque<Integer> stack = new LinkedList<>();
        int i = 0, j = 0, len = popped.length;
        for (; i < len; ++i) {
            stack.addFirst(pushed[i]);
            while(!stack.isEmpty()&&stack.peekFirst()==popped[j]){
                stack.pollFirst();
                ++j;
            }
        }
        return stack.isEmpty();
    }
}
```

参考代码1（1ms）：没什么本质的区别，就是改用数组模拟stack

```java
class Solution {
    public boolean validateStackSequences(int[] pushed, int[] popped) {
        if(pushed.length == 0 || popped.length == 0){
            return true;
        }
        int[] stack = new int[pushed.length];
        int size = 0;
        int poppedPoint = 0;
        for(int i = 0; i < pushed.length; i++){
            stack[size++] = pushed[i];
            while(size > 0 && popped[poppedPoint] == stack[size-1]){
                poppedPoint++;
                size--;
            }
        }
        if(size == 0){
            return true;
        } else {
            return false;
        }
    }
}
```

参考代码2（0ms）：也是类似的操作，不过更绝，直接用指针在原数组上模拟栈操作

```java
class Solution {
    public boolean validateStackSequences(int[] pushed, int[] popped) {
        if (pushed.length != popped.length) return false;

        int i = 0, j = 0, s = -1;
        while (i < pushed.length) {
            if (s > -1 && pushed[s] == popped[j]) {
                s -= 1;
                j += 1;
            } else {
                s += 1;
                pushed[s] = pushed[i];
                i += 1;
            }
        }

        while (s > -1 && pushed[s] == popped[j]) {
            s -= 1;
            j += 1;
        }

        return s == -1;
    }
}
```

参考代码3（3ms，84.17%）：

> [面试题31. 栈的压入、弹出序列（模拟，清晰图解）](https://leetcode-cn.com/problems/zhan-de-ya-ru-dan-chu-xu-lie-lcof/solution/mian-shi-ti-31-zhan-de-ya-ru-dan-chu-xu-lie-mo-n-2/)

```java
class Solution {
    public boolean validateStackSequences(int[] pushed, int[] popped) {
        Stack<Integer> stack = new Stack<>();
        int i = 0;
        for(int num : pushed) {
            stack.push(num); // num 入栈
            while(!stack.isEmpty() && stack.peek() == popped[i]) { // 循环判断与出栈
                stack.pop();
                i++;
            }
        }
        return stack.isEmpty();
    }
}
```

### 面试题45. 把数组排成最小的数

> [面试题45. 把数组排成最小的数](https://leetcode-cn.com/problems/ba-shu-zu-pai-cheng-zui-xiao-de-shu-lcof/)
>
> [面试题45. 把数组排成最小的数（自定义排序，清晰图解）](https://leetcode-cn.com/problems/ba-shu-zu-pai-cheng-zui-xiao-de-shu-lcof/solution/mian-shi-ti-45-ba-shu-zu-pai-cheng-zui-xiao-de-s-4/)

语言：java

思路：这个比较巧妙，建议直接看上面的解析。

代码1（10ms，42.53%）：使用java的快排

```java
class Solution {
    public String minNumber(int[] nums) {
        String[] strs = new String[nums.length];
        for(int i = 0,len = strs.length;i<len;++i)
            strs[i] = String.valueOf(nums[i]);
        Arrays.sort(strs,(x,y)->(x+y).compareTo(y+x));
        StringBuilder sb = new StringBuilder();
        for(String str : strs)
            sb.append(str);
        return sb.toString();
    }
}
```

代码2（5ms，98.83%）：手写快排

```java
class Solution {
    public String minNumber(int[] nums) {
            String[] strs = new String[nums.length];
            for (int i = 0, len = strs.length; i < len; ++i)
                strs[i] = String.valueOf(nums[i]);
            KuaiPai(strs, 0, strs.length - 1);
            StringBuilder sb = new StringBuilder();
            for (String str : strs)
                sb.append(str);
            return sb.toString();
        }

        public void KuaiPai(String[] strs, int start, int end) {
            if(start>=end)
                return;
            String tmp = strs[start];
            int left = start, right = end;
            while(left<right){
                while(right>left&&(strs[right]+strs[start]).compareTo(strs[start]+strs[right])>=0)
                    -- right;
                while(left<right&&(strs[left]+strs[start]).compareTo(strs[start]+strs[left])<=0)
                    ++ left;
                tmp = strs[left];
                strs[left] = strs[right];
                strs[right] = tmp;
            }
            strs[left] = strs[start];
            strs[start] = tmp;
            KuaiPai(strs,start, left-1);
            KuaiPai(strs,left+1, end);
        }
}
```

参考代码1（2ms）：思想没区别，就是快排等写法各种优化了。

```java
class Solution {
    public String minNumber(int[] nums) {
        quicksort(nums);

        StringBuilder stringBuilder = new StringBuilder();
        for (int n : nums) {
            stringBuilder.append(n);
        }
        return stringBuilder.toString();
    }

    public void quicksort(int[] nums) {
        int p = 0, q = nums.length - 1;
        quicksort(nums, p, q);
    }

    public void quicksort(int[] nums, int p, int q) {
        if (p < q) {
            int r = partition(nums, p, q);
            quicksort(nums, p, r - 1);
            quicksort(nums, r + 1, q);
        }
    }

    public int partition(int[] nums, int p, int q) {
        int pivot = nums[p];
        int i = p;
        for (int j = i + 1; j <= q; j++) {
            // if (nums[j] < pivot) {
            if (compare(nums[j], pivot)) {
                i++;
                swap(nums, i, j);
            }
        }
        swap(nums, i, p);
        return i;
    }

    private void swap(int[] nums, int i, int j) {
        int temp = nums[i];
        nums[i] = nums[j];
        nums[j] = temp;
    }

    private boolean compare(int x, int y) {
        if (((long) x * (int) Math.pow(10, numscount(y)) + y)
            < ((long) y * (int) Math.pow(10, numscount(x)) + x)) {
            return true;
        } else {
            return false;
        }
    }

    private int numscount(int n) {
        if (n == 0) {
            return 1;
        }
        int x = 0;
        while (n > 0) {
            x++;
            n = n / 10;
        }
        return x;
    }
}
```

### 面试题34. 二叉树中和为某一值的路径

语言：[面试题34. 二叉树中和为某一值的路径](https://leetcode-cn.com/problems/er-cha-shu-zhong-he-wei-mou-yi-zhi-de-lu-jing-lcof/)

思路：DFS，用list暂存路径，要是满足sum，就遍历list暂存的节点，new一个存Integer的List，存到结果集中

代码（4ms，9%，丢人的时间）：

```java
class Solution {
    List<TreeNode> list;
    List<List<Integer>> res = new LinkedList<>();
    Integer sum;

    public List<List<Integer>> pathSum(TreeNode root, int sum) {
        if(root!=null){
            list = new LinkedList<>();
            this.sum = sum;
            dfs(root,0,list);
        }
        return res;
    }
    public void dfs(TreeNode root, int cur, List<TreeNode> list) {
        if (root != null) {
            cur += root.val;
            list.add(root);
            if(root.left==null&&root.right==null&&cur==sum){
                List<Integer> tmp = new LinkedList<>();
                for (TreeNode node : list) {
                    tmp.add(node.val);
                }
                res.add(tmp);
            }else{
                if(root.left!=null)
                    dfs(root.left, cur, list);
                if(root.right!=null)
                    dfs(root.right, cur, list);
            }
            list.remove(root);
        }
    }
}
```

参考代码1（1ms，100%）：

用减法判断为零而不是累加，省一个变量。

再者，不需要先存节点再转换成数字。（原本我是以为必须移除当前节点，看来只要移除最后一个节点就好了），毕竟移除最后一个，要是左子树行不通，还会走右子树，又是都不行，也会移除当前节点。（我之前的思路有点问题）

> [面试题34. 二叉树中和为某一值的路径（回溯法，清晰图解）](https://leetcode-cn.com/problems/er-cha-shu-zhong-he-wei-mou-yi-zhi-de-lu-jing-lcof/solution/mian-shi-ti-34-er-cha-shu-zhong-he-wei-mou-yi-zh-5/)

```java
class Solution {
    LinkedList<List<Integer>> res = new LinkedList<>();
    LinkedList<Integer> path = new LinkedList<>(); 
    public List<List<Integer>> pathSum(TreeNode root, int sum) {
        recur(root, sum);
        return res;
    }
    void recur(TreeNode root, int tar) {
        if(root == null) return;
        path.add(root.val);
        tar -= root.val;
        if(tar == 0 && root.left == null && root.right == null)
            res.add(new LinkedList(path));
        recur(root.left, tar);
        recur(root.right, tar);
        path.removeLast();
    }
}
```

参考后重写（1ms）：

```java
class Solution{
    LinkedList<List<Integer>> res= new LinkedList<>();
    LinkedList<Integer> path = new LinkedList<>();
    public List<List<Integer>> pathSum(TreeNode root, int sum) {
        dfs(sum,root);
        return res;
    }

    public void dfs(int sum,TreeNode root){
        if(root==null)
            return ;
        path.add(root.val);
        sum-=root.val;
        if(sum==0&&root.left==null&&root.right==null)
            res.add(new LinkedList<>(path));
        dfs(sum,root.left);
        dfs(sum,root.right);
        path.removeLast();
    }
}
```

### 面试题14- I. 剪绳子

> [面试题14- I. 剪绳子](https://leetcode-cn.com/problems/jian-sheng-zi-lcof/)
>
> [面试题14- I. 剪绳子（数学推导 / 贪心思想，清晰图解）](https://leetcode-cn.com/problems/jian-sheng-zi-lcof/solution/mian-shi-ti-14-i-jian-sheng-zi-tan-xin-si-xiang-by/)
>
> [【剪绳子】动态规划](https://leetcode-cn.com/problems/jian-sheng-zi-lcof/solution/jian-sheng-zi-dong-tai-gui-hua-by-97wgl/)
>
> [暴力搜索->记忆化搜索->动态规划](https://leetcode-cn.com/problems/integer-break/solution/bao-li-sou-suo-ji-yi-hua-sou-suo-dong-tai-gui-hua-/)

语言：java

思路：建议直接看上面解析得了。

代码（1ms,44.76%）：

```java
class Solution {
    public int cuttingRope(int n) {
        int[] res = new int[n + 1];
        res[2] = 1;
        for (int i = 3; i <= n; ++i) {
            for (int j = 1; j <= i - 1; ++j) {
                res[i] = Math.max(res[i], Math.max(j * res[i - j], j * (i - j)));
            }
        }
        return res[n];
    }
}
```

参考代码1（0ms）：数学方法

```java
class Solution {
    public int cuttingRope(int n) {
        if(n<=3){
            return n-1;
        }
        int a = n/3 , b = n%3;
        if(b==0){
            return (int)Math.pow(3,a);
        }
        if(b==1){
            return (int)Math.pow(3,a-1) * 4;
        }
        return (int)Math.pow(3,a) * 2;

    }
}
```

### 面试题38. 字符串的排列

> [面试题38. 字符串的排列](https://leetcode-cn.com/problems/zi-fu-chuan-de-pai-lie-lcof/)

语言：java

思路：递归DFS，每次往下，都固定前面一部分的排列，往后的部分则用for循环每次替换一个位置。每次替换之前，需要判断当前字符是否出现过，如果出现过（说明替换过了），就没必要替换了。调换后，往下DFS递归，回溯回来后需要记得在调换回来（恢复原样），这样才不会漏排列组合。

代码（14ms）：

```java
class Solution {
    List<String> res = new LinkedList<>();
    char[] chars;

    public String[] permutation(String s) {
        chars = s.toCharArray();
        dfs(0);
        return res.toArray(new String[res.size()]);
    }

    public void dfs(int x) {
        if (x == chars.length - 1) { // 遍历到最后一个字符了，那么返回结果串
            res.add(String.valueOf(chars));
        }
        HashSet<Character> charSet = new HashSet<>();
        for (int i = x; i < chars.length; ++i) {
            if (!charSet.contains(chars[i])) { // 如果当前位置的字符X已经出现过，该位置已经使用过X字符，可以直接跳过
                charSet.add(chars[i]);
                swap(i,x);
                dfs(x+1);// 固定下标<=x的所有元素，下一递归从x+1开始往后替换
                swap(i,x); // 还原现场，继续下一次替换。
            }
        }
    }

    public void swap(int a,int b){
        char tmp = chars[a];
        chars[a] = chars[b];
        chars[b] = tmp;
    }

}
```

参考代码1（10ms，93.15%）：

> [面试题38. 字符串的排列（回溯法，清晰图解）](https://leetcode-cn.com/problems/zi-fu-chuan-de-pai-lie-lcof/solution/mian-shi-ti-38-zi-fu-chuan-de-pai-lie-hui-su-fa-by/)

```java
class Solution {
    List<String> res = new LinkedList<>();
    char[] c;
    public String[] permutation(String s) {
        c = s.toCharArray();
        dfs(0);
        return res.toArray(new String[res.size()]);
    }
    void dfs(int x) {
        if(x == c.length - 1) {
            res.add(String.valueOf(c)); // 添加排列方案
            return;
        }
        HashSet<Character> set = new HashSet<>();
        for(int i = x; i < c.length; i++) {
            if(set.contains(c[i])) continue; // 重复，因此剪枝
            set.add(c[i]);
            swap(i, x); // 交换，将 c[i] 固定在第 x 位 
            dfs(x + 1); // 开启固定第 x + 1 位字符
            swap(i, x); // 恢复交换
        }
    }
    void swap(int a, int b) {
        char tmp = c[a];
        c[a] = c[b];
        c[b] = tmp;
    }
}
```

参考代码2（4ms）：相同思路。不过这里剪枝操作，直接对char[]数组进行判断，无需再new一个Set来暂存。从当前要替换的位置开始往后查找（包括当前位置）是否有和当前要替换的位置相同的字符，有的话就不需要再替换元素了（因为肯定重复）。

比如ababab，当前perm中，curPos = 0，那么for循环j = 2时，就没必要替换curPost和j的元素了，因为curPos这个位置已经有用过'a'和'b'两个字符了

```java
class Solution {
    private List<String> list = new ArrayList<>();

    public String[] permutation(String s) {
        perm(s.toCharArray(), 0, s.length() - 1);
        return list.toArray(new String[0]);
    }

    public void perm(char[] seq, int curPos, int n) {
        if(curPos == n) {
            list.add(new String(seq));            
        } else {
            for(int i = curPos; i <= n; i++) {
                if(!findSame(seq, curPos, i)) {
                    swap(seq, curPos, i);
                    perm(seq, curPos + 1, n);
                    swap(seq, curPos, i);
                }
            }
        }
    }

    private boolean findSame(char[] seq, int from, int candidate) {
        for(int j = from; j < candidate; j++) {
            if(seq[j] == seq[candidate]) {
                return true;
            }
        }
        return false;
    }

    private void swap(char[] chars, int i, int j) {
        char tmp = chars[i];
        chars[i] = chars[j];
        chars[j] = tmp;
    }
}
```

参考后重写（6ms，98.82%）：

```java
class Solution {

    List<String> res = new LinkedList<>();

    public String[] permutation(String s) {
        dfs(s.toCharArray(), 0, s.length() - 1);
        return res.toArray(new String[0]);
    }

    public void dfs(char[] chars, int start, int end) {
        if (start == end) {
            res.add(new String(chars));
        }
        for (int i = start; i <= end; ++i) {
            if (!exist(chars, start, i)) {
                swap(chars, start, i);
                dfs(chars, start + 1, end);
                swap(chars, start, i);
            }
        }
    }

    public boolean exist(char[] arrs, int start, int end) {
        for (int i = start; i < end; ++i) {
            if (arrs[i] == arrs[end]) {// 如果当前位置之前出现过相同的字符
                return true;
            }
        }
        return false;
    }

    public void swap(char[] chars, int a, int b) {
        char tmp = chars[a];
        chars[a] = chars[b];
        chars[b] = tmp;
    }
}
```

### 面试题46. 把数字翻译成字符串

> [面试题46. 把数字翻译成字符串](https://leetcode-cn.com/problems/ba-shu-zi-fan-yi-cheng-zi-fu-chuan-lcof/)
>
> [面试题46. 把数字翻译成字符串（动态规划，清晰图解）](https://leetcode-cn.com/problems/ba-shu-zi-fan-yi-cheng-zi-fu-chuan-lcof/solution/mian-shi-ti-46-ba-shu-zi-fan-yi-cheng-zi-fu-chua-6/)
>
> [递归求解，双百](https://leetcode-cn.com/problems/ba-shu-zi-fan-yi-cheng-zi-fu-chuan-lcof/solution/di-gui-qiu-jie-shuang-bai-by-xiang-shang-de-gua-ni/)

语言：java

思路：建议直接看上述思路

代码1（0ms）：

```java
class Solution {
    public int translateNum(int num) {
        if(num<10)
            return 1;
        int yu = num % 100;
        if(yu<10||yu>25){
            return translateNum(num/10);
        }
        else
            return translateNum(num/10)+translateNum(num/100);
    }
}
```

代码2（0ms）：

```java
class Solution {
    public int translateNum(int num) {
        int a = 1, b = 1;
        String str = String.valueOf(num);
        for (int i = str.length() - 2, tmp; i >= 0; --i) {
            tmp = str.substring(i, i + 2).compareTo("10") >= 0 && str.substring(i, i + 2).compareTo("25") <= 0 ? a + b : a;
            b = a;
            a = tmp;
        }
        return a;
    }
}
```

### 面试题33. 二叉搜索树的后序遍历序列

> [面试题33. 二叉搜索树的后序遍历序列](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-de-hou-xu-bian-li-xu-lie-lcof/)

语言：java

思路：递归判断。后序遍历（小大中），拆分左右子树递归判断是否符合后序遍历。

代码（0ms）：

```java
class Solution {
    public boolean verifyPostorder(int[] postorder) {
        return verifyDFS(postorder, 0, postorder.length - 1);
    }

    public boolean verifyDFS(int[] postorder, int left, int right) {

        if (left >= right)
            return true;
        int a = left;
        while (a < right && postorder[a] < postorder[right]) ++a;
        int b = a;
        while (b < right && postorder[b] > postorder[right]) ++b;
        return b == right && verifyDFS(postorder, left, a - 1) && verifyDFS(postorder, a, right-1);
    }
}
```

参考代码1（0ms）：递归，拆分左右子树判断

> [面试题33. 二叉搜索树的后序遍历序列（递归分治 / 单调栈，清晰图解）](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-de-hou-xu-bian-li-xu-lie-lcof/solution/mian-shi-ti-33-er-cha-sou-suo-shu-de-hou-xu-bian-6/)

```java
class Solution {
    public boolean verifyPostorder(int[] postorder) {
        return recur(postorder, 0, postorder.length - 1);
    }
    boolean recur(int[] postorder, int i, int j) {
        if(i >= j) return true;
        int p = i;
        while(postorder[p] < postorder[j]) p++;
        int m = p;
        while(postorder[p] > postorder[j]) p++;
        return p == j && recur(postorder, i, m - 1) && recur(postorder, m, j - 1);
    }
}
```

参考代码2（1ms，21.1%）：单调栈，神奇的思路

> [单调递增栈辅助，逆向遍历数组](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-de-hou-xu-bian-li-xu-lie-lcof/solution/dan-diao-di-zeng-zhan-by-shi-huo-de-xia-tian/)
>
> [面试题33. 二叉搜索树的后序遍历序列（递归分治 / 单调栈，清晰图解）](https://leetcode-cn.com/problems/er-cha-sou-suo-shu-de-hou-xu-bian-li-xu-lie-lcof/solution/mian-shi-ti-33-er-cha-sou-suo-shu-de-hou-xu-bian-6/)

```java
class Solution {
    public boolean verifyPostorder(int[] postorder) {
        Stack<Integer> stack = new Stack<>();
        int root = Integer.MAX_VALUE;
        for(int i = postorder.length - 1; i >= 0; i--) {
            if(postorder[i] > root) return false;
            while(!stack.isEmpty() && stack.peek() > postorder[i])
            	root = stack.pop();
            stack.add(postorder[i]);
        }
        return true;
    }
}
```

### 面试题59 - II. 队列的最大值

> [面试题59 - II. 队列的最大值](https://leetcode-cn.com/problems/dui-lie-de-zui-da-zhi-lcof/)

语言：java

思路：类似的，记得之前有个要求快速获取最小值[面试题30. 包含min函数的栈](https://leetcode-cn.com/problems/bao-han-minhan-shu-de-zhan-lcof/)。用一个双向队列存储max。

代码（38ms，61.1%）：

```java
class MaxQueue {

    Deque<Integer> deque;
    Deque<Integer> maxDeque;

    public MaxQueue() {
        deque = new LinkedList<>();
        maxDeque = new LinkedList<>();
    }
    //  3 4 2 5 1 6 7
    //  4
    public int max_value() {
        return maxDeque.isEmpty() ? -1 : maxDeque.peekFirst();
    }

    public void push_back(int value) {
        deque.addLast(value);
        while(!maxDeque.isEmpty()&&maxDeque.peekLast()<value) maxDeque.pollLast();
        maxDeque.addLast(value);
    }

    public int pop_front() {
        if(deque.isEmpty())
            return -1;
        int front = deque.pollFirst();
        if(!maxDeque.isEmpty()&&front == maxDeque.peekFirst())
            maxDeque.pollFirst();
        return front;
    }
}
```

参考代码1（32ms）：用数组模拟的，更快一点。

```java
class MaxQueue {
    int MAXQueueTail = 0;
    int MAXQueueHead = 0;
    int QueueTail = 0;
    int QueueHead = 0;
    int[] Queue,MAXQueue;

    public MaxQueue() {
        Queue = new int[10000];
        MAXQueue = new int[10000];
    }

    public int max_value() {
        if(MAXQueueHead == MAXQueueTail){
            // 头尾相等的时候，表示此时队列为空，没有最大值
            return -1;
        }
        return MAXQueue[MAXQueueHead];
    }

    public void push_back(int value) {
        Queue[QueueTail++] = value;
        while(MAXQueueHead != MAXQueueTail && MAXQueue[MAXQueueTail-1] < value){
            // MAXQueueTail-1 因为MAXQueueTail处的值是0，还没有被初始化
            // 比value小的值，一定会在value出栈前，先出栈，
            // 队列中的最大值最少都是value，就没有保存比value小的值的必要了
            MAXQueueTail--;
        }
        MAXQueue[MAXQueueTail++] = value;

    }

    public int pop_front() {
        if(QueueHead == QueueTail){
            // 队列为空
            return -1;
        }
        int res = Queue[QueueHead];
        if(res == MAXQueue[MAXQueueHead]){
            MAXQueueHead++;
        }
        QueueHead++;
        return res;
    }
}
```

### 面试题13. 机器人的运动范围

> [面试题13. 机器人的运动范围](https://leetcode-cn.com/problems/ji-qi-ren-de-yun-dong-fan-wei-lcof/)

语言：java

思路：DFS。就简单地判断是否符合要求。这里需要注意的就是边界100是肯定不可能到达的，所以就不用考虑了。虽然m和n<=100，但是题目有明确说了顶多走到(m-1,n-1)，也就是撑死到（99，99）.

代码（0ms，100%）：

```java
class Solution {

    public int movingCount(int m, int n, int k) {
        boolean[][] map = new boolean[m][n];
        return dfs(0,0,m, n, k, map);
    }

    public int dfs(int x, int y, int m, int n, int k, boolean[][] map) {
        if (x >= m || y >= n || map[x][y] ||(x/10+x%10+y/10+y%10)>k)
            return 0;
        map[x][y] = true;
        return 1 + dfs(x+1,y,m,n,k,map)+dfs(x,y+1,m,n,k,map);
    }
}
```

参考代码1（7ms，21.10%）：BFS

> [面试题13. 机器人的运动范围（ DFS / BFS ，清晰图解）](https://leetcode-cn.com/problems/ji-qi-ren-de-yun-dong-fan-wei-lcof/solution/mian-shi-ti-13-ji-qi-ren-de-yun-dong-fan-wei-dfs-b/)

```java
class Solution {
    public int movingCount(int m, int n, int k) {
        boolean[][] visited = new boolean[m][n];
        int res = 0;
        Queue<int[]> queue= new LinkedList<int[]>();
        queue.add(new int[] { 0, 0, 0, 0 });
        while(queue.size() > 0) {
            int[] x = queue.poll();
            int i = x[0], j = x[1], si = x[2], sj = x[3];
            if(i >= m || j >= n || k < si + sj || visited[i][j]) continue;
            visited[i][j] = true;
            res ++;
            queue.add(new int[] { i + 1, j, (i + 1) % 10 != 0 ? si + 1 : si - 8, sj });
            queue.add(new int[] { i, j + 1, si, (j + 1) % 10 != 0 ? sj + 1 : sj - 8 });
        }
        return res;
    }
}
```

### 面试题26. 树的子结构

> [面试题26. 树的子结构](https://leetcode-cn.com/problems/shu-de-zi-jie-gou-lcof/)

语言：java

思路：先写dfs递归方程。首先，如果当前A某个子树和B整个比较完了，那么肯定B会走到null，而只有比较完了，B才会变成null，所以

+ B==null，返回true。只有比较的过程往下递归，B才可能变成null
+ 如果B!=null但是A==null，那肯定就当前这回递归行不通了，直接返回false
+ 如果A和B值相同，就判断A和B的左右子树是否也相同结构。

编写完dfs递归方程后，返回原方程思考，如果简单只调用dfs(A，B)就出现A根节点和B根节点不同，但是也向下进行比较了，会直接返回false；也就是除了从当前节点以外，还必须考虑A的左右两个子节点是否可能含有B。因为左右子树都可能，所以用逻辑或||调用判断A的左子树和A的右子树是否和B根节点相同。而当A为null或者B为null时，说明A左右子树向下还没有找到和B相同的结构，且树走到结尾了，直接返回false；

代码（0ms，100%）：

```java
class Solution {

    public boolean isSubStructure(TreeNode A, TreeNode B) {
        if(A==null||B==null)
            return false;
        return dfs(A, B)||isSubStructure(A.left,B)||isSubStructure(A.right,B);
    }

    public boolean dfs(TreeNode A, TreeNode B) {
        if(B==null)
            return true;
        if(A==null)
            return false;
        return A.val==B.val && dfs(A.left,B.left)&&dfs(A.right,B.right);
    }
}
```

参考代码1（0ms）: 思路没区别。

>[面试题26. 树的子结构（先序遍历 + 包含判断，清晰图解）](https://leetcode-cn.com/problems/shu-de-zi-jie-gou-lcof/solution/mian-shi-ti-26-shu-de-zi-jie-gou-xian-xu-bian-li-p/)

```java
class Solution {
    public boolean isSubStructure(TreeNode A, TreeNode B) {
        return (A != null && B != null) && (recur(A, B) || isSubStructure(A.left, B) || isSubStructure(A.right, B));
    }
    boolean recur(TreeNode A, TreeNode B) {
        if(B == null) return true;
        if(A == null || A.val != B.val) return false;
        return recur(A.left, B.left) && recur(A.right, B.right);
    }
}
```

### 面试题48. 最长不含重复字符的子字符串

> [面试题48. 最长不含重复字符的子字符串](https://leetcode-cn.com/problems/zui-chang-bu-han-zhong-fu-zi-fu-de-zi-zi-fu-chuan-lcof/)

语言：java

思路：遍历字符串，用指针l表示最大不重复字符串的左边界。每次都改变左边界，最后max要么时原本的max，要么是最新的i-l+1。这里改变左边界，是考虑到每次往下读一个字符，可能和之前那个最大串有重复，有重复的话，就把重复字符的下一个位置作为左边界；没重复更好，直接保持原样就好。

代码（4ms，92.96%）：

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        int max = 0;
        char[] chars = s.toCharArray();
        int l=0;
        for(int i = 0,len = chars.length;i<len;++i){
            l=leftBorder(chars, l,i-1,chars[i]);
            max = Math.max(max,i-l+1);
        }
        return max;
    }

    public int leftBorder(char[] chars,int start,int end,char c){
        for(int i = start;i<=end;++i){
            if(chars[i]== c)
                return i+1;
        }
        return start;
    }
}
```

参考代码1（3ms）：直接用一个数组标记某一字符上次出现的位置。

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        int ans=0;
        int[] index=new int[256];
        Arrays.fill(index,-1);
        int start=0;
        int end;
        for(end=0;end<s.length();end++){
            char c=s.charAt(end);
            if(index[c]>=start){

                start=index[c]+1;
            }
            ans=Math.max(ans,end-start+1);
            index[c]=end;
        }
        // ans=Math.max(ans,end-start);
        return ans;
    }
}
```

参考代码2（2ms）：我是把判断重复放到单独函数里，这里就是直接放进来，其他没什么太大区别。

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        if(s.length() == 0){
            return 0;
        }
        int result = 1;
        int num = 1;
        int k = 0;
        int j;
        char[] chars = s.toCharArray();
        for(int i = 1;i<s.length();i++){
            for(j =k ;j< i;j++ ){
                if(chars[j]==chars[i]){
                    num = i-j;
                    k = j+1;
                    if(num >result){
                        result = num;
                    }
                    break;
                }
            }
            if(j==i){
                num = num +1;
                if(num > result){
                    result = num;

                }
            }
        }
        return  result;
    }
}
```

参考代码3（8ms，72.35%）：

> [面试题48. 最长不含重复字符的子字符串（动态规划 / 双指针 + 哈希表，清晰图解）](https://leetcode-cn.com/problems/zui-chang-bu-han-zhong-fu-zi-fu-de-zi-zi-fu-chuan-lcof/solution/mian-shi-ti-48-zui-chang-bu-han-zhong-fu-zi-fu-d-9/)

```java
class Solution {
    public int lengthOfLongestSubstring(String s) {
        Map<Character, Integer> dic = new HashMap<>();
        int res = 0, tmp = 0;
        for(int j = 0; j < s.length(); j++) {
            int i = dic.containsKey(s.charAt(j)) ? dic.get(s.charAt(j)) : -1; // 获取索引 i
            dic.put(s.charAt(j), j); // 更新哈希表
            tmp = tmp < j - i ? tmp + 1 : j - i; // dp[j - 1] -> dp[j]
            res = Math.max(res, tmp); // max(dp[j - 1], dp[j])
        }
        return res;
    }
}
```

### 面试题43. 1～n整数中1出现的次数

> [面试题43. 1～n整数中1出现的次数](https://leetcode-cn.com/problems/1nzheng-shu-zhong-1chu-xian-de-ci-shu-lcof/)
>
> [面试题43. 1～n 整数中 1 出现的次数（清晰图解）](https://leetcode-cn.com/problems/1nzheng-shu-zhong-1chu-xian-de-ci-shu-lcof/solution/mian-shi-ti-43-1n-zheng-shu-zhong-1-chu-xian-de-2/)
>
> [java递归](https://leetcode-cn.com/problems/1nzheng-shu-zhong-1chu-xian-de-ci-shu-lcof/solution/javadi-gui-by-xujunyi/)

语言：java

思路：尝试的方式超时or超内存了。建议直接看上述解题文章

代码（0ms，100%）：

```java
class Solution {
    public int countDigitOne(int n) {
        // 2304 23 * 10
        // 2314 23 *10 + 4 + 1
        // 2324 (23+1) * 10
        int digit = 1,res = 0;
        int high = n / 10 , cur = n % 10,low = 0;
        while(high!=0||cur!=0){
            if(cur==0)res+= high * digit;
            else if (cur==1)res+=high * digit+low+1;
            else res += (high+1)*digit;
            low+=cur*digit;
            cur = high % 10;
            high /= 10;
            digit *= 10;
        }
        return res;
    }
}
```

参考代码1（0ms）：

```java
class Solution {
    public int countDigitOne(int n) {
        if (n == 0) {
            return 0;
        }

        String value = String.valueOf(n);
        int high = value.charAt(0) - '0';
        int pow = (int) Math.pow(10, value.length() - 1);
        int last = n - high * pow;

        if (high == 1) {
            return countDigitOne(last) + countDigitOne(pow - 1) + last + 1;
        } else {
            return countDigitOne(last) + high * countDigitOne(pow - 1) + pow;
        }
    }
}
```

### 面试题12. 矩阵中的路径

> [面试题12. 矩阵中的路径](https://leetcode-cn.com/problems/ju-zhen-zhong-de-lu-jing-lcof/)

语言：java

思路：中规中矩的DFS，用二维数组记录走过的路径，一维数组记录需要对比的目标字符串方格。

代码（5ms，98.77%）：

```java
class Solution {
    public boolean exist(char[][] board, String word) {
        int x = board.length, y = board[0].length, target = word.length()-1;
        char[] words = word.toCharArray();
        boolean[][] visited = new boolean[x][y];
        for (int i = 0; i < x; ++i) {
            for (int j = 0; j < y; ++j) {
                if (board[i][j] == words[0]) {
                    if (dfs(i, j, x, y, 0, target, board, visited, words))
                        return true;
                }
            }
        }
        return false;
    }

    public boolean dfs(int i, int j, int x, int y, int stage, int target, char[][] map, boolean[][] visited, char[] words) {
        if (i < 0 || j < 0 || i >= x || j >= y || map[i][j] != words[stage]||visited[i][j])
            return false;
        if (stage == target)
            return true;

        visited[i][j] = true;

        if (dfs(i, j + 1, x, y, stage + 1, target, map, visited, words) // 右
            || dfs(i + 1, j, x, y, stage + 1, target, map, visited, words) // 下
            || dfs(i, j - 1, x, y, stage + 1, target, map, visited, words) // 左
            || dfs(i - 1, j, x, y, stage + 1, target, map, visited, words) ){// 上
            return true;
        }

        visited[i][j] = false;// 还原路径
        return false;
    }
}
```

参考代码1（8ms，40.91%）：

> [面试题12. 矩阵中的路径（深度优先搜索 DFS ，清晰图解）](https://leetcode-cn.com/problems/ju-zhen-zhong-de-lu-jing-lcof/solution/mian-shi-ti-12-ju-zhen-zhong-de-lu-jing-shen-du-yo/)

```java
class Solution {
    public boolean exist(char[][] board, String word) {
        char[] words = word.toCharArray();
        for(int i = 0; i < board.length; i++) {
            for(int j = 0; j < board[0].length; j++) {
                if(dfs(board, words, i, j, 0)) return true;
            }
        }
        return false;
    }
    boolean dfs(char[][] board, char[] word, int i, int j, int k) {
        if(i >= board.length || i < 0 || j >= board[0].length || j < 0 || board[i][j] != word[k]) return false;
        if(k == word.length - 1) return true;
        char tmp = board[i][j];
        board[i][j] = '/';
        boolean res = dfs(board, word, i + 1, j, k + 1) || dfs(board, word, i - 1, j, k + 1) || 
            dfs(board, word, i, j + 1, k + 1) || dfs(board, word, i , j - 1, k + 1);
        board[i][j] = tmp;
        return res;
    }
}
```

参考代码2（4ms）：大同小异，没啥区别

```java
class Solution {
    boolean[][] visited;

    public boolean exist(char[][] board, String word) {
        if(word==null||word.length()==0)return true;
        char firsech=word.charAt(0);
        int width=board.length;
        int lon=board[0].length;
        visited=new boolean[width][lon];

        for(int i=0;i<width;i++) {
            for(int j=0;j<lon;j++) {
                if(board[i][j]==firsech) {
                    if(subexist(board, word, i, j, 1))return true;
                }
            }
        }
        return false;
    }

    boolean subexist(char[][] board, String word,int i,int j,int n) {
        if(word.length()==n)return true;
        visited[i][j]=true;

        if(i>0&&board[i-1][j]==word.charAt(n)&&!visited[i-1][j])
            if(subexist(board, word, i-1, j, n+1))return true;
        if(i<board.length-1&&board[i+1][j]==word.charAt(n)&&!visited[i+1][j])
            if(subexist(board, word, i+1, j, n+1))return true;
        if(j>0&&board[i][j-1]==word.charAt(n)&&!visited[i][j-1])
            if(subexist(board, word, i, j-1, n+1))return true;
        if(j<board[0].length-1&&board[i][j+1]==word.charAt(n)&&!visited[i][j+1])
            if(subexist(board, word, i, j+1, n+1))return true;

        visited[i][j]=false;
        return false;
    }
}
```

### 面试题44. 数字序列中某一位的数字

> [面试题44. 数字序列中某一位的数字](https://leetcode-cn.com/problems/shu-zi-xu-lie-zhong-mou-yi-wei-de-shu-zi-lcof/)
>
> [面试题44. 数字序列中某一位的数字（迭代 + 求整 / 求余，清晰图解）](https://leetcode-cn.com/problems/shu-zi-xu-lie-zhong-mou-yi-wei-de-shu-zi-lcof/solution/mian-shi-ti-44-shu-zi-xu-lie-zhong-mou-yi-wei-de-6/)
>
> [详解 找规律](https://leetcode-cn.com/problems/shu-zi-xu-lie-zhong-mou-yi-wei-de-shu-zi-lcof/solution/zhe-shi-yi-dao-shu-xue-ti-ge-zhao-gui-lu-by-z1m/)

语言：java

思路：又是纯数学+规律题，建议直接看上面解析

代码（0ms）：这注意要用long，而不是int，自己就踩坑了

```java
class Solution {
    public int findNthDigit(int n) {
        int digit = 1;
        long start = 1;
        long count = 9;
        while (n > count) { // 1.
            n -= count;
            digit += 1;
            start *= 10;
            count = digit * start * 9;
        }
        long num = start + (n - 1) / digit;
        return Long.toString(num).charAt((n - 1) % digit) - '0'; 
    }
}
```

### 面试题16. 数值的整数次方

> [面试题16. 数值的整数次方](https://leetcode-cn.com/problems/shu-zhi-de-zheng-shu-ci-fang-lcof/)
>
> [面试题16. 数值的整数次方（快速幂，清晰图解）](https://leetcode-cn.com/problems/shu-zhi-de-zheng-shu-ci-fang-lcof/solution/mian-shi-ti-16-shu-zhi-de-zheng-shu-ci-fang-kuai-s/)
>
> [递归写法（分治思想）与非递归写法（将指数看成二进制数）](https://leetcode-cn.com/problems/shu-zhi-de-zheng-shu-ci-fang-lcof/solution/di-gui-xie-fa-fen-zhi-si-xiang-yu-fei-di-gui-xie-f/)

语言：java

思路：可以说还是纯数学题，建议直接看上面的讲解

代码（1ms，93.25%）：

```java
class Solution {
    public double myPow(double x, int n) {
        double res = 1.0;
        long power = n;
        if(n<0){
            x = 1/x;
            power = -power;
        }
        while(power>0){
            if((power&1)==1)
                res *= x;
            x *= x;
            power >>= 1;
        }
        return res;
    }
}
```

参考代码1（0ms）：递归写法。假设是2^-5，那么递归就是reutrn 2的-2次方运算 ^2 *  2的-1次方运算

```java
class Solution {
    public double myPow(double x, int n) {
        if(n == 1)
            return x;
        if(n == 0)
            return 1;
        if(n == -1)
            return 1 / x;
        double half = myPow(x, n / 2);
        double rest = myPow(x, n % 2);
        return half * half * rest;
    }
}
```

### 面试题14- II. 剪绳子 II

> [面试题14- II. 剪绳子 II](https://leetcode-cn.com/problems/jian-sheng-zi-ii-lcof/)
>
> [面试题14- II. 剪绳子 II（数学推导 / 贪心思想 + 快速幂求余，清晰图解）](https://leetcode-cn.com/problems/jian-sheng-zi-ii-lcof/solution/mian-shi-ti-14-ii-jian-sheng-zi-iitan-xin-er-fen-f/)

语言：java

思路：贪心算法，但本质也还是数学问题

代码（0ms）：

```java
class Solution {
    public int cuttingRope(int n) {
        if(n<4)
            return n-1;
        long res = 1;
        while(n>4){
            res = (res*3)%1000000007;
            n-=3;
        }
        res= (res*n)%1000000007;
        return (int)res;
    }
}
```

### 面试题67. 把字符串转换成整数

> [面试题67. 把字符串转换成整数](https://leetcode-cn.com/problems/ba-zi-fu-chuan-zhuan-huan-cheng-zheng-shu-lcof/)

语言：java

思路：没啥特别的，就是题目条件比较苛刻，需要对符号等判断多分情况处理

代码（2ms，99.92%）：

```java
class Solution {
    public int strToInt(String str) {
        char[] chars = str.toCharArray();
        int len = chars.length;
        int i = 0;
        int res = 0;
        boolean neg = false;
        boolean pos = false;
        boolean num = false;
        while (i<len&&chars[i] == ' ') ++i;
        for (int j = i, tmp; j < len; ++j) {
            if (chars[j] == '-') {
                if (neg||pos||num)
                    break;
                neg = true;
                continue;
            }
            if(chars[j] == '+'){
                if(neg||pos||num)
                    break;
                pos = true;
                continue;
            }
            if (chars[j] < '0' || chars[j] > '9')
                break;
            num = true;
            tmp = chars[j] - '0';
            if (res + tmp> 214748371){
                if(neg)
                    return Integer.MIN_VALUE;
                return Integer.MAX_VALUE;
            }
            res = res * 10 + tmp;
        }
        return neg ? -res : res;
    }
}
```

参考代码1（1ms）：符号处理上稍微不太一样

```java
class Solution {
    public int strToInt(String str) {
        int idx = 0;
        int length = str.length();
        while(idx < length && str.charAt(idx) == ' '){idx++;}

        boolean isNegative = false;
        int ans = 0;

        if(idx < length && (str.charAt(idx) == '+' || str.charAt(idx) == '-')){
            isNegative = str.charAt(idx) == '-'? true : false;
            idx++;
        }

        while(idx < length && isDigit(str.charAt(idx))){
            int num = str.charAt(idx) - '0';
            if(ans > Integer.MAX_VALUE / 10 || (ans == Integer.MAX_VALUE / 10 && num > 7)){
                return isNegative? Integer.MIN_VALUE : Integer.MAX_VALUE;
            }
            idx++;
            ans = ans * 10 + num;
        }
        return isNegative? -ans : ans;
    }
    private boolean isDigit(char c){
        if(c >= '0' && c <= '9'){
            return true;
        }else{
            return false;
        }
    }
}
```

参考代码2（2ms，99.92%）：直接用long，更省事

> [面试题67. 把字符串转换成整数（清晰图解）](https://leetcode-cn.com/problems/ba-zi-fu-chuan-zhuan-huan-cheng-zheng-shu-lcof/solution/mian-shi-ti-67-ba-zi-fu-chuan-zhuan-huan-cheng-z-4/)

```java
class Solution {
    public int strToInt(String str) {
        char[] c = str.trim().toCharArray();
        if(c.length == 0) return 0;
        long res = 0;
        int i = 1, sign = 1;
        if(c[0] == '-') sign = -1;
        else if(c[0] != '+') i = 0;
        for(int j = i; j < c.length; j++) {
            if(c[j] < '0' || c[j] > '9') break;
            res = res * 10 + (c[j] - '0');
            if(res > Integer.MAX_VALUE) return sign == 1 ? Integer.MAX_VALUE : Integer.MIN_VALUE;
        }
        return sign * (int)res;
    }
}
```

### 面试题20. 表示数值的字符串

> [面试题20. 表示数值的字符串](https://leetcode-cn.com/problems/biao-shi-shu-zhi-de-zi-fu-chuan-lcof/)
>
> [Java版本题解，逻辑清晰。](https://leetcode-cn.com/problems/biao-shi-shu-zhi-de-zi-fu-chuan-lcof/solution/javaban-ben-ti-jie-luo-ji-qing-xi-by-yangshyu6/)
>
> [确定有限自动机DFA](https://leetcode-cn.com/problems/biao-shi-shu-zhi-de-zi-fu-chuan-lcof/solution/que-ding-you-xian-zi-dong-ji-dfa-by-justyou/)

语言：java

思路：本来看着以为就简单for和if就好了。没想到折腾挺久，还是没整好。主要还是没能明确题目对于什么是数值的判断。感觉题目并没能把”准确的数字“必须怎么组成说明清楚。

代码（2ms，100%）：这里直接贴上面的参考代码，不折腾了（自己写的版本从无到有，改了5-6次了，都不行）

```java
class Solution {
    public boolean isNumber(String s) {
        if(s == null || s.length() == 0){
            return false;
        }
        //标记是否遇到相应情况
        boolean numSeen = false;
        boolean dotSeen = false;
        boolean eSeen = false;
        char[] str = s.trim().toCharArray();
        for(int i = 0;i < str.length; i++){
            if(str[i] >= '0' && str[i] <= '9'){
                numSeen = true;
            }else if(str[i] == '.'){
                //.之前不能出现.或者e
                if(dotSeen || eSeen){
                    return false;
                }
                dotSeen = true;
            }else if(str[i] == 'e' || str[i] == 'E'){
                //e之前不能出现e，必须出现数
                if(eSeen || !numSeen){
                    return false;
                }
                eSeen = true;
                numSeen = false;//重置numSeen，排除123e或者123e+的情况,确保e之后也出现数
            }else if(str[i] == '-' || str[i] == '+'){
                //+-出现在0位置或者e/E的后面第一个位置才是合法的
                if(i != 0 && str[i-1] != 'e' && str[i-1] != 'E'){
                    return false;
                }
            }else{//其他不合法字符
                return false;
            }
        }
        return numSeen;
    }
}
```

### 面试题41. 数据流中的中位数

> [面试题41. 数据流中的中位数](https://leetcode-cn.com/problems/shu-ju-liu-zhong-de-zhong-wei-shu-lcof/)
>
> [面试题41. 数据流中的中位数（优先队列 / 堆，清晰图解）](https://leetcode-cn.com/problems/shu-ju-liu-zhong-de-zhong-wei-shu-lcof/solution/mian-shi-ti-41-shu-ju-liu-zhong-de-zhong-wei-shu-y/)
>
> [优先队列，无废话简单易懂](https://leetcode-cn.com/problems/shu-ju-liu-zhong-de-zhong-wei-shu-lcof/solution/you-xian-dui-lie-wu-fei-hua-jian-dan-yi-dong-by-je/)
>
> [图解 排序+二分查找+优先队列](https://leetcode-cn.com/problems/shu-ju-liu-zhong-de-zhong-wei-shu-lcof/solution/you-xian-dui-lie-by-z1m/)

语言：java

思路：自己老实地用list，每次插入前先找到位置再插入，果不其然地超时了。建议直接看上面的解析。

代码（75ms，90.81%）：

```java
class MedianFinder {
    PriorityQueue<Integer> maxStack, minStack;

    public MedianFinder() {
        maxStack = new PriorityQueue<>((x, y) -> y - x); // 大顶堆，存较小部分
        minStack = new PriorityQueue<>(); // 小顶堆，存较大部分
    }

    public void addNum(int num) {
        if(minStack.size()==maxStack.size()){
            maxStack.add(num);
            minStack.add(maxStack.poll());
        }else{
            minStack.add(num);
            maxStack.add(minStack.poll());
        }
    }

    public double findMedian() {
        if(minStack.size()==maxStack.size()){
            return (minStack.peek() + maxStack.peek()) / 2.0;
        }else{
            return minStack.peek();
        }
    }
}
```

### 面试题37. 序列化二叉树

> [面试题37. 序列化二叉树](https://leetcode-cn.com/problems/xu-lie-hua-er-cha-shu-lcof/)

语言：java

思路：层次遍历BFS

代码：（18ms，75.68%）：

```java
public class Codec {

    // Encodes a tree to a single string.
    public String serialize(TreeNode root) {
        if (root == null)
            return "[]";
        Queue<TreeNode> queue = new LinkedList<>();
        queue.add(root);
        StringBuilder sb = new StringBuilder();
        sb.append("[");
        while (!queue.isEmpty()) {
            TreeNode tmp = queue.poll();
            if(tmp!=null){
                sb.append(tmp.val).append(",");
                queue.add(tmp.left);
                queue.add(tmp.right);
            }
            else
                sb.append("null").append(",");
        }
        sb.deleteCharAt(sb.length() - 1);
        sb.append("]");
        return sb.toString();
    }

    // Decodes your encoded data to tree.
    public TreeNode deserialize(String data) {
        if (data.equals("[]"))
            return null;
        String[] vals = data.substring(1, data.length() - 1).split(",");
        TreeNode root = new TreeNode(Integer.parseInt(vals[0]));
        Queue<TreeNode> queue = new LinkedList<>();
        queue.add(root);
        TreeNode tmp, newTmp;
        int i = 1;
        while (!queue.isEmpty()) {
            tmp = queue.poll();
            if (!vals[i].equals("null")) {
                newTmp = new TreeNode(Integer.parseInt(vals[i]));
                tmp.left = newTmp;
                queue.add(newTmp);
            }
            ++i;
            if (!vals[i].equals("null")) {
                newTmp = new TreeNode(Integer.parseInt(vals[i]));
                tmp.right = newTmp;
                queue.add(newTmp);
            }
            ++i;
        }
        return root;
    }
}
```

参考代码1（25ms，55.75%）：也是层次遍历，不是很动为啥比我的慢，感觉看着没什么太大区别，可能是StringBuild的append夹杂String字符串连接操作的原因？

> [面试题37. 序列化二叉树（层序遍历 BFS ，清晰图解）](https://leetcode-cn.com/problems/xu-lie-hua-er-cha-shu-lcof/solution/mian-shi-ti-37-xu-lie-hua-er-cha-shu-ceng-xu-bian-/)

```java
public class Codec {
    public String serialize(TreeNode root) {
        if(root == null) return "[]";
        StringBuilder res = new StringBuilder("[");
        Queue<TreeNode> queue = new LinkedList<>() {{ add(root); }};
        while(!queue.isEmpty()) {
            TreeNode node = queue.poll();
            if(node != null) {
                res.append(node.val + ",");
                queue.add(node.left);
                queue.add(node.right);
            }
            else res.append("null,");
        }
        res.deleteCharAt(res.length() - 1);
        res.append("]");
        return res.toString();
    }

    public TreeNode deserialize(String data) {
        if(data.equals("[]")) return null;
        String[] vals = data.substring(1, data.length() - 1).split(",");
        TreeNode root = new TreeNode(Integer.parseInt(vals[0]));
        Queue<TreeNode> queue = new LinkedList<>() {{ add(root); }};
        int i = 1;
        while(!queue.isEmpty()) {
            TreeNode node = queue.poll();
            if(!vals[i].equals("null")) {
                node.left = new TreeNode(Integer.parseInt(vals[i]));
                queue.add(node.left);
            }
            i++;
            if(!vals[i].equals("null")) {
                node.right = new TreeNode(Integer.parseInt(vals[i]));
                queue.add(node.right);
            }
            i++;
        }
        return root;
    }
}
```

参考代码2（2ms，95.32%）：前序遍历+递归

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode(int x) { val = x; }
 * }
 */
public class Codec {
    StringBuilder sb;
    int id;

    // Encodes a tree to a single string.
    public String serialize(TreeNode root) {
        sb=new StringBuilder();
        serializeUtil(root);
        return sb.toString();
    }

    public void serializeUtil(TreeNode node){
        if(node==null){
            sb.append("#");
            return;
        }

        sb.append((char)(node.val+'0'));
        serializeUtil(node.left);
        serializeUtil(node.right);
    }

    // Decodes your encoded data to tree.
    public TreeNode deserialize(String data) {
        id=0;
        return deserializeUtil(data.toCharArray());
    }

    public TreeNode deserializeUtil(char[] data){
        if(data[id]=='#'){
            id++;
            return null;
        }

        TreeNode root=new TreeNode(data[id++]-'0');
        root.left=deserializeUtil(data);
        root.right=deserializeUtil(data);

        return root;
    }

}
```

参考代码2后重写（2ms）：

```java
public class Codec {

    StringBuilder sb;
    int i;

    // Encodes a tree to a single string.
    public String serialize(TreeNode root) {
        sb = new StringBuilder();
        serializeDFS(root);
        return sb.toString();
    }

    public void serializeDFS(TreeNode root) {
        if(root==null){
            sb.append("#");
            return;
        }
        sb.append((char)(root.val+'0'));
        serializeDFS(root.left);
        serializeDFS(root.right);
    }

    // Decodes your encoded data to tree.
    public TreeNode deserialize(String data) {
        i = 0;
        return deserializeDFS(data.toCharArray());
    }

    public TreeNode deserializeDFS(char[] data) {
        if(data[i]=='#'){
            ++i;
            return null;
        }
        TreeNode node = new TreeNode(data[i++]-'0');
        node.left = deserializeDFS(data);
        node.right = deserializeDFS(data);
        return node;
    }
}
```

### 面试题51. 数组中的逆序对

> [面试题51. 数组中的逆序对](https://leetcode-cn.com/problems/shu-zu-zhong-de-ni-xu-dui-lcof/)
>
> [数组中的逆序对--官方讲解](https://leetcode-cn.com/problems/shu-zu-zhong-de-ni-xu-dui-lcof/solution/shu-zu-zhong-de-ni-xu-dui-by-leetcode-solution/)

语言：java

思路：本来天真地想暴力双层for，果不其然超时了。根据官方讲解可以用归并排序解题，这里尝试下

代码（34ms，92.20%）：

```java
class Solution {
    public int reversePairs(int[] nums) {
        return reversePairs(nums, 0, nums.length - 1, new int[nums.length]);
    }

    public int reversePairs(int[] raw, int left, int right, int[] temp) {
        if (left < right) {
            int mid = left + (right - left) / 2;
            int leftPairs = reversePairs(raw, left, mid, temp);
            int rightPairs = reversePairs(raw, mid + 1, right, temp);
            if (raw[mid] <= raw[mid + 1])
                return leftPairs + rightPairs;
            int mergePairs = mergePairs(raw, left, mid, right, temp);
            return leftPairs + rightPairs + mergePairs;
        } else
            return 0;
    }

    public int mergePairs(int[] raw, int left, int mid, int right, int[] temp) {
        int i = left, j = mid+1, t = 0, pairs = 0;
        while (i <= mid && j <= right) {
            if (raw[i] <= raw[j]) {
                temp[t++] = raw[i++];
            } else {
                temp[t++] = raw[j++];
                pairs += mid-i+1;
            }
        }
        while (i <= mid) {
            temp[t++] = raw[i++];
        }
        while (j <= right) {
            temp[t++] = raw[j++];
        }
        t = 0;
        while (left <= right) {
            raw[left++] = temp[t++];
        }
        return pairs;
    }
}
```

参考代码1（27ms）：merge采用了非递归的形式。

```java
class Solution {
    public int reversePairs(int[] nums) {
        if(nums==null||nums.length<1)
            return 0;
        int[] copy=new int[nums.length];
        System.arraycopy(nums,0,copy,0,copy.length);

        return mergeSort(nums,copy,0,nums.length-1);
    }

    public int mergeSort(int[] nums,int[] copy,int start,int end) {
        if(start==end){
            copy[start]=nums[start];
            return 0;
        }
        int middle=(start+end)>>1;
        int leftcount=mergeSort(copy,nums,start,middle);
        int rightcount=mergeSort(copy,nums,middle+1,end);
        int count=0;
        int i=middle,j=end;
        int lastindex=end;
        while(i>=start&&j>=middle+1){
            if(nums[i]>nums[j]){
                copy[lastindex--]=nums[i--];
                count+=j-middle;
            }
            else
                copy[lastindex--]=nums[j--];
        }
        while(i>=start)
            copy[lastindex--]=nums[i--];
        while(j>=middle+1) 
            copy[lastindex--]=nums[j--];

        return count+leftcount+rightcount;
    }
}
```

### 面试题19. 正则表达式匹配

> [面试题19. 正则表达式匹配](https://leetcode-cn.com/problems/zheng-ze-biao-da-shi-pi-pei-lcof/)
>
> [逐行详细讲解，由浅入深，dp和递归两种思路](https://leetcode-cn.com/problems/zheng-ze-biao-da-shi-pi-pei-lcof/solution/zhu-xing-xiang-xi-jiang-jie-you-qian-ru-shen-by-je/)
>
> [动态规划超详细解答，由繁入简。](https://leetcode-cn.com/problems/zheng-ze-biao-da-shi-pi-pei-lcof/solution/dong-tai-gui-hua-chao-xiang-xi-jie-da-you-fan-ru-j/)
>
> [正则表达式匹配 - 递归求解](https://leetcode-cn.com/problems/zheng-ze-biao-da-shi-pi-pei-lcof/solution/zheng-ze-biao-da-shi-pi-pei-di-gui-qiu-jie-by-jarv/)

语言：java

思路：本想要直接暴力解题，但是自己本地尝试了下，发现很多情况还是没能很好地考虑到。建议直接看上面解析，学习下别人的解题方式.

代码（2ms，100%）：

```java
class Solution {
    public boolean isMatch(String s, String p) {
        int lenS = s.length(),lenP = p.length();
        boolean[][] dp = new boolean[lenS+1][lenP+1];
        char[] charS = s.toCharArray();
        char[] charP = p.toCharArray();
        for(int i = 0;i<=lenS;++i){
            for(int j = 0;j<=lenP;++j){
                // (1)正则串为空,那么只有原字符串也为空才匹配==true
                if(j==0)
                    dp[i][j] = i==0;
                else{
                    // (2)考虑 正则串 遇到 * 这个特殊情况 (遇到.和遇到普通字符a-z分析方式没什么区别)
                    if(charP[j-1]=='*'){
                        // 如果长度少于2，那么*前面没有有效的字符，肯定有问题
                        if(j>=2){
                            // (2-1)*不起作用，也就是*匹配0个字符，比如s是ab,而p是abc*,那么*就直接匹配0个字符
                            dp[i][j] |= dp[i][j-2];
                            // (2-2)*起作用，*匹配1个或者多个字符
                            if(i>0&&(charS[i-1]==charP[j-2]||charP[j-2]=='.'))
                                dp[i][j] |= dp[i-1][j];
                        }

                    }else{
                        // (3)不是*,那匹配就必须是字母相同或者 p正则串是.
                        if(i>0&&(charS[i-1]==charP[j-1]||charP[j-1]=='.'))
                            dp[i][j] = dp[i-1][j-1];
                    }
                }
            }
        }
        return dp[lenS][lenP];
    }
}
```











## 算法与数据结构

### 1. 快速排序-java

思路：

1. 找基准base。
2. 右边往左找 right < base的位置；然后左边向右找 left > base的位置；两者交换数字。直到left和right指针相遇，结束一轮快排。
3. 把基准和当前left的位置互换，然后对left位置的左半部分和右半部分，递归快排。

```java
@Test
public void KuaiPai(){
    int[] arr = {6,7,1,3,8,2,4,9,5};
    DiGui(arr,0 ,arr.length-1);
    for(int item:arr){
        System.out.printf("%d ", item);
    }
}

public void DiGui(int[] arr,int left,int right){
    int base = arr[left];
    int start = left;
    int end = right;
    while(start<end){
        while(arr[end]>= base && end>start)
            --end;
        while(arr[start]<= base&&start<end)
            ++start;
        if(start<end){
            int tmp = arr[start];
            arr[start] = arr[end];
            arr[end] = tmp;
        }
    }
    arr[left] = arr[start];
    arr[start] = base;
    if(start>left)
        DiGui(arr,left,start-1);
    if(start<right)
        DiGui(arr,start+1, right);
}
```

### 1. 快速排序+快速选择(常用于筛选前N个最大or最小)-java

​	后面又写了个快速排序和排序选择（快速选择）明确划分开的版本。（因为有些应用场景只要快速选择，不需要完整的快速排序）。可以根据需求选择快排需要的区间、升降序。

```java
public Test{
    @Test
    public void testSort() {

        // 选择前11小的数字
        int[] arr1 = {10, 10, 10, 9, 9, 9, 8, 8, 8, 7, 7, 7, 6, 6, 6, 5, 5, 5, 4, 4, 4, 3, 3, 3, 2, 2, 2, 1, 1, 1};
        quickTopN(arr1, true, 0, arr1.length - 1, 10);
        System.out.println("选择前11小的数字");
        for (int item : arr1) {
            System.out.printf("%d ", item);
        }
        System.out.println();


        // 选择前11大的数字
        int[] arr2 = {1,1,1,2,2,2,3,3,3,4,4,4,5,5,5,6,6,6,7,7,7,8,8,8,9,9,9,10,10,10};
        quickTopN(arr2, false, 0, arr2.length - 1, 10);
        System.out.println("选择前11大的数字");
        for (int item : arr2) {
            System.out.printf("%d ", item);
        }
        System.out.println();


        // 快排，升序
        int[] arr3 = {1,9,3,4,7,6,2,4,8,1,5,2,6,5,2,4,7,5,1,3,6,9,5,1,3,4,7,1,2,8,3,5,8,2,4,2,1,5,7,6,2,1,4,6,9,2,4,7,1,3,5,8,1,2,6,6,2,1,7,1,2,6,4,8,1,3,5,7,1,2};
        quickSort(arr3, true, 0,arr3.length-1);
        System.out.println("快排，升序");
        for (int item : arr3) {
            System.out.printf("%d ", item);
        }
        System.out.println();

        // 快排，降序
        int[] arr4 = {4,5,7,1,3,5,2,1,9,3,4,1,2,5,8,71,2,1,5,4,1,3,9,5,1,2,5,4,7,2,3,3,1,5,4,8,6,2,4,7,1,6,1,2,5,4,3,9,2,4,2,2,7,6,1,36,5,4,8,2,1,3,5,7,1,54,2,2,81,3,5,7,21,6,5,2,78,2,4,5,9};
        quickSort(arr4, false, 0,arr4.length-1);
        System.out.println("快排，降序");
        for (int item : arr4) {
            System.out.printf("%d ", item);
        }
        System.out.println();


    }

    /**
     * TopN,快速选择(类快排)
     *
     * @param rawArr    原始数组
     * @param isAsc     是否升序，false则降序
     * @param start     需要筛选的闭区间的左边界
     * @param end       需要筛选的闭区间的右边界
     * @param targetEnd 需要得到的有序的闭区间的右边界
     * @apiNote 例如 rawArr=[1,6,4,2,3,5] isAsc=false, start=0 end=5, targetEnd=3,
     * 表示需要 得到长度6的数组rawArr的前4大的数字(0,1,2,3下标是4个数字)
     */
    public int[] quickTopN(int[] rawArr, boolean isAsc, int start, int end, int targetEnd) {
        int basePos = quickSelect(rawArr, isAsc, start, end);
        if (basePos == targetEnd) return rawArr;
        return basePos >= targetEnd ? quickTopN(rawArr, isAsc, start, basePos - 1, targetEnd) : quickTopN(rawArr, isAsc, basePos + 1, end, targetEnd);
    }

    /**
     * 快速排序
     * @param rawArr 需要快排的原始数组
     * @param isAsc 是否升序
     * @param start 需要快排的闭区间的左边界
     * @param end 需要快排的闭区间的右边界
     */
    public void quickSort(int[] rawArr, boolean isAsc,int start,int end) {
        int mid = quickSelect(rawArr, isAsc, start, end);
        if(start<mid)
            quickSort(rawArr,isAsc,start, mid-1);
        if(mid<end)
            quickSort(rawArr,isAsc,mid+1, end);

    }


    /**
     * 快速选择
     * @param rawArr 需要快速选择的原数组
     * @param isAsc 是否升序
     * @param start 需要快速选择的闭区间的左边界
     * @param end 需要快速选择的闭区间的右边界
     * @return 返回基准的下标
     */
    public int quickSelect(int[] rawArr, boolean isAsc, int start, int end) {
        int baseVal = rawArr[start];
        int left = start, right = end;
        while (left < right) {
            if (isAsc) {
                while (left < right && rawArr[right] >= baseVal)
                    --right;
                while (left < right && rawArr[left] <= baseVal)
                    ++left;

            } else {
                while (left < right && rawArr[right] <= baseVal)
                    --right;
                while (left < right && rawArr[left] >= baseVal)
                    ++left;
            }
            if(left<right){
                int tmp = rawArr[left];
                rawArr[left] = rawArr[right];
                rawArr[right] = tmp;
            }
        }
        rawArr[start] = rawArr[left];
        rawArr[left] = baseVal;
        return left;
    }
}
```

参考代码1：

> [最小K个数](https://leetcode-cn.com/problems/smallest-k-lcci/)

```java
class Solution{
  public int[] smallestK(int[] arr, int k) {
    if (k >= arr.length) {
      return arr;
    }

    int low = 0;
    int high = arr.length - 1;
    while (low < high) {
      int pos = partition(arr, low, high);
      if (pos == k - 1) {
        break;
      } else if (pos < k - 1) {
        low = pos + 1;
      } else {
        high = pos - 1;
      }
    }

    int[] dest = new int[k];
    System.arraycopy(arr, 0, dest, 0, k);
    return dest;
  }

  private int partition(int[] arr, int low, int high) {
    int pivot = arr[low];
    while (low < high) {
      while (low < high && arr[high] >= pivot) {
        high--;
      }

      arr[low] = arr[high];
      while (low < high && arr[low] <= pivot) {
        low++;
      }
      arr[high] = arr[low];
    }
    arr[low] = pivot;
    return low;
  } 
}
```

参考代码2：

```java
class Solution {
  public int[] smallestK(int[] arr, int k) {
    // 快排 分堆
    int low=0,hi=arr.length-1;
    while (low<hi){
      int pos=partition(arr,low,hi);
      if(pos==k-1) break;
      else if(pos>k-1) hi=pos-1;
      else low=pos+1;
    }
    int[] dest=new int[k];
    System.arraycopy(arr,0,dest,0,k);
    return dest;
  }
  private int partition(int[] arr,int low,int hi){
    int v=arr[low];
    int i=low,j=hi+1;
    while (true){
      while (arr[++i]<v) if(i==hi) break;
      while (arr[--j]>v) if(j==low) break;
      if(i>=j) break;
      exchange(arr,i,j);
    }
    exchange(arr,low,j);
    return j;
  }
  private void exchange(int[] arr,int i,int j){
    int temp=arr[i];
    arr[i]=arr[j];
    arr[j]=temp;
  }
}
```

### 2. 最小堆、最大堆

#### 2.1 Java内置的PriorityQueue(二叉小顶堆)

```java
public class Test{
    /**
     * 最小堆(Java的PriorityQueue默认最小堆)
     */
    @Test
    public void minStackTest() {
        PriorityQueue<Integer> minStack = new PriorityQueue<>();
        int[] arr = {1, 5, 3, 43, 43, 453, 2, 3, 84, 62, 12};
        for (int i : arr) {
            minStack.add(i);
        }
        int j = 0;
        for (int i : minStack) {
            System.out.printf("%d ", i);
        }

    }
}
// 输出结果如下：
// 1 3 2 5 12 453 3 43 84 62 43
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502233044622.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FzaGlhbWQ=,size_16,color_FFFFFF,t_70)

```java
public class Test{
    /**
     * 最大堆，需要new一个Comparator
     */
    @Test
    public void maxStackTest() {
        PriorityQueue<Integer> maxStack = new PriorityQueue<>(11, new Comparator<Integer>() {
            @Override
            public int compare(Integer o1, Integer o2) {
                return o2-o1;
            }
        });
        int[] arr = {1, 5, 3, 43, 43, 453, 2, 3, 84, 62, 12}; // 11 个数字
        for (int i : arr) {
            maxStack.add(i);
        }
        int j = 0;
        for (int i : maxStack) {
            System.out.printf("%d ", i);
        }
    }
}
// 输出结果如下：
// 453 84 43 43 62 3 2 1 3 5 12
```

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200502233929416.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0FzaGlhbWQ=,size_16,color_FFFFFF,t_70)

#### 2.2 使用PriorityQueue完成Top-N问题

```java
public class Test{
    @Test
    public void testTopMinN(){
        int[] arr = {1,2,5,47,9,5,1,3,2,5,4,7,4,5,6,6,9,8,5,4,1,3,5,7,1,3,6,2,4,1,8,5,41,3,9,5,47,8,52,12,6,5,4,7,1,2,6,32,4846,468,468,848,4,21,5648,461,54,645,78,13546,843,1,4684,31,389,897,65,879613,465,453,};
        // 获取最小的10个数，用最大堆(替换掉原本最小的10个数中最大的)；
        // 获取最大的10个数，用最小堆(替换掉原本最大的10个数中最小的)；
        // 数据结构中，最小/大堆 是树状的，所以替换最顶部的根节点，再判断是否需要改变堆的结构。
        PriorityQueue<Integer> maxStack = new PriorityQueue<>(10, new Comparator<Integer>() {
            @Override
            public int compare(Integer o1, Integer o2) {
                return o2-o1;
            }
        });
        for(int i : arr){
            if(maxStack.size()<10){
                maxStack.add(i);
            }else{
                if(i < maxStack.peek()){
                    maxStack.poll();
                    maxStack.add(i);
                }
            }
        }
        for(int i : maxStack){
            System.out.printf("%d ",i);
        }
    }
}
// 输出结果如下：
// 2 2 2 1 1 1 1 1 1 1 
```

### 3. 归并排序

#### 3.1 归并排序-java

```java
public class MergeSort{
    @Test
    public void testMergeSort() {
        int[] arr = new int[]{15, 435, 43, 543, 1, 24, 46, 12, 1, 234, 84, 68, 1321, 231, 2, 846, 2, 0, 4, 453,
                              2, 14, 8, 453, 123, 48, 468, 453, 132, 48, 3, 15, 48, 23, 132, 45, 46, 2, 18, 7, 123,
                              487, 53, 378, 3, 24, 87, 6, 4, 47, 8, 123, 435, 87, 12, 45, 86, 12, 45, 44, 8, 124, 684, 8653,
                              46, 8, 34, 234, 2321, 4, 8, 533, 453, 7, 5, 4, 11, 3, 8, 71, 2, 6, 5, 7, 7, 9, 4, 1,
                              23, 1, 5, 5, 9, 3, 5, 7, 1, 1, 23, 5, 4, 7, 8, 21, 2, 85, 2, 6, 6, 2, 3, 6, 8, 5, 1,
                              2, 2, 0, 1, 4, 5, 8, 6, 2, 4, 88, 52, 2, 0};
        System.out.println("原数组：" + Arrays.toString(arr));
        mergeSort(arr, 0, arr.length - 1, new int[arr.length]);
        //        sort(arr);
        System.out.println("归并排序后：" + Arrays.toString(arr));
    }

    public void mergeSort(int[] raw, int left, int right, int[] temp) {
        if (left < right) {
            int mid = left + (right - left) / 2;
            mergeSort(raw, left, mid, temp);
            mergeSort(raw, mid + 1, right, temp);
            if(raw[mid]>raw[mid+1])
                merge(raw, left, mid, right, temp);
        }
    }

    public void merge(int[] raw, int left, int mid, int right, int[] temp) {
        int i = left, j = mid + 1, t = 0;
        while (i <= mid && j <= right) {
            if (raw[i] <= raw[j]) {
                temp[t++] = raw[i++];
            } else {
                temp[t++] = raw[j++];
            }
        }
        while (i <= mid) {
            temp[t++] = raw[i++];
        }
        while (j <= right) {
            temp[t++] = raw[j++];
        }
        t = 0;
        while (left <= right) {
            raw[left++] = temp[t++];
        }
    }
}
```

输出结果：

```none
原数组：[15, 435, 43, 543, 1, 24, 46, 12, 1, 234, 84, 68, 1321, 231, 2, 846, 2, 0, 4, 453, 2, 14, 8, 453, 123, 48, 468, 453, 132, 48, 3, 15, 48, 23, 132, 45, 46, 2, 18, 7, 123, 487, 53, 378, 3, 24, 87, 6, 4, 47, 8, 123, 435, 87, 12, 45, 86, 12, 45, 44, 8, 124, 684, 8653, 46, 8, 34, 234, 2321, 4, 8, 533, 453, 7, 5, 4, 11, 3, 8, 71, 2, 6, 5, 7, 7, 9, 4, 1, 23, 1, 5, 5, 9, 3, 5, 7, 1, 1, 23, 5, 4, 7, 8, 21, 2, 85, 2, 6, 6, 2, 3, 6, 8, 5, 1, 2, 2, 0, 1, 4, 5, 8, 6, 2, 4, 88, 52, 2, 0]
归并排序后：[0, 0, 0, 1, 1, 1, 1, 1, 1, 1, 1, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 3, 3, 3, 3, 3, 4, 4, 4, 4, 4, 4, 4, 4, 5, 5, 5, 5, 5, 5, 5, 5, 6, 6, 6, 6, 6, 6, 7, 7, 7, 7, 7, 7, 8, 8, 8, 8, 8, 8, 8, 8, 8, 9, 9, 11, 12, 12, 12, 14, 15, 15, 18, 21, 23, 23, 23, 24, 24, 34, 43, 44, 45, 45, 45, 46, 46, 46, 47, 48, 48, 48, 52, 53, 68, 71, 84, 85, 86, 87, 87, 88, 123, 123, 123, 124, 132, 132, 231, 234, 234, 378, 435, 435, 453, 453, 453, 453, 468, 487, 533, 543, 684, 846, 1321, 2321, 8653]
```

### 4. KMP算法

> 根据B站视频复习KMP算法，简单用Java实现了一下。

```java
public class KMP {


    /**
     * 获取前缀表prefix(方法一)
     * 根据这个B站视频学习后编写 => KMP字符串匹配算法2 https://www.bilibili.com/video/BV1hW411a7ys/
     * @param pattern
     */
    public static int[] prefix_table(String pattern) {
        int n = pattern.length();
        int[] prefix = new int[n];
        // 这里len为匹配到的前后缀相同部分的长度
        int len = 0;
        for (int i = 1; i < n; ) {
            if (pattern.charAt(i) == pattern.charAt(len)) {
                ++len;
                prefix[i] = len;
                ++i;
            } else {
                // 之前已经匹配过一部分后(len > 0)，如果当前不匹配，则回溯上一个匹配时的长度
                // 由于匹配过一部分了，所以当前的len = 之前匹配过的长度+1，取上次匹配长度就是prefix[len-1]
                if (len > 0) {
                    len = prefix[len - 1];
                } else {
                    // 如果连第一个字符都不匹配，那么直接找下一个位置
                    ++i;
                }
            }
        }
        return prefix;
    }

    /**
     * 右移一位前缀表，第一位置-1，方便后续KMP计算
     *
     * @param prefix
     */
    public static void move_prefix_table(int[] prefix) {
        if (prefix.length - 1 >= 0) {
            System.arraycopy(prefix, 0, prefix, 1, prefix.length - 1);
        }
        prefix[0] = -1;
    }

    /**
     * KMP查询子串，返回所有子串的位置(开头匹配的下标位置)
     *
     * @param text
     * @param pattern
     */
    public static List<Integer> kmp_search(String text, String pattern) {
        List<Integer> kmpResList = new ArrayList<>();
        int[] prefix = prefix_table(pattern);
        move_prefix_table(prefix);
        int lenText = text.length();
        int lenPattern = pattern.length();
        for (int i = 0, j = 0; i < lenText; ) {
            if (pattern.charAt(j) == text.charAt(i)) {
                ++i;
                ++j;
                // 匹配到一个完整子串后，再到下一个可能匹配的位置
                // 这里相当于子串在 prefix[lenPattern-1]之前的都是和当前text的i前面一串匹配上了，所以只需要看这往后的字符
                if (j == lenPattern) {
                    kmpResList.add(i - j);
                    j = prefix[lenPattern - 1];
                }
            } else {
                j = prefix[j];
                if (j == -1) {
                    ++i;
                    j = 0;
                }
            }
        }
        return kmpResList;
    }


    /**
     * 获取next数组(方法二)
     * @param pattern
     * 根据B站视频学习后编写 => 帮你把KMP算法学个通透！（求next数组代码篇） https://www.bilibili.com/video/BV1M5411j7Xx/
     */
    public static int[] next(String pattern){
        int n = pattern.length();
        int[] next = new int[n];
        // i指向后缀的最后一个位置;
        // j指向前缀的最后一个位置
        int i = 1,j=0;
        for(;i<n;++i){
            while(j>0 && pattern.charAt(i)!=pattern.charAt(j)){
                // 没有匹配上，则j回溯到上一个位置
                j = next[j-1];
            }
            if(pattern.charAt(i)==pattern.charAt(j)){
                next[i] = ++j;
            }
        }
        return next;
    }

    /**
     * 使用next数组完成KMP算法
     * @param text
     * @param pattern
     * @return
     */
    public static List<Integer> kmp_search2(String text, String pattern){
        List<Integer> resList = new ArrayList<>();
        int[] next = next(pattern);
        int textLen = text.length();
        int patternLen = pattern.length();
        for(int i = 0,j = 0;i<textLen;){
            if(text.charAt(i) == pattern.charAt(j)){
                ++i;
                ++j;
                if(j == patternLen){
                    resList.add(i-j);
                    j = next[patternLen-1];
                }
            }else{
                if(j>0){
                    j = next[j-1];
                }else{
                    ++i;
                }
            }
        }
        return resList;
    }

    public static void main(String[] args) {
        String pattern = "ABABCABAA";
        String text = "ABABABCABAABABABAB";
//        List<Integer> resList = kmp_search(text, pattern);
        List<Integer> resList = kmp_search2(text, pattern);
        for (Integer index : resList) {
            System.out.println(index);
        }
    }
}
```

输出结果

```shell
2
```

