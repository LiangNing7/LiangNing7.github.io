+++
title = 'Backtracking'
date = 2026-06-01T15:37:29+08:00
draft = false
categories = [
    "algorithm"
]

tags = [
    "algorithm",
    "interview"
]

series = [
    "algorithm"
]
series_order = 1
+++



# 回溯法

## 核心思想

回溯法是一种**系统地搜索问题所有解**的算法，常用于：

* 排列（Permutation）
* 组合（Combination）
* 子集（Subset）
* N 皇后 / 数独 / 图着色等约束问题

其本质是：

> 通过深度优先搜索（DFS）的方式，枚举所有可能的解；
> 当发现当前路径不可能产生合法解时，**立即回退（剪枝）**，从上一步继续尝试其他可能。

**伪代码模板**：

```c++
void backtrack(状态参数...) {
    if (满足结束条件) {
        收集结果;
        return;
    }

    for (选择 in 可选列表) {
        做出选择;
        backtrack(新的状态);
        撤销选择;  // 回溯
    }
}
```

## AcWing 92. 递归实现指数型枚举

> https://www.acwing.com/problem/content/94/

### 题目

从 1∼n 这 n 个整数中随机选取任意多个，输出所有可能的选择方案。

#### 输入格式

输入一个整数 n。

#### 输出格式

每行输出一种方案。

同一行内的数必须升序排列，相邻两个数用恰好 1 个空格隔开。

对于没有选任何数的方案，输出空行。

本题有自定义校验器（SPJ），各行（不同方案）之间的顺序任意。

#### 数据范围

1≤n≤15

#### 输入样例：

```
3
```

#### 输出样例：

```
3
2
2 3
1
1 3
1 2
1 2 3
```

### 代码

```c++
// 枚举 选 OR 不选 
#include <bits/stdc++.h>
using namespace std;
const int N = 20;
int n;
bool st[N];

void dfs(int u) {
    if (u == n + 1) {
        for (int i = 1;i<=n;i++){
            if (st[i]) cout << i << " ";
        }
        cout << endl;
        return;
    }
    st[u] = false;
    dfs(u+1);
    
    st[u] = true;
    dfs(u+1);
}

int main() {
    cin >> n;
    dfs(1);
    return 0;
}
```

```c++
// 枚举 [u...n]
#include <bits/stdc++.h>
using namespace std;
const int N = 20;
int path[N]; // st[N] 隐藏在了 u 中，因为是从 u...n 中选一个，即 u...n 必定都没被选过，所以不需要 st[n].
int n,cnt;

void dfs(int u){
    for (int i = 1;i<=cnt;i++) cout << path[i] << " ";
    cout << endl;
    for (int i = u;i<=n;i++){
        path[++cnt] = i;
        dfs(i+1);
        cnt--;
    }
}

int main() {
    cin >> n;
    dfs(1);
    return 0;
}
```

## LeetCode 分割字符串

> https://leetcode.cn/problems/M99OJA/

给定一个字符串 `s` ，请将 `s` 分割成一些子串，使每个子串都是 **回文串** ，返回 s 所有可能的分割方案。

### 题目

**回文串** 是正着读和反着读都一样的字符串。

**示例 1：**

```
输入：s = "google"
输出：[["g","o","o","g","l","e"],["g","oo","g","l","e"],["goog","l","e"]]
```

**示例 2：**

```
输入：s = "aab"
输出：[["a","a","b"],["aa","b"]]
```

**示例 3：**

```
输入：s = "a"
输出：[["a"]]
```

**提示：**

- `1 <= s.length <= 16`
- `s `仅由小写英文字母组成

### 代码

```go
func partition(s string) [][]string {
    n := len(s)
    res := make([][]string,0)
    st := make([]bool,n+1)

    check := func(l,r int) bool {
        for l < r{
            if s[l] == s[r]{
                l++
                r--
            } else{
                return false
            }
        }
        return true
    }

    var dfs func(int)
    dfs = func(u int) {
        if (u == n) {
            ans := make([]string,0)
            start := 0
            for i := 0;i < n;i++ {
                if st[i] {
                    ans = append(ans,s[start:i+1])
                    start = i + 1
                }
            }
            res = append(res,ans)
        }

        for i := u; i < n;i++ {
            if check(u,i){
                st[i] = true
                dfs(i+1)
                st[i] = false
            }
        }
    }

    dfs(0)
    return res
}
```

## AcWing 94. 递归实现排列型枚举

> https://www.acwing.com/problem/content/96/

### 题目

把 1∼n 这 n 个整数排成一行后随机打乱顺序，输出所有可能的次序。

#### 输入格式

一个整数 n。

#### 输出格式

按照从小到大的顺序输出所有方案，每行 1 个。

首先，同一行相邻两个数用一个空格隔开。

其次，对于两个不同的行，对应下标的数一一比较，字典序较小的排在前面。

#### 数据范围

1≤n≤9

#### 输入样例：

```
3
```

#### 输出样例：

```
1 2 3
1 3 2
2 1 3
2 3 1
3 1 2
3 2 1
```

### 代码

```c++
#include <bits/stdc++.h>
using namespace std;
const int N = 10;
int n;
int path[N];
bool st[N];

void dfs(int u) {
    if (u == n + 1){
        for (int i = 1;i<=n;i++){
            cout << path[i] <<  " ";
        }
        cout << endl;
        return;
    }
    for (int i = 1;i <= n;i++){
        if (!st[i]){
            st[i] = true;
            path[u] = i;
            dfs(u+1);
            st[i] = false;
        }
    }
}

int main(){
    cin >> n;
    dfs(1);
    return 0;
}
```

## AcWing 93. 递归实现组合型枚举

> https://www.acwing.com/problem/content/95/

### 题目

从 1∼n 这 n 个整数中随机选出 m 个，输出所有可能的选择方案。

#### 输入格式

两个整数 n,m ,在同一行用空格隔开。

#### 输出格式

按照从小到大的顺序输出所有方案，每行 1 个。

首先，同一行内的数升序排列，相邻两个数用一个空格隔开。

其次，对于两个不同的行，对应下标的数一一比较，字典序较小的排在前面（例如 `1 3 5 7` 排在 `1 3 6 8` 前面）。

#### 数据范围

n>0 ,
0≤m≤n ,
n+(n−m)≤25

#### 输入样例：

```
5 3
```

#### 输出样例：

```
1 2 3 
1 2 4 
1 2 5 
1 3 4 
1 3 5 
1 4 5 
2 3 4 
2 3 5 
2 4 5 
3 4 5 
```

### 代码

```c++
#include <bits/stdc++.h>
using namespace std;
const int N = 30;
int path[N];
bool st[N];
int n,m;

void dfs(int u,int start){
    if (n - start < m - u) return;
    if (u == m+1) {
        for (int i = 1;i <= m;i++) cout << path[i] << " ";
        cout << endl;
        return;
    }
    for (int i = start;i <= n;i++){
        if (!st[i]){
            st[i] = true;
            path[u] = i;
            dfs(u+1,i+1);
            st[i] = false;
        }
    }
}

int main(){
    cin >> n >>m;
    dfs(1,1);
    return 0;
}
```

## AcWing 843. n-皇后

> https://www.acwing.com/problem/content/description/845/

### 题目

n−皇后问题是指将 n 个皇后放在 n×n 的国际象棋棋盘上，使得皇后不能相互攻击到，即任意两个皇后都不能处于同一行、同一列或同一斜线上。

![image-20260531143118593](https://images.liangning7.cn/typora/image-20260531143118593.png)

现在给定整数 n，请你输出所有的满足条件的棋子摆法。

#### 输入格式

共一行，包含整数 n。

#### 输出格式

每个解决方案占 n 行，每行输出一个长度为 n 的字符串，用来表示完整的棋盘状态。

其中 `.` 表示某一个位置的方格状态为空，`Q` 表示某一个位置的方格上摆着皇后。

每个方案输出完成后，输出一个空行。

**注意：行末不能有多余空格。**

输出方案的顺序任意，只要不重复且没有遗漏即可。

#### 数据范围

1≤n≤9

#### 输入样例：

```
4
```

#### 输出样例：

```
.Q..
...Q
Q...
..Q.

..Q.
Q...
...Q
.Q..
```

### 代码

```c++
#include <bits/stdc++.h>
using namespace std;
const int N = 20;
char g[N][N];
int col[N],dg[N],udg[N];
int n;

void dfs(int u) {
    if (u == n) {
        for (int i = 0;i < n;i++) puts(g[i]);
        cout << endl;
        return;
    }
    for (int i = 0; i < n;i++){
        if (!col[i] && !dg[u + i] && !udg[n - u + i]){
            col[i] = dg[u+i] = udg[n-u+i] = true;
            g[u][i] = 'Q';
            dfs(u+1);
            g[u][i] = '.';
            col[i] = dg[u+i] = udg[n-u+i] = false;
        }
    }
}

int main() {
    cin >> n;
    for (int i = 0;i < n;i++){
        for (int j = 0;j < n;j++){
            g[i][j] = '.';
        }
    }
    dfs(0);
    return 0;
}
```

## 括号生成

> https://leetcode.cn/problems/generate-parentheses/description/

### 题目

数字 `n` 代表生成括号的对数，请你设计一个函数，用于能够生成所有可能的并且 **有效的** 括号组合。

**示例 1：**

```
输入：n = 3
输出：["((()))","(()())","(())()","()(())","()()()"]
```

**示例 2：**

```
输入：n = 1
输出：["()"]
```

**提示：**

- `1 <= n <= 8`

### 代码

```go
func generateParenthesis(n int) []string {
    res := make([]string,0)

    var dfs func(int,int,string)
    dfs = func(l int,r int,s string) {
        if len(s) == 2*n {
            res = append(res,s)
            return 
        }
        if l < n {
            dfs(l+1,r,s+"(")
        }
        if r < l {
            dfs(l,r+1,s+")")
        }
    }
    dfs(0,0,"")

    return res
}
```

## 划分为 K 个相等的子集

> https://leetcode.cn/problems/partition-to-k-equal-sum-subsets/description/

### 题目

给定一个整数数组 `nums` 和一个正整数 `k`，找出是否有可能把这个数组分成 `k` 个非空子集，其总和都相等。

**示例 1：**

```
输入： nums = [4, 3, 2, 3, 5, 2, 1], k = 4
输出： True
说明： 有可能将其分成 4 个子集（5），（1,4），（2,3），（2,3）等于总和。
```

**示例 2:**

```
输入: nums = [1,2,3,4], k = 3
输出: false
```

**提示：**

- `1 <= k <= len(nums) <= 16`
- `0 < nums[i] < 10000`
- 每个元素的频率在 `[1,4]` 范围内

### 代码

```go
func canPartitionKSubsets(nums []int, k int) bool {
    sum := 0
    n := len(nums)
    for i := 0; i < n;i++ {
        sum += nums[i]
    } 
    if sum % k != 0 {
        return false
    }
    sort.Slice(nums, func(i, j int) bool {
		return nums[i] > nums[j]
	})
    var target int = sum / k
    st := make([]bool,n)
    var dfs func(int,int,int) bool

    dfs = func(k int,cnt int,u int) bool {
        if k == 1 {
            return true
        }
        if cnt == target {
            return dfs(k-1,0,0)
        }
        for i := u;i < n;i++ {
            if st[i] || cnt + nums[i] > target {
                continue
            }
            st[i] = true
            if dfs(k,cnt+nums[i],i+1) {
                return true
            }
            st[i] = false

            if cnt == 0 || cnt+nums[i] == target {
                return false
            }
        }
        return false
    }
    return dfs(k,0,0)
}
```

## 给表达式添加运算符

> https://leetcode.cn/problems/expression-add-operators/

### 题目

给定一个仅包含数字 `0-9` 的字符串 `num` 和一个目标值整数 `target` ，在 `num` 的数字之间添加 **二元** 运算符（不是一元）`+`、`-` 或 `*` ，返回 **所有** 能够得到 `target` 的表达式。

注意，返回表达式中的操作数 **不应该** 包含前导零。

**注意**，一个数字可以包含多个数位。

**示例 1:**

```
输入: num = "123", target = 6
输出: ["1+2+3", "1*2*3"] 
解释: “1*2*3” 和 “1+2+3” 的值都是6。
```

**示例 2:**

```
输入: num = "232", target = 8
输出: ["2*3+2", "2+3*2"]
解释: “2*3+2” 和 “2+3*2” 的值都是8。
```

**示例 3:**

```
输入: num = "3456237490", target = 9191
输出: []
解释: 表达式 “3456237490” 无法得到 9191 。
```

**提示：**

- `1 <= num.length <= 10`
- `num` 仅含数字
- $-2^{31}$ <= target <= $2^{31} - 1$

### 代码

```go
func addOperators(num string, target int) []string {
    n := len(num)
    ans := make([]string,0)
    var dfs func(u,res,mul int,expr []byte)
    dfs = func(u,res,mul int,expr []byte) {
        if u == n {
            if res == target {
                ans = append(ans,string(expr))
            }
            return
        }
        signIdx := len(expr)
        if u > 0 {
            expr = append(expr,0)
        }
        for j,val := u,0; j < n && (j == u || num[u] != '0');j++ {
            val = val * 10 + int(num[j] - '0')
            expr = append(expr,num[j])
            if u == 0 {
                dfs(j+1,val,val,expr)
            }  else {
                expr[signIdx] = '+';dfs(j+1,res+val,val,expr)
                expr[signIdx] = '-';dfs(j+1,res-val,-val,expr)
                expr[signIdx] = '*';dfs(j+1,res-mul+mul*val,mul*val,expr)
            }
        }
    }
    dfs(0,0,0,make([]byte,0,2*n-1))
    return ans
}
```


