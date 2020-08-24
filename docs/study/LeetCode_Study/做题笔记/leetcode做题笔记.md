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

### 