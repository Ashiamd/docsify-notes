# LeetCode做题笔记

### 739. 每日温度

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

### 71. 简化路径

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

###  93. 复原IP地址

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

### 695. 岛屿的最大面积

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

### 75. 颜色分类

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

### 最小的K个数

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

###  518. 零钱兑换 II

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

### 416. 分割等和子集

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

### 474. 一和零

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

### 530. 二叉搜索树的最小绝对差

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

### 977. 有序数组的平方

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

### 52. N皇后 II

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

### 844. 比较含退格的字符串

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

### 143. 重排链表

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

### 925. 长按键入

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

### 763. 划分字母区间

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

### 47. 全排列 II

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

### 17. 电话号码的字母组合

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

### 39. 组合总和

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

### 40. 组合总和 II

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

### 51. N皇后

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

### 416. 分割等和子集

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

### 474. 一和零

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

### 494. 目标和

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

### 1025. 除数博弈

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

### 112. 路径总和

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

### 面试题 01.04. 回文排列

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

### 647. 回文子串

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

### 1392. 最长快乐前缀

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

### 572. 另一个树的子树

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

### 1143. 最长公共子序列

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

### 剑指 Offer 48. 最长不含重复字符的子字符串

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

### 781. 森林中的兔子

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

### 310. 最小高度树

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

### 264. 丑数 II

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

