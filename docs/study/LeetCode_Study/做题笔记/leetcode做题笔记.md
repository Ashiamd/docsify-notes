# LeetCode做题笔记

## 739. 每日温度

语言：java

思路：典型的单调栈，一看就是递减栈。如果遇到比栈顶大的数字，就循环出栈，每个出栈数字，计算下标差值。

代码（14ms，87.26%）：

```java
class Solution {
    public int[] dailyTemperatures(int[] T){

        int len =T.length;
        int[] res = new int[len];

        LinkedList<Integer> stack = new LinkedList<>();
        for(int i = 0;i<len;++i){
            while(!stack.isEmpty()&&T[stack.peekFirst()]<T[i]){
                int top = stack.pollFirst();
                res[top] = i - top;
            }
            stack.addFirst(i);
        }
        return res;
    }
}
```

参考代码1（3ms）：从后往前，外层for循环表示给每个res\[i]计算结果值，内层for循环表示寻找比res[i]大的值，运用了类似KMP算法的技巧，快速跳过绝对不可能出现匹配的位置。动态规划DP。

> [评论区](https://leetcode-cn.com/problems/daily-temperatures/comments/)

```java
/**
 * 根据题意，从最后一天推到第一天，这样会简单很多。因为最后一天显然不会再有升高的可能，结果直接为0。
 * 再看倒数第二天的温度，如果比倒数第一天低，那么答案显然为1，如果比倒数第一天高，又因为倒数第一天
 * 对应的结果为0，即表示之后不会再升高，所以倒数第二天的结果也应该为0。
 * 自此我们容易观察出规律，要求出第i天对应的结果，只需要知道第i+1天对应的结果就可以：
 * - 若T[i] < T[i+1]，那么res[i]=1；
 * - 若T[i] > T[i+1]
 *   - res[i+1]=0，那么res[i]=0;
 *   - res[i+1]!=0，那就比较T[i]和T[i+1+res[i+1]]（即将第i天的温度与比第i+1天大的那天的温度进行比较）
 */
class Solution {
    public int[] dailyTemperatures(int[] T) {
        if (T.length == 0)
            return new int[0];
        int[] res = new int[T.length];
        for (int i = res.length - 2; i >= 0; i--) {
            for (int j = i + 1; j < res.length; j += res[j]) {
                if (T[j] > T[i]) {
                    res[i] = j - i;
                    break;
                }
                if (res[j] == 0) {
                    break;
                }
            }
        }
        return res;
    }
}
```

## 71. 简化路径

语言：java

思路：简单粗暴地先将字符串以`/`分割成`List`，再用`Deque`模拟路径切换。

代码（6ms，72.88%）：

```java
class Solution {
  public String simplifyPath(String path) {
    List<String> paths = Arrays.asList(path.split("/"));
    Deque<String> res = new LinkedList<>();
    for(int i = 1;i<paths.size();++i){
      String tmp  = paths.get(i).trim();
      if(tmp.isEmpty()|| ".".equals(tmp)){
        continue;
      }else if("..".equals(tmp)){
        if(!res.isEmpty()) {
          res.removeLast();
        }
      }else{
        res.add(tmp);
      }
    }
    StringBuilder sb = new StringBuilder();
    for(String s:res){
      sb.append("/").append(s);
    }
    String s = sb.toString();
    return s.equals("")?"/":s;
  }
}
```

参考代码1（1ms，100%）：

直接逐字符判断（主要这个对判断条件的编写比较熟练）。

用空间换时间。（再者最开始故意在最后加个`/`，防止最后一次遇到`/..`时没有处理完）

```java
class Solution {
  public String simplifyPath(String path) {
    path += '/';
    char[] chs = path.toCharArray();
    int top = -1;
    for (char c : chs) {
      if (top == -1 || c != '/') {
        chs[++top] = c;
        continue;
      }
      if (chs[top] == '/') {
        continue;
      }
      if (chs[top] == '.' && chs[top - 1] == '/') {
        top--;
        continue;
      }
      if (chs[top] == '.' && chs[top - 1] == '.' && chs[top - 2] == '/') {
        top -= 2;
        while (top > 0 && chs[--top] != '/') ;
        continue;
      }
      chs[++top] = c;
    }
    if (top > 0 && chs[top] == '/') top--;
    return new String(chs, 0, top + 1);
  }
}
```

参考代码2（5ms，91.252%）：和我的思路是一样的。

```java
class Solution {
  public String simplifyPath(String path) {
    String[] paths = path.split("\\/");
    LinkedList<String> stack = new LinkedList<>();

    for(String p:paths){
      if(p.equals(".") || p.equals("")){
        continue;
      } else if(p.equals("..")){
        stack.pollLast();
      } else {
        stack.offer(p);
      }
    }

    StringBuffer sb = new StringBuffer();
    if(stack.isEmpty()){
      return "/";
    }
    for(String s:stack){
      sb.append("/");
      sb.append(s);
    }
    return sb.toString();
  }
}
```

参考代码3（9ms，31.30%）：

> [栈](https://leetcode-cn.com/problems/simplify-path/solution/zhan-by-powcai/)

这个其实和前面的双向队列一个意思。

```java
class Solution {
    public String simplifyPath(String path) {
        Deque<String> stack = new LinkedList<>();
        for (String item : path.split("/")) {
            if (item.equals("..")) {
                if (!stack.isEmpty()) stack.pop();
            } else if (!item.isEmpty() && !item.equals(".")) stack.push(item);
        }
        String res = "";
        for (String d : stack) res = "/" + d + res;
        return res.isEmpty() ? "/" : res;  
    }
}
```

## 93. 复原IP地址

语言：java

思路：超暴力for循环拆分ip，然后判断数值范围是否符合，符合的加入结果集。

代码（11ms，12.41%）：

```java
class Solution {
  public List<String> restoreIpAddresses(String s) {
    List<String> res = new LinkedList<>();
    int len = s.length();
    if (len < 4 || len > 12) {
      return res;
    }

    // a=3,b=6,c=9
    Integer part1;
    Integer part2;
    Integer part3;
    Integer part4;

    for (int a = 1; a <= 3 && a <= len - 3; ++a) {
      part1 = Integer.parseInt(s.substring(0, a));
      if (part1 <= 255&&(a==1||part1>=Math.pow(10,a-1))) {
        for (int b = a+1; b <= a + 3 && b <= len - 2; ++b) {
          part2 = Integer.parseInt(s.substring(a, b));
          if (part2 <= 255&&(b == a+1||part2>=Math.pow(10,b-a-1))) {
            for (int c = b+1; c <= b + 3 && c <= len - 1; ++c) {
              part3 = Integer.parseInt(s.substring(b, c));
              if (part3 <= 255&&(c == b+1||part3>=Math.pow(10,c-b-1))) {
                if (c >= len - 3) {
                  part4 = Integer.parseInt(s.substring(c, len));
                  if (part4 <= 255&&(len == (c+1)||part4>=Math.pow(10,len-c-1))) {
                    res.add(part1 + "." + part2 + "." + part3 + "." + part4);
                  }
                }
              }
            }
          }
        }
      }
    }
    return res;
  }
}
```

参考代码1（1ms）：DFS

> [复原IP地址--官方题解](https://leetcode-cn.com/problems/restore-ip-addresses/solution/fu-yuan-ipdi-zhi-by-leetcode-solution/)

```java
class Solution {
    static final int SEG_COUNT = 4;
    List<String> ans = new ArrayList<String>();
    int[] segments = new int[SEG_COUNT];

    public List<String> restoreIpAddresses(String s) {
        segments = new int[SEG_COUNT];
        dfs(s, 0, 0);
        return ans;
    }

    public void dfs(String s, int segId, int segStart) {
        // 如果找到了 4 段 IP 地址并且遍历完了字符串，那么就是一种答案
        if (segId == SEG_COUNT) {
            if (segStart == s.length()) {
                StringBuffer ipAddr = new StringBuffer();
                for (int i = 0; i < SEG_COUNT; ++i) {
                    ipAddr.append(segments[i]);
                    if (i != SEG_COUNT - 1) {
                        ipAddr.append('.');
                    }
                }
                ans.add(ipAddr.toString());
            }
            return;
        }

        // 如果还没有找到 4 段 IP 地址就已经遍历完了字符串，那么提前回溯
        if (segStart == s.length()) {
            return;
        }

        // 由于不能有前导零，如果当前数字为 0，那么这一段 IP 地址只能为 0
        if (s.charAt(segStart) == '0') {
            segments[segId] = 0;
            dfs(s, segId + 1, segStart + 1);
        }

        // 一般情况，枚举每一种可能性并递归
        int addr = 0;
        for (int segEnd = segStart; segEnd < s.length(); ++segEnd) {
            addr = addr * 10 + (s.charAt(segEnd) - '0');
            if (addr > 0 && addr <= 0xFF) {
                segments[segId] = addr;
                dfs(s, segId + 1, segEnd + 1);
            } else {
                break;
            }
        }
    }
}
```

参考代码2（3ms）：回溯剪枝

> [回溯算法（画图分析剪枝条件）](https://leetcode-cn.com/problems/restore-ip-addresses/solution/hui-su-suan-fa-hua-tu-fen-xi-jian-zhi-tiao-jian-by/)

```java
public class Solution {

  public List<String> restoreIpAddresses(String s) {
    int len = s.length();
    List<String> res = new ArrayList<>();
    if (len > 12 || len < 4) {
      return res;
    }

    Deque<String> path = new ArrayDeque<>(4);
    dfs(s, len, 0, 4, path, res);
    return res;
  }

  // 需要一个变量记录剩余多少段还没被分割

  private void dfs(String s, int len, int begin, int residue, Deque<String> path, List<String> res) {
    if (begin == len) {
      if (residue == 0) {
        res.add(String.join(".", path));
      }
      return;
    }

    for (int i = begin; i < begin + 3; i++) {
      if (i >= len) {
        break;
      }

      if (residue * 3 < len - i) {
        continue;
      }

      if (judgeIpSegment(s, begin, i)) {
        String currentIpSegment = s.substring(begin, i + 1);
        path.addLast(currentIpSegment);

        dfs(s, len, i + 1, residue - 1, path, res);
        path.removeLast();
      }
    }
  }

  private boolean judgeIpSegment(String s, int left, int right) {
    int len = right - left + 1;
    if (len > 1 && s.charAt(left) == '0') {
      return false;
    }

    int res = 0;
    while (left <= right) {
      res = res * 10 + s.charAt(left) - '0';
      left++;
    }

    return res >= 0 && res <= 255;
  }
}
```

参考1后重写（1ms，99.91%）：

惭愧，最近老久没写题了，第一反应这题就是DFS+剪枝，但是没写成，后面就还是老实暴力解法了。

这里DFS的注意点就是

+ 每次DFS判断一个部分（拆分IP为4部分）
+ 某部分开头是0，那么只能这部分为0，直接下一轮DFS。
+ 返回条件
  + 已经4部分（不管是否还有剩余，无剩余字符说明正常则添加到答案中，否则不添加就好了）
  + 已经遍历到字符串尾部（还没有4部分就没得判断了）

```java
class Solution00{

  int[] segments = new int[4];
  List<String> res = new LinkedList<>();

  public List<String> restoreIpAddresses(String s) {
    dfs(s,0,0);
    return res;
  }

  /**
         *
         * @param s 原字符串
         * @param segId 第X部分的IP地址（拆分IP为四部分）
         * @param start 第X部分从下标start开始
         */
  public void dfs(String s,int segId,int start){
    // 总共就4部分 0,1,2,3。
    if(segId==4){
      if(start==s.length()){
        StringBuilder sb = new StringBuilder();
        sb.append(segments[0]);
        for(int i = 1 ;i<4;++i){
          sb.append(".").append(segments[i]);
        }
        res.add(sb.toString());
      }
      return ;
    }

    // 提前遍历完（不足4部分）
    if(start == s.length()){
      return;
    }

    // 首位0，那么只能是0，不允许 "023 => 23"的形式
    if(s.charAt(start)=='0'){
      segments[segId] = 0;
      dfs(s,segId+1, start+1);
      return ;
    }

    for(int i= start,sum = 0;i<s.length();++i){
      sum *=10;
      sum += s.charAt(i) - '0';
      if(sum <=255){
        segments[segId] = sum;
        dfs(s,segId+1,i+1);
      }else{
        break;
      }
    }
  }
}
```

## 695. 岛屿的最大面积

语言：java

思路：常见的岛屿问题。这里算面积，就把面积变量当作静态成员变量，然后其他代码和常见的岛屿问题一致。

代码（2ms，100%）：

```java
class Solution {
  int area = 0;

  public int maxAreaOfIsland(int[][] grid) {
    int res = 0;
    for (int i = 0, tmp; i < grid.length; ++i) {
      for (int j = 0; j < grid[0].length; ++j) {
        if (grid[i][j] == 1) {
          dfs(grid, i, j);
          res = Math.max(res, area);
          area = 0;
        }
      }
    }
    return res;
  }


  public void dfs(int[][] grid, int x, int y) {
    if(x<0||x>=grid.length||y<0||y>=grid[0].length||grid[x][y]!=1){
      return ;
    }
    grid[x][y] = 0;
    ++area;
    dfs(grid,x-1,y); // 上
    dfs(grid,x+1,y); // 下
    dfs(grid,x,y-1); // 左
    dfs(grid,x,y+1); // 右
  }
}
```

## 75. 颜色分类

语言：java

思路：最右边都是2，左边要么是0，要么是1。

+ 数字2是最无歧义的，所以表示数字2所在的数组下标的指针用`twoIdx`表示，且初值为`nums.length-1`，即最后一个元素的位置。（就算没有2也无所谓，反正一定会遍历到最后一个位置）,另外数字0和数字1下标从0开始，这两个数字不确定有谁，都是从第一个元素开始遍历。
+ 最外层循环条件，即表示数字0或者数字1的指针下标`zeroIdx`和`oneIdx`要小于`twoIdx`。这个没啥歧义，不管到底输入数组中有没有数字2，`twoIdx`反正充当右边界一般的存在。
+ 我们主要操作的指针就是`oneIdx`,这个正好夹在0和2之间的数字1的下标。（个人感觉方便判断）
+ 循环内，注意先判断`nums[oneIdx]>1`的情况，后考虑`nums[oneIdx]<1`的情况。
  + `nums[oneIdx]>1`时，和`nums[twoIdx]`对调后，此时`nums[oneIdx]`的数字是`<=2`的，就算还是2，之后下一轮循环还是可以替换。（而且`oneIdx`和`twoIdx`两个指针一个从左到右，一个从右到左，互不影响）
  + `nuns[oneIdx]<1`时，同理和`nums[zeroIdx]`对调，由于`zeroIdx`和`oneIdx`都是从左往右，之后需要考虑怎么移动`oneIdx`。
+ 题目要求"000...11..2222"这种形式，那么`oneIdx`应该尽可能让他一直指向数字1所在的位置。所以循环中每轮当`nums[oneIdx]<=1`，就`++oneIdx`，如果原本`nums[oneIdx]==0`，那么也在前面的判断时和`nums[zeroIdx]`对调了。

代码（0ms，100%）：

```java
class Solution {
  public void sortColors(int[] nums) {
    int zeroIdx = 0,oneIdx = 0,twoIdx = nums.length-1;
    while(zeroIdx<=twoIdx&&oneIdx<=twoIdx){
      if(nums[oneIdx]>1){
        nums[oneIdx] = nums[twoIdx];
        nums[twoIdx--]=2;
      }
      if(nums[oneIdx]<1){
        nums[oneIdx] = nums[zeroIdx];
        nums[zeroIdx++] = 0;
      }
      if(nums[oneIdx]<=1){
        ++oneIdx;
      }
    }
  }
}
```

## 面试题 17.14. 最小K个数

> [面试题 17.14. 最小K个数](https://leetcode-cn.com/problems/smallest-k-lcci/)

语言：java

思路：最简单的就是堆排序（因为java有现成的PriorityQueue），其次就是需要手写的快速选择。

代码1（38ms，11%）：堆排序

```java
class Solution {
  public int[] smallestK(int[] arr, int k) {
    PriorityQueue<Integer> maxStack = new PriorityQueue<>((x, y) -> y - x);
    for (int num : arr) {
      if (maxStack.size() < k) {
        maxStack.add(num);
      } else if (maxStack.size()>0&&maxStack.peek() > num) {
        maxStack.poll();
        maxStack.add(num);
      }
    }
    int[] res = new int[k];
    for (int i = 0; i < k; ++i) {
      res[i] = maxStack.poll();
    }
    return res;
  }
}
```

代码2（2ms，99.277%）：快速选择

```java
class Solution {
  public int[] smallestK(int[] arr, int k) {
    int left = 0, right = arr.length - 1;
    while (left<right) {
      int pos = quickSelect(arr, left, right);
      if (pos == k - 1) {
        break;
      } else if (pos > k - 1) {
        right = pos - 1;
      } else {
        left = pos + 1;
      }
    }
    int[] res =  new int[k];
    System.arraycopy(arr, 0, res, 0, k);
    return res;
  }

  public int quickSelect(int[] arr, int left, int right) {
    int pivot = arr[left];
    int start = left;
    while (true) {
      while (left < right && arr[right] >= pivot) {
        --right;
      }
      while (left < right && arr[left] <= pivot) {
        ++left;
      }
      if (left >= right) {
        break;
      }
      exchange(arr, left, right);
    }
    exchange(arr, start, left);
    return left;
  }

  public void exchange(int[] arr, int a, int b) {
    int tmp = arr[a];
    arr[a] = arr[b];
    arr[b] = tmp;
  }
}
```

参考代码1（1ms，100%）：

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

## 518. 零钱兑换 II

> [518. 零钱兑换 II](https://leetcode-cn.com/problems/coin-change-2/)

语言：java

思路：动态规划。大的零钱兑换拆分成小的零钱兑换。

代码（3ms，82.22%）：

```java
class Solution {
  public int change(int amount, int[] coins) {
    int[] dp = new int[amount+1];
    dp[0] = 1;
    for(int coin:coins){
      for(int i = coin;i<=amount;++i){
        dp[i] += dp[i-coin];
      }
    }
    return dp[amount];
  }
}
```

参考代码1（2ms，100%）：看着是一样的，但是莫名快1ms？

```java
class Solution {
  public int change(int amount, int[] coins) {
    int[] dp = new int[amount + 1];
    dp[0] = 1;
    for (int coin : coins) {
      for (int i = coin; i <= amount; i++) {
        dp[i] += dp[i - coin];
      }
    }
    return dp[amount];
  }
}
```

## 416. 分割等和子集

> [416. 分割等和子集](https://leetcode-cn.com/problems/partition-equal-subset-sum/)
>
> [分割等和子集--官方题解](https://leetcode-cn.com/problems/partition-equal-subset-sum/solution/fen-ge-deng-he-zi-ji-by-leetcode-solution/)
>
> [动态规划（转换为 0-1 背包问题）](https://leetcode-cn.com/problems/partition-equal-subset-sum/solution/0-1-bei-bao-wen-ti-xiang-jie-zhen-dui-ben-ti-de-yo/)

语言：java

思路：这个动态规划不是很好想，建议直接看官方题解。

代码（88ms，5.00%）：慢到极致

```java
class Solution {
  public boolean canPartition(int[] nums) {

    // (1) 数组长度<2，那么不可能拆分成两个非空数组
    if (nums.length < 2) {
      return false;
    }

    // (2) 计算数组和
    int sum = 0, max = 0;
    for (int num : nums) {
      sum += num;
      if (max < num) {
        max = num;
      }
    }
    // (3) 数组和为奇数，不可能拆分成两个等和数组
    if (sum%2 == 1) {
      return false;
    }

    // (4) 等下寻找数组和为一半的其中一个数组就好了 sum = sum / 2;
    sum /= 2;

    // (5) 如果 max大于 总和的一半，说明不可能拆分数组
    if (max > sum) {
      return false;
    }

    // (6) 动态规划， i 属于 [0 ～ length) , j 属于[0,sum]，dp[i][j]表示从[0，i]中拿任意个数字，且和为j
    boolean[][] dp = new boolean[nums.length][sum + 1];
    for (boolean[] bool : dp) {
      bool[0] = true; // dp[i][0] = true
    }
    dp[0][nums[0]] = true;

    // j >= num[i]时, dp[i][j] = dp[i-1][j] | dp[i][j-num[i]];
    // j < num[i] 时, dp[i][j] = dp[i-1][j]
    for (int i = 1; i < nums.length; ++i) {
      for (int j = 1; j <= sum; ++j) {
        if (j >= nums[i]) {
          dp[i][j] = dp[i - 1][j] | dp[i-1][j - nums[i]];
        } else {
          dp[i][j] = dp[i - 1][j];
        }
      }
    }
    return dp[nums.length-1][sum];
  }
}
```

参考代码1（47ms，24.31%）：

> [分割等和子集--官方题解](https://leetcode-cn.com/problems/partition-equal-subset-sum/solution/fen-ge-deng-he-zi-ji-by-leetcode-solution/)

```java
class Solution {
  public boolean canPartition(int[] nums) {
    int n = nums.length;
    if (n < 2) {
      return false;
    }
    int sum = 0, maxNum = 0;
    for (int num : nums) {
      sum += num;
      maxNum = Math.max(maxNum, num);
    }
    if (sum % 2 != 0) {
      return false;
    }
    int target = sum / 2;
    if (maxNum > target) {
      return false;
    }
    boolean[][] dp = new boolean[n][target + 1];
    for (int i = 0; i < n; i++) {
      dp[i][0] = true;
    }
    dp[0][nums[0]] = true;
    for (int i = 1; i < n; i++) {
      int num = nums[i];
      for (int j = 1; j <= target; j++) {
        if (j >= num) {
          dp[i][j] = dp[i - 1][j] | dp[i - 1][j - num];
        } else {
          dp[i][j] = dp[i - 1][j];
        }
      }
    }
    return dp[n - 1][target];
  }
}
```

参考代码2（21ms，69.797%）：优化空间复杂度

> [分割等和子集--官方题解](https://leetcode-cn.com/problems/partition-equal-subset-sum/solution/fen-ge-deng-he-zi-ji-by-leetcode-solution/)

```java
class Solution {
  public boolean canPartition(int[] nums) {
    int n = nums.length;
    if (n < 2) {
      return false;
    }
    int sum = 0, maxNum = 0;
    for (int num : nums) {
      sum += num;
      maxNum = Math.max(maxNum, num);
    }
    if (sum % 2 != 0) {
      return false;
    }
    int target = sum / 2;
    if (maxNum > target) {
      return false;
    }
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;
    for (int i = 0; i < n; i++) {
      int num = nums[i];
      for (int j = target; j >= num; --j) {
        dp[j] |= dp[j - num];
      }
    }
    return dp[target];
  }
}
```

参考2后重写（23ms，65.83%）：

```java
public class Solution {

  public boolean canPartition(int[] nums) {

    // (1) 数组长度<2，那么不可能拆分成两个非空数组
    if (nums.length < 2) {
      return false;
    }

    // (2) 计算数组和
    int sum = 0, max = 0;
    for (int num : nums) {
      sum += num;
      if (max < num) {
        max = num;
      }
    }
    // (3) 数组和为奇数，不可能拆分成两个等和数组
    if ((sum & 1) == 1) {
      return false;
    }

    // (4) 等下寻找数组和为一半的其中一个数组就好了 sum = sum / 2;
    sum /= 2;

    // (5) 如果 max大于 总和的一半，说明不可能拆分数组
    if (max > sum) {
      return false;
    }

    // (6) 动态规划, j 属于[0,sum]，dp[j]表示从[0，i]中拿任意个数字，且和为j
    boolean[] dp = new boolean[sum + 1];
    dp[0] = true;

    // j == sum , return true
    // j < num[i] 时, dp[i][j] = dp[i-1][j]
    for (int i = 0; i < nums.length; ++i) {
      for (int j = sum; j >= nums[i]; --j) {
        dp[j] |= dp[j - nums[i]];
      }
    }
    return dp[sum];
  }
}
```

## 474. 一和零

> [474. 一和零](https://leetcode-cn.com/problems/ones-and-zeroes/)

语言：java

思路：参考该文章[动态规划（转换为 0-1 背包问题）](https://leetcode-cn.com/problems/partition-equal-subset-sum/solution/0-1-bei-bao-wen-ti-xiang-jie-zhen-dui-ben-ti-de-yo/)后，没想到一次写成。

类似0-1背包问题，这里把字符"0"和字符"1"当作消耗品，然后用来购买`strs`字符串。

状态转移方程：

`dp[j][k] = Math.max(dp[j][k],dp[j-strs[i].zeroCount][k-strs[i].oneCount]+1)`，其中j表示字符0的库存，k表示字符1的库存。这里逆序遍历j和k，因为j和k是消耗品，分别原库存是m和n。

`dp[j][k]`表示字符0和1的库存分别为j和k的情况下最多能换取的字符串数量。

代码（31ms，99.32%）：

```java
class Solution {
  public int findMaxForm(String[] strs, int m, int n) {
    int[][] dp = new int[m + 1][n + 1];
    //        int max = 0;
    //dp[i][j] = Math.max(dp[i-strs[i].zero][j-str[i].one]+1,dp[i][j]);
    for (int i = 0; i < strs.length; ++i) {
      int zeroCount = zeroCount(strs, i);
      int oneCount = oneCount(strs, i);
      for (int j = m; j >= zeroCount; --j) {
        for (int k = n; k >= oneCount; --k) {
          dp[j][k] = Math.max(dp[j][k], dp[j-zeroCount][k-oneCount]+1);
        }
      }
    }
    return dp[m][n];
  }

  public int zeroCount(String[] strs, int index) {
    int count = 0;
    for (int i = 0; i < strs[index].length(); ++i) {
      if (strs[index].charAt(i) == '0') {
        ++count;
      }
    }
    return count;
  }

  public int oneCount(String[] strs, int index) {
    int count = 0;
    for (int i = 0; i < strs[index].length(); ++i) {
      if (strs[index].charAt(i) == '1') {
        ++count;
      }
    }
    return count;
  }
}
```

参考代码1（31ms，99.32%）：思路一样，就是记录0和1的数量的逻辑简化了。

```java
class Solution {
  public int findMaxForm(String[] strs, int m, int n) {
    int[][] dp = new int[m + 1][n + 1];
    int len = strs.length;
    int[][] matrix = new int[len][2];
    for(int i = 0; i < len; i++){
      String str = strs[i];
      for(int j = 0; j < str.length(); j++){
        if(str.charAt(j) == '0') matrix[i][0]++; 
        else matrix[i][1]++;
      }
      int zero = matrix[i][0];
      int one = matrix[i][1];
      for(int x = m; x >= zero; x--){
        for(int y = n; y >= one; y--){
          dp[x][y] = Math.max(dp[x][y], 1 + dp[x - zero][y - one]);
        }
      }
    }
    return dp[m][n];
  }
}
```

## 530. 二叉搜索树的最小绝对差

> [530. 二叉搜索树的最小绝对差](https://leetcode-cn.com/problems/minimum-absolute-difference-in-bst/)

语言：java

思路：先DFS前序遍历，用最小堆存储所有节点，然后再逐一计算差值。

代码（7ms，6.71%）：慢到极致

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

  int res = Integer.MAX_VALUE;
  PriorityQueue<Integer> queue = new PriorityQueue<>();

  public int getMinimumDifference(TreeNode root) {
    dfs(root);
    int first = queue.poll(),second;
    while(!queue.isEmpty()){
      second = queue.poll();
      res = Math.min(res,Math.abs(first-second));
      first = second;
    }
    return res;
  }


  public void dfs(TreeNode cur) {
    if (cur == null) {
      return;
    }
    queue.add(cur.val);
    dfs(cur.left);
    dfs(cur.right);
  }

}
```

参考代码1（0ms）：直接中序遍历，边计算差值。（这里我才反应起来，原来这个题目是二叉搜索树，那么中序遍历保证数值从小到大排序，这样只要遍历过程中计算差值即可）

```java
class Solution {
  int ans = Integer.MAX_VALUE, prev = -1;
  public int getMinimumDifference(TreeNode root) {
    getMinimumDifference0(root);
    return ans;
  }
  private void getMinimumDifference0(TreeNode node) {
    if (node != null) {
      getMinimumDifference0(node.left);
      if (prev != -1) ans = Math.min(ans, node.val - prev);
      prev = node.val;
      getMinimumDifference0(node.right);
    }
  }
}
```

## 977. 有序数组的平方

> [977. 有序数组的平方](https://leetcode-cn.com/problems/squares-of-a-sorted-array/)

语言：java

思路：偷懒的直接计算平方，然后调用库函数快排。

代码（4ms，15.68%）：

```java
class Solution {
  public int[] sortedSquares(int[] A) {
    int len = A.length, head = 0, tail = len - 1, index = 0;
    int[] res = new int[len];
    for (int i = 0; i < len; ++i) {
      res[i] = A[i] * A[i];
    }
    Arrays.sort(res);
    return res;
  }
}
```

参考代码1（1ms，100%）：双指针，主要需要注意的是从尾部开始填充。因为题目保证非递减，所以从可能是最大值的两个边界同时向中间判断。比起所有遍历过的数字的最小值，最大值可以确定，所以从数组最后一个数字开始往前填充。

```java
class Solution {
  public int[] sortedSquares(int[] A) {
    int start = 0;
    int end = A.length-1;
    int i = end;
    int[] B = new int[A.length];
    while(i >= 0){
      B[i--] = A[start]*A[start] >= A[end]*A[end]? A[start]*A[start++]:A[end]*A[end--];
    }
    return B;
  }
}
```

代码2（1ms，100%）：两边双指针，找平方后比较大的数字，往新数组的最右边存储。

```java
class Solution {
  public int[] sortedSquares(int[] nums) {
    int left = 0,right = nums.length -1,newNumsRight = right;
    int[] newNums = new int[nums.length];
    while(left <= right) {
      int leftNum = nums[left] * nums[left];
      int rightNum = nums[right] * nums[right];
      if(rightNum >= leftNum) {
        newNums[newNumsRight--] = rightNum;
        --right;
      } else {
        newNums[newNumsRight--] = leftNum;
        ++left;
      }
    }
    return newNums;
  }
}
```

## 52. N皇后 II

>[52. N皇后 II](https://leetcode-cn.com/problems/n-queens-ii/)

语言：java

思路：DFS。原本我写了一个，让count计数器为static的时候，后台会误判！！！！这个我踩坑了。

代码（2ms，56.40%）：

```java
class Solution {
  
  int count = 0;

  public int totalNQueens(int n) {
    // (1) 地图 map[x] = y;
    int[] map = new int[n];

    // (2) 第一行每个位置都试一遍。
    for(int col = 0;col<n;++col){
      map[0] = col;
      dfs(map,n,1);
    }

    return count;
  }

  public void dfs(int[] map, int n, int row) {
    // 走到边界，return
    if (row == n) {
      ++count;
      return;
    }
    for(int col = 0;col<n;++col){
      map[row] = col;
      if(canSet(map,row,col)){
        dfs(map,n,row+1);
      }
    }
  }

  public boolean canSet(int[] map,int row,int col){

    for(int i = 0;i<row;++i){
      // 竖直方向 判断
      if(map[i]==col){
        return false;
      }
      // 撇方向 判断
      if( i + map[i] ==row+col){
        return false;
      }
      // 捺方向 判断
      if(i - map[i] == row-col){
        return false;
      }
    }
    return true;
  }
}
```

参考代码1（0ms）：

> [N皇后 II--官方题解](https://leetcode-cn.com/problems/n-queens-ii/solution/nhuang-hou-ii-by-leetcode-solution/)

```java
class Solution {
  public int totalNQueens(int n) {
    return solve(n, 0, 0, 0, 0);
  }

  public int solve(int n, int row, int columns, int diagonals1, int diagonals2) {
    if (row == n) {
      return 1;
    } else {
      int count = 0;
      int availablePositions = ((1 << n) - 1) & (~(columns | diagonals1 | diagonals2));
      while (availablePositions != 0) {
        int position = availablePositions & (-availablePositions);
        availablePositions = availablePositions & (availablePositions - 1);
        count += solve(n, row + 1, columns | position, (diagonals1 | position) << 1, (diagonals2 | position) >> 1);
      }
      return count;
    }
  }
}
```

参考代码2（2ms，56.40%）：我原本代码和这个差不多，就判断冲突的方法写的形式略不同

```java
class Solution {
  int n;
  int[] res; //记录每种方案的皇后放置索引
  int count = 0; //总方案数
  public int totalNQueens(int n) {
    this.n = n;
    this.res = new int[n];
    check(0); // 第0行开始放置
    return count;
  }
  //放置第k行
  public void check(int k) {
    if(k == n) {
      count++;
      return;
    }
    for(int i = 0; i < n; i++) {
      res[k] = i;  // 将位置i 放入索引数组第k个位置
      if(!judge(k)) {
        check(k+1); //不冲突的话，回溯放置下一行
      }
      //冲突的话试下一个位置
    }
  }
  //判断第k行的放置是否与之前位置冲突
  public boolean judge(int k) {
    for(int i = 0; i < k; i++) {
      if(res[k] == res[i] || Math.abs(k-i) == Math.abs(res[k]-res[i])) {
        return true;
      }
    }
    return false;
  }
}
```

## 844. 比较含退格的字符串

> [844. 比较含退格的字符串](https://leetcode-cn.com/problems/backspace-string-compare/)

语言：java

思路：双栈，先添加，后比较。

代码（2ms，74.87%）：

```java
class Solution {
  public boolean backspaceCompare(String S, String T) {
    Deque<Character> sDeque = new LinkedList<>();
    Deque<Character> tDeque = new LinkedList<>();

    for(char c : S.toCharArray()){
      if(c=='#'){
        if(!sDeque.isEmpty()){
          sDeque.pollFirst();
        }
      }else{
        sDeque.addFirst(c);
      }
    } 

    for(char c : T.toCharArray()){
      if(c=='#'){
        if(!tDeque.isEmpty()){
          tDeque.pollFirst();
        }
      }else{
        tDeque.addFirst(c);
      }
    }

    while(!sDeque.isEmpty() && !tDeque.isEmpty()){
      if(!sDeque.pollFirst().equals(tDeque.pollFirst())){
        return false;
      }
    }
    return sDeque.isEmpty()&&tDeque.isEmpty();
  }
}
```

参考代码1（0ms）：用双指针，模拟栈操作。

```java
class Solution {
  public boolean backspaceCompare(String S, String T) {

    int s = S.length() - 1;
    int t = T.length() - 1;

    int sBack = 0;
    int tBack = 0;

    while (s >= 0 && t >= 0) {
      while (s >= 0) {
        if (S.charAt(s) == '#') {
          sBack++;
          s--;
        } else {
          if (sBack == 0) {
            break;
          }
          sBack--;
          s--;
        }
      }
      while (t >= 0) {
        if (T.charAt(t) == '#') {
          tBack++;
          t--;
        } else {
          if (tBack == 0) {
            break;
          }
          tBack--;
          t--;
        }
      }

      //都到了真实字符
      if (s >= 0 && t >= 0 ) {
        if (S.charAt(s) != T.charAt(t)) {
          return false;
        }
        s--;
        t--;
      }

    }
    //对于剩余的字符串，因为全部退格后可能为空字符串，所以继续处理
    while (s >= 0) {
      if (S.charAt(s) == '#') {
        sBack++;
        s--;
      } else {
        if (sBack == 0) {
          break;
        }
        sBack--;
        s--;
      }
    }
    while (t >= 0) {
      if (T.charAt(t) == '#') {
        tBack++;
        t--;
      } else {
        if (tBack == 0) {
          break;
        }
        tBack--;
        t--;
      }
    }
    //都到了末尾
    if (s < 0 && t < 0) {
      return true;
    }
    //只有一个到了末尾
    return false;

  }
}
```

代码2（0ms，100%）：两个指针遍历两个字符串。两个字符串都先根据规则遇到'#'从后往前删除字符，直到某一个位置起是有效的字符时再进行比较。

```java
class Solution {
  public boolean backspaceCompare(String s, String t) {
    int i = s.length()-1, j = t.length()-1;
    int sFlag = 0, tFlag = 0;
    while(i>= 0 || j >= 0) {
      //根据 # 消除 s的 字符
      while(i>=0) {
        if(s.charAt(i)== '#') {
          --i;
          ++sFlag;
        } else if (sFlag > 0) {
          --i;
          --sFlag;
        } else {
          break;
        }
      } 
      //根据 # 消除 t的 字符
      while(j>=0) {
        if(t.charAt(j)== '#') {
          --j;
          ++tFlag;
        } else if (tFlag > 0) {
          --j;
          --tFlag;
        } else {
          break;
        }
      }
      // 消除后，如果不相等，返回false.相等则 同时跳过一个字符
      if(i>=0&&j>=0 ) {
        if(s.charAt(i) != t.charAt(j)) {
          return false;
        }
        --i;
        --j;
        continue;
      }
      // 如果 最后有一个没遍历完，说明还有剩余字符（比另一个字符串多字符）
      if(i!=j){
        return false;
      }
    }
    return true;
  }
}
```

## 143. 重排链表

> [143. 重排链表](https://leetcode-cn.com/problems/reorder-list/)

语言：java

思路：没想到比较巧的方法，根据官方题解二，说实际就是链表前半段和倒序后的链表后半段组合。这里尝试下。

代码（2ms，79.06%）：

```java
class Solution {
  public void reorderList(ListNode head) {
    if(head==null){
      return;
    }
    // (1) 获取链表中点
    ListNode mid = mid(head);
    // (2) 颠倒后半段链表
    ListNode second = mid.next;
    mid.next = null;
    second = reverse(second);
    // (3) 组合前半段和倒序的后半段
    merge(head,second);
  }

  public ListNode mid(ListNode head) {
    ListNode slow = head,fast = head;
    while (fast.next != null && fast.next.next != null) {
      slow = slow.next;
      fast = fast.next.next;
    }
    return slow;
  }

  public ListNode reverse(ListNode head) {
    ListNode pre = null,cur = head,next;
    while (cur != null) {
      next = cur.next;
      cur.next = pre;
      pre = cur;
      cur = next;
    }
    return pre;
  }

  public void merge(ListNode first, ListNode second) {
    ListNode first_next,second_next;
    while (first != null && second != null) {
      first_next = first.next;
      second_next = second.next;

      first.next = second;
      first = first_next;

      second.next = first;
      second = second_next;
    }
  }
}
```

参考代码1（1ms，100%）：思路一样，写法略有差别

```java
class Solution {
  public void reorderList(ListNode head) {

    //找中点
    ListNode slow = head;
    ListNode lastSlow = head;
    ListNode quick = head;
    //特殊情况
    if(head == null || head.next == null)
      return;
    while(quick != null){
      if(quick.next == null)
        quick = quick.next;
      else
        quick = quick.next.next;

      lastSlow = slow;
      slow = slow.next;
    }
    //将链表分成两半，前斩断前段
    lastSlow.next = null;
    //将后半段链表反转
    ListNode lastNode = null;
    while(slow != null){
      ListNode nexrTemp = slow.next;
      slow.next = lastNode;
      //移动
      lastNode = slow;
      slow = nexrTemp;
    }

    ListNode head2 = lastNode;
    //将两分段链表拼接成一个
    ListNode dummy = new ListNode(Integer.MAX_VALUE);//哑节点
    ListNode cur = dummy;
    int count = 0;//插入节点计数
    while (head != null && head2 != null){
      count++;
      if(count % 2 == 1){ //奇数
        cur.next = head;
        head = head.next;
        cur = cur.next;
      }else { //偶数
        cur.next = head2;
        head2 = head2.next;
        cur = cur.next;
      }
    }
    //拼接剩余
    while(head != null){
      cur.next = head;
      head = head.next;
      cur = cur.next;
    }
    while(head2 != null){
      cur.next = head2;
      head2 = head2.next;
      cur = cur.next;
    }

    head = dummy.next;
  }
}
```

## 925. 长按键入

语言：java

思路：两个字符串同时遍历，比较；党遍历到的位置字符不同时，考虑typed是否当前字符和之前的是重复的，是则认为是不小心重复输入了，跳过所有重复的字符。

代码（1ms，86.83%）：

```java
class Solution {
  public boolean isLongPressedName(String name, String typed) {
    int i = 0, j = 0;
    while (j < typed.length()) {
      if (i < name.length() && name.charAt(i) == typed.charAt(j)) {
        i++;
        j++;
      } else if (j > 0 && typed.charAt(j) == typed.charAt(j - 1)) {
        j++;
      } else {
        return false;
      }
    }
    return i == name.length();
  }
}
```

参考代码1（0ms）：和我原本的写法类似，思路是一样的。

```java
class Solution {
  public boolean isLongPressedName(String name, String typed) {
    char[] ch1 = name.toCharArray();
    char[] ch2 = typed.toCharArray();
    if(ch1.length > ch2.length) return false;
    int i = 0, j = 0;
    while(i < ch1.length && j < ch2.length){
      if(ch1[i] == ch2[j]){
        i++;
        j++;
      }else if(j > 0 && ch2[j - 1] == ch2[j]){
        j++;
      }else{
        return false;
      }
    }
    while(j < ch2.length){
      if(ch2[j] != ch2[j - 1]){
        return false;
      }
      j++;
    }
    return i == ch1.length;
  }
}
```

## 763. 划分字母区间

> [763. 划分字母区间](https://leetcode-cn.com/problems/partition-labels/)

语言：java

思路：滑动窗口类题目。按照每个字母首次出现的位置进行排序，然后判断交集；无交集直接添加上一个到结果集中，有交集则修改滑动窗口右边界，继续往下判断。

代码（8ms，34.35%）：就是效率比较低，但是这个写法还是比较好理解的。

```java
class Solution {
  public List<Integer> partitionLabels(String S) {

    // 初始化数组，最多26字母
    PartNode[] characters = new PartNode[26];
    for (int i = 0; i < 26; ++i) {
      characters[i] = new PartNode();
    }
    for (int i = 0, len = S.length(), pos; i < len; ++i) {
      pos = S.charAt(i) - 'a';
      if (characters[pos].start == Integer.MAX_VALUE) {
        characters[pos].start = i;
      }
      characters[pos].end = i;
    }

    // 根据 第一次出现的位置 从小到大 递增排序
    Arrays.sort(characters, (x, y) -> x.start - y.start);

    List<Integer> res = new LinkedList<>();

    //a b c d e f
    int start = 0, end = 0;
    for (int i = 0; i < 26; ++i) {

      // (1) 当前字母所在字符串和前面的 无重叠
      if (characters[i].start > end) {
        // 最后的字母 ( 没出现过的字母，绝对排在最后 )
        if (characters[i].start == Integer.MAX_VALUE) {
          res.add(end - start + 1);
          start = Integer.MAX_VALUE;
          break;
        }

        // 添加上一个字母所在字符串的长度，并修改下次字符串的起点、终点
        res.add(end - start + 1);
        start = characters[i].start;
        end = characters[i].end;
      } else{
        // (2) 重叠,继续向下判断是否重叠
        end = Math.max(characters[i].end,end);
      }

    }
    // 最后一个出现的字母的所在字符串没被加入到res中
    if (start != Integer.MAX_VALUE) {
      res.add(end - start + 1);
    }

    return res;
  }


  /**
     * 存储字母 第一次出现 和 最后一次出现 的位置。
     */
  class PartNode {
    public Integer start;
    public Integer end;

    public PartNode() {
      start = Integer.MAX_VALUE;
      end = Integer.MAX_VALUE;
    }
  }
}
```

参考代码1（3ms，96.91%）：贪心算法 + 双指针。

这个思路也很清晰，就是只要存储每个字母最后出现的位置，然后重新遍历字符串，维护两个位置变量（可理解为窗口）：`start`、`end`，`end = Math.max(Math.max(last, lasts[chs[right] - 'a'))`表示每次都让右边界尽量大（这样子就直接考虑了字符串重叠的情况），当走出重叠区时`i == end`，直接把窗口的长度加入结果集，然后又更新窗口的左边界`start = end+1`

> [划分字母区间--官方题解](https://leetcode-cn.com/problems/partition-labels/solution/hua-fen-zi-mu-qu-jian-by-leetcode-solution/)
>
> 上述做法使用贪心的思想寻找每个片段可能的最小结束下标，因此可以保证每个片段的长度一定是符合要求的最短长度，如果取更短的片段，则一定会出现同一个字母出现在多个片段中的情况。由于每次取的片段都是符合要求的最短的片段，因此得到的片段数也是最多的。
>

```java
class Solution {
  public List<Integer> partitionLabels(String S) {
    int[] last = new int[26];
    int length = S.length();
    for (int i = 0; i < length; i++) {
      last[S.charAt(i) - 'a'] = i;
    }
    List<Integer> partition = new ArrayList<Integer>();
    int start = 0, end = 0;
    for (int i = 0; i < length; i++) {
      end = Math.max(end, last[S.charAt(i) - 'a']);
      if (i == end) {
        partition.add(end - start + 1);
        start = end + 1;
      }
    }
    return partition;
  }
}
```

参考代码2（2ms，100%）：和官方题解大同小异，就是写法略不同而已。

```java
class Solution {
  public List<Integer> partitionLabels(String S) {
    List<Integer> ans = new ArrayList<>();

    char[] chs = S.toCharArray();
    int[] lasts = new int[26];
    for (int i = 0; i < chs.length; i++) {
      lasts[chs[i] - 'a'] = i;
    }

    int last = 0;
    int right = 0;
    do {
      int left = right;

      do {
        last = Math.max(last, lasts[chs[right] - 'a']);
        right++;
      } while (right <= last);

      ans.add(right - left);
    } while (right < chs.length);

    return ans;
  }
}
```

## 47. 全排列 II

> [47. 全排列 II](https://leetcode-cn.com/problems/permutations-ii/)

语言：java

思路：这个感觉还挺复杂的，一时间只想到用传统全排列写法，然后在去重，但是效率太低。建议直接看其他人讲解。

参考代码1（1ms，100%）：

> [47. 全排列 II:【彻底理解排列中的去重问题】详解](https://leetcode-cn.com/problems/permutations-ii/solution/47-quan-pai-lie-iiche-di-li-jie-pai-lie-zhong-de-q/)

```java
class Solution {
    //存放结果
    List<List<Integer>> result = new ArrayList<>();
    //暂存结果
    List<Integer> path = new ArrayList<>();

    public List<List<Integer>> permuteUnique(int[] nums) {
        boolean[] used = new boolean[nums.length];
        Arrays.fill(used, false);
        Arrays.sort(nums);
        backTrack(nums, used);
        return result;
    }

    private void backTrack(int[] nums, boolean[] used) {
        if (path.size() == nums.length) {
            result.add(new ArrayList<>(path));
            return;
        }
        for (int i = 0; i < nums.length; i++) {
            // used[i - 1] == true，说明同⼀树⽀nums[i - 1]使⽤过
            // used[i - 1] == false，说明同⼀树层nums[i - 1]使⽤过
            // 如果同⼀树层nums[i - 1]使⽤过则直接跳过
            if (i > 0 && nums[i] == nums[i - 1] && used[i - 1] == false) {
                continue;
            }
            //如果同⼀树⽀nums[i]没使⽤过开始处理
            if (used[i] == false) {
                used[i] = true;//标记同⼀树⽀nums[i]使⽤过，防止同一树支重复使用
                path.add(nums[i]);
                backTrack(nums, used);
                path.remove(path.size() - 1);//回溯，说明同⼀树层nums[i]使⽤过，防止下一树层重复
                used[i] = false;//回溯
            }
        }
    }
}
```

参考后重写：

```java
class Solution {
  public List<List<Integer>> permuteUnique(int[] nums) {
    List<List<Integer>> res = new ArrayList<>();
    Arrays.sort(nums);
    recall(nums, res, 0, nums.length, new ArrayList<>(), new boolean[nums.length]);
    return res;
  }

  public void recall(int[] nums, List<List<Integer>> res, int curDepth, int maxDepth, List<Integer> road, boolean[] used) {
    if (curDepth == maxDepth) {
      res.add(new ArrayList<>(road));
    } else {
      for (int i = 0; i < nums.length; ++i) {
        if (i > 0 && nums[i] == nums[i - 1] && !used[i - 1]) {
          continue;
        }
        if (!used[i]) {
          road.add(nums[i]);
          used[i] = true;
          recall(nums, res, curDepth + 1, maxDepth, road, used);
          used[i] = false;
          road.remove(road.size() - 1);
        }
      }
    }
  }
}
```

## 17. 电话号码的字母组合

> [17. 电话号码的字母组合](https://leetcode-cn.com/problems/letter-combinations-of-a-phone-number/)

语言：java

思路：类似全排列II，但是需要考虑的是，每个数字按键里面一次只能挑一个字母。全排列是排列问题，这里是组合问题。这里直接DFS大胆遍历所有情况即可。

代码（0ms，100%）：

```java
class Solution {
  public List<String> letterCombinations(String digits) {
    char[][] maps = new char[][]{{}, {}, {'a', 'b', 'c'}, {'d', 'e', 'f'}, {'g', 'h', 'i'}
                                 , {'j', 'k', 'l'}, {'m', 'n', 'o'}, {'p', 'q', 'r', 's'}, {'t', 'u', 'v'}, {'w', 'x', 'y', 'z'}};
    List<String> res = new ArrayList<>();
    int[] nums = new int[digits.length()];
    for (int i = 0; i < digits.length(); ++i) {
      nums[i] = digits.charAt(i) - '0';
    }
    recall(0,digits.length(),nums,maps,new StringBuilder(), res);
    return res;
  }

  public void recall(int curDepth,int maxDepth,int[] digits, char[][] maps,StringBuilder sb, List<String> res) {
    if(curDepth==maxDepth){
      //这个if是，输入”“的情况，res必须是[]
      if(sb.length()>0){
        res.add(sb.toString());
      }
    }else {
      for(int i = 0;i<maps[digits[curDepth]].length;++i){
        sb.append(maps[digits[curDepth]][i]);
        recall(curDepth+1,maxDepth,digits,maps,sb,res);
        sb.deleteCharAt(sb.length()-1);
      }
    }
  }
}
```

## 39. 组合总和

> [39. 组合总和](https://leetcode-cn.com/problems/combination-sum/)

语言：java

思路：还是按照类似全排列的思想去做题，但是这里同样是更暴力的DFS，遍历所有情况。

+ 需要排序
+ 暴力的DFS，但是需要用begin记录起始遍历位置=>避免回头路`1112`和`1121`。

代码（3ms，77.35%）：

```java
class Solution {
  public List<List<Integer>> combinationSum(int[] candidates, int target) {
    Arrays.sort(candidates);
    List<List<Integer>> res = new ArrayList<>();
    recall(candidates, 0, candidates.length, target, new LinkedList<>(), res);
    return res;
  }

  public void recall(int[] candidates, int begin, int length, int target, Deque<Integer> road, List<List<Integer>> res) {
    if (target == 0) {
      res.add(new ArrayList<>(road));
    } else {
      // begin是避免走回头路，重复组合
      for (int i = begin; i < length; ++i) {
        // 如果当前元素大于目标值，没必要再往下DFS，因为再往后遍历的数字更大
        if (candidates[i] > target) {
          break;
        }
        road.addLast(candidates[i]);
        recall(candidates, i, length, target - candidates[i], road, res);
        road.removeLast();
      }
    }
  }
}
```

参考代码1（4ms，52.33%）：

> [组合总和--官方解答](https://leetcode-cn.com/problems/combination-sum/solution/zu-he-zong-he-by-leetcode-solution/)

```java
class Solution {
  public List<List<Integer>> combinationSum(int[] candidates, int target) {
    List<List<Integer>> ans = new ArrayList<List<Integer>>();
    List<Integer> combine = new ArrayList<Integer>();
    dfs(candidates, target, ans, combine, 0);
    return ans;
  }

  public void dfs(int[] candidates, int target, List<List<Integer>> ans, List<Integer> combine, int idx) {
    if (idx == candidates.length) {
      return;
    }
    if (target == 0) {
      ans.add(new ArrayList<Integer>(combine));
      return;
    }
    // 直接跳过
    dfs(candidates, target, ans, combine, idx + 1);
    // 选择当前数
    if (target - candidates[idx] >= 0) {
      combine.add(candidates[idx]);
      dfs(candidates, target - candidates[idx], ans, combine, idx);
      combine.remove(combine.size() - 1);
    }
  }
}
```

## 40. 组合总和 II

> [40. 组合总和 II](https://leetcode-cn.com/problems/combination-sum-ii/)

语言：java

思路：

+ 和[39. 组合总和](https://leetcode-cn.com/problems/combination-sum/)不同的是，每个数字只能使用一次。那这里就需要像全排列做题一样，用一个`boolean[]  used`来记录用过的元素。

+ 需要像[47. 全排列 II](https://leetcode-cn.com/problems/permutations-ii/)一样，避免DFS递归树同层出现相同数字的情况。

  ```java
  if(i>0&&candidates[i]==candidates[i-1]&&!used[i-1]){
    continue;
  }
  ```

代码（2ms，99.95%）：

```java
class Solution {
  public List<List<Integer>> combinationSum2(int[] candidates, int target) {
    List<List<Integer>> res = new ArrayList<>();
    Arrays.sort(candidates);
    recall(candidates,0,candidates.length,target,new boolean[candidates.length],new LinkedList<>(),res);
    return res;
  }

  public void recall(int[] candidates,int begin,int len,int target, boolean[] used, Deque<Integer> road,List<List<Integer>> res){
    if(target == 0){
      res.add(new ArrayList<>(road));
    }else{
      for(int i = begin;i<len;++i){
        if(candidates[i]> target){
          break;
        }
        // eg: 避免数组里多个1时，会有重复情况
        if(i>0&&candidates[i]==candidates[i-1]&&!used[i-1]){
          continue;
        }
        if(!used[i]){
          road.addLast(candidates[i]);
          used[i] = true;
          recall(candidates,i,len,target-candidates[i],used,road,res);
          used[i] = false;
          road.removeLast();
        }
      }
    }
  }
}
```

参考代码1（3ms，82.41%）：

> [回溯算法 + 剪枝（Java、Python）](https://leetcode-cn.com/problems/combination-sum-ii/solution/hui-su-suan-fa-jian-zhi-python-dai-ma-java-dai-m-3/)

```java
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Arrays;
import java.util.Deque;
import java.util.List;

public class Solution {

  public List<List<Integer>> combinationSum2(int[] candidates, int target) {
    int len = candidates.length;
    List<List<Integer>> res = new ArrayList<>();
    if (len == 0) {
      return res;
    }

    // 关键步骤
    Arrays.sort(candidates);

    Deque<Integer> path = new ArrayDeque<>(len);
    dfs(candidates, len, 0, target, path, res);
    return res;
  }

  /**
     * @param candidates 候选数组
     * @param len        冗余变量
     * @param begin      从候选数组的 begin 位置开始搜索
     * @param target     表示剩余，这个值一开始等于 target，基于题目中说明的"所有数字（包括目标数）都是正整数"这个条件
     * @param path       从根结点到叶子结点的路径
     * @param res
     */
  private void dfs(int[] candidates, int len, int begin, int target, Deque<Integer> path, List<List<Integer>> res) {
    if (target == 0) {
      res.add(new ArrayList<>(path));
      return;
    }
    for (int i = begin; i < len; i++) {
      // 大剪枝：减去 candidates[i] 小于 0，减去后面的 candidates[i + 1]、candidates[i + 2] 肯定也小于 0，因此用 break
      if (target - candidates[i] < 0) {
        break;
      }

      // 小剪枝：同一层相同数值的结点，从第 2 个开始，候选数更少，结果一定发生重复，因此跳过，用 continue
      if (i > begin && candidates[i] == candidates[i - 1]) {
        continue;
      }

      path.addLast(candidates[i]);
      // 调试语句 ①
      // System.out.println("递归之前 => " + path + "，剩余 = " + (target - candidates[i]));

      // 因为元素不可以重复使用，这里递归传递下去的是 i + 1 而不是 i
      dfs(candidates, len, i + 1, target - candidates[i], path, res);

      path.removeLast();
      // 调试语句 ②
      // System.out.println("递归之后 => " + path + "，剩余 = " + (target - candidates[i]));
    }
  }

  public static void main(String[] args) {
    int[] candidates = new int[]{10, 1, 2, 7, 6, 1, 5};
    int target = 8;
    Solution solution = new Solution();
    List<List<Integer>> res = solution.combinationSum2(candidates, target);
    System.out.println("输出 => " + res);
  }
}
```

## 51. N皇后

> [51. N 皇后](https://leetcode-cn.com/problems/n-queens/)

语言：java

思路：

+ 每次放棋子前，先考虑是否能放。
+ 因为每一行只用取一个值，所以用一维数组够了
+ Main方法里，需要指定第一行的棋子放哪里

代码（14ms，7.32%）：巨慢无比，我估计是打印结果的地方我写的不好。

```java
class Solution {
  public List<List<String>> solveNQueens(int n) {
    List<List<String>> res = new ArrayList<>();
    int[] map = new int[n];
    for(int i = 0;i<n;++i){
      map[0] = i;
      recall(map,1,n,res);
    }
    return res;
  }

  public void recall(int[] map,int curDepth,int maxDepth,List<List<String>> res){
    if(curDepth == maxDepth){
      List<String> oneAnswer = new ArrayList<>();
      for(int i = 0;i<maxDepth;++i){
        int col = map[i];
        String line = String.join("", Collections.nCopies(col, ".")) + "Q" +
          String.join("", Collections.nCopies(maxDepth - col - 1, "."));
        oneAnswer.add(line);
      }
      res.add(oneAnswer);
    }else{
      for(int i = 0;i<maxDepth;++i){
        if(canSet(map,curDepth,i)){
          map[curDepth] = i;
          recall(map,curDepth+1,maxDepth,res);
        }
      }
    }
  }

  public boolean canSet(int[] map,int row,int col){
    for(int i = 0;i<row;++i){
      // 竖直方向
      if(map[i]==col){
        return false;
      }
      // 撇方向
      if(map[i] - col == i - row){
        return false;
      }
      // 捺方向
      if(map[i] - col == row - i){
        return false;
      }
    }
    return true;
  }
}
```

参考代码1（6ms，43.63%）:

>[N皇后--官方题解](https://leetcode-cn.com/problems/n-queens/solution/nhuang-hou-by-leetcode-solution/)

```java
class Solution {
  public List<List<String>> solveNQueens(int n) {
    List<List<String>> solutions = new ArrayList<List<String>>();
    int[] queens = new int[n];
    Arrays.fill(queens, -1);
    Set<Integer> columns = new HashSet<Integer>();
    Set<Integer> diagonals1 = new HashSet<Integer>();
    Set<Integer> diagonals2 = new HashSet<Integer>();
    backtrack(solutions, queens, n, 0, columns, diagonals1, diagonals2);
    return solutions;
  }

  public void backtrack(List<List<String>> solutions, int[] queens, int n, int row, Set<Integer> columns, Set<Integer> diagonals1, Set<Integer> diagonals2) {
    if (row == n) {
      List<String> board = generateBoard(queens, n);
      solutions.add(board);
    } else {
      for (int i = 0; i < n; i++) {
        if (columns.contains(i)) {
          continue;
        }
        int diagonal1 = row - i;
        if (diagonals1.contains(diagonal1)) {
          continue;
        }
        int diagonal2 = row + i;
        if (diagonals2.contains(diagonal2)) {
          continue;
        }
        queens[row] = i;
        columns.add(i);
        diagonals1.add(diagonal1);
        diagonals2.add(diagonal2);
        backtrack(solutions, queens, n, row + 1, columns, diagonals1, diagonals2);
        queens[row] = -1;
        columns.remove(i);
        diagonals1.remove(diagonal1);
        diagonals2.remove(diagonal2);
      }
    }
  }

  public List<String> generateBoard(int[] queens, int n) {
    List<String> board = new ArrayList<String>();
    for (int i = 0; i < n; i++) {
      char[] row = new char[n];
      Arrays.fill(row, '.');
      row[queens[i]] = 'Q';
      board.add(new String(row));
    }
    return board;
  }
}
```

参考代码2（1ms，100.00%）：

> [N皇后--官方题解](https://leetcode-cn.com/problems/n-queens/solution/nhuang-hou-by-leetcode-solution/)

```java
class Solution {
  public List<List<String>> solveNQueens(int n) {
    int[] queens = new int[n];
    Arrays.fill(queens, -1);
    List<List<String>> solutions = new ArrayList<List<String>>();
    solve(solutions, queens, n, 0, 0, 0, 0);
    return solutions;
  }

  public void solve(List<List<String>> solutions, int[] queens, int n, int row, int columns, int diagonals1, int diagonals2) {
    if (row == n) {
      List<String> board = generateBoard(queens, n);
      solutions.add(board);
    } else {
      int availablePositions = ((1 << n) - 1) & (~(columns | diagonals1 | diagonals2));
      while (availablePositions != 0) {
        int position = availablePositions & (-availablePositions);
        availablePositions = availablePositions & (availablePositions - 1);
        int column = Integer.bitCount(position - 1);
        queens[row] = column;
        solve(solutions, queens, n, row + 1, columns | position, (diagonals1 | position) << 1, (diagonals2 | position) >> 1);
        queens[row] = -1;
      }
    }
  }

  public List<String> generateBoard(int[] queens, int n) {
    List<String> board = new ArrayList<String>();
    for (int i = 0; i < n; i++) {
      char[] row = new char[n];
      Arrays.fill(row, '.');
      row[queens[i]] = 'Q';
      board.add(new String(row));
    }
    return board;
  }
}
```

## 416. 分割等和子集

> [416. 分割等和子集](https://leetcode-cn.com/problems/partition-equal-subset-sum/)

语言：java

思路：动态规划，用一维度数组`dp[target+1]`来保存，这里dp表示背包的容量，放进去的数字即是商品价值，同时也是商品重量。

代码（24ms，82.03%）：

```java
class Solution {
  public boolean canPartition(int[] nums) {
    int sum = 0;
    for (int num : nums) {
      sum += num;
    }
    // 奇数，本身不可能拆分成两个等和数组
    if ((sum & 1) == 1) {
      return false;
    }
    int target = sum / 2;
    int[] dp = new int[target + 1];
    // 遍历物品
    for (int item = 0; item < nums.length; ++item) {
      // 背包一开始是有足够容量的，之后拿了东西后就减小，所以倒序遍历
      // 因为有些容量可以直接浪费掉，所以每次capacity只-1就好了
      // 如果容量不够了，也不用再遍历了，所以 capacity>=nums[item]
      for (int capacity = target; capacity >= nums[item]; --capacity) {
        // 当前背包 两种选择，要么不拿当前的商品，要么拿。
        dp[capacity] = Math.max(dp[capacity], dp[capacity - nums[item]] + nums[item]);
      }
    }
    // 判断 背包容量为target时，是不是正好装得下target
    // 之所以能这么判断，是因为如果能够划分数组，那么容量为target的背包能装的价值总量为target的可能有多种情况，但是一定不会超过target。
    // 毕竟这里 价值 == 重量
    return dp[target] == target;
  }
}
```

参考代码1（20ms，91.38%）：

> [动态规划（转换为 0-1 背包问题）](https://leetcode-cn.com/problems/partition-equal-subset-sum/solution/0-1-bei-bao-wen-ti-xiang-jie-zhen-dui-ben-ti-de-yo/)

```java
public class Solution {

  public boolean canPartition(int[] nums) {
    int len = nums.length;
    int sum = 0;
    for (int num : nums) {
      sum += num;
    }
    if ((sum & 1) == 1) {
      return false;
    }

    int target = sum / 2;
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;

    if (nums[0] <= target) {
      dp[nums[0]] = true;
    }
    for (int i = 1; i < len; i++) {
      for (int j = target; nums[i] <= j; j--) {
        if (dp[target]) {
          return true;
        }
        dp[j] = dp[j] || dp[j - nums[i]];
      }
    }
    return dp[target];
  }
}
```

## 474. 一和零

> [474. 一和零](https://leetcode-cn.com/problems/ones-and-zeroes/)

语言：java

思路：

+ 看上去可以暴力DFS，也可以动态规划的样子。这里试着用动态规划写写看。
+ `dp[m+1][n+1]`表示能装m个0和n个1的背包最多能放几个子集。
+ 状态转移方程：`dp[m][n] = Math.max(dp[m][n],dp[m-zeroArr[i]][n-oneArr[i]]+1);`
  + 即要么不把当前元素放到背包
  + 或者当前元素放背包，对应的剩余容量减小，然后子集数量+1
  + `dp[0][0] = 0`，因为题目里面所有字符串长度>0，所以`m=0 且 n=0`时，背包必定装不下任何元素

代码（61ms，35.89%）：超级慢，应该是三重for循环的原因

```java
class Solution {
  public int findMaxForm(String[] strs, int m, int n) {
    int strsLen = strs.length;
    int[] oneArr = new int[strsLen];
    int[] zeroArr = new int[strsLen];
    int[][] dp = new int[m+1][n+1];
    // 统计每个字符串里面的 0 和 1 的个数
    for(int i = 0;i<strsLen;++i){
      char[] chars = strs[i].toCharArray();
      for(char c:chars){
        if(c=='1'){
          ++oneArr[i];
        }else{
          ++zeroArr[i];
        }
      }
    }
    // 动态规划 - 状态转移方程
    for(int item = 0;item<strsLen;++item){
      for(int i = m;i>=0;--i){
        for(int j = n;j>=0;--j){
          if(i>=zeroArr[item]&&j>=oneArr[item]){
            dp[i][j] = Math.max(dp[i][j],dp[i-zeroArr[item]][j-oneArr[item]]+1);
          }
        }
      }
    }
    return dp[m][n];
  }
}
```

参考代码1（41ms，54.92%）：

+ 主要是考虑到是**"有限背包"**，所以计算字符串的0和1的个数。可以在遍历字符串数组的时候临时统计，因为只遍历一次，不会重复放入同一个物品。=>而我多了一次for循环用来专门统计，亏时间了。

> [一和零--官方题解](https://leetcode-cn.com/problems/ones-and-zeroes/solution/yi-he-ling-by-leetcode/)
>
> **注意由于每个字符串只能使用一次（即有限背包**），因此在更新 `dp(i, j)` 时，`i` 和 `j` 都需要从大到小进行枚举。

```java
public class Solution {
  public int findMaxForm(String[] strs, int m, int n) {
    int[][] dp = new int[m + 1][n + 1];
    for (String s: strs) {
      int[] count = countzeroesones(s);
      for (int zeroes = m; zeroes >= count[0]; zeroes--)
        for (int ones = n; ones >= count[1]; ones--)
          dp[zeroes][ones] = Math.max(1 + dp[zeroes - count[0]][ones - count[1]], dp[zeroes][ones]);
    }
    return dp[m][n];
  }
  public int[] countzeroesones(String s) {
    int[] c = new int[2];
    for (int i = 0; i < s.length(); i++) {
      c[s.charAt(i)-'0']++;
    }
    return c;
  }
}
```

参考代码2（40ms，57.51%）：

> [动态规划（转换为 0-1 背包问题）](https://leetcode-cn.com/problems/ones-and-zeroes/solution/dong-tai-gui-hua-zhuan-huan-wei-0-1-bei-bao-wen-ti/)
>
> 和官方题解其实差不多，没啥太大区别

```java
public class Solution {

  public int findMaxForm(String[] strs, int m, int n) {
    int[][] dp = new int[m + 1][n + 1];
    dp[0][0] = 0;
    for (String s : strs) {
      int[] zeroAndOne = calcZeroAndOne(s);
      int zeros = zeroAndOne[0];
      int ones = zeroAndOne[1];
      for (int i = m; i >= zeros; i--) {
        for (int j = n; j >= ones; j--) {
          dp[i][j] = Math.max(dp[i][j], dp[i - zeros][j - ones] + 1);
        }
      }
    }
    return dp[m][n];
  }

  private int[] calcZeroAndOne(String str) {
    int[] res = new int[2];
    for (char c : str.toCharArray()) {
      res[c - '0']++;
    }
    return res;
  }
}
```

过去提交过的代码（44ms，49.79%）：

+ 其实也差不多，主要主要时间差距的地方就是获取每个字符串的0和1个数的代码实现

```java
class Solution {
  public int findMaxForm(String[] strs, int m, int n) {
    int[][] dp = new int[m + 1][n + 1];
    //        int max = 0;
    //dp[i][j] = Math.max(dp[i-strs[i].zero][j-str[i].one]+1,dp[i][j]);
    for (int i = 0; i < strs.length; ++i) {
      int zeroCount = zeroCount(strs, i);
      int oneCount = oneCount(strs, i);
      for (int j = m; j >= zeroCount; --j) {
        for (int k = n; k >= oneCount; --k) {
          dp[j][k] = Math.max(dp[j][k], dp[j-zeroCount][k-oneCount]+1);
        }
      }
    }
    return dp[m][n];
  }

  public int zeroCount(String[] strs, int index) {
    int count = 0;
    for (int i = 0; i < strs[index].length(); ++i) {
      if (strs[index].charAt(i) == '0') {
        ++count;
      }
    }
    return count;
  }

  public int oneCount(String[] strs, int index) {
    int count = 0;
    for (int i = 0; i < strs[index].length(); ++i) {
      if (strs[index].charAt(i) == '1') {
        ++count;
      }
    }
    return count;
  }
}
```

## 494. 目标和

> [494. 目标和](https://leetcode-cn.com/problems/target-sum/)

语言：java

思路：可以用暴力DFS，也可以用动态规划。这里尝试动态规划

+ 这个题目有个烦人点就是+-号问题。
+ 假设准备取正的集合和为Z，准备取负的集合和为F，目标值S，nums数组原本的和为sum
  + `Z-F = S` => `Z-F+F = S + F` => `Z + Z = S + Z + F` => `2Z = S + sum`
  + 所以到头来，我们只要考虑正数情况，不需要考虑负数
+ `dp[i]`表示目标和到i的方式有几种。
  + `dp[0]=1`，数组非空，全部元素都必须用上，即一定会用到`dp[0]`，每次用到`dp[0]`说明找到一种情况，+1。

代码（3ms，89.22%）：

```java
class Solution {
  public int findTargetSumWays(int[] nums, int S) {
    // `Z-F = S` => `Z-F+F = S + F` => `Z + Z = S + Z + F` => `2Z = S + sum`
    int sum = 0;
    for (int num : nums) {
      sum += num;
    }
    // S + sum 必须 == 2Z，即不能是奇数
    if (S > sum || ((S + sum) & 1) == 1) {
      return 0;
    }
    // 实际 target，我们只考虑 正数和 计算
    int target = (S + sum) / 2;
    int[] dp = new int[target + 1];
    dp[0] = 1;
    for (int num : nums) {
      for (int j = target; j >= num; --j) {
        dp[j] += dp[j - num];
      }
    }
    return dp[target];
  }
}
```

过去提交过的代码（可能是参考代码？2ms，99.98%）：整体差不多

```java
class Solution {
  public int findTargetSumWays(int[] nums, int S) {
    // S(p) - S(n) = S
    // 2 * S(p)  = S + sum

    int sum = 0;

    for(int num:nums){
      sum += num;
    }

    if(sum<S||(sum+S)%2==1){
      return 0;
    }

    sum = (sum+S)/2;

    int[] dp = new int[sum+1];
    dp[0] = 1;

    for(int num:nums){
      for(int j = sum;j>=num;--j){
        dp[j] += dp[j-num];
      }
    }
    return dp[sum];
  }
}
```

## 1025. 除数博弈

> [1025. 除数博弈](https://leetcode-cn.com/problems/divisor-game/)

语言：java

思路：这里主要需要思考的是，"两个玩家都以最佳状态参与游戏"，怎么样是最佳。

+ 先找规律试试，根据题目的提示，可以得知，如果为2，直接就是true，为3则false。
+ 这里`N-x`替换原本的`N`，也算是一种提示，暗示可以用动态规划=>当前状态依赖过去的计算
+ `dp[i]`表示数字为i时，爱丽丝是否能赢
+ `dp[2] = true; d[3] = false` => 题目给出的
+ `dp[1] = false`，因为爱丽丝此时无法操作
+ 纸上发现规律，基本只要考虑-1，-2，-3的情况，而实际上只要是偶数就赢了

代码1（0ms，100%）：

```java
class Solution {
  public boolean divisorGame(int N) {
    return N%2==0;
  }
}
```

代码2（0ms，100%）：只考虑-1，-2，-3的情况

```java
class Solution {
  public boolean divisorGame(int N) {
    boolean[] dp = new boolean[N+2];
    dp[2] = true;
    for(int i = 4;i<=N;++i){
      if(i%2==0){
        dp[i] = !dp[i-2];
      }
      if(!dp[i] && i%3==0){
        dp[i] = !dp[i-3];
      }
      if(!dp[i]){
        dp[i] = !dp[i-1];
      }
    }
    return dp[N];
  }
}
```

## 112. 路径总和

> [112. 路径总和](https://leetcode-cn.com/problems/path-sum/)

语言：java

思路：暴力DFS，遍历所有情况

+ 必须是根到叶子结点，所以节点数量至少2个
+ **targetSum本来就可以是0**

代码（0ms，100%）：看着简单题，结果没想到还错了几次

```java
class Solution {
  public boolean hasPathSum(TreeNode root, int targetSum) {
    return root != null && recur(root, targetSum);
  }

  public boolean recur(TreeNode root, int targetSum) {
    if(root == null){
      return false;
    }
    if(root.left==null&&root.right==null){
      return targetSum - root.val == 0;
    }
    return recur(root.left, targetSum-root.val) || recur(root.right,targetSum-root.val);
  }
}
```

## 面试题 01.04. 回文排列

> [面试题 01.04. 回文排列](https://leetcode-cn.com/problems/palindrome-permutation-lcci/)

语言：java

思路：感觉这题就是脑经急转弯，字符串里的字母类别中，至多只有一个字母出现次数为单数即true。

+ 为方便计算同一个字母次数，先对s字符串里的字母排序

代码（0ms，100.00%）：

```java
class Solution {
  public boolean canPermutePalindrome(String s) {
    char[] chars = s.toCharArray();
    int len = chars.length;
    Arrays.sort(chars);
    int oddCount = 0;// 同一个字符个数为奇数的 字符类别总数
    for (int left = 0, right = 0; right < len; ) {
      while (right < len && chars[left] == chars[right]) {
        ++right;
      }
      if ((right - left) % 2 == 1) {
        ++oddCount;
      }
      left = right;
    }
    return oddCount <= 1;
  }
}
```

## 647. 回文子串

> [647. 回文子串](https://leetcode-cn.com/problems/palindromic-substrings/)

语言：java

思路：暴力双层for循环，效率超级慢。直接每个子串都判断回文。

代码（582ms，5.02%）：感觉自己对"字符串处理"相关的题目不是很熟练。

```java
class Solution {
  public int countSubstrings(String s) {
    int count = 0;
    int len = s.length();
    for(int left = 0;left<len;++left){
      for(int right = left;right<len;++right){
        if(is(s,left,right)){
          ++count;
        }
      }
    }
    return count;
  }

  public boolean is(String s ,int left,int right){
    while(left<right){
      if(s.charAt(left)!=s.charAt(right)){
        return false;
      }
      ++left;
      --right;
    }
    return true;
  }
}
```

参考代码1（4ms，78.15%）：

> [回文子串--官方题解](https://leetcode-cn.com/problems/palindromic-substrings/solution/hui-wen-zi-chuan-by-leetcode-solution/)
>
> 2*n，主要是为了统一长度为偶数和奇数的情况
>
> 需要纸上找规律， 把回文左右边界的计算规律总结出公式

```java
class Solution {
  public int countSubstrings(String s) {
    int n = s.length(), ans = 0;
    for (int i = 0; i < 2 * n - 1; ++i) {
      int l = i / 2, r = i / 2 + i % 2;
      while (l >= 0 && r < n && s.charAt(l) == s.charAt(r)) {
        --l;
        ++r;
        ++ans;
      }
    }
    return ans;
  }
}
```

参考代码2：

> [回文子串--官方题解](https://leetcode-cn.com/problems/palindromic-substrings/solution/hui-wen-zi-chuan-by-leetcode-solution/)
>
> Manacher 算法 => 有点复杂。

```java
class Solution {
  public int countSubstrings(String s) {
    int n = s.length();
    StringBuffer t = new StringBuffer("$#");
    for (int i = 0; i < n; ++i) {
      t.append(s.charAt(i));
      t.append('#');
    }
    n = t.length();
    t.append('!');

    int[] f = new int[n];
    int iMax = 0, rMax = 0, ans = 0;
    for (int i = 1; i < n; ++i) {
      // 初始化 f[i]
      f[i] = i <= rMax ? Math.min(rMax - i + 1, f[2 * iMax - i]) : 1;
      // 中心拓展
      while (t.charAt(i + f[i]) == t.charAt(i - f[i])) {
        ++f[i];
      }
      // 动态维护 iMax 和 rMax
      if (i + f[i] - 1 > rMax) {
        iMax = i;
        rMax = i + f[i] - 1;
      }
      // 统计答案, 当前贡献为 (f[i] - 1) / 2 上取整
      ans += f[i] / 2;
    }

    return ans;
  }
}
```

## 1392. 最长快乐前缀

> [最长快乐前缀](https://leetcode-cn.com/problems/longest-happy-prefix/)

语言：java

思路：KMP算法，利用里面求next数组的思想，进行求解

代码（11ms，66.30%）：

```java
class Solution {
  public String longestPrefix(String s) {
    int n = s.length();
    int[] prefix = new int[n];
    int len = 0;
    for (int i = 1; i < n;) {
      if(s.charAt(i) == s.charAt(len)){
        ++len;
        prefix[i] = len;
        ++i;
      }else{
        if(len > 0){
          len = prefix[len-1];
        }else{
          ++i;
        }
      }
    }
    return s.substring(0, prefix[n-1]);
  }
}
```

参考代码1（6ms，100%）：

```java
class Solution {
  public String longestPrefix(String s) {
    if (s == null || s.length() < 2) return "";
    char[] chs = s.toCharArray();
    int len = s.length();
    int[] next = new int[len + 1];
    next[0] = -1;
    // next[1] = 0;
    for (int l = 0, r = 1; r < len;) {
      if (chs[l] == chs[r]) {
        next[++r] = ++l;
      } else if (l > 0) {
        l = next[l];
      } else {
        r++;
      }
    }
    return s.substring(0, next[len]);
  }
}
```

参考代码2（7ms，97.24%）：

> [利用KMP算法中的next数组求法解答，时间超100%](https://leetcode-cn.com/problems/longest-happy-prefix/solution/li-yong-kmpsuan-fa-zhong-de-nextshu-zu-q-57a7/)

```java
class Solution {
  public String longestPrefix(String s) {
    if (s == null || s.length() < 2) return "";
    char[] sChars = s.toCharArray();
    int[] nexts = new int[s.length() + 1];
    nexts[0] = -1;
    nexts[1] = 0;
    int cur = 2;
    int preNext = 0;
    while (cur < nexts.length) {
      if (sChars[cur - 1] == sChars[preNext]) {
        nexts[cur++] = ++preNext;
      } else if (preNext > 0) {
        preNext = nexts[preNext];
      } else {
        nexts[cur++] = 0;
      }
    }
    return s.substring(0, nexts[nexts.length - 1]);
  }
}
```

参考后重写（10ms，71.82%）：

```java
class Solution {
  public String longestPrefix(String s) {
    int len = s.length();
    // 这里len+1，之后使用的时候，从下标1开始使用
    int[] next = new int[len + 1];
    for (int left = 0, right = 1; right < len; ) {
      if (s.charAt(right) == s.charAt(left)) {
        next[++right] = ++left;
      } else if (left > 0) {
        left = next[left];
      } else {
        ++right;
      }
    }
    return s.substring(0, next[len]);
  }
}
```

## 572. 另一个树的子树

> [572. 另一个树的子树](https://leetcode-cn.com/problems/subtree-of-another-tree/)

语言：java

思路：最容易想到的，感觉就是DFS判断了，这里试一下这种粗暴方法。

+ 判断当前节点是不是就是和子树一模一样；
+ 如果当前子树不是，那就判断左右子树是不是可能含有目标子树（用或||）

代码（13ms，6.77%）：巨慢。

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
  public boolean isSubtree(TreeNode s, TreeNode t) {
    if (s == null && t == null) {
      return true;
    }
    if (s == null || t == null) {
      return false;
    }
    return dfs(s,t);
  }

  public boolean dfs(TreeNode s,TreeNode t ){
    if(s == null){
      return false;
    }
    return judge(s,t) || dfs(s.left,t) || dfs(s.right,t);
  }

  public boolean judge(TreeNode s, TreeNode t){
    if(s == null && t == null){
      return true;
    }
    if(s == null || t == null || s.val != t.val){
      return false;
    }
    return judge(s.left,t.left) && judge(s.right, t.right);
  }
}
```

参考代码1：同样也是DFS，但是却快了6ms

>[另一个树的子树--官方题解](https://leetcode-cn.com/problems/subtree-of-another-tree/solution/ling-yi-ge-shu-de-zi-shu-by-leetcode-solution/)

```java
class Solution {
  public boolean isSubtree(TreeNode s, TreeNode t) {
    return dfs(s, t);
  }

  public boolean dfs(TreeNode s, TreeNode t) {
    if (s == null) {
      return false;
    }
    return check(s, t) || dfs(s.left, t) || dfs(s.right, t);
  }

  public boolean check(TreeNode s, TreeNode t) {
    if (s == null && t == null) {
      return true;
    }
    if (s == null || t == null || s.val != t.val) {
      return false;
    }
    return check(s.left, t.left) && check(s.right, t.right);
  }
}
```

参考代码2（5ms，86.0%）：先序遍历+KMP判断，感觉很奇特的解法。

>[另一个树的子树--官方题解](https://leetcode-cn.com/problems/subtree-of-another-tree/solution/ling-yi-ge-shu-de-zi-shu-by-leetcode-solution/)

```java
class Solution {
  List<Integer> sOrder = new ArrayList<Integer>();
  List<Integer> tOrder = new ArrayList<Integer>();
  int maxElement, lNull, rNull;

  public boolean isSubtree(TreeNode s, TreeNode t) {
    maxElement = Integer.MIN_VALUE;
    getMaxElement(s);
    getMaxElement(t);
    lNull = maxElement + 1;
    rNull = maxElement + 2;

    getDfsOrder(s, sOrder);
    getDfsOrder(t, tOrder);

    return kmp();
  }

  public void getMaxElement(TreeNode t) {
    if (t == null) {
      return;
    }
    maxElement = Math.max(maxElement, t.val);
    getMaxElement(t.left);
    getMaxElement(t.right);
  }

  public void getDfsOrder(TreeNode t, List<Integer> tar) {
    if (t == null) {
      return;
    }
    tar.add(t.val);
    if (t.left != null) {
      getDfsOrder(t.left, tar);
    } else {
      tar.add(lNull);
    }
    if (t.right != null) {
      getDfsOrder(t.right, tar);
    } else {
      tar.add(rNull);
    }
  }

  public boolean kmp() {
    int sLen = sOrder.size(), tLen = tOrder.size();
    int[] fail = new int[tOrder.size()];
    Arrays.fill(fail, -1);
    for (int i = 1, j = -1; i < tLen; ++i) {
      while (j != -1 && !(tOrder.get(i).equals(tOrder.get(j + 1)))) {
        j = fail[j];
      }
      if (tOrder.get(i).equals(tOrder.get(j + 1))) {
        ++j;
      }
      fail[i] = j;
    }
    for (int i = 0, j = -1; i < sLen; ++i) {
      while (j != -1 && !(sOrder.get(i).equals(tOrder.get(j + 1)))) {
        j = fail[j];
      }
      if (sOrder.get(i).equals(tOrder.get(j + 1))) {
        ++j;
      }
      if (j == tLen - 1) {
        return true;
      }
    }
    return false;
  }
}
```

参考KMP解法后，重写（6ms，82.05%）：

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
  List<Integer> sPreList, tPreList;
  int maxNum = Integer.MIN_VALUE, leftNull, rightNull;

  public boolean isSubtree(TreeNode s, TreeNode t) {
    sPreList = new ArrayList<>();
    tPreList = new ArrayList<>();
    getMaxNum(s);
    getMaxNum(t);
    leftNull = maxNum + 1;
    rightNull = maxNum + 2;
    preOrder(s, sPreList);
    preOrder(t, tPreList);
    return kmp(sPreList,tPreList);
  }

  public void getMaxNum(TreeNode root) {
    if (root == null) {
      return;
    }
    maxNum = Math.max(root.val, maxNum);
    getMaxNum(root.left);
    getMaxNum(root.right);
  }

  public void preOrder(TreeNode root, List<Integer> tree) {
    if (root == null) {
      return;
    }
    tree.add(root.val);
    if (root.left == null) {
      tree.add(leftNull);
    }else{
      preOrder(root.left, tree);
    }
    if (root.right == null) {
      tree.add(rightNull);
    }else{
      preOrder(root.right, tree);
    }
  }

  public boolean kmp(List<Integer> s,List<Integer> t){
    int sSize = s.size();
    int tSize = t.size();
    if(sSize<tSize){
      return false;
    }
    int[] next = new int[tSize+1];
    // 求next数组
    for(int i = 1,j=0;i<tSize;){
      if(t.get(i).equals(t.get(j))){
        next[++i] = ++j;
      }else if(j>0){
        j = next[j];
      }else{
        ++i;
      }
    }
    // kmp匹配
    for(int i = 0,j=0;i<sSize;){
      if(s.get(i).equals(t.get(j))){
        ++i;
        ++j;
        if(j==tSize){
          return true;
        }
      }else if(j>0){
        j = next[j];
      }else{
        ++i;
      }
    }
    return false;
  }
}
```

## 1143. 最长公共子序列

> [1143. 最长公共子序列](https://leetcode-cn.com/problems/longest-common-subsequence/)

语言：java

思路：本来想暴力求解，但是发现暴力求解其实也不好写。

参考代码1（11ms，81.36%）：好吧，居然要动态规划。

> [最长公共子序列--官方题解](https://leetcode-cn.com/problems/longest-common-subsequence/solution/zui-chang-gong-gong-zi-xu-lie-by-leetcod-y7u0/)

```java
class Solution {
  public int longestCommonSubsequence(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m + 1][n + 1];
    for (int i = 1; i <= m; i++) {
      char c1 = text1.charAt(i - 1);
      for (int j = 1; j <= n; j++) {
        char c2 = text2.charAt(j - 1);
        if (c1 == c2) {
          dp[i][j] = dp[i - 1][j - 1] + 1;
        } else {
          dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
        }
      }
    }
    return dp[m][n];
  }
}
```

参考后重写（12ms，78.55%）：我这个用的index，而不是len，写得好丑，再改一下。

```java
class Solution {
  public int longestCommonSubsequence(String text1, String text2) {
    int text1Len = text1.length();
    int text2Len = text2.length();
    int[][] dp = new int[text1Len][text2Len];
    for(int i = 0;i<text1Len;++i){
      for(int j = 0;j<text2Len;++j){
        if(text1.charAt(i) == text2.charAt(j)){
          if(i==0||j==0){
            dp[i][j] = 1;
          }else{
            dp[i][j] = dp[i-1][j-1]+1;
          }
        }else{
          if(i!=0&&j!=0){
            dp[i][j] = Math.max(dp[i-1][j],dp[i][j-1]);
          }
          else if(i==0&&j==0){
            continue;
          }
          else if(i==0){
            dp[i][j] = dp[i][j-1];
          }else if(j==0){
            dp[i][j] = dp[i-1][j];
          }
        }
      }
    }
    return dp[text1Len-1][text2Len-1];
  }
}
```

重写（以len的形式，而不是用index，9ms，86.25%）：

```java
class Solution {
  public int longestCommonSubsequence(String text1, String text2) {
    int text1Len = text1.length();
    int text2Len = text2.length();
    int[][] dp = new int[text1Len+1][text2Len+1];
    for(int i = 1;i<=text1Len;++i){
      char c1 = text1.charAt(i-1);
      for(int j = 1;j<=text2Len;++j){
        char c2 = text2.charAt(j-1);
        if(c1==c2){
          dp[i][j] = dp[i-1][j-1] + 1;
        }else{
          dp[i][j] = Math.max(dp[i-1][j],dp[i][j-1]);
        }
      }
    }
    return dp[text1Len][text2Len];
  }
}
```

## 剑指 Offer 48. 最长不含重复字符的子字符串

>[剑指 Offer 48. 最长不含重复字符的子字符串](https://leetcode-cn.com/problems/zui-chang-bu-han-zhong-fu-zi-fu-de-zi-zi-fu-chuan-lcof/)

语言：java

思路：一开始感觉可以用动态规划dp，dp[i\][j\]表示下标i到家的无重复字符串的最长子串。但是一看到s的长度可以到达5*10^4这个数量级，就感觉这样子不行。

后面想想可以维护一个大小变动的滑动窗口，然后HashMap记录用到的字符和对应的下标。

代码（7ms，79.40%）：

```java
class Solution {
  public int lengthOfLongestSubstring(String s) {
    char[] chars = s.toCharArray();
    int len = chars.length;
    int left = 0, right = 0;
    int max = 0;
    HashMap<Character, Integer> map = new HashMap<>();
    while (right < len) {
      char cur = chars[right];
      Integer last = map.get(cur);
      map.put(cur, right);
      if (last != null && last >= left) {
        left = last + 1;
      }else{
        max = Math.max(max,right-left+1);
      }
      ++right;
    }
    return max;
  }
}
```

## 781. 森林中的兔子

>[781. 森林中的兔子](https://leetcode-cn.com/problems/rabbits-in-forest/)

语言：java

思路：

+ 每个兔子喊的数字n，最后能容纳n+1只。比如兔子喊2，那么最多和他一组的还有2个，也就是这个小组里最多3个兔子（2+1）。
+ 如果一个组满人了，出现同样大小的组，就等于另外开一个新的。每次加入一个，就把原本的key对应的value减一，如果到0，下次就是新建一个组。（HashMap存储）
+ 最后兔子总数 = 数组内元素 + 每个组最后剩余空间。

代码（3ms，68.18%）：

```java
class Solution {
  public int numRabbits(int[] answers) {
    int len = answers.length;
    // 至少有len只兔子
    int res = len;
    HashMap<Integer, Integer> map = new HashMap<>();
    for (int num : answers) {
      Integer groupCount = map.get(num);
      if (groupCount != null) {
        groupCount = groupCount > 0 ? groupCount - 1 : num;
        map.put(num,groupCount);
      }else{
        map.put(num,num);
      }
    }
    for(int num : map.values()){
      res+=num;
    }
    return res;
  }
}
```

参考代码1（3ms，68.18%）：

> [森林中的兔子--官方题解](https://leetcode-cn.com/problems/rabbits-in-forest/solution/sen-lin-zhong-de-tu-zi-by-leetcode-solut-kvla/)

```java
class Solution {
  public int numRabbits(int[] answers) {
    Map<Integer, Integer> count = new HashMap<Integer, Integer>();
    for (int y : answers) {
      count.put(y, count.getOrDefault(y, 0) + 1);
    }
    int ans = 0;
    for (Map.Entry<Integer, Integer> entry : count.entrySet()) {
      int y = entry.getKey(), x = entry.getValue();
      ans += (x + y) / (y + 1) * (y + 1);
    }
    return ans;
  }
}
```

参考代码2（0ms）：思路其实没有什么太大区别，用的数组（主要题目有限定了出现的数字范围）

```java
class Solution {
  public int numRabbits(int[] answers) {
    int res = 0;
    int[] count = new int[1000];
    for(int temp:answers){
      if(count[temp]==0){
        res += (temp+1);
        count[temp] = temp;
      }else{
        count[temp] = count[temp]-1;
      }
    }
    return res;
  }
}
```

## 310. 最小高度树

>[310. 最小高度树](https://leetcode-cn.com/problems/minimum-height-trees/)

语言：java

思路：动态规划，A和B相连时，求A的高度，即B的高度+1，之后B再往下找与之相连的。（然而这样写了之后，代码运行不通过，虽然知道哪个步骤我的逻辑错了，但是一时间想不到怎么改好。）

参考代码1（13ms，90.51%）：

> [BFS 超级简单 注释超级详细](https://leetcode-cn.com/problems/minimum-height-trees/solution/zui-rong-yi-li-jie-de-bfsfen-xi-jian-dan-zhu-shi-x/)
>
> 类似拓扑排序。这个代码和另一个题解类似=>[通用图形BFS](https://leetcode-cn.com/problems/minimum-height-trees/solution/tong-yong-tu-xing-bfs-by-user8772/)
>
> 先找出所有度为1的节点，然后进行BFS，之后度为1的节点BFS中遇到的共同节点，即出于图中心位置的点，也就是我们要的答案。

```java
class Solution {

  public List<Integer> findMinHeightTrees(int n, int[][] edges) {
    List<Integer> res = new ArrayList<>();
    /*如果只有一个节点，那么他就是最小高度树*/
    if (n == 1) {
      res.add(0);
      return res;
    }
    /*建立各个节点的出度表*/
    int[] degree = new int[n];
    /*建立图关系，在每个节点的list中存储相连节点*/
    List<List<Integer>> map = new ArrayList<>();
    for (int i = 0; i < n; i++) {
      map.add(new ArrayList<>());
    }
    for (int[] edge : edges) {
      degree[edge[0]]++;
      degree[edge[1]]++;/*出度++*/
      map.get(edge[0]).add(edge[1]);/*添加相邻节点*/
      map.get(edge[1]).add(edge[0]);
    }
    /*建立队列*/
    Queue<Integer> queue = new LinkedList<>();
    /*把所有出度为1的节点，也就是叶子节点入队*/
    for (int i = 0; i < n; i++) {
      if (degree[i] == 1) queue.offer(i);
    }
    /*循环条件当然是经典的不空判断*/
    while (!queue.isEmpty()) {
      res = new ArrayList<>();/*这个地方注意，我们每层循环都要new一个新的结果集合，
            这样最后保存的就是最终的最小高度树了*/
      int size = queue.size();/*这是每一层的节点的数量*/
      for (int i = 0; i < size; i++) {
        int cur = queue.poll();
        res.add(cur);/*把当前节点加入结果集，不要有疑问，为什么当前只是叶子节点为什么要加入结果集呢?
                因为我们每次循环都会新建一个list，所以最后保存的就是最后一个状态下的叶子节点，
                这也是很多题解里面所说的剪掉叶子节点的部分，你可以想象一下图，每层遍历完，
                都会把该层（也就是叶子节点层）这一层从队列中移除掉，
                不就相当于把原来的图给剪掉一圈叶子节点，形成一个缩小的新的图吗*/
        List<Integer> neighbors = map.get(cur);
        /*这里就是经典的bfs了，把当前节点的相邻接点都拿出来，
                * 把它们的出度都减1，因为当前节点已经不存在了，所以，
                * 它的相邻节点们就有可能变成叶子节点*/
        for (int neighbor : neighbors) {
          degree[neighbor]--;
          if (degree[neighbor] == 1) {
            /*如果是叶子节点我们就入队*/
            queue.offer(neighbor);
          }
        }
      }
    }
    return res;/*返回最后一次保存的list*/
  }
}
```

参考后重写（373ms，5.98%）：??用了LinkedList就变成300+ms，将近400ms

```java
class Solution {
  public List<Integer> findMinHeightTrees(int n, int[][] edges) {
    if(n==1){
      return Collections.singletonList(0);
    }
    int[] degree = new int[n];
    List<List<Integer>> treeMap = new LinkedList<>();
    List<Integer> res = null;
    Queue<Integer> queue = new LinkedList<>();
    for(int i = 0;i<n;++i){
      treeMap.add(new LinkedList<>());
    }
    for(int[] edge: edges){
      degree[edge[0]] ++;
      degree[edge[1]] ++;
      treeMap.get(edge[0]).add(edge[1]);
      treeMap.get(edge[1]).add(edge[0]);
    }
    for(int i = 0;i<n;++i){
      if(degree[i] == 1){
        queue.offer(i);
      }
    }
    while(!queue.isEmpty()){
      int size = queue.size();
      res = new ArrayList<>();
      for(int i = 0;i<size;++i){
        int cur = queue.poll();
        res.add(cur);
        List<Integer> nextEdges = treeMap.get(cur);
        for(int num : nextEdges){
          --degree[num];
          if(degree[num]==1){
            queue.offer(num);
          }
        }
      }
    }
    return res;
  }
}
```

重写改用ArrayList（13ms，90.51%）：使用ArrayList和使用LinkedList的时间差距居然这么大。

```java
class Solution {
  public List<Integer> findMinHeightTrees(int n, int[][] edges) {
    List<Integer> res = new ArrayList<>();
    if(n==1){
      res.add(0);
      return res;
    }
    int[] degree = new int[n];
    List<List<Integer>> treeMap = new ArrayList<>();
    Queue<Integer> queue = new LinkedList<>();
    for(int i = 0;i<n;++i){
      treeMap.add(new ArrayList<>());
    }
    for(int[] edge: edges){
      degree[edge[0]] ++;
      degree[edge[1]] ++;
      treeMap.get(edge[0]).add(edge[1]);
      treeMap.get(edge[1]).add(edge[0]);
    }
    for(int i = 0;i<n;++i){
      if(degree[i] == 1){
        queue.offer(i);
      }
    }
    while(!queue.isEmpty()){
      int size = queue.size();
      res = new ArrayList<>();
      for(int i = 0;i<size;++i){
        int cur = queue.poll();
        res.add(cur);
        List<Integer> nextEdges = treeMap.get(cur);
        for(int num : nextEdges){
          --degree[num];
          if(degree[num]==1){
            queue.offer(num);
          }
        }
      }
    }
    return res;
  }
}
```

## 264. 丑数 II

>[264. 丑数 II](https://leetcode-cn.com/problems/ugly-number-ii/)

语言：java

思路：遇到过类似的题目，但是还是错了，卑微。

参考代码1（65ms，19.33%）：每次从最小堆里面取最小的数据，然后再把取出来的值\*2，\*3，\*5，放入最小堆中。需要保证没有重复元素。

> [丑数 II--官方题解](https://leetcode-cn.com/problems/ugly-number-ii/solution/chou-shu-ii-by-leetcode-solution-uoqd/)

```java
class Solution {
    public int nthUglyNumber(int n) {
        int[] factors = {2, 3, 5};
        Set<Long> seen = new HashSet<Long>();
        PriorityQueue<Long> heap = new PriorityQueue<Long>();
        seen.add(1L);
        heap.offer(1L);
        int ugly = 0;
        for (int i = 0; i < n; i++) {
            long curr = heap.poll();
            ugly = (int) curr;
            for (int factor : factors) {
                long next = curr * factor;
                if (seen.add(next)) {
                    heap.offer(next);
                }
            }
        }
        return ugly;
    }
}
```

参考代码2（3ms，81.63%）：动态规划DP，dp[i\]表示第i个丑数

> [丑数 II--官方题解](https://leetcode-cn.com/problems/ugly-number-ii/solution/chou-shu-ii-by-leetcode-solution-uoqd/)

```java
class Solution {
  public int nthUglyNumber(int n) {
    int[] dp = new int[n + 1];
    dp[1] = 1;
    int p2 = 1, p3 = 1, p5 = 1;
    for (int i = 2; i <= n; i++) {
      int num2 = dp[p2] * 2, num3 = dp[p3] * 3, num5 = dp[p5] * 5;
      dp[i] = Math.min(Math.min(num2, num3), num5);
      if (dp[i] == num2) {
        p2++;
      }
      if (dp[i] == num3) {
        p3++;
      }
      if (dp[i] == num5) {
        p5++;
      }
    }
    return dp[n];
  }
}
```

参考后重写（4ms，45.20%）：

主要就是乘2，乘3，乘5，必须是对上一次最小的数进行计算。而动态规划最擅长的就是依赖之前状态的值进行计算了。

这里用到3个指针，表明分别该用哪个数来乘2、乘3和乘5

```java
class Solution {
  public int nthUglyNumber(int n) {
    int index = 1, min = 1, twoIndex = 0, threeIndex = 0, fiveIndex = 0;
    int[] dp = new int[n];
    dp[0] = 1;
    while (index < n) {
      int twoNum = dp[twoIndex] * 2;
      int threeNum = dp[threeIndex] * 3;
      int fiveNum = dp[fiveIndex] * 5;
      dp[index] = Math.min(twoNum,Math.min(threeNum,fiveNum));
      if(dp[index] == twoNum){
        ++twoIndex;
      }
      if(dp[index] == threeNum){
        ++threeIndex;
      }
      if(dp[index] == fiveNum){
        ++fiveIndex;
      }
      ++index;
    }
    return dp[n-1];
  }
}
```

## 670. 最大交换

> [670. 最大交换 - 力扣（LeetCode）](https://leetcode.cn/problems/maximum-swap/)

语言：java

思路：许久没有做题了，脑子烂掉。

+ 最优情况即整个数字串就是从大到小排序好的，无需调换位置。例如9973
+ 坏一点的情况，例如9937，则是和最优情况差一点，<u>右边某部分存在不是 从大到小排序好的</u>，则需要调换。

所以，先对原数字串从到小排序得到新数字串，然后左到右遍历新数字串，和原本数字串对比，找到第一个不一样的数字，这个数字需要被调换位置。接着为了让数字尽量大，从右边往左在原来的数字串中找到对应的数字，将这两个下标进行调换。

代码（1ms，38.17%）

```java
class Solution {
  public int maximumSwap(int num) {
    char[] rawArray = String.valueOf(num).toCharArray();
    char[] numberCharArray = String.valueOf(num).toCharArray();
    Arrays.sort(numberCharArray);
    int len = numberCharArray.length;
    for (int i = 0; i < len - 1 - i; ++i) {
      swap(numberCharArray, i, len - i - 1);
    }
    int left = 0, right = len - 1;
    while (right > left) {
      while (right > left && numberCharArray[left] == rawArray[left]) {
        ++left;
      }
      while (right > left && rawArray[right] != numberCharArray[left]) {
        --right;
      }
      if (right > left) {
        swap(rawArray, left, right);
        break;
      }
    }
    int result = 0;
    for (int i = 0; i < len; ++i) {
      result *= 10;
      result += rawArray[i] - '0';
    }
    return result;
  }

  public void swap(char[] arr, int i, int j) {
    char tmp = arr[i];
    arr[i] = arr[j];
    arr[j] = tmp;
  }
}
```

参考代码1（0ms，100%）：

> [【爪哇缪斯】图解LeetCode - 最大交换 - 力扣（LeetCode）](https://leetcode.cn/problems/maximum-swap/solution/by-muse-77-hwnt/)

```java
class Solution {
  public int maximumSwap(int num) {
    char[] numArr = String.valueOf(num).toCharArray();        
    int[] maxIndexs = new int[numArr.length];

    int index = numArr.length - 1;
    for (int i = numArr.length - 1; i >= 0; i--) {
      if (numArr[i] > numArr[index]) index = i;
      maxIndexs[i] = index;
    }

    for (int i = 0; i < numArr.length; i++) {
      if (numArr[i] != numArr[maxIndexs[i]]) {
        char temp = numArr[i];
        numArr[i] = numArr[maxIndexs[i]];
        numArr[maxIndexs[i]] = temp;
        break;
      }
    }

    return Integer.valueOf(new String(numArr));
  }
}
```

参考后重写：

（1）什么情况需要调换位置 => 某下标 右边部分 存在数字比自己当前数字大 => 如何表示 某下标右边的值比自己大 => 试图用当前下标表示右边比自己大的数字

（2）怎么调换位置最划算 => 同样是需要兑换位置的情况，左边的数字尽量靠近左边，右边的数字尽量靠近右边，且右边的数字尽量大

（3）最后怎么调换位置

解决（1）：`rightMaxIndex[i] = j` （i <= j），表示 i 右边 比自己大的最大值数字对应的下标j

在（1）的前提下，思考（2），从右边往左遍历数字串，找到右边侧最大值，记录下标到rightMaxIndex[i]，并且下标只记录最大的下标值（即j尽量大）。

在（1）、（2）下思考（3），当`数字串[rightMaxIndex[i]]!=数字串[i]`时，表示某下标对应的数字其右边存在比自己大的数字（其右边部分比自己大的数字中下标最大的下标值是j），对换i和j对应位置的数字，最好替换左边的数字，所以从左到右遍历，找到第一个符合这个情况的，然后对换位置

```java
class Solution {
  public int maximumSwap(int num) {
    char[] rawArray = String.valueOf(num).toCharArray();
    int len = rawArray.length;
    int[] rightMaxIndex = new int[len];
    int max = -1;
    for (int i = len - 1, maxIndex = i; i >= 0; --i) {
      if (rawArray[i] > rawArray[maxIndex]) {
        maxIndex = i;
      }
      rightMaxIndex[i] = maxIndex;
    }
    for (int i = 0; i < len; ++i) {
      if (rawArray[i] != rawArray[rightMaxIndex[i]]) {
        swap(rawArray, i, rightMaxIndex[i]);
        break;
      }
    }
    int result = 0;
    for (int i = 0; i < len; ++i) {
      result *= 10;
      result += rawArray[i] - '0';
    }
    return result;
  }
  // 2736 => 1133

  public void swap(char[] arr, int i, int j) {
    char tmp = arr[i];
    arr[i] = arr[j];
    arr[j] = tmp;
  }
}
```

## 27. 移除元素

语言：java

思路：

1. 最简单暴力的方法，就是一次for循环找到需要删除的数字，第二次for从后往前找不是要删除的数字，和预删除数字对换，然后整理len-1。如果从后往前都是要删除的数字，则只需要len-1。
2. 双指针法(学习)：一个指针用于寻找非删除的值，一个指针用于存储非删除的值。即把指针1找到的不用删除的数据，放到指针2所在位置中（指针2相当于一个新数组，只不过空间刚好和原来数组重叠）。

代码1（0ms，100%）：

```java
class Solution {
  public int removeElement(int[] nums, int val) {
    int len = nums.length;
    for(int i = 0; i < len; ++i) {
      if(nums[i] == val) {
        for(int j = len -1; j >= i ; --j) {
          // 后面的数字不需要删除，则和前面要删除的数字对换
          if(nums[j] != val) {
            int tmp = nums[j];
            nums[j] = nums[i];
            nums[i] = tmp;
            // 删除一个数字后，len-1
            len = len - 1;
            break;
          } 
          // 原本靠后的数字就==要删除的数字，所以直接len-1
          len = len - 1;
        }
      }
    }
    return len;
  }
}
```

代码2（0ms，100%）：

```java
class Solution {
  public int removeElement(int[] nums, int val) {
    // i 用于寻找原数组中不用删除的元素，j用于存放不用删除的元素
    int j = 0;
    for(int i = 0,len = nums.length;i< len; ++ i) {
      if(nums[i] != val) {
        nums[j] = nums[i];
        ++j;
      }
    }
    return j;
  }
}
```

代码3，双向指针（0ms，100%）：目标是用两个指针，实现把删除的元素挪到右边。这里需要注意的是边界什么时候可以取=号。

+ 最外层while（left <=right）因为在[left,right]找数据，left可以=right
+ 中间left、right移动时，可取=号，因为[left，right]找数据
+ 只有left<right才有替换的必要。替换后，left和right当前位置无意义，可以继续挪动指针

```java
class Solution {
  public int removeElement(int[] nums, int val) {
    int len = nums.length;
    int left = 0,right = len -1;
    while(left <= right) {
      // 左到右，寻找需要删除的数据
      while(left <= right && nums[left] != val) {
        ++left;
      }
      // 右到左，寻找可以保留的数据
      while(right >= left && nums[right] == val) {
        --right;
      }
      if(left < right) {
        nums[left++] = nums[right--];
      }
    } 
    return left;
  }
}
```

## 26. 删除有序数组中的重复项

语言：java

思路：双指针，一个指针找不一样的数字，一个指针存储不重复的数字

代码（0ms，100%）：

```java
class Solution {
  public int removeDuplicates(int[] nums) {
    // 题目说 至少1个数字，那么从j从nums[1]开始，用于记录不重复的数字，而i找不重复的数字，找到则存到j中
    int j = 1;
    for(int i =1,len = nums.length;i < len; ++i) {
      if(nums[i] != nums[i-1]) {
        nums[j++] = nums[i];
      }
    } // 223344
    return j;
  }
}
```

参考代码1（0ms，100%）：进行了局部判断的优化

> [【双指针】删除重复项-带优化思路 - 删除有序数组中的重复项 - 力扣（LeetCode）](https://leetcode.cn/problems/remove-duplicates-from-sorted-array/solution/shuang-zhi-zhen-shan-chu-zhong-fu-xiang-dai-you-hu/)
>
> 原题解p+1用于存储不重复的数字，如果完全没有数字重复，就会有多余的重复赋值的步骤。
>
> 而没有重复数字时，p和q只相差1，所以当p和q相差 > 1的时候才有必要显式赋值。

```java
public int removeDuplicates(int[] nums) {
  if(nums == null || nums.length == 0) return 0;
  int p = 0;
  int q = 1;
  while(q < nums.length){
    if(nums[p] != nums[q]){
      if(q - p > 1){
        nums[p + 1] = nums[q];
      }
      p++;
    }
    q++;
  }
  return p + 1;
}
```

## 283. 移动零

语言：java

思路：相当于类似把0删除，即挪到数组最后面。最后在把后面位置填充0

代码（1ms，100%）：

```java
class Solution {
  public void moveZeroes(int[] nums) {
    int j =0,len = nums.length;
    for(int i = 0;i< len ;++i) {
      if(nums[i]!=0) {
        nums[j++] = nums[i];
      }
    }
    while(j<len) {
      nums[j++] = 0;
    }
  }
}
```

参考代码1： 

> [动画演示 283.移动零 - 移动零 - 力扣（LeetCode）](https://leetcode.cn/problems/move-zeroes/solution/dong-hua-yan-shi-283yi-dong-ling-by-wang_ni_ma/)
>
> 下方评论区

```java
public void moveZeroes(int[] nums)  {
  int length;
  if (nums == null || (length = nums.length) == 0) {
    return;
  }
  int j = 0;
  for (int i = 0; i < length; i++) {
    if (nums[i] != 0) {
      if (i > j) {// 当i > j 时，只需要把 i 的值赋值给j，并把原位置的值置0。同时这里也把交换操作换成了赋值操作，减少了一条操作语句，理论上能更节省时间。
        nums[j] = nums[i];
        nums[i] = 0;
      }
      j++;
    }
  }
}
```

## 209. 长度最小的子数组

语言：java

思路：最外情况即包含整个数组。使用滑动窗口，先只移动右边界，直到第一个能够满足的情况。如果满足条件，则尝试移动左边界，直到不满足条件为止，再此挪动右边界。

代码（1ms，99.99%）：

```java
class Solution {
  public int minSubArrayLen(int target, int[] nums) {
    int min = Integer.MAX_VALUE;
    int left = 0,right = 0,sum = 0, len = nums.length;
    while(left <= right) { // 一开始 left = right，所以 可以 =
      if(sum>=target) { // 如果 当前 和 >= target，可以尝试缩小左边界
        min = Math.min(min, right-left);
        sum-=nums[left];
        ++left;
      }else if(right < len){ // 如果 right 可以继续挪动，则尝试移动右边界
        sum += nums[right];
        ++right;
      } else { // 如果到这里，说明 sum < target并且右边界不能继续扩大了，sum只会越来越小，直接退出
        break;
      }
    }
    return min == Integer.MAX_VALUE ? 0 : min;
  }
}
```

参考代码1（1ms，99.99%）：

> [长度最小的子数组 - 长度最小的子数组 - 力扣（LeetCode）](https://leetcode.cn/problems/minimum-size-subarray-sum/solution/chang-du-zui-xiao-de-zi-shu-zu-by-leetcode-solutio/)
>
> 一样是滑动窗口，不过代码更简洁

```java
class Solution {
    public int minSubArrayLen(int s, int[] nums) {
        int n = nums.length;
        if (n == 0) {
            return 0;
        }
        int ans = Integer.MAX_VALUE;
        int start = 0, end = 0;
        int sum = 0;
        while (end < n) {
            sum += nums[end];
            while (sum >= s) {
                ans = Math.min(ans, end - start + 1);
                sum -= nums[start];
                start++;
            }
            end++;
        }
        return ans == Integer.MAX_VALUE ? 0 : ans;
    }
}
```

## 904. 水果成篮

语言：java

思路：滑动窗口，默认先只移动有边界，直到达到2种果实的最大上限限制后，在移动左边界。这里和普通的滑动窗口不一样的是，如何判断滑动窗口内种类超过2种（这个我判断的方法写的不好，一直没过，蛋疼）。

参考代码（58ms，21.75%）：

> [水果成篮 - 水果成篮 - 力扣（LeetCode）](https://leetcode.cn/problems/fruit-into-baskets/solution/shui-guo-cheng-lan-by-leetcode/)

```java
class Solution {
  public int totalFruit(int[] tree) {
    int ans = 0, i = 0;
    Counter count = new Counter();
    for (int j = 0; j < tree.length; ++j) {
      count.add(tree[j], 1);
      while (count.size() >= 3) {
        count.add(tree[i], -1);
        if (count.get(tree[i]) == 0)
          count.remove(tree[i]);
        i++;
      }

      ans = Math.max(ans, j - i + 1);
    }

    return ans;
  }
}

class Counter extends HashMap<Integer, Integer> {
  public int get(int k) {
    return containsKey(k) ? super.get(k) : 0;
  }

  public void add(int k, int v) {
    put(k, get(k) + v);
  }
}
```

参考代码2（5ms，97.39%）：

> 评论区题解。我最早的写法类似这个，但是对2个篮子的记录处理写得不妥过不了。

```java
// 本题要求，选择一个最长只有两个元素的子序列
class Solution {
  public int totalFruit(int[] fruits) {
    if(fruits.length == 1 && fruits.length == 2) {
      return fruits.length;
    }
    int basket1 = -1, basket2 = -1; //记录当前篮子里的水果
    int sum = 0;
    int curFruit = -1, curFruitLoc = 0; //记录当前的水果，和当前水果的起始位置
    int subSum = 0;
    int j = 0; // 记录篮子起始位置
    for (int i = 0; i < fruits.length; ++i) {
      if (fruits[i] == basket1 || fruits[i] == basket2)
      {
        if (fruits[i] != curFruit) {// 记录在篮子里的连续最近，在更换篮子里水果的时候使用
          curFruit = fruits[i];
          curFruitLoc = i;
        }
      }
      else {
        j = curFruitLoc;
        curFruitLoc = i;
        if (basket1 == curFruit) { // 更新水果篮
          basket2 = fruits[i];
          curFruit = basket2;

        }
        else {
          basket1 = fruits[i];
          curFruit = basket1;
        }
      }
      subSum = (i - j + 1); // 计算最长子序列
      sum = sum > subSum ? sum : subSum;
    }
    return sum;
  }
}
```

参考后重写（6ms，87%）：

```java
// 本题要求，选择一个最长只有两个元素的子序列
class Solution {
  public int totalFruit(int[] fruits) {
    int len = fruits.length;
    if(len <= 2) {
      return len;
    }
    // [0,0,1,1]
    // [3,3,3,1,2,1,1,2,3,3,4]
    int right = 0 ,left = 0,basket1 = -1,basket2 = -1,lastFruit =-1,lastIndex = 0,max = 0;
    while(right < len) {
      // 当前遍历的果子 和 之前 框里 的一致
      if(fruits[right] == basket1 || fruits[right] == basket2) {
        if(fruits[right]!=lastFruit) { // 当right边界不是连续的果子时，记录边界点，比如2234的3,后面更换篮子用
          lastFruit = fruits[right];
          lastIndex = right;
        }

      }else {
        // 如果和框里的不一致，说明出现第3种果子，替换掉果子种类最早的一种（left=前一次遇到的第二种果子）
        left = lastIndex;
        lastIndex = right;
        // 决定把本次遇到的不一样的果子放到哪个框里（二选一）
        if(lastFruit == basket1) {
          basket2 = fruits[right];
          lastFruit = fruits[right];
        } else {
          basket1 = fruits[right];
          lastFruit = fruits[right];
        }
      }
      max = Math.max(max, right-left+1);
      ++right;
    }
    return max;
  }
}
```

## 79. 最小覆盖子串

语言：java

思路：能看出来是滑动窗口，暴力的滑动窗口的话，需要至少2个Map，一个记录当前遍历的字符，一个记录要求匹配的字符。

参考代码（2ms，96.55%）：

> [最小覆盖子串 - 最小覆盖子串 - 力扣（LeetCode）](https://leetcode.cn/problems/minimum-window-substring/solution/zui-xiao-fu-gai-zi-chuan-by-leetcode-solution/)
> 评论区大神：

```java
// 常规思路是右指针一直右移，直到窗口中包含t，然后左指针一直右移，直到窗口中不包含t，此过程中要一直验证窗口中是否包含t，时间复杂度高
// 思想：滑动窗口（优化版） 面对窗口中是否包含某一字符串这一问题，可以用数组统计每个字符出现的次数的方式。在该题中，右指针是一直右移直到窗口包含t，此时左指针不一定移动，只有当左指针指向的字符在窗口出现的次数太多时，即抛弃该字符窗口内仍包含t，此时才移动左指针。
// 时间复杂度：O(N) 空间复杂度：O(C)
class Solution {
  public String minWindow(String s, String t) {
    char[] cs = s.toCharArray(), ct = t.toCharArray();

    // 将字符串t中每个字母出现的次数统计出来，这里--可以理解为有这么多的坑要填
    int[] count = new int[128];
    for(char c:ct) count[c]--;

    String res = "";
    for(int i=0,j=0,cnt=0; i<cs.length; i++){
      // 利用字符cs[i]去填count数组的坑
      count[cs[i]]++;
      // 如果填完坑之后发现，坑没有满或者刚好满，那么这个填坑是有效的，否则如果坑本来就是满的，这次填坑是无效的
      // 注意其他非t中出现的字符，count数组的值是0，原来坑就是满的，那么填入count数组中，count[cs[i]]肯定大于0
      if(count[cs[i]]<=0) cnt++;
      // 如果cnt等于ct.length，那么说明窗口内已经包含t了，这时就要考虑移动左指针了，只有当左指针指向的字符是冗余的情况下，即count[cs[j]]>0，才能保证去掉该字符后，窗口中仍然包含t
      // 注意cnt达到字符串t的长度后，它的值就不会再变化了，因为窗口内包含t之后，就会一直包含
      while(cnt==ct.length && count[cs[j]]>0){
        count[cs[j]]--;
        j++;
      }
      // 当窗口内包含t后，计算此时窗口内字符串的长度，更新res
      if(cnt==ct.length){
        if(res.equals("") || res.length()>(i-j+1))
          res = s.substring(j, i+1);
      }
    }

    return res;
  }
}
```

## 19. 删除链表的倒数第 N 个结点

语言：java

思路：一般来说倒数，即需要从后往前数。但是从后往前数，这个动作不一定需要先遍历到最后一个节点再执行，提前数好倒数N个节点的窗口，然后挪动整个窗口，最后右边界在最后一个节点，我们就找到倒数第N个节点了。

代码（0ms，100%）：

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode() {}
 *     ListNode(int val) { this.val = val; }
 *     ListNode(int val, ListNode next) { this.val = val; this.next = next; }
 * }
 */
class Solution {
  public ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode result = new ListNode();
    ListNode lastN = result;
    result.next = head;
    head = result;
    while(n-- > 0) {
      head = head.next;
    }
    while(head.next != null){
      head = head.next;
      lastN = lastN.next;
    }
    lastN.next = lastN.next.next;
    return result.next;
  }
}
```

## 面试题02.07. 链表相交

语言：java

思路：以前做过，这个思路比较巧妙。两个链表长度可能不一致，如果要他们长度一致，那么就是两个链表各自遍历完自己的链表后，再遍历别人的链表。如果两个链表有交集，那么他们经过相同的路程长后（A链表+B链表），一定会相遇。

代码（1ms，97.42%）：

```java
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public class Solution {
  public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
    ListNode aNode = headA,bNode = headB;
    while(aNode!= null || bNode != null) {
      if(aNode == bNode) {
        return aNode;
      }
      aNode = aNode == null? headB : aNode.next;
      bNode = bNode == null? headA : bNode.next;
    }
    return null;
  }
}
```

参考代码1（1ms，97.42%）：思路更加清晰，其实遍历的次数也一样，就是先统计两个链表的长度，然后较长的一个先移动（长度差）个节点，之后再两个链表节点挨个判断是否相交。

```java
public class Solution {
  public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
    ListNode curA = headA;
    ListNode curB = headB;
    int lenA = 0, lenB = 0;
    while (curA != null) { // 求链表A的长度
      lenA++;
      curA = curA.next;
    }
    while (curB != null) { // 求链表B的长度
      lenB++;
      curB = curB.next;
    }
    curA = headA;
    curB = headB;
    // 让curA为最长链表的头，lenA为其长度
    if (lenB > lenA) {
      //1. swap (lenA, lenB);
      int tmpLen = lenA;
      lenA = lenB;
      lenB = tmpLen;
      //2. swap (curA, curB);
      ListNode tmpNode = curA;
      curA = curB;
      curB = tmpNode;
    }
    // 求长度差
    int gap = lenA - lenB;
    // 让curA和curB在同一起点上（末尾位置对齐）
    while (gap-- > 0) {
      curA = curA.next;
    }
    // 遍历curA 和 curB，遇到相同则直接返回
    while (curA != null) {
      if (curA == curB) {
        return curA;
      }
      curA = curA.next;
      curB = curB.next;
    }
    return null;
  }

}
```

## 349. 两个数组的交集

语言：java

思路：题目限制数值只会出现在0～1000，用数组代替Set存储出现过的数字，后续再统计哪些在另一个数组出现过就好了。

代码（1ms，98.83%）：

```java
class Solution {
  public int[] intersection(int[] nums1, int[] nums2) {
    int[] numMap = new int[1000];
    int count = 0;
    for(int i = 0,len = nums1.length; i< len; ++i) {
      numMap[nums1[i]] = 1;
    }
    for(int i = 0,len = nums2.length; i< len; ++i) {
      if(numMap[nums2[i]]==1) {
        numMap[nums2[i]] = 2;
        count+=1;
      }
    }
    int[] result = new int[count];
    for(int i= 0,j = 0;i<1000;++i){
      if(numMap[i]==2) {
        result[j++] = i;
      }
    }
    return result;
  }
}
```

## 383. 赎金信

语言：java

思路：记录第一个字符串出现的每个字符的数量；然后遍历另一个字符串，如果出现完相同数量的字符，就返回true

代码（1ms，99.92%）：

```java
class Solution {
  public boolean canConstruct(String ransomNote, String magazine) {
    int[] alphabet = new int[26];
    char[] aChars = ransomNote.toCharArray();
    char[] bChars = magazine.toCharArray();
    int count = aChars.length;
    for(int i = 0;i<count;++i){
      ++ alphabet[aChars[i]-'a'];
    }
    for(int i = 0,len = bChars.length; i < len; ++i) {
      int index = bChars[i]-'a';
      if(alphabet[index] > 0) {
        --alphabet[index];
        --count;
      }
      if(count==0) {
        return true;
      }
    }
    return false;
  }
}
```

## 18. 四数之和

语言：java

思路：第一反应和三数之和很像，想办法把四数之和化简成三数之和。先排序，然后这里双层for遍历nums获取`nums[i]+nums[j]`，右边的部分则还是靠双指针。主要需要注意的就是，怎么跳过相同的已经出现过的元祖数组。

代码（18ms，33.20%）：

```java
class Solution {
  public List<List<Integer>> fourSum(int[] nums, int target) {
    Arrays.sort(nums);
    List<List<Integer>> result = new ArrayList<>();
    for(int i = 0, len = nums.length;i< len-3;++i) {
      if(nums[i] > 0 && target <= 0) {
        return result;
      }
      while(i> 0 && i<len-3&&nums[i] == nums[i-1]) {
        ++i;
      } 
      for(int j = i+1; j< len-2;++j) {
        int left = j + 1;
        int right = len-1;
        while(right > left) {
          int sum = nums[i]+nums[j]+nums[left]+nums[right];
          if(sum == target) {
            result.add(Arrays.asList(nums[i],nums[j],nums[left],nums[right]));
            while(j< len-2 && nums[j] == nums[j+1]) {
              ++j;
            }
            while(right > left && nums[right] == nums[right-1]) {
              --right;
            } 
            while(right > left && nums[left] == nums[left+1]) {
              ++left;
            }
            --right;
            ++left;
          } else if(sum > target) {
            --right;
          } else{
            ++left;
          }
        }      
      } 
    }
    return result;
  }
}
```

参考代码1（2ms，100%）：一样的思路，就是减支做得更彻底

> [四数之和 - 四数之和 - 力扣（LeetCode）](https://leetcode.cn/problems/4sum/solution/si-shu-zhi-he-by-leetcode-solution/)

```java
class Solution {
  public List<List<Integer>> fourSum(int[] nums, int target) {
    List<List<Integer>> quadruplets = new ArrayList<List<Integer>>();
    if (nums == null || nums.length < 4) {
      return quadruplets;
    }
    Arrays.sort(nums);
    int length = nums.length;
    for (int i = 0; i < length - 3; i++) {
      if (i > 0 && nums[i] == nums[i - 1]) {
        continue;
      }
      if ((long) nums[i] + nums[i + 1] + nums[i + 2] + nums[i + 3] > target) {
        break;
      }
      if ((long) nums[i] + nums[length - 3] + nums[length - 2] + nums[length - 1] < target) {
        continue;
      }
      for (int j = i + 1; j < length - 2; j++) {
        if (j > i + 1 && nums[j] == nums[j - 1]) {
          continue;
        }
        if ((long) nums[i] + nums[j] + nums[j + 1] + nums[j + 2] > target) {
          break;
        }
        if ((long) nums[i] + nums[j] + nums[length - 2] + nums[length - 1] < target) {
          continue;
        }
        int left = j + 1, right = length - 1;
        while (left < right) {
          long sum = (long) nums[i] + nums[j] + nums[left] + nums[right];
          if (sum == target) {
            quadruplets.add(Arrays.asList(nums[i], nums[j], nums[left], nums[right]));
            while (left < right && nums[left] == nums[left + 1]) {
              left++;
            }
            left++;
            while (left < right && nums[right] == nums[right - 1]) {
              right--;
            }
            right--;
          } else if (sum < target) {
            left++;
          } else {
            right--;
          }
        }
      }
    }
    return quadruplets;
  }
}
```

## 541. 反转字符串 II

语言：java

思路：和反转字符串第一题差不多，区别就是多了些if条件而已。需要注意的是交换的边界(i+k-1)

代码（0ms，100%）：

```java
class Solution {
  public String reverseStr(String s, int k) {
    char[] chars = s.toCharArray();
    int len = chars.length;
    for(int i = 0;i<len;i+=2*k) {
      if(i+k > len) {
        reverse(chars, i, len-1);
        break;
      } else if(i+2*k >len) {
        reverse(chars, i, i+k-1);
        break;
      } else {
        reverse(chars, i, i+k-1);
      }
    }
    return new String(chars);
  }

  public void reverse(char[] array,int left, int right) {
    while(left < right) {
      array[left]^=array[right];
      array[right]^=array[left];
      array[left]^=array[right];
      ++left;
      --right;
    }
  }
}
```

## 151. 反转字符串中的单词

语言：java

思路：题目进阶要求中，说尝试O(1)空间复杂度的原地算法。题目包含两个要求，一、字符串中单词顺序反转；二、去除前后空格和中间多余空格（每个单词之间只有一个空格）。

（1）单词顺序反转，可以先反转整个字符串；然后在对中间每次遇到的字符串在反转一遍
（2）去除多余空格，双指针，一个用于找单词，一个用于存放单词（两个指针一开始都在字符串的第一个位置），然后存单词的指针，每次存完后额外往后跳一个空格位置。最后再删除最后额外多跳过的这个空格。

这里先删除多余空格，然后按要求反转所有单词的顺序

代码（2ms，96.82%）：

```java
class Solution {
  public String reverseWords(String s) {
    String noBlankStr = removeBlanks(s);
    char[] chars = noBlankStr.toCharArray();
    reverse(chars, 0, noBlankStr.length());
    int left = 0,right = 0, len = chars.length;
    while(right < len) {
      if(chars[right]==' '){
        reverse(chars,left,right);
        left = right + 1;
      }
      ++right;
    }
    //反转最后一个单词，因为上面遇到空格才会执行一次反转
    reverse(chars,left,right);
    return new String(chars);
  }

  public void reverse(char[] chars, int left,int right) {
    int l = left, r = right-1;
    while(l < r) {
      chars[l] ^= chars[r];
      chars[r] ^= chars[l];
      chars[l] ^= chars[r];
      ++l;
      --r;
    }
  }

  public String removeBlanks(String s) {
    int slow = 0,len = s.length() ,fast = len-1;
    boolean hasWord = false;
    // 先去掉尾部空格
    while(fast > 0 && s.charAt(fast) == ' '){
      --fast;
    }
    // 去掉头部空格
    while(slow < len && s.charAt(slow) == ' '){
      ++slow;
    }
    // 获取去掉头尾空格后的 char[]
    char[] chars = s.substring(slow, fast+1).toCharArray();
    fast = 0;
    slow = 0;
    len = chars.length;
    while(fast < len) {
      if(chars[fast]!=' '){
        chars[slow++] = chars[fast];
        hasWord = true;
      }else if(hasWord){
        chars[slow++] = ' ';
        hasWord = false;
      }
      ++fast;
    }
    // 中间可能还有空格，对我们来说有用的字符串刚好到[0,slow)
    return new String(chars).substring(0,slow);
  }
}
```

参考代码1（6ms，62.16%）：

```java
class Solution {
  /**
     * 不使用Java内置方法实现
     * <p>
     * 1.去除首尾以及中间多余空格
     * 2.反转整个字符串
     * 3.反转各个单词
     */
  public String reverseWords(String s) {
    // System.out.println("ReverseWords.reverseWords2() called with: s = [" + s + "]");
    // 1.去除首尾以及中间多余空格
    StringBuilder sb = removeSpace(s);
    // 2.反转整个字符串
    reverseString(sb, 0, sb.length() - 1);
    // 3.反转各个单词
    reverseEachWord(sb);
    return sb.toString();
  }

  private StringBuilder removeSpace(String s) {
    // System.out.println("ReverseWords.removeSpace() called with: s = [" + s + "]");
    int start = 0;
    int end = s.length() - 1;
    while (s.charAt(start) == ' ') start++;
    while (s.charAt(end) == ' ') end--;
    StringBuilder sb = new StringBuilder();
    while (start <= end) {
      char c = s.charAt(start);
      if (c != ' ' || sb.charAt(sb.length() - 1) != ' ') {
        sb.append(c);
      }
      start++;
    }
    // System.out.println("ReverseWords.removeSpace returned: sb = [" + sb + "]");
    return sb;
  }

  /**
     * 反转字符串指定区间[start, end]的字符
     */
  public void reverseString(StringBuilder sb, int start, int end) {
    // System.out.println("ReverseWords.reverseString() called with: sb = [" + sb + "], start = [" + start + "], end = [" + end + "]");
    while (start < end) {
      char temp = sb.charAt(start);
      sb.setCharAt(start, sb.charAt(end));
      sb.setCharAt(end, temp);
      start++;
      end--;
    }
    // System.out.println("ReverseWords.reverseString returned: sb = [" + sb + "]");
  }

  private void reverseEachWord(StringBuilder sb) {
    int start = 0;
    int end = 1;
    int n = sb.length();
    while (start < n) {
      while (end < n && sb.charAt(end) != ' ') {
        end++;
      }
      reverseString(sb, start, end - 1);
      start = end + 1;
      end = start + 1;
    }
  }
}
```

参考代码2（7ms，50.91%）：使用队列存储单词，然后重新拼接。

> [翻转字符串里的单词 - 反转字符串中的单词 - 力扣（LeetCode）](https://leetcode.cn/problems/reverse-words-in-a-string/solution/fan-zhuan-zi-fu-chuan-li-de-dan-ci-by-leetcode-sol/)

```java
class Solution {
  public String reverseWords(String s) {
    int left = 0, right = s.length() - 1;
    // 去掉字符串开头的空白字符
    while (left <= right && s.charAt(left) == ' ') {
      ++left;
    }

    // 去掉字符串末尾的空白字符
    while (left <= right && s.charAt(right) == ' ') {
      --right;
    }

    Deque<String> d = new ArrayDeque<String>();
    StringBuilder word = new StringBuilder();

    while (left <= right) {
      char c = s.charAt(left);
      if ((word.length() != 0) && (c == ' ')) {
        // 将单词 push 到队列的头部
        d.offerFirst(word.toString());
        word.setLength(0);
      } else if (c != ' ') {
        word.append(c);
      }
      ++left;
    }
    d.offerFirst(word.toString());

    return String.join(" ", d);
  }
}
```

## 28. 找出字符串中第一个匹配项的下标

语言：java

思路：一个指针遍历haystack数组，一个指针遍历needle数组。

+ 匹配时，两个指针都++。
+ 遇到不匹配时，
  + needle指针为0，则haystack指针继续遍历
  + needle指针>0（已经匹配过几个字符），haystack指针-=needle指针，从之前匹配的第一个字符的下一个位置重新匹配

代码（0ms，100%）：

```java
class Solution {
  public int strStr(String haystack, String needle) {
    int hPointer = 0, nPointer = 0,hLen = haystack.length(), nLen = needle.length();
    char[] hChars = haystack.toCharArray(), nChars = needle.toCharArray();
    while(hPointer < hLen) {
      if(nPointer == nLen) {
        return hPointer-nLen;
      }
      if(hChars[hPointer] == nChars[nPointer]) {
        ++nPointer;
      } else {
        if(nPointer > 0) {
          hPointer -= nPointer;
        }
        nPointer = 0;
      }
      ++hPointer;
    }
    return nPointer == nLen? hLen-nLen : -1;
  }
}
```

代码2（0ms，100%）：改用KMP算法，KMP果然是容易忘记的算法，哈哈。

```java
class Solution {
  public int strStr(String haystack, String needle) {
    int hLen = haystack.length();
    int nLen = needle.length();
    // 匹配串比原来字符串长，直接不用比了
    if(nLen > hLen) {
      return -1;
    }
    int[] next = getNextArray(needle);
    int i = 0,j = 0;
    while(i<hLen && j < nLen) {
      // 字符不匹配，匹配串的指针找到最近一次能匹配字符的位置
      while(j > 0 && haystack.charAt(i) != needle.charAt(j)) {
        j = next[j-1];
      }
      // 有字符匹配，则继续遍历
      if(haystack.charAt(i) == needle.charAt(j)) {
        ++j;
      }
      ++i;
    }
    return j == nLen ? i-j : -1;
  }

  public int[] getNextArray(String needle) {
    int len = needle.length(), j = 0;
    int[] next = new int[len];
    // i相当于是后缀串的指针(匹配的第一个字符相当于当前[0,i]字符串的后缀第一个字符)，只有长度>1才有后缀，所以从1开始
    for(int i = 1; i < len; ++i) {
      // 处理不相同时，j回退的情况。j相当于是前缀串的头指针
      while(j > 0 && needle.charAt(j) != needle.charAt(i)) {
        j = next[j-1];
      }
      if(needle.charAt(j) == needle.charAt(i)) {
        // 前后缀多个字符相同时，i和j相当于当时匹配的第一个位置[x,往后)的字符串的遍历指针
        next[i] = ++j;
      }
    }
    return next;
  }
}
```

## 459. 重复的子字符串

语言：java

思路：只想到比较暴力的写法，看了网络视频后得知可用KMP巧妙处理。用KMP求最大相同前后缀，然后求前后缀不重叠的部分，它就是组成重复串的最小字符串。如果整个字符串长度被该字符串长度整除，则说明该字符串都由该子串组成

> [字符串这么玩，可有点难度！ | LeetCode：459.重复的子字符串_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1cg41127fw/?spm_id_from=333.788&vd_source=ba4d176271299cb334816d3c4cbc885f)
>
> [代码随想录 (programmercarl.com)](https://programmercarl.com/0459.重复的子字符串.html#其他语言版本)

代码（8ms，81.71%）：

```java
class Solution {
  public boolean repeatedSubstringPattern(String s) {
    int len = s.length();
    int[] next = getNextArray(s);
    // "abac" ，next[len-1] > 0 保证 至少存在相同前后缀；后面一个判断则是判断是不是整个字符串由前后缀中不重叠的部分组成（这部分就是最小重复字符串）
    return next[len-1] > 0 && 0 == (len % (len - next[len-1]));
  }

  public int[] getNextArray(String s) {
    int j = 0,len = s.length();
    int[] next = new int[len];
    for(int i = 1; i < len; ++i) {
      while(j> 0 && s.charAt(j) != s.charAt(i)) {
        j = next[j-1];
      }
      if(s.charAt(j) == s.charAt(i)) {
        next[i] = ++j;
      }
    }
    return next;
  }
}
```

参考代码1（5ms，91.15%）：整体思路也是KMP求出重复的子串（理论上的），然后判断原字符串是否真由该字符串堆砌成

> [代码随想录 (programmercarl.com)](https://programmercarl.com/0459.重复的子字符串.html#其他语言版本)

```java
class Solution {
  public boolean repeatedSubstringPattern(String s) {
    if (s.equals("")) return false;

    int len = s.length();
    // 原串加个空格(哨兵)，使下标从1开始，这样j从0开始，也不用初始化了
    s = " " + s;
    char[] chars = s.toCharArray();
    int[] next = new int[len + 1];

    // 构造 next 数组过程，j从0开始(空格)，i从2开始
    for (int i = 2, j = 0; i <= len; i++) {
      // 匹配不成功，j回到前一位置 next 数组所对应的值
      while (j > 0 && chars[i] != chars[j + 1]) j = next[j];
      // 匹配成功，j往后移
      if (chars[i] == chars[j + 1]) j++;
      // 更新 next 数组的值
      next[i] = j;
    }

    // 最后判断是否是重复的子字符串，这里 next[len] 即代表next数组末尾的值
    if (next[len] > 0 && len % (len - next[len]) == 0) {
      return true;
    }
    return false;
  }
}
```

参考代码2（2ms，99.95%）：很巧妙，（1）基串能够整除；（2）基串在头和尾是一致的；（2）“只去掉头”和“只去掉尾”的字符串需要相等

```java
class Solution {
  public boolean repeatedSubstringPattern(String s) {
    int lens = s.length(), i = 0;
    while (++i < lens) {
      if (lens % i != 0) continue;
      if (s.substring(lens - i, lens).equals(s.substring(0, i))) // 判断x是不是基串
        if (s.substring(i, lens).equals(s.substring(0, lens - i))) return true; // 判断拿去x后是否相等
    }
    return false;
  }
}
```

## 20. 有效的括号

语言：java

思路：用栈做匹配即可，这里直接拿字符串的字符数组作为stack。

代码（0ms，100%）：
```java
class Solution {
  public boolean isValid(String s) {
    char[] chars = s.toCharArray();
    int i = 0, stackIndex = 0, len = chars.length;
    while(i < len) {
      switch(chars[i]) {
        case '(':
        case '[':
        case '{':
          chars[stackIndex++] = chars[i];
          break;
        case ')':
          if(stackIndex == 0 || chars[--stackIndex] != '(') {
            return false;
          }
          break;
        case ']':
          if(stackIndex == 0 || chars[--stackIndex] != '[') {
            return false;
          }
          break;
        case '}':
          if(stackIndex == 0 || chars[--stackIndex] != '{') {
            return false;
          }
          break;
      }
      ++i;
    }
    return stackIndex == 0;
  }
}
```

## 1047. 删除字符串中的所有相邻重复项

语言：java

思路：经典的栈消消乐，用原本的chars作为栈，然后消除相同元素即可。

代码（3ms，100%）：

```java
class Solution {
  public String removeDuplicates(String s) {
    char[] chars = s.toCharArray();
    int stackIndex = -1;
    // abb
    for(int i = 0, len = s.length(); i < len; ++i) {
      if(stackIndex >= 0 && chars[stackIndex] == chars[i]) {
        --stackIndex;
      } else {
        chars[++stackIndex] = chars[i];
      }
    }
    return new String(chars).substring(0,stackIndex+1);
  }
}
```

## 107. 二叉树的层序遍历 II

语言：java

思路：

+ 第一种方法，BFS利用队列从上到下层次遍历，最后翻转结果（就得到题目要求的自底向上的层次遍历）
+ 第二种方法，可以用DFS，递归时记录层数，最后翻转结果

代码1（方法1，BFS+翻转遍历结果）（1ms，91.77%）：

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
  public List<List<Integer>> levelOrderBottom(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if(null == root) {
      return result;
    }
    Queue<TreeNode> queue = new LinkedList<>();
    queue.add(root);
    while(!queue.isEmpty()) {
      int size = queue.size();
      List<Integer> tmp = new LinkedList<>();
      while(size-- > 0) {
        TreeNode cur = queue.poll();
        tmp.add(cur.val);
        if(null != cur.left) {
          queue.add(cur.left);
        }
        if(null != cur.right) {
          queue.add(cur.right);
        }
      }
      // 每次在头一个位置插入，相当于翻转
      result.add(0, tmp);
    }
    return result;
  }
}
```

代码2（DFS，记录层数，最后翻转结果）（0ms，100%）：

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
  public List<List<Integer>> levelOrderBottom(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if(null == root) {
      return result;
    }
    dfs(result, root, 1);
    Collections.reverse(result);
    return result;
  }

  public void dfs(List<List<Integer>> result, TreeNode root, int depth) {
    if(null == root) {
      return;
    }
    if(depth > result.size()) {
      result.add(new LinkedList<>());
    }
    result.get(depth-1).add(root.val);
    dfs(result,root.left, depth+1);
    dfs(result,root.right, depth+1);
  }
}
```

## 199. 二叉树的右视图

语言：java

思路：乍一看以为是只需要遍历最右侧节点，实际上是尽可能遍历"最靠右边"的节点。主要也是层次遍历，BFS、DFS都可以。

代码1（BFS，每层遍历时只添加最右边一个元素）（1ms，81.63%）：

```java
class Solution {
  public List<Integer> rightSideView(TreeNode root) {
    List<Integer> result = new LinkedList<>();
    while(null==root) {
      return result;
    }
    Queue<TreeNode> queue = new LinkedList<>();
    queue.add(root);
    while(!queue.isEmpty()) {
      int size = queue.size();
      // 每层只添加最右边的元素到结果集
      if(size > 0) {
        result.add(queue.peek().val);
      }
      while(size--> 0) {
        TreeNode cur = queue.poll();
        // 先右后左，保证下次result一定先取最右边元素
        if(cur.right!=null) {
          queue.add(cur.right);
        }
        if(cur.left!=null) {
          queue.add(cur.left);
        }
      }
    }
    return result;
  }
}
```

代码2（DFS，每层递归时只添加一个元素）(0ms，100%)：

```java
class Solution {
  public List<Integer> rightSideView(TreeNode root) {
    List<Integer> result = new LinkedList<>();
    while(null==root) {
      return result;
    }
    dfs(result, root, 1);
    return result;
  }

  public void dfs(List<Integer> result, TreeNode root, int depth) {
    if(null == root) {
      return;
    }
    // 保证每层只添加一个元素
    if(depth > result.size()) {
      result.add(root.val);
    }
    // 先右后左，保证尽量取最右边的元素
    dfs(result, root.right, depth+1);
    dfs(result, root.left, depth+1);
  }
}
```

## 637. 二叉树的层平均值

语言：java

思路：就是在层次遍历的基础上，BFS可直接求平均值。DFS则还需要维护层次遍历的求和List、每层的个数List，比较麻烦。

代码1（BFS）（2ms，95.02%）：

```java
class Solution {
  public List<Double> averageOfLevels(TreeNode root) {
    List<Double> result = new LinkedList<>();
    if(null == root) {
      return result;
    } 
    Queue<TreeNode> queue = new LinkedList<>();
    queue.add(root);
    while(!queue.isEmpty()) {
      int size = queue.size();
      // 这里 需要long，不然测试用例会有 两个MAX最大值相加的情况
      long sum = 0;
      for(int i = 0;i < size; ++i) {
        TreeNode cur = queue.poll();
        sum += cur.val;
        if(cur.left != null) {
          queue.add(cur.left);
        }
        if(cur.right != null) {
          queue.add(cur.right);
        }
      }
      result.add(sum/(double)size);
    }
    return result;
  }
}
```

代码2（DFS）（3ms，23.97%）：

```java
class Solution {
  public List<Double> averageOfLevels(TreeNode root) {
    if(null == root) {
      return new ArrayList<>();
    } 
    List<Double> result = new LinkedList<>();
    List<Double> sumList = new LinkedList<>();
    List<Integer> countList = new LinkedList<>();
    dfs(sumList, countList, root, 0);
    for(int i = 0, len = sumList.size();i<len; ++i) {
      result.add(sumList.get(i) / countList.get(i));
    }
    return result;
  }

  public void dfs(List<Double> sumList,List<Integer> countList, TreeNode root, int depth) {
    if(null == root) {
      return ;
    }
    if(depth == sumList.size()){
      sumList.add((double)root.val);
      countList.add(1);
    }else {
      sumList.set(depth, sumList.get(depth) + root.val);
      countList.set(depth, countList.get(depth) + 1);
    }
    dfs(sumList,countList, root.left, depth+1);
    dfs(sumList,countList, root.right, depth+1);
  }
}
```

参考代码（1ms，100%）：和我的DFS差不多，但是就是莫名要快一点。

```java
class Solution {
  public List<Double> averageOfLevels(TreeNode root) {
    List<Integer> counts = new ArrayList<Integer>();
    List<Double> sums = new ArrayList<Double>();
    dfs(root, 0, counts, sums);
    List<Double> averages = new ArrayList<Double>();
    int size = sums.size();
    for (int i = 0; i < size; i++) {
      averages.add(sums.get(i) / counts.get(i));
    }
    return averages;
  }

  public void dfs(TreeNode root, int level, List<Integer> counts, List<Double> sums) {
    if (root == null) {
      return;
    }
    if (level < sums.size()) {
      sums.set(level, sums.get(level) + root.val);
      counts.set(level, counts.get(level) + 1);
    } else {
      sums.add(1.0 * root.val);
      counts.add(1);
    }
    dfs(root.left, level + 1, counts, sums);
    dfs(root.right, level + 1, counts, sums);
  }
}
```

## 429. N 叉树的层序遍历

语言：java

思路：和普通的二叉树BFS没什么区别，借用队列实现；或者DFS也可以

代码1（BFS）（2ms，92.86%）：

```java
class Solution {
  public List<List<Integer>> levelOrder(Node root) {
    if(null == root) {
      return new ArrayList<>();
    }
    List<List<Integer>> result = new LinkedList<>();
    Queue<Node> queue = new LinkedList<>();
    queue.add(root);
    while(!queue.isEmpty()) {
      int size = queue.size();
      List<Integer> tmp = new LinkedList<>();
      while(size-- > 0) {
        Node cur = queue.poll();
        tmp.add(cur.val);
        if(!cur.children.isEmpty()) {
          queue.addAll(cur.children);
        }
      }
      result.add(tmp); 
    }
    return result;
  }
}
```

代码2（DFS）（1ms，94.95%）：

```java
class Solution {
  public List<List<Integer>> levelOrder(Node root) {
    if(null == root) {
      return new ArrayList<>();
    }
    List<List<Integer>> result = new LinkedList<>();
    dfs(result, Collections.singletonList(root), 0);
    return result;
  }

  public void dfs(List<List<Integer>> resultList, List<Node> root, int depth) {
    if(null == root || root.isEmpty()) {
      return;
    }
    if(depth== resultList.size()) {
      resultList.add(new LinkedList<>());
    }
    List<Integer> tmp = resultList.get(depth);
    ++depth;
    for(int i = 0, len = root.size();i < len ;++i) {
      tmp.add(root.get(i).val);
      dfs(resultList, root.get(i).children, depth);
    }
  }
}
```

## 515. 在每个树行中找最大值

语言：java

思路：还是层次遍历，因为找最大值需要每层都遍历一次。DFS、BFS都可以。

代码1（BFS）（2ms，83.87%）：

```java
class Solution {
  public List<Integer> largestValues(TreeNode root) {
    List<Integer> maxResult = new LinkedList<>();
    if(null == root) {
      return maxResult;
    }
    Queue<TreeNode> queue = new LinkedList<>();
    queue.add(root);
    while(!queue.isEmpty()) {
      int size = queue.size();
      int maxValue = Integer.MIN_VALUE;
      for(int i = 0;i<size; ++i) {
        TreeNode cur =  queue.poll();
        maxValue = Math.max(maxValue, cur.val);
        if(cur.left != null) {
          queue.add(cur.left);
        }
        if(cur.right != null) {
          queue.add(cur.right);
        }
      }
      maxResult.add(maxValue);
    }
    return maxResult;
  }
}
```

代码2（DFS）（1ms，96.48%）：

```java
class Solution {
  public List<Integer> largestValues(TreeNode root) {
    List<Integer> maxResult = new LinkedList<>();
    if(null == root) {
      return maxResult;
    }
    dfs(maxResult, root, 0);
    return maxResult;
  }

  public void dfs(List<Integer> maxResult, TreeNode root, int depth) {
    if(null == root) {
      return;
    }
    if(depth == maxResult.size()) {
      maxResult.add(Integer.MIN_VALUE);
    }
    if(maxResult.get(depth) < root.val) {
      maxResult.set(depth, root.val);
    }
    dfs(maxResult, root.left, depth + 1);
    dfs(maxResult, root.right, depth + 1);
  }
}
```

参考代码（0ms。100%）：一样是DFS，就是填充结果的地方和我不一样

```java
class Solution {
  public List<Integer> largestValues(TreeNode root) {
    if (root == null) {
      return new ArrayList<Integer>();
    }
    List<Integer> res = new ArrayList<Integer>();
    dfs(res, root, 0);
    return res;
  }

  public void dfs(List<Integer> res, TreeNode root, int curHeight) {
    if (curHeight == res.size()) {
      res.add(root.val);
    } else {
      res.set(curHeight, Math.max(res.get(curHeight), root.val));
    }
    if (root.left != null) {
      dfs(res, root.left, curHeight + 1);
    }
    if (root.right != null) {
      dfs(res, root.right, curHeight + 1);
    }
  }
}
```

## 117. 填充每个节点的下一个右侧节点指针 II

语言：java

思路：BFS

代码1（BFS）（1ms，66.71%）：

```java
class Solution {
  public Node connect(Node root) {
    if(null == root) {
      return root;
    }
    Queue<Node> queue = new LinkedList<>();
    queue.add(root);
    while(!queue.isEmpty()) {
      int size = queue.size();
      Node pre = null;
      while(size-- > 0) {
        Node cur = queue.poll();
        if(pre!=null) {
          pre.next = cur;
        }
        pre = cur;
        if(cur.left!=null) {
          queue.add(cur.left);
        }
        if(cur.right!=null) {
          queue.add(cur.right);
        }
      }
    }
    return root;
  }
}
```

参考代码1（0ms）：BFS，但是常量级空间。

```java
class Solution {
  public Node connect(Node root) {
    if (root == null)
      return root;
    //cur我们可以把它看做是每一层的链表
    Node cur = root;
    while (cur != null) {
      //遍历当前层的时候，为了方便操作在下一
      //层前面添加一个哑结点（注意这里是访问
      //当前层的节点，然后把下一层的节点串起来）
      Node dummy = new Node(0);
      //pre表示访下一层节点的前一个节点
      Node pre = dummy;
      //然后开始遍历当前层的链表
      while (cur != null) {
        if (cur.left != null) {
          //如果当前节点的左子节点不为空，就让pre节点
          //的next指向他，也就是把它串起来
          pre.next = cur.left;
          //然后再更新pre
          pre = pre.next;
        }
        //同理参照左子树
        if (cur.right != null) {
          pre.next = cur.right;
          pre = pre.next;
        }
        //继续访问这一行的下一个节点
        cur = cur.next;
      }
      //把下一层串联成一个链表之后，让他赋值给cur，
      //后续继续循环，直到cur为空为止
      cur = dummy.next;
    }
    return root;
  }
}
```

参考后重写（0ms）：

```java
class Solution {
  public Node connect(Node root) {
    if(null == root) {
      return null;
    }
    Node cur = root;
    // 外层while，深度遍历，最左边节点
    while(cur != null) {
      // 内层while，下一层的层级遍历 (downHead是临时节点，链表头指针)
      Node downHead = new Node(0);
      Node pre = downHead;
      while(cur!=null) {
        if(cur.left!=null) {
          pre.next = cur.left;
          pre = pre.next;
        }
        if(cur.right!=null) {
          pre.next = cur.right;
          pre = pre.next;
        }
        cur = cur.next;
      }
      // cur 遍历到下一层
      cur = downHead.next;
      // 释放临时节点内存
      downHead.next = null;
      downHead = null;
    }
    return root;
  }
}
```

## 111. 二叉树的最小深度

语言：java

思路：DFS，这里有个坑，这里的最小深度，首先要到某个节点其没有任何左右节点，然后才算深度。

代码（11ms，16.42%）：

```java
class Solution {
  public int minDepth(TreeNode root) {
    if(root == null) {
      return 0;
    }
    return dfs(root, 1);
  }

  public int dfs(TreeNode root, int depth) {
    if(root.left == null && root.right == null) {
      return depth;
    }
    if(root.left!=null && root.right!=null) {
      return Math.min(dfs(root.left, depth+1), dfs(root.right, depth+1));
    }else if(root.left == null) {
      return dfs(root.right,depth+1);
    } else return dfs(root.left,depth+1);
  } 
}
```

参考代码1（12ms，9.42%）：DFS

> [二叉树的最小深度 - 二叉树的最小深度 - 力扣（LeetCode）](https://leetcode.cn/problems/minimum-depth-of-binary-tree/solution/er-cha-shu-de-zui-xiao-shen-du-by-leetcode-solutio/)

```java
class Solution {
  public int minDepth(TreeNode root) {
    if (root == null) {
      return 0;
    }

    if (root.left == null && root.right == null) {
      return 1;
    }

    int min_depth = Integer.MAX_VALUE;
    if (root.left != null) {
      min_depth = Math.min(minDepth(root.left), min_depth);
    }
    if (root.right != null) {
      min_depth = Math.min(minDepth(root.right), min_depth);
    }

    return min_depth + 1;
  }
}
```

参考代码2（1ms）：BFS

```java
class Solution {
  public int minDepth(TreeNode root) {
    if (root == null) {
      return 0;
    }

    Queue<TreeNode> q = new LinkedList<>();
    //Set<TreeNode> visited = new HashSet<>();

    q.offer(root);
    int step = 1;

    while (!q.isEmpty()) {
      int size = q.size();
      for (int i = 0; i < size; i++) {
        TreeNode node = q.poll();
        if (node.left == null && node.right == null) {
          return step;
        }

        if (node.left != null) {
          q.offer(node.left);
        }

        if (node.right != null) {
          q.offer(node.right);
        }
      }
      step++;
    }

    return step;
  }
}
```

参考后重写（BFS）（0ms，100%）：
```java
class Solution {
  public int minDepth(TreeNode root) {
    if (root == null) {
      return 0;
    }
    int depth = 1;
    Queue<TreeNode> queue = new LinkedList<>();
    queue.add(root);
    while(!queue.isEmpty()) {
      int size = queue.size();
      while(size-- > 0) {
        TreeNode cur = queue.poll();
        if(cur.left == null && cur.right == null) {
          return depth;
        }
        if(cur.left != null) {
          queue.add(cur.left);
        }
        if(cur.right != null) {
          queue.add(cur.right);
        }
      }
      ++depth;
    }
    return depth;
  }
}
```

## 226. 翻转二叉树

语言：java

思路：DFS，边往下遍历的时候边做翻转

代码（0ms，100%）：

```java
class Solution {
  public TreeNode invertTree(TreeNode root) {
    if(null == root) {
      return root;
    }
    TreeNode tmp = root.right;
    root.right = root.left;
    root.left = tmp;
    invertTree(root.left);
    invertTree(root.right);
    return root;
  }
}
```

## 559. N 叉树的最大深度

语言：java

思路：最大深度 = 根节点的高度，求高度可以用后序遍历（深度则是前序遍历）

代码（DFS）（0ms，100%）：

```java
class Solution {
  public int maxDepth(Node root) {
    if(null == root) {
      return 0;
    }
    int max = 1;
    for(Node node : root.children) {
      max = Math.max(maxDepth(node)+1, max);
    }
    return max;
  }
}
```

## 222. 完全二叉树的节点个数

语言：java

思路：DFS遍历所有节点，每个节点计数。这里后序遍历就好了

代码1（DFS）（0ms）：

```java
class Solution {
  public int countNodes(TreeNode root) {
    if(null == root) return 0;
    return countNodes(root.left) + countNodes(root.right) + 1;
  }
}
```

代码2（0ms）：利用完全二叉树、满二叉树的定义计算节点数量。要点就是判断当前子树是不是满二叉树（判断方式即当前root的最左和最右向下遍历的深度一致）

```java
class Solution {
  public int countNodes(TreeNode root) {
    if(null == root) return 0;
    TreeNode left = root;
    TreeNode right = root;
    int lDepth = 0, rDepth = 0;
    // 当前节点到最左子节点深度
    while(left!=null) {
      left = left.left;
      ++lDepth;
    }
    // 当前节点到最右子节点深度
    while(right!=null) {
      right = right.right;
      ++rDepth;
    }
    // 左深度 = 右深度，表示当前子树是满二叉树
    if(lDepth == rDepth) {
      return (1 << lDepth) - 1;
    }
    // 非满二叉树的情况，像正常后序遍历一样求节点个数
    return countNodes(root.left) + countNodes(root.right) + 1;
  }
}
```

## 110. 平衡二叉树

语言：java

思路：DFS求左右子树高度差就好了

代码1（1ms，45.10%）：

```java
class Solution {
    public boolean isBalanced(TreeNode root) {
        if(null == root) return true;
      	// 当前节点满足 && 左 满足 &&  右 满足
        return Math.abs(dfs(root.left)-dfs(root.right)) <=1 && isBalanced(root.left) && isBalanced(root.right);
    }
		// 求树高度
    public int dfs(TreeNode root) {
        if(null == root) return 0;
        return Math.max(dfs(root.left), dfs(root.right)) + 1;
    }
}
```

代码2（0ms）：减枝优化逻辑，当向下求左右子树高度，发现某处已经不是平衡二叉树了，那么直接返回-1，表示整体已经不是平衡二叉树

```java
class Solution {
  public boolean isBalanced(TreeNode root) {
    return dfs(root) > -1;
  }

  public int dfs(TreeNode root) {
    if(null == root) return 0;
    int left = dfs(root.left) ;
    // 这里最好改成提前判断 left == -1 的情况，这样要是左子树已经判断不是平衡二叉树了，就不需要判断右子树了
    int right = dfs(root.right);
    if(left == -1 || right == -1 || Math.abs(left-right) > 1) {
      return -1;
    }
    return Math.max(left, right) + 1;
  }
}
```

参考代码1（0ms）：一样DFS，就是left提前判断是否平衡二叉树，减枝更彻底

> [110. 平衡二叉树 - 力扣（Leetcode）](https://leetcode.cn/problems/balanced-binary-tree/solutions/8737/balanced-binary-tree-di-gui-fang-fa-by-jin40789108/)

```java
class Solution {
  public boolean isBalanced(TreeNode root) {
    return recur(root) != -1;
  }

  private int recur(TreeNode root) {
    if (root == null) return 0;
    int left = recur(root.left);
    if(left == -1) return -1;
    int right = recur(root.right);
    if(right == -1) return -1;
    return Math.abs(left - right) < 2 ? Math.max(left, right) + 1 : -1;
  }
}
```

## 257. 二叉树的所有路径

语言：java

思路：DFS前序遍历

代码（1ms，100%）：

```java
class Solution {
  public List<String> binaryTreePaths(TreeNode root) {
    List<String> result = new LinkedList<>();
    if (null == root) {
      return result;
    }
    dfs(result, root, new LinkedList<>());
    return result;
  }

  public void dfs(List<String> result, TreeNode root, Deque<TreeNode> deque) {
    deque.addLast(root);
    if(root.left == null && root.right ==null) {
      StringBuilder sb = new StringBuilder();
      for(TreeNode cur : deque) {
        sb.append(cur.val).append("->");
      }
      sb.delete(sb.length()-2, sb.length());
      result.add(sb.toString());
    }
    if(null!=root.left) {
      dfs(result,root.left, deque);
    }
    if(null!= root.right) {
      dfs(result,root.right, deque);
    }
    deque.removeLast();
  }
}
```

## 404. 左叶子之和

语言：java

思路：求的是左叶子之和，所以可以考虑中/后序遍历DFS，左中右的中没有用，所以选左右中（后序）

代码（0ms，100%）：

```java
class Solution {
  public int sumOfLeftLeaves(TreeNode root) {
    if(null == root) {
      return 0;
    }
    return dfs(root,false);
  }

  public int dfs(TreeNode root, boolean isLeft) {
    if(null == root.left && null == root.right) {
      return isLeft ? root.val : 0;
    }
    int left = 0,right = 0;
    if(null != root.left) {
      left = dfs(root.left, true);
    }
    if(null!= root.right) {
      right = dfs(root.right, false);
    }
    return left + right;
  }
}
```

## 513. 找树左下角的值

语言：java

思路：最底层，其次最左边，层次优先，可以用BFS

代码1（BFS）（2ms，14.97%）：

```java
class Solution {
  public int findBottomLeftValue(TreeNode root) {
    if(null == root) {
      return 0;
    }
    int result = 0;
    Queue<TreeNode> queue = new LinkedList<>();
    queue.add(root);
    while(!queue.isEmpty()) {
      int size = queue.size();
      for(int i = 0;i <size;++i) {
        TreeNode cur = queue.poll();
        if(i==0) {
          result = cur.val;
        }
        if(null!=cur.left) {
          queue.add(cur.left);
        }
        if(null!=cur.right) {
          queue.add(cur.right);
        }
      }
    }
    return result;
  }
}
```

代码2（DFS）（0ms，100%）：后序遍历

```java
class Solution {
  public int findBottomLeftValue(TreeNode root) {
    if(null == root) {
      return 0;
    }
    int[] record = new int[2];
    dfs(root,1, record);
    return record[1];
  }

  public void dfs(TreeNode root,int depth, int[] depthValuePair) {
    if(null == root) {
      return;
    }
    dfs(root.left,depth+1,depthValuePair);
    dfs(root.right,depth+1,depthValuePair);
    if(depth > depthValuePair[0]) {
      depthValuePair[0] = depth;
      depthValuePair[1] = root.val;
    }
  }
}
```

## 113. 路径总和 II

语言：java

思路：后序遍历DFS

代码（1ms，100%）：

```java
class Solution {
  public List<List<Integer>> pathSum(TreeNode root, int targetSum) {
    List<List<Integer>> pathList = new LinkedList<>();
    if(null == root) {
      return pathList;
    }
    dfs(pathList, new LinkedList<>(), root, targetSum);
    return pathList;
  }

  public void dfs(List<List<Integer>> resultList,List<Integer> path, TreeNode root, int targetSum) {
    if(null == root.left && null == root.right) {
      if(targetSum == root.val) {
        List<Integer> tmp = new LinkedList<>();
        tmp.addAll(path);
        tmp.add(root.val);
        resultList.add(tmp);
      }
    }
    if(null != root.left) {
      path.add(root.val);
      dfs(resultList, path, root.left, targetSum-root.val);
      path.remove(path.size()-1);
    }
    if(null != root.right) {
      path.add(root.val);
      dfs(resultList, path, root.right, targetSum-root.val);
      path.remove(path.size()-1);
    }
  }
}
```

参考代码1：同样DFS，主要精简在添加节点到临时List和最后回溯弹出节点。

> [113. 路径总和 II - 力扣（Leetcode）](https://leetcode.cn/problems/path-sum-ii/solutions/427759/lu-jing-zong-he-ii-by-leetcode-solution/)

```java
class Solution {
  List<List<Integer>> ret = new LinkedList<List<Integer>>();
  Deque<Integer> path = new LinkedList<Integer>();

  public List<List<Integer>> pathSum(TreeNode root, int targetSum) {
    dfs(root, targetSum);
    return ret;
  }

  public void dfs(TreeNode root, int targetSum) {
    if (root == null) {
      return;
    }
    path.offerLast(root.val);
    targetSum -= root.val;
    if (root.left == null && root.right == null && targetSum == 0) {
      ret.add(new LinkedList<Integer>(path));
    }
    dfs(root.left, targetSum);
    dfs(root.right, targetSum);
    path.pollLast();
  }
}
```

## 106. 从中序与后序遍历序列构造二叉树

语言：java

思路：中序和后序的区别，就是 中和右节点的遍历顺序；通过后序遍历确定中节点，在中序遍历中找对应中节点的右子树，进而继续划分左右子树.

代码（3ms，39.79%）：主要麻烦点就是后续遍历，怎么确认下一次拆分的左区间和右区间

```java
class Solution {
  public TreeNode buildTree(int[] inorder, int[] postorder) {
    // 1、后序 找 中
    // 2、中序 划分 左/右
    // 3、构造中节点
    // 4、对于左/右子树，拆分新inorder[], postorder[]，向下继续划分
    return buildTree(inorder,postorder,0,inorder.length-1,0,postorder.length-1);
  }

  public TreeNode buildTree(int[] inorder, int[] postorder, int leftIn, int rightIn, int leftPost,int rightPost) {
    if(leftIn < 0 || rightIn >= inorder.length || rightIn < 0 || rightPost>= postorder.length || leftIn > rightIn || leftPost > rightPost) {
      return null;
    }
    int midValue = postorder[rightPost];
    // 中节点
    TreeNode mid = new TreeNode(midValue);
    // 中序遍历的 中节点 pos
    int midPos = pos(inorder,leftIn, rightIn, midValue);
    // 通过midPos可以得到 midPos左子树的个数，从而推算出 后续遍历前面几个节点是 属于左子树的
    mid.left = buildTree(inorder,postorder, leftIn, midPos-1, leftPost, leftPost+midPos-leftIn-1);
    mid.right = buildTree(inorder,postorder, midPos+1, rightIn, leftPost+midPos-leftIn, rightPost-1);
    return mid;
  }

  public int pos(int[] arr, int left, int right,int num) {
    while(left <= right) {
      if(arr[left]==num) {
        return left;
      }
      ++left;
    }
    return left;
  }
}
```

参考代码1（0ms）：有点过于抽象，难理解

```java
class Solution {
  int[] inorder, postorder;
  public TreeNode buildTree(int[] _inorder, int[] _postorder) {
    inorder = _inorder;
    postorder = _postorder;
    return recursion(postorder.length-1, postorder.length-1, 0);
  }
  public TreeNode recursion(int index, int start, int end){
    if(start < end){
      return null;
    }else if(start == end){
      return new TreeNode(postorder[index]);
    }
    TreeNode root = new TreeNode(postorder[index]);
    for(int i = start; i >= end; i--){
      if(inorder[i] == postorder[index]){
        index--;
        root.right = recursion(index, start, i+1);
        root.left = recursion(index + i - start, i-1, end);
        break;
      }
    }
    return root;
  }
}
```

参考代码2（1ms，99.58%）：

> [代码随想录 (programmercarl.com)](https://programmercarl.com/0106.从中序与后序遍历序列构造二叉树.html#java)

```java
class Solution {
  Map<Integer, Integer> map;  // 方便根据数值查找位置
  public TreeNode buildTree(int[] inorder, int[] postorder) {
    map = new HashMap<>();
    for (int i = 0; i < inorder.length; i++) { // 用map保存中序序列的数值对应位置
      map.put(inorder[i], i);
    }

    return findNode(inorder,  0, inorder.length, postorder,0, postorder.length);  // 前闭后开
  }

  public TreeNode findNode(int[] inorder, int inBegin, int inEnd, int[] postorder, int postBegin, int postEnd) {
    // 参数里的范围都是前闭后开
    if (inBegin >= inEnd || postBegin >= postEnd) {  // 不满足左闭右开，说明没有元素，返回空树
      return null;
    }
    int rootIndex = map.get(postorder[postEnd - 1]);  // 找到后序遍历的最后一个元素在中序遍历中的位置
    TreeNode root = new TreeNode(inorder[rootIndex]);  // 构造结点
    int lenOfLeft = rootIndex - inBegin;  // 保存中序左子树个数，用来确定后序数列的个数
    root.left = findNode(inorder, inBegin, rootIndex,
                         postorder, postBegin, postBegin + lenOfLeft);
    root.right = findNode(inorder, rootIndex + 1, inEnd,
                          postorder, postBegin + lenOfLeft, postEnd - 1);

    return root;
  }
}
```

## 654. 最大二叉树

语言：java

思路：每次先构造根节点，然后才是左右节点。即前序DFS，构造二叉树都是前序

代码（2 ms，87.67%）：

```java
class Solution {
  public TreeNode constructMaximumBinaryTree(int[] nums) {
    return null == nums || nums.length ==0 ? null : dfs(nums, 0, nums.length-1);
  }


  public TreeNode dfs(int[] nums,int left,int right) {
    if(left > right) {
      return null;
    }
    if(left == right) {
      return new TreeNode(nums[left]);
    }
    int maxIndex = maxIndex(nums,left,right);
    TreeNode root = new TreeNode(nums[maxIndex]);
    root.left = dfs(nums, left, maxIndex-1);
    root.right = dfs(nums, maxIndex+1, right);
    return root;
  }

  public int maxIndex(int[] nums, int left, int right) {
    int maxIndex = left;
    while(left <= right) {
      if(nums[left] > nums[maxIndex]) {
        maxIndex = left;
      }
      ++left;
    }
    return maxIndex;
  }
}
```

## 617.合并二叉树

语言：java

思路：简单的前序遍历DFS

代码（0ms，100%）：

```java
class Solution {
  public TreeNode mergeTrees(TreeNode root1, TreeNode root2) {
    if(null == root1) {
      return root2;
    }
    if(null == root2) {
      return root1;
    }
    root1.val += root2.val;
    root1.left = mergeTrees(root1.left, root2.left);
    root1.right = mergeTrees(root1.right, root2.right);
    return root1;
  }
}
```

## 700.二叉搜索树中的搜索

语言：java

思路：前序遍历DFS即可

代码1（0ms，100%）：

```java
class Solution {
  public TreeNode searchBST(TreeNode root, int val) {
    if(null == root) {
      return null;
    }
    if(root.val == val) {
      return root;
    }
    TreeNode left = searchBST(root.left, val);
    return left != null ? left : searchBST(root.right, val);
  }
}
```

代码2（0ms，100%）：利用二叉搜索树性质，减枝搜索

```java
class Solution {
  public TreeNode searchBST(TreeNode root, int val) {
    if(null == root) {
      return null;
    }
    if(root.val == val) {
      return root;
    } 
    return root.val > val ? searchBST(root.left, val) : searchBST(root.right, val) ;
  }
}
```

代码3（0ms，100%）：迭代法

```java
class Solution {
  public TreeNode searchBST(TreeNode root, int val) {
    while(null != root) {
      if(root.val == val) break;
      root = root.val > val ? root.left : root.right;
    }
    return root;
  }
}
```

## 501. 二叉搜索树中的众数

语言：java

思路：简单思路就是直接Map存每个数字出现次数，以及某数字出现次数最大值；复杂点的，可以去掉Map，用一个List存储众数（中间实时更新该List）

代码（0ms，100%）：

```java
class Solution {
  int maxCount = 0;
  int curCount = 0;
  List<Integer> tmpList = new ArrayList<>();
  TreeNode pre;
  public int[] findMode(TreeNode root) {
    dfs(root);
    int[] result = new int[tmpList.size()];
    for(int i =0;i< tmpList.size();++i) {
      result[i] = tmpList.get(i);
    }
    return result;
  }

  public void dfs(TreeNode root) {
    if(null == root) {
      return;
    }
    dfs(root.left);
    // 如果上一个为空，说明现在遍历第一个非节点
    if(pre==null) {
      curCount = 1;
      // 与上一相同，则继续 计数
    } else if (pre.val == root.val) {
      curCount +=1;
      // 与上一个不同，重新计数（这个是二叉搜索树，后面肯定不会再出现和pre.val一样的root.val）
    } else {
      curCount = 1;
    }
    // 出现新的 计数最大的众数，清空之前维护的众数集合
    if(curCount > maxCount) {
      tmpList.clear();
      tmpList.add(root.val);
      maxCount = curCount;
      // 与现有的众树出现次数相同
    }else if(curCount == maxCount) {
      tmpList.add(root.val);
    }
    pre = root;
    dfs(root.right);
  }
}
```

## 236. 二叉树的最近公共祖先

语言：java

思路：需要回溯才返回结果，所以后序（左右中）；用一个额外节点表示已经找到result，减枝返回

代码（6ms，99.99%）：

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
  TreeNode result = null; // 公共祖先

  public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if(result!= null) {
      return result;
    }
    if(root == null) {
      return null;
    }
    if(root == p || root == q) {
      return root;
    }
    TreeNode left = lowestCommonAncestor(root.left,p,q);
    if(null!= result ) {
      return result;
    }
    if(left == p && root == q || left == q && root == p) {
      result = root;
      return root;
    }
    TreeNode right = lowestCommonAncestor(root.right,p,q);
    if(null!= result ) {
      return result;
    }
    if(right == p && root == q || right == q && root == p) {
      result = root;
      return root;
    }
    if(left == p && right == q || left == q && right == p) {
      result = root;
      return root;
    }
    return left != null? left : right;
  }
}
```

参考代码（6ms，99.99%）：

> [236. 二叉树的最近公共祖先 - 力扣（Leetcode）](https://leetcode.cn/problems/lowest-common-ancestor-of-a-binary-tree/solutions/240096/236-er-cha-shu-de-zui-jin-gong-gong-zu-xian-hou-xu/)

```java
class Solution {
  public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if(root == null || root == p || root == q) return root;
    TreeNode left = lowestCommonAncestor(root.left, p, q);
    TreeNode right = lowestCommonAncestor(root.right, p, q);
    if(left == null && right == null) return null; // 1.
    if(left == null) return right; // 3.
    if(right == null) return left; // 4.
    return root; // 2. if(left != null and right != null)
  }
}
```

## 235. 二叉搜索树的最近公共祖先

语言：java

思路：整体还是后序DFS（左右中），因为需要回溯才能返回数据；通过二叉搜索树的性质，可以提前减枝（提前确定接下去是左子树还是右子树）。

代码1（DFS）（5ms，99.98%）

```java
class Solution {
  public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    if(null==root || p == root || q == root ) {
      return root;
    }
    if(root.val < p.val && root.val < q.val) {
      return lowestCommonAncestor(root.right, p , q);
    }
    if(root.val > p.val && root.val > q.val) {
      return lowestCommonAncestor(root.left, p , q);
    }
    return root;
  }
}
```

代码2（迭代法）（5ms，99.98%）：
```java
class Solution {
  public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
    while(null != root) {
      if(root.val > p.val && root.val > q.val) {
        root = root.left;
      }
      else if(root.val < p.val && root.val < q.val) {
        root = root.right;
      } else {
        return root;
      }
    }
    return null;
  }
}
```

## 701. 二叉搜索树中的插入操作

语言：java

思路：搜索树（左右中）后序DFS或者迭代法。其实某种意义上，相当于找到目标节点，在原二叉搜索树中应该在的位置。

代码1（后序DFS）（0ms，100%）：

```java
class Solution {
  public TreeNode insertIntoBST(TreeNode root, int val) {
    if(null == root) {
      return new TreeNode(val);
    }
    dfs(root,val);
    return root;
  }

  public void dfs(TreeNode root, int val) {
    if(null == root) {
      return;
    }
    if(root.val > val) {
      if(root.left != null) {
        dfs(root.left, val);
      }else {
        root.left = new TreeNode(val);
        return;
      }    
    } 
    if(root.val < val) {
      if(root.right != null) {
        dfs(root.right, val);
      }else {
        root.right = new TreeNode(val);
        return;
      }    
    }
  }
}
```

代码2（迭代法）（0ms，100%）：

```java
class Solution {
  public TreeNode insertIntoBST(TreeNode root, int val) {
    if(null == root) {
      return new TreeNode(val);
    }
    TreeNode result = root;
    while(null != root) {
      if(root.val > val) {
        if(root.left !=null) {
          root = root.left;
          continue;
        } else {
          root.left = new TreeNode(val);
          break;
        }
      }
      if(root.val < val) {
        if(root.right !=null) {
          root = root.right;
          continue;
        } else {
          root.right = new TreeNode(val);
          break;
        }
      }
    }
    return result;
  }
}
```

参考代码1（递归法）（0ms，100%）：更加简洁。每次向下递归相当于重新构造子树

> [代码随想录 (programmercarl.com)](https://programmercarl.com/0701.二叉搜索树中的插入操作.html#java)

```java
class Solution {
  public TreeNode insertIntoBST(TreeNode root, int val) {
    if (root == null) // 如果当前节点为空，也就意味着val找到了合适的位置，此时创建节点直接返回。
      return new TreeNode(val);

    if (root.val < val){
      root.right = insertIntoBST(root.right, val); // 递归创建右子树
    }else if (root.val > val){
      root.left = insertIntoBST(root.left, val); // 递归创建左子树
    }
    return root;
  }
}
```

参考代码2（迭代法）（0ms，100%）：减少中间null值判断，最后再判断一次插入位置，但需要额外维护指针

> [代码随想录 (programmercarl.com)](https://programmercarl.com/0701.二叉搜索树中的插入操作.html#java)

```java
class Solution {
  public TreeNode insertIntoBST(TreeNode root, int val) {
    if (root == null) return new TreeNode(val);
    TreeNode newRoot = root;
    TreeNode pre = root;
    while (root != null) {
      pre = root;
      if (root.val > val) {
        root = root.left;
      } else if (root.val < val) {
        root = root.right;
      } 
    }
    if (pre.val > val) {
      pre.left = new TreeNode(val);
    } else {
      pre.right = new TreeNode(val);
    }

    return newRoot;
  }
}
```

## 450. 删除二叉搜索树中的节点

语言：java

思路：DFS题目相当于（1）删除指定节点；（2）用两个子树构造二叉搜索树

代码（DFS找节点，迭代合并子树构造二叉搜索树）（0ms，100%）：

```java
class Solution {
  public TreeNode deleteNode(TreeNode root, int key) {
    if(null == root) {
      return root;
    }
    if(root.val > key) {
      root.left = deleteNode(root.left, key);
    }
    else if(root.val < key) {
      root.right = deleteNode(root.right, key);
    } else {
      return mergeTreeNode(root.left, root.right);
    }
    return root;
  }

  public TreeNode mergeTreeNode(TreeNode left, TreeNode right) {
    if(right == null) {
      return left;
    }
    TreeNode tmpLeft = right;
    while(tmpLeft.left !=null) {
      tmpLeft = tmpLeft.left;
    }
    tmpLeft.left = left;
    return right;
  }
}
```

参考代码1（迭代法）（0ms，100%）：

> [代码随想录 (programmercarl.com)](https://programmercarl.com/0450.删除二叉搜索树中的节点.html#迭代法)

```java
class Solution {
  public TreeNode deleteNode(TreeNode root, int key) {
    root = delete(root,key);
    return root;
  }

  private TreeNode delete(TreeNode root, int key) {
    if (root == null) return null;

    if (root.val > key) {
      root.left = delete(root.left,key);
    } else if (root.val < key) {
      root.right = delete(root.right,key);
    } else {
      if (root.left == null) return root.right;
      if (root.right == null) return root.left;
      TreeNode tmp = root.right;
      while (tmp.left != null) {
        tmp = tmp.left;
      }
      root.val = tmp.val;
      root.right = delete(root.right,tmp.val);
    }
    return root;
  }
}
```

参考代码2（递归法）（0ms，100%）：

> [代码随想录 (programmercarl.com)](https://programmercarl.com/0450.删除二叉搜索树中的节点.html#迭代法)

```java
class Solution {
  public TreeNode deleteNode(TreeNode root, int key) {
    if (root == null) return root;
    if (root.val == key) {
      if (root.left == null) {
        return root.right;
      } else if (root.right == null) {
        return root.left;
      } else {
        TreeNode cur = root.right;
        while (cur.left != null) {
          cur = cur.left;
        }
        cur.left = root.left;
        root = root.right;
        return root;
      }
    }
    if (root.val > key) root.left = deleteNode(root.left, key);
    if (root.val < key) root.right = deleteNode(root.right, key);
    return root;
  }
}
```

## 669. 修剪二叉搜索树

语言：java

思路：遍历的节点在区间外时，尝试寻找其符合区间的子树；其他情况和"删除二叉搜索树中的节点"基本一致。这里用DFS重新构造遍历的子树，只保留符合条件的节点。

代码1（DFS）（0ms，100%）：

```java
class Solution {
  public TreeNode trimBST(TreeNode root, int low, int high) {
    if(null == root) {
      return root;
    }
    if(root.val < low) {
      return trimBST(root.right, low, high);
    }
    if(root.val > high ) {
      return trimBST(root.left, low, high);
    }
    root.left = trimBST(root.left, low, high);
    root.right = trimBST(root.right, low, high);
    return root;
  }
}
```

代码2（迭代法）（0ms，100%）：先找到符合区间的root，然后裁剪这个root的左右子树

```java
class Solution {
  public TreeNode trimBST(TreeNode root, int low, int high) {
    // 寻找符合区间的 根 节点
    while(null != root && (root.val < low || root.val > high)) {
      if(root.val < low) {
        root = root.right;
      }
      else if(root.val > high) {
        root = root.left;
      }
    }
    if(null == root) {
      return root;
    }
    // 对 左子树减枝
    TreeNode cur = root;
    while(cur!=null && cur.left!=null) {
      if(cur.left.val  < low) {
        cur.left = cur.left.right;
        continue;
      }
      cur = cur.left;
    }
    cur = root;
    // 对 右子树减枝
    while(cur !=null && cur.right!=null) {
      if(cur.right.val > high) {
        cur.right = cur.right.left;
        continue;
      }
      cur = cur.right;
    }
    return root;
  }
}
```

参考代码1（0ms，100%）：迭代法，先找到符合区间的root；然后对左右子树进行减枝

> [669. 修剪二叉搜索树 - 力扣（Leetcode）](https://leetcode.cn/problems/trim-a-binary-search-tree/solutions/1813384/xiu-jian-er-cha-sou-suo-shu-by-leetcode-qe7q1/)

```java
class Solution {
  public TreeNode trimBST(TreeNode root, int low, int high) {
    while (root != null && (root.val < low || root.val > high)) {
      if (root.val < low) {
        root = root.right;
      } else {
        root = root.left;
      }
    }
    if (root == null) {
      return null;
    }
    for (TreeNode node = root; node.left != null; ) {
      if (node.left.val < low) {
        node.left = node.left.right;
      } else {
        node = node.left;
      }
    }
    for (TreeNode node = root; node.right != null; ) {
      if (node.right.val > high) {
        node.right = node.right.left;
      } else {
        node = node.right;
      }
    }
    return root;
  }
}
```

## 538. 把二叉搜索树转换为累加树

语言：java

思路：想像成有序数组，则从右往左，每个数字加上前面的数字和。这里用一个pre指针模拟该过程（右中左）

代码1（DFS）（0ms，100%）：

```java
class Solution {
  TreeNode pre = null;
  public TreeNode convertBST(TreeNode root) {
    if(null == root) {
      return null;
    }
    dfs(root);
    return root;
  }

  public void dfs(TreeNode root) {
    if(null == root) {
      return;
    }
    dfs(root.right);
    if(pre!=null) {
      root.val += pre.val;
    }
    pre = root;
    dfs(root.left);
  }
}
```

代码2（迭代法）（0ms，100%）：拆解问题就是后中左遍历，然后每次遍历到的节点val变成之前遍历的总和+自己val

```java
class Solution {
  int pre = 0;
  public TreeNode convertBST(TreeNode root) {
    if(null == root) {
      return null;
    }
    Deque<TreeNode> stack = new LinkedList<>();
    TreeNode cur = root;
    while(null != cur || !stack.isEmpty()) {
      if(cur!=null) {
        stack.addLast(cur);
        cur = cur.right;
      } else {
        cur = stack.pollLast();
        cur.val += pre;
        pre = cur.val;
        cur = cur.left;
      }
    }
    return root;
  }
}
```

参考代码1：迭代法

> [代码随想录 (programmercarl.com)](https://programmercarl.com/0538.把二叉搜索树转换为累加树.html#递归)

```c++
class Solution {
  private:
  int pre; // 记录前一个节点的数值
  void traversal(TreeNode* root) {
    stack<TreeNode*> st;
    TreeNode* cur = root;
    while (cur != NULL || !st.empty()) {
      if (cur != NULL) {
        st.push(cur);
        cur = cur->right;   // 右
      } else {
        cur = st.top();     // 中
        st.pop();
        cur->val += pre;
        pre = cur->val;
        cur = cur->left;    // 左
      }
    }
  }
  public:
  TreeNode* convertBST(TreeNode* root) {
    pre = 0;
    traversal(root);
    return root;
  }
};
```

## 77. 组合

语言：java

思路：回溯遍历每一种情况，然后存储。因为是组合，避免重复，所以让后续的数字都比之前的大

代码（13ms，13.95%）：

```java
class Solution {
  public List<List<Integer>> combine(int n, int k) {
    List<List<Integer>> result = new ArrayList<>();
    trackBacking(result, new LinkedList<>(), n,1,k);
    return result;
  }

  public void trackBacking(List<List<Integer>> result,List<Integer> path, int n,int start, int k) {
    if(k == 0) {
      result.add(new ArrayList<>(path));
      return;
    }
    for(int i = start;i <= n;++i) {
      path.add(i);
      trackBacking(result, path, n,i+1, k-1);
      path.remove(path.size()-1);
    }
  }
}
```

参考代码1（1ms，99.99%）：整体思路一致，就是提前减枝。找规律得出 `begin > n - k + 1` 就没必要再往下递归了

> [77. 组合 - 力扣（Leetcode）](https://leetcode.cn/problems/combinations/solutions/13436/hui-su-suan-fa-jian-zhi-python-dai-ma-java-dai-ma-/)

```java
import java.util.ArrayDeque;
import java.util.ArrayList;
import java.util.Deque;
import java.util.List;

public class Solution {

  public List<List<Integer>> combine(int n, int k) {
    List<List<Integer>> res = new ArrayList<>();
    if (k <= 0 || n < k) {
      return res;
    }

    // 为了防止底层动态数组扩容，初始化的时候传入最大长度
    Deque<Integer> path = new ArrayDeque<>(k);
    dfs(1, n, k, path, res);
    return res;
  }

  private void dfs(int begin, int n, int k, Deque<Integer> path, List<List<Integer>> res) {
    if (k == 0) {
      res.add(new ArrayList<>(path));
      return;
    }

    // 基础版本的递归终止条件：if (begin == n + 1) {
    if (begin > n - k + 1) {
      return;
    }
    // 不选当前考虑的数 begin，直接递归到下一层
    dfs(begin + 1, n, k, path, res);

    // 不选当前考虑的数 begin，递归到下一层的时候 k - 1，这里 k 表示还需要选多少个数
    path.addLast(begin);
    dfs(begin + 1, n, k - 1, path, res);
    // 深度优先遍历有回头的过程，因此需要撤销选择
    path.removeLast();
  }
}
```

## 216. 组合总和 III

语言：java

思路：回溯，剪枝方式：如果和已经超过预期值n或者数字个数超过k则直接返回

代码（0ms，100%）：

```java
class Solution {
  List<List<Integer>> result = new ArrayList<>();
  public List<List<Integer>> combinationSum3(int k, int n) {
    backTracking(new LinkedList<>(), 1, k, n);
    return result;
  }

  public void backTracking(List<Integer> path,int start ,int k, int n) {
    if(k == 0) {
      if(n==0) {
        result.add(new ArrayList<>(path));
      }
      return;
    }
    for(int i = start; i <= 9 && n - i >=0; ++i) {
      path.add(i);
      backTracking(path, i+1,k-1,n - i);
      path.remove(path.size()-1);
    }
  }
}
```

## 93. 复原 IP 地址

语言：java

思路：回溯。减枝思路：

+ 整体字符串长度 < 4 或者 > 12，那么肯定是无效IP（最小IP：0000，最大255255255255）
+ 中间切割IP时，单个IP切割后，剩余字符串长度应该 <= 还需切割的IP子个数 *3，比如第一个数字获取后，剩下三个数字要获取，后面剩余的字符串长度 最多不超过 3 * 3 = 9。（因为每个子数字都是最多3位）
+ 每个回溯过程，i最多遍历3位数字，因为每个IP子数字最多255
+ 每个子数字，0开头只有1位才生效，数字超过1位则0开头肯定无效。
+ 每个子数字，只能是0～255的值。

代码（1ms，95.75%）：

```java
class Solution {
  List<String> result = new LinkedList<>();
  public List<String> restoreIpAddresses(String s) {
    // 0000, 255255255255 两个极限情况
    if(s.length() < 4 || s.length() > 12) {
      return result;
    }
    char[] chars = s.toCharArray();
    backTracking(chars, 0, chars.length, 0, new int[4]);
    return result;
  }

  // path 存的是 切割的 右边界Index
  public void backTracking(char[] chars, int start, int end, int numberIndex, int[] path) {
    if(numberIndex == 4) {
      StringBuilder sb = new StringBuilder();
      for(int i = 0,pre= 0;i < 4; ++i) {
        sb.append(new String(chars,pre, path[i]-pre+1));
        if(i < 3 ) {
          sb.append(".");
        }
        pre = path[i]+1;
      }
      result.add(sb.toString());
      return;
    }
    for(int i = start; i < start + 3 && i < end; ++i) {
      if(end - i -1 <= (4-numberIndex-1) * 3 && isValid(chars, start, i)) {
        path[numberIndex] = i;
        backTracking(chars, i+1, end, numberIndex+1, path);
      }
    }
  }

  public boolean isValid(char[] chars,int start, int end) {
    // 仅一个数字 (0～9)
    if(start == end) {
      return true;
    }
    // 超过1个数字，并且开头是0，不合法
    if(chars[start] == '0') {
      return false;
    }
    // 超过3个数字，肯定不合法
    if(end-start+1 > 3) {
      return false;
    }
    // 求数字和，是否在有效的 1～255 内 
    int num = 0;
    while(start <= end) {
      num *= 10;
      num += chars[start++] - '0';
    }
    return num <= 255;
  }
} 
```

## 90. 子集 II

语言：java

思路：回溯。关键字不重复，所以先排序，后面跳过重复元素。

代码（1ms，99.76%）：

```java
class Solution {
  List<List<Integer>> result = new LinkedList<>();
  public List<List<Integer>> subsetsWithDup(int[] nums) {
    // 不重复，一般就需要先排序，方便跳过重复的元素
    Arrays.sort(nums);
    backTracking(nums, new LinkedList<>(), 0, nums.length);
    return result;
  }

  public void backTracking(int[] nums, List<Integer> path, int start, int end) {
    result.add(new LinkedList<>(path));
    for(int i = start; i < end; ++i) {
      path.add(nums[i]);
      backTracking(nums, path, i+1, end);
      path.remove(path.size()-1);
      while(i+1 < end && nums[i] == nums[i+1]) {
        ++i;
      }
    }
  }
}
```

## 491. 递增子序列

语言：java

思路：要求至少两个元素，即每次遍历的节点满2个节点就可以添加到结果集中。本题是子序列，还不能直接通过排序再跳过重复元素去重。`[1,2,3,1,1]`可能出现开始的1和后面任意一个1组成`[1,1]`，然后后面单独两个1，又重复组成`[1,1]`的情况。这里改成每层遍历时，使用一个Set临时存储当前层遍历过的元素，重复的就不再遍历

代码（6ms，26.51%）：

```java
class Solution {
  List<List<Integer>> result = new LinkedList<>();
  public List<List<Integer>> findSubsequences(int[] nums) {
    backTracking(nums, new LinkedList<>(), 0, nums.length);
    return result;
  }

  public void backTracking(int[] nums, List<Integer> path,int start, int end) {
    if(path.size() > 1) {
      result.add(new LinkedList<>(path));
    }
    Set<Integer> usedSet = new HashSet<>();
    for(int i = start; i < end; ++i) {
      if(usedSet.contains(nums[i])) {
        continue;
      }
      if(path.size() == 0 || nums[i] >= path.get(path.size()-1)) {
        usedSet.add(nums[i]);
        path.add(nums[i]);
        backTracking(nums, path, i+1, end);
        path.remove(path.size()-1);
      }
    }
  }
}
```

参考代码1（5ms，62.31%）：用数组代替Set做去重。

> [代码随想录 (programmercarl.com)](https://programmercarl.com/0491.递增子序列.html#总结)

```java
class Solution {
  private List<Integer> path = new ArrayList<>();
  private List<List<Integer>> res = new ArrayList<>();
  public List<List<Integer>> findSubsequences(int[] nums) {
    backtracking(nums,0);
    return res;
  }

  private void backtracking (int[] nums, int start) {
    if (path.size() > 1) {
      res.add(new ArrayList<>(path));
    }

    int[] used = new int[201];
    for (int i = start; i < nums.length; i++) {
      if (!path.isEmpty() && nums[i] < path.get(path.size() - 1) ||
          (used[nums[i] + 100] == 1)) continue;
      used[nums[i] + 100] = 1;
      path.add(nums[i]);
      backtracking(nums, i + 1);
      path.remove(path.size() - 1);
    }
  }
}
```

参考代码2（4ms，89.7%）：其实光说思路，和我的基本一样，就是写法有点不一样。利用了Set.add的返回值，有做contains判断的特性。

```java
class Solution {
  List<List<Integer>> lists = new ArrayList<>();

  public List<List<Integer>> findSubsequences(int[] nums) {
    dfs(new ArrayList<>(), 0, nums);
    return lists;
  }

  // 含有2个元素，3个，4个
  private void dfs(ArrayList<Integer> list, int left, int[] nums) {
    if (list.size() >= 2) {
      lists.add(new ArrayList<>(list));
    }

    Set<Integer> set = new HashSet<>();
    for (int i = left; i < nums.length; i++) {
      if (!set.add(nums[i])) {
        continue;
      }

      if (list.size() == 0 || list.get(list.size() - 1) <= nums[i]) {
        list.add(nums[i]);
        dfs(list, i + 1, nums);
        list.remove(list.size() - 1);
      }
    }
  }
}
```

## 332. 重新安排行程

语言：java

思路：先把行程二元组按照(value1, value2) 字母序进行升序排序，其中"JFK"比较特殊，如果value1是"JFK"则需要排序到最前面。

其次，按照回溯法DFS遍历，如果能使所有行程二元组都used一遍，则找到答案。由于事先做了排序，找到答案时一定是最小行程组合。

代码（28ms，7.65%）：

```java
class Solution {
  public List<String> findItinerary(List<List<String>> tickets) {
    // 1. 对tickets 排序
    Collections.sort(tickets, Comparator.comparing((List<String> a) -> a.get(0)).thenComparing(a -> a.get(1)));
    // 2. 构造一个 Map <String, List<Integer>>， key: tickets.get(下标).get(0)，value: tickets下标集合(小到大)
    Map<String, List<Integer>> nodeIndexMap = new HashMap<>();
    for(int i = 0;i < tickets.size();++i) {
      List<String> path = tickets.get(i);
      String pathNode = path.get(0);
      List<Integer> pathNodeIndexList = nodeIndexMap.getOrDefault(pathNode, new ArrayList<>());
      pathNodeIndexList.add(i);
      nodeIndexMap.put(pathNode, pathNodeIndexList);
    }
    // 3. 回溯法DFS尝试方案，直到找到唯一路径为止
    LinkedList<String> path = new LinkedList<>();
    path.add("JFK");
    return backTracking(tickets, path, new boolean[tickets.size()], tickets.size());
  }

  public List<String> backTracking(List<List<String>> tickets, List<String> path, boolean[] used, int end) {
    if(path.size() == end + 1) {
      return path;
    }
    String lastNode = path.get(path.size()-1);
    for(int i = 0; i < end; ++i) {
      if(lastNode.equals(tickets.get(i).get(0)) && !used[i]) {
        used[i] = true;
        path.add(tickets.get(i).get(1));
        List<String> tmpPath = backTracking(tickets, path,used,end);
        if(path.size() == end + 1) {
          return tmpPath;
        }
        path.remove(path.size()-1);
        used[i] = false;
      }
    }
    return path;
  }
}
```

参考代码1（12ms，42.13%）：

（1）这里只对每个路径的出口排序，推测是因为入口往后是固定的，即每个下一步入口是固定的，只需尽可能选择出口更小的就好了。

（2）boolean返回值，因为只需要找到一个路径（到叶子节点），就可以直接返回结果了。

> [代码随想录 (programmercarl.com)](https://programmercarl.com/0332.重新安排行程.html#其他语言版本)

```java
class Solution {
  private LinkedList<String> res;
  private LinkedList<String> path = new LinkedList<>();

  public List<String> findItinerary(List<List<String>> tickets) {
    Collections.sort(tickets, (a, b) -> a.get(1).compareTo(b.get(1)));
    path.add("JFK");
    boolean[] used = new boolean[tickets.size()];
    backTracking((ArrayList) tickets, used);
    return res;
  }

  public boolean backTracking(ArrayList<List<String>> tickets, boolean[] used) {
    if (path.size() == tickets.size() + 1) {
      res = new LinkedList(path);
      return true;
    }

    for (int i = 0; i < tickets.size(); i++) {
      if (!used[i] && tickets.get(i).get(0).equals(path.getLast())) {
        path.add(tickets.get(i).get(1));
        used[i] = true;

        if (backTracking(tickets, used)) {
          return true;
        }

        used[i] = false;
        path.removeLast();
      }
    }
    return false;
  }
}
```

参考代码2（9ms，62.20%）：思路基本没变，就是事先把tickets转成有序的Map作遍历和递归回溯。

（1）Map的key：路径的入口，value：新Map（key：路径的出口，value：该路径出现的次数）

> [代码随想录 (programmercarl.com)](https://programmercarl.com/0332.重新安排行程.html#其他语言版本)

```java
class Solution {
  private Deque<String> res;
  private Map<String, Map<String, Integer>> map;

  private boolean backTracking(int ticketNum){
    if(res.size() == ticketNum + 1){
      return true;
    }
    String last = res.getLast();
    if(map.containsKey(last)){//防止出现null
      for(Map.Entry<String, Integer> target : map.get(last).entrySet()){
        int count = target.getValue();
        if(count > 0){
          res.add(target.getKey());
          target.setValue(count - 1);
          if(backTracking(ticketNum)) return true;
          res.removeLast();
          target.setValue(count);
        }
      }
    }
    return false;
  }

  public List<String> findItinerary(List<List<String>> tickets) {
    map = new HashMap<String, Map<String, Integer>>();
    res = new LinkedList<>();
    for(List<String> t : tickets){
      Map<String, Integer> temp;
      if(map.containsKey(t.get(0))){
        temp = map.get(t.get(0));
        temp.put(t.get(1), temp.getOrDefault(t.get(1), 0) + 1);
      }else{
        temp = new TreeMap<>();//升序Map
        temp.put(t.get(1), 1);
      }
      map.put(t.get(0), temp);

    }
    res.add("JFK");
    backTracking(tickets.size());
    return new ArrayList<>(res);
  }
}
```

参考代码3（4ms，100%）：
```java
class Solution {
  // key: 入口， value: 出口升序排序的集合
  private Map<String, PriorityQueue<String>> mapOfFindItinerary1;
  // 答案 (中间存节点的List)
  private List<String> resOfFindItinerary1;
  public List<String> findItinerary(List<List<String>> tickets) {
    mapOfFindItinerary1 = new HashMap<>();
    resOfFindItinerary1 = new ArrayList<>(tickets.size());
    // 先构造 key: 入口， value：出口升序排序的集合
    for (List<String> ticket : tickets) {
      String src = ticket.get(0);
      String dst = ticket.get(1);
      if (!mapOfFindItinerary1.containsKey(src)) {
        PriorityQueue<String> pq = new PriorityQueue<>();
        mapOfFindItinerary1.put(src, pq);
      }
      mapOfFindItinerary1.get(src).add(dst);
    }
    // DFS遍历，这里大佬用poll特性保证每个边只用一次，着实佩服
    dfs("JFK");
    // 因为是回溯的时候添加节点，所以顺序是反向的，需要 reverse颠倒
    Collections.reverse(resOfFindItinerary1);
    return resOfFindItinerary1;
  }

  private void dfs(String src) {
    PriorityQueue<String> pq = mapOfFindItinerary1.get(src);
    while (pq != null && !pq.isEmpty())
      dfs(pq.poll());
    (resOfFindItinerary1).add(src);
  }
}
```

参考代码4（7ms，81.4%）：

> [332. 重新安排行程 - 力扣（Leetcode）](https://leetcode.cn/problems/reconstruct-itinerary/solutions/389885/zhong-xin-an-pai-xing-cheng-by-leetcode-solution/)
>
> Hierholzer 算法用于在连通图中寻找欧拉路径，其流程如下：
>
> 1、从起点出发，进行深度优先搜索。
>
> 2、每次沿着某条边从某个顶点移动到另外一个顶点的时候，都需要删除这条边。
>
> 3、如果没有可移动的路径，则将所在节点加入到栈中，并返回。

```java
class Solution {
  Map<String, PriorityQueue<String>> map = new HashMap<String, PriorityQueue<String>>();
  List<String> itinerary = new LinkedList<String>();

  public List<String> findItinerary(List<List<String>> tickets) {
    for (List<String> ticket : tickets) {
      String src = ticket.get(0), dst = ticket.get(1);
      if (!map.containsKey(src)) {
        map.put(src, new PriorityQueue<String>());
      }
      map.get(src).offer(dst);
    }
    dfs("JFK");
    Collections.reverse(itinerary);
    return itinerary;
  }

  public void dfs(String curr) {
    while (map.containsKey(curr) && map.get(curr).size() > 0) {
      String tmp = map.get(curr).poll();
      dfs(tmp);
    }
    itinerary.add(curr);
  }
}
```

## 37. 解数独

语言：java

思路：由于每次回溯需要擦除的错误填充，跨越X行Y列，所以每次回溯过程需要直接判断整个棋盘的填充，而不是每次只遍历一行。

代码（6ms，49.81%）：

```java
class Solution {
  public void solveSudoku(char[][] board) {
    backTracking(board);
  }

  public boolean backTracking(char[][] board) {
    for(int row = 0; row < 9; ++row) {
      for(int col = 0; col < 9; ++col) {
        if(board[row][col] != '.')
          continue;
        for(char num = '1'; num <= '9'; ++num) {
          if(valid(board,row,col,num)) {
            board[row][col] = num;
            if(backTracking(board)){
              return true;
            }
            board[row][col] = '.';
          }
        }
        return false;
      }
    }
    return true;
  }

  public boolean valid(char[][] board,int row, int col, char num) {
    // 1、当前行是否重复
    for(int i = 0; i < 9; ++ i) {
      if(board[row][i] == num) return false;
    } 
    // 2、当前列是否重复
    for(int i = 0; i < 9; ++i) {
      if(board[i][col] == num) return false;
    }
    // 3、当前9宫格是否重复
    for(int i = row/3*3; i< row/3*3+3; ++i) {
      for(int j = col/3*3; j < col/3*3+3;++j) {
        if(board[i][j]== num) return false;
      }
    }
    return true;
  }
}
```

参考代码1（6ms，49.81%）：思路一样的

> [代码随想录 (programmercarl.com)](https://programmercarl.com/0037.解数独.html#其他语言版本)

```java
class Solution {
  public void solveSudoku(char[][] board) {
    solveSudokuHelper(board);
  }

  private boolean solveSudokuHelper(char[][] board){
    //「一个for循环遍历棋盘的行，一个for循环遍历棋盘的列，
    // 一行一列确定下来之后，递归遍历这个位置放9个数字的可能性！」
    for (int i = 0; i < 9; i++){ // 遍历行
      for (int j = 0; j < 9; j++){ // 遍历列
        if (board[i][j] != '.'){ // 跳过原始数字
          continue;
        }
        for (char k = '1'; k <= '9'; k++){ // (i, j) 这个位置放k是否合适
          if (isValidSudoku(i, j, k, board)){
            board[i][j] = k;
            if (solveSudokuHelper(board)){ // 如果找到合适一组立刻返回
              return true;
            }
            board[i][j] = '.';
          }
        }
        // 9个数都试完了，都不行，那么就返回false
        return false;
        // 因为如果一行一列确定下来了，这里尝试了9个数都不行，说明这个棋盘找不到解决数独问题的解！
        // 那么会直接返回， 「这也就是为什么没有终止条件也不会永远填不满棋盘而无限递归下去！」
      }
    }
    // 遍历完没有返回false，说明找到了合适棋盘位置了
    return true;
  }

  /**
     * 判断棋盘是否合法有如下三个维度:
     *     同行是否重复
     *     同列是否重复
     *     9宫格里是否重复
     */
  private boolean isValidSudoku(int row, int col, char val, char[][] board){
    // 同行是否重复
    for (int i = 0; i < 9; i++){
      if (board[row][i] == val){
        return false;
      }
    }
    // 同列是否重复
    for (int j = 0; j < 9; j++){
      if (board[j][col] == val){
        return false;
      }
    }
    // 9宫格里是否重复
    int startRow = (row / 3) * 3;
    int startCol = (col / 3) * 3;
    for (int i = startRow; i < startRow + 3; i++){
      for (int j = startCol; j < startCol + 3; j++){
        if (board[i][j] == val){
          return false;
        }
      }
    }
    return true;
  }
}
```

## 509. 斐波那契数

语言：java

思路：经典的动态规划入门题。

代码（0ms，100%）：

```java
class Solution {
  public int fib(int n) {
    if(n < 2) return n;
    int[] nums = new int[n+1];
    nums[0] = 0;
    nums[1] = 1;
    for(int i = 2; i <= n;++i) {
      nums[i] = nums[i-1] + nums[i-2];
    }
    return nums[n];
  }
}
```

