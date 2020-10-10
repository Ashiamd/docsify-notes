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

