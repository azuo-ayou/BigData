

### 1.两数之和：hash

将已经遍历的数据存到hash表中，key数据，value是数组索引

```python
class Solution:
    def twoSum(self, nums: List[int], target: int) -> List[int]:
        hashtable = dict()
        for i in range(len(nums)):
            if target-nums[i] in hashtable:
                return [hashtable[target-nums[i]],i]
            hashtable[nums[i]] = i 
        return []
```

### 2.两数相加：一起遍历

思路：不考虑长度，如果短的话，那就赋值0；最后要考虑到可能还会进一位

```python
class Solution:
    def addTwoNumbers(self, l1: Optional[ListNode], l2: Optional[ListNode]) -> Optional[ListNode]:
        head = None
        tail = None
        # 判断上次是否需要+1
        carry = 0
        while l1 or l2:
            n1 = l1.val if l1 else 0
            n2 = l2.val if l2 else 0
            resnum = n1 + n2 + carry
            if head is None:
                print(1)
                head = tail = ListNode(resnum % 10)
            else:
                tail.next = ListNode(resnum % 10)
                tail = tail.next
            carry = resnum // 10

            if l1:
                l1 = l1.next
            if l2:
                l2 = l2.next
        if carry > 0:
            tail.next = ListNode(carry)
        return head
            
```



### 3.无重复字符的最长子串

滑动窗口：右指针一直滑动，将数据存入set，当遇到重复的元素，从左指针开始删除，直到没有重复的元素，过程中记录长度最大时候的起始位置

```python
class Solution:
    def lengthOfLongestSubstring(self, s: str) -> int:
        begin = 0
        end = 0
        right = 0
        res = 0
        strs = set()
        n = len(s)
        for i in range(n):
            while right<n and s[right] not in strs:
                strs.add(s[right])
                right = right + 1
            if res < len(strs):
                begin = i 
                end = right-1
                res = len(strs)
            strs.remove(s[i])
        
        print(s[begin:end+1])
        return res
```

### 5.最长回文子串：动态规划

思路$dp[i][j]$表示从下标i到j的字符串是否是回文子串，显然$dp[i][i]$都是true，那么状态转移方程为


$$
dp[i][j] = dp[i]==dp[j] 且 dp[i+1][j-1]
$$
**注意点：两层循环，外层遍历长度，内层遍历每个子串开始位置。**

```python
class Solution:
    def longestPalindrome(self, s: str) -> str:
        n = len(s)
        if n < 2:
            return s
        dp = [[False]*n for _ in range(n)]
        for i in range(n):
            dp[i][i] = True 
        
        max_len = 1
        begin = 0
        # 两步循环，L是长度，i开始位置
        # 循环遍历每个长度的子串，判断是否dp[i][j]为true
        for L in range(2,n+1):
            for i in range(n):
                j = L+i-1
                if j >= n:
                    break
                if s[i] != s[j]:
                    dp[i][j] = False
                else:
                    if j-i<3:
                        dp[i][j] = True
                    else:
                        dp[i][j] = dp[i+1][j-1]
                
                if dp[i][j] and j - i + 1 > max_len:
                    max_len = j - i + 1 
                    begin = i
      
        return s[begin:begin + max_len]
```



### 11.盛最多水的容器

思路：解题思路是双指针，分别从左右两端向中间走，每次求得面积，取最大的；每次移动高度小的指针（因为移动高度大的指针，无论如何都会使得面积变小，这个可以很容易证明，也可以参考leetcode解析）

```python
class Solution:
    def maxArea(self, height: List[int]) -> int:
        n = len(height)
        left = 0
        right = n-1
        ans = 0
        while left < right:
            res = (right - left)*min(height[right],height[left])
            ans = max(res,ans)
            if height[right] > height[left]:
                left = left + 1
            else:
                right = right - 1 
        return ans
```





### 17.点话号码的字母组合：深度优先遍历dfs

直接回溯法即可：深度优先遍历，当长度=数字长度，就返回；最后记得每一层的result要有pop操作，也就是循环当前数字的其他字母的时候要，先删除这个数字的前一个字母。

```python
class Solution:
    def letterCombinations(self, digits: str) -> List[str]:
        if not digits:
            return list()
        phoneMap = {
            "2": "abc",
            "3": "def",
            "4": "ghi",
            "5": "jkl",
            "6": "mno",
            "7": "pqrs",
            "8": "tuv",
            "9": "wxyz",
        }
        def dfs(index):
            if index == len(digits):
                ress.append("".join(res))
                return
            for i in phoneMap[digits[index]]:
                res.append(i)
                dfs(index+1)
                res.pop()
        ress=list()
        res=list()
        dfs(0)
        return ress
```



### 19.删除链表的倒数第N个节点：双指针（快慢指针）

题目思路并不难，难点是如何理解节点的删除，之前的赋值是都不会改变原来的list，只有当改变某个节点的val或者next的时候整个list才会被改变

```python
class Solution:
    def removeNthFromEnd(self, head: Optional[ListNode], n: int) -> Optional[ListNode]:
        tmp = ListNode(0,head)
        first = head
        second = tmp
        for i in range(n):
            first = first.next
        while first:
            first = first.next
            second = second.next
        # 怎么理解这一步的节点删除，
        second.next = second.next.next
        return tmp.next
```



### 20.有效括号：栈

思路：属于简单题，想到栈就可以解决；唯一需要注意的点就是需要判断栈不为空

```python
class Solution:
    def isValid(self, s: str) -> bool:
        if len(s) % 2 == 1:
            return False
        
        pairs = {
            ")": "(",
            "]": "[",
            "}": "{",
        }
        stack = list()
        for i in s:
            if i not in pairs:
                stack.append(i)
            else:
                if not stack or stack[-1] != pairs[i]:
                    return False
                else:
                    stack.pop()
        return not stack

```



### 21.合并两个有序连表：递归和迭代

这个也算事简单题，首先要想到递归和迭代两种方法，每种方法都各有特色

```python
class Solution:
    def mergeTwoLists(self, list1: Optional[ListNode], list2: Optional[ListNode]) -> Optional[ListNode]:

        # 递归实现
        # if not list1:
        #     return list2
        # if not list2:
        #     return list1
        # if list1.val > list2.val:
        #     return ListNode(list2.val,self.mergeTwoLists(list1,list2.next))
        # else:
        #     return ListNode(list1.val,self.mergeTwoLists(list1.next,list2))
       
        # 迭代实现
        tmp = ListNode(-1)
        pre = tmp
        while list1 and list2:
            if list1.val < list2.val:
                pre.next = list1
                list1 = list1.next
            else:
                pre.next = list2
                list2 = list2.next
            pre = pre.next 
        pre.next = list1 if list1 is not None else list2
        return tmp.next
```



### 22.括号生成：递归

思路：这里的递归要从n=1到n=2理解，n=1的时候是（），那么n=2，添加一个括号，那么这个括号之可能出现在这三个地方0（1）2；

所以，只需要遍历n-1结果每一种情况的每一个位置，然后去重即可，代码如下

```python
class Solution:
    def generateParenthesis(self, n: int) -> List[str]:
        # 直接看评论大佬，n=1,();n=2,就是 0 （1） 2 三个位置分别放一个（）然后去重
        # 用一个
        if  n == 1:
            return ["()"]
        res = set()
        for i in self.generateParenthesis(n-1):
            for j in range(len(i)+1):
                # print(i[0:j]+"--"+i[j:]) 用于测试边界条件
                res.add(i[0:j]+"()"+i[j:])
        return list(res)
```

### 31.下一个排列

整数数组的一个 排列  就是将其所有成员以序列或线性顺序排列。

例如，arr = [1,2,3] ，以下这些都可以视作 arr 的排列：[1,2,3]、[1,3,2]、[3,1,2]、[2,3,1] 。
整数数组的 下一个排列 是指其整数的下一个字典序更大的排列。更正式地，如果数组的所有排列根据其字典顺序从小到大排列在一个容器中，那么数组的 下一个排列 就是在这个有序容器中排在它后面的那个排列。如果不存在下一个更大的排列，那么这个数组必须重排为字典序最小的排列（即，其元素按升序排列）。

例如，arr = [1,2,3] 的下一个排列是 [1,3,2] 。
类似地，arr = [2,3,1] 的下一个排列是 [3,1,2] 。
而 arr = [3,2,1] 的下一个排列是 [1,2,3] ，因为 [3,2,1] 不存在一个字典序更大的排列。
给你一个整数数组 nums ，找出 nums 的下一个排列。

**示例 1：**

```py
输入：nums = [1,2,3]
输出：[1,3,2]
```



思路

1. 从后往前找到第一个升序的连续数字i，j
2. 从后往前找到j，找到第一个大于arr[i]的下表g，交换i,g的位置
3. 让从j到end，变成升序

不知道为什么这段代码报错

```python
class Solution:
    def nextPermutation(self, nums: List[int]) -> None:
        """
        Do not return anything, modify nums in-place instead.
        """
        n = len(nums)
        n1,n2 = 0,0
        for i in range(n-1):
            if nums[n-2-i] < nums[n-1-i]:
                n1 = n-2-i 
                n2 = n-1-i
                break 
        if n2 == 0:
            nums.sort()
            return
        # print(nums)
        for i in range(n-1,n2-1,-1):
            # print(nums)
            if nums[i] > nums[n1]:
                nums[i],nums[n1] = nums[n1],nums[i]
                break
        tmpnums = nums[n2:]
        tmpnums.sort()
        # print(nums[:n2])
        # print(tmpnums)
        nums = nums[:n2] + tmpnums
        
        print(nums)
```





### 34.在排序数组中查找元素的第一个和最后一个位置：二分法

经典的二分法：需要弄清楚两个问题1、什么样的边界条件可以得到最左边的元素，2、什么样的边界可以得到右边的元素

```python
class Solution:
    def searchRange(self, nums: List[int], target: int) -> List[int]:
        if not nums:
            return list([-1,-1])
        l = 0
        r = len(nums)-1
        while(l<r):
            mid = (l+r)//2
            if nums[mid] >= target:
                r = mid 
            else:
                l = mid+1
        if nums[r] != target:
            return list([-1,-1])
        L = r 
        l = 0
        r = len(nums)-1
        while(l<r):
            mid = (l+r+1)//2
            if nums[mid] <= target:
                l = mid 
            else:
                r = mid -1
        return list([L,r])
```

### 39.组合总数：经典的回溯+剪枝

总结一下回溯的集中情况

1. 完全的全排列，用过的数据还能再用，结果也就是$2^n$种情况，这时候只需要回溯，不需要设置状态如[1,2]的排列有 11、12、21、22
2. 全排列：如[1,2]的全排有两种情况12和21，这时候需要设置state，请看leetcode46题
3. 本题：需要区分顺序，12和21认为是同一种情况；那么在遍历的时候，每次遍历需要从上一次开始的地方开始，也就是设置begin



思路分析：根据示例 1：输入: candidates = [2, 3, 6, 7]，target = 7。

候选数组里有 2，如果找到了组合总和为 7 - 2 = 5 的所有组合，再在之前加上 2 ，就是 7 的所有组合；
同理考虑 3，如果找到了组合总和为 7 - 3 = 4 的所有组合，再在之前加上 3 ，就是 7 的所有组合，依次这样找下去。




```python
class Solution:
    def combinationSum(self, candidates: List[int], target: int) -> List[List[int]]:
        def dfs(begin,target):
        
            if target == 0:
                tmp = sub_res[::]
                res.append(tmp)
                return
            elif target < 0:
                return

            for i in range(begin,len(candidates)):

                sub_res.append(candidates[i])
                dfs(i,target-candidates[i])
                sub_res.pop()
                


        res = list()
        sub_res = list()
        dfs(0,target)
        return res
```

### 42.接雨水：多种方法

1. 按列求：每一列能接的雨水是：min（左边最高墙，右边最搞墙）- 当前列的高度
2. 在1的基础上，我们不必每次遍历求得左右两边的最高墙
3. 类似括号的栈

```python
class Solution:
    def trap(self, height: List[int]) -> int:
        max_left = 0
        n = len(height)
        max_right = [0 for _ in range(n)]
        sums = 0
        
        # 从后往前遍历，取每一列的右边最大值
        for i in range(n-2,0,-1):
            max_right[i] = max(max_right[i+1],height[i+1])
        for i in range(1,n-1):
            max_left = max(max_left,height[i-1])
            minnum = min(max_left,max_right[i])
            if minnum > height[i]:
                sums = sums + minnum - height[i]
        return sums
       
```







### 46.全排列：深刻理解回溯算法和深度优先遍历

https://leetcode.cn/problems/permutations/solution/hui-su-suan-fa-python-dai-ma-java-dai-ma-by-liweiw/必看的大佬

思路：回溯+剪枝

注意两点

1. 使用过的元素及时的回溯
2. 使用过的状态要标记，用完以后修改标记

```python
class Solution:
    def permute(self, nums: List[int]) -> List[List[int]]:
        def dfs(cur_index,max_index):
   
            if cur_index == max_index:
      					# 由于list是引用类型，所以需要复制一份出来
                tmp = sub_res[::]
                res.append(tmp)
                return
            for i in range(len(nums)):
                if not state[i]:
                  	# 添加元素到case
                    sub_res.append(nums[i])
                    # 修改状态，当前数字已经被用过了
                    state[i] = True
                    dfs(cur_index+1,max_index)
                    # 删除当前添加的元素
                    sub_res.pop()
                    # 设置当前元素还没有被用过
                    state[i] = False
        max_index = len(nums)
        state = [False for _ in range(max_index)]
        # state = [[False for _ in range(max_index)] for _ in range(max_index)]
        res = list()
        sub_res = list()
        dfs(0,max_index)
        return res
```



### 48.旋转图像（旋转矩阵）水平翻转 和 对角线翻转

思路：先水平（上下）翻转，然后对角线翻转

引申

1. 逆时针旋转呢？ **可以先左右翻转，再对角翻转**
2. 制定翻转n次呢？



```python
class Solution:
    def rotate(self, matrix: List[List[int]]) -> None:
        """
        Do not return anything, modify matrix in-place instead.
        """
        n = len(matrix)
        # 水平翻转
        for i in range(n//2):
            for j in range(n):
                matrix[i][j],matrix[n-i-1][j] = matrix[n-i-1][j],matrix[i][j]

        # 对角线翻转
        for i in range(n):
            for j in range(i):
                matrix[i][j],matrix[j][i] = matrix[j][i],matrix[i][j]
```









### 53.最大子数组和：动态规划

$$
dp[i] = max(dp[i]+nums[i],num[i])
$$

```python
class Solution:
    def maxSubArray(self, nums: List[int]) -> int:
        # 动态规划，dp[i]表示以num[i]结尾的最大子数组
        n = len(nums)
        dp = [0] * n 
        dp[0] = nums[0]
        ans = nums[0]
        for i in range(1,n):
            dp[i] = max(dp[i-1]+nums[i],nums[i])
            ans = max(dp[i],ans)
        return ans
```



### 55.跳跃游戏：贪心算法

思路：遍历数组，每个位置能到达的最远距离是 $i+num[i]$,只要当 $i+num[i]>n-1$即可，要注意的是需要先判断是否能达到当前位置，$i<most_distance$



```python
class Solution:
    def canJump(self, nums: List[int]) -> bool:
        n = len(nums)
        most_distance = 0
        for i in range(n):
            if i <= most_distance: #前提是必须到达这一步
                most_distance = max(most_distance,i+nums[i])
                if most_distance >= n-1:
                    print(i)
                    return True
        return False
```



### 62.不同路径：从左上角到右下角

方法一：排列组合

需要移动$m+n-2$次，其中$m-1$次向下，$n-1$次向右移动，也就是从$m+n-2$中选择$m-1$次向下移动的方案数(或者$n-1$次向右移动的方案，是一样的)，即

方法二：动态规划

$dp[i][j]$表示到达ij位置需要的路径和，那么状态转移方程为：
$$
dp[i][j] = dp[i][j-1] + dp[j][i-1]
$$
注：

1. 这时候边界条件都是1
2. 可以连续计算，省去边界条件



```python
# 以下分别是两种方法
class Solution:
    def uniquePaths(self, m: int, n: int) -> int:
        # pre = [1]*n
        # cur = [1]*n

        # # 想办法列遍历
        # for i in range(1,m):
        #     for j in range(1,n):
        #         cur[j] = pre[j] + cur[j-1]
        #     print(pre)
        #     pre = cur[::]
        # return cur[n-1]
        n1,n2 = 1,1
        for i in range(m,m+n-1):
            n1 = n1*i 
        for j in range(1,n):
            n2 = n2*j
        return (n1//n2)

```



### 64.最小路径和：动态规划

思路很简单，边界条件分情况处理一下就行，状态转移方程就不写了，代码如下

```python
class Solution:
    def minPathSum(self, grid: List[List[int]]) -> int:
        m = len(grid)
        n = len(grid[0])
        dp = [[0 for _ in range(n)] for _ in range(m)]
        for i in range(m):
            for j in range(n):
                if i == 0:
                    dp[i][j] = dp[i][j-1] + grid[i][j]
                elif j == 0:
                    dp[i][j] = dp[i-1][j] + grid[i][j]
                else:
                    dp[i][j] = min(dp[i-1][j],dp[i][j-1]) + grid[i][j]
        print(dp)
        return dp[m-1][n-1]
```



### 70.爬楼梯：就是一个经典的迭代问题

注：青蛙跳台阶

```python
class Solution:
    def climbStairs(self, n: int) -> int:
        if n == 1:
            return 1
        if n == 2:
            return 2
        
        cur,pre=2,1
        for i in range(2,n):
            tem = cur + pre 
            cur,pre = tem,cur
        return cur
```

#### 请务必注意爬楼梯和零钱兑换的问题：



**爬楼梯是排列，零钱兑换是组合**

爬楼梯:循环楼梯在外层

1. 爬上楼梯需要的最少步数：dp[]
2. 爬上楼梯的方法总是：dp[]

https://leetcode.cn/problems/coin-change-ii/solution/ling-qian-dui-huan-iihe-pa-lou-ti-wen-ti-dao-di-yo/





### 72.编辑距离：动态规划

将一个单词转换成另一个单词需要的最少的操作步骤，这道题可以用动态规划来做：$dp[i][j]$表示从word1前i字符，转化到word2前j个字符需要最少的步骤
$$
一、当word1[i] != word[j]：dp[i][j] = min(dp[i-1][j],dp[i][j-1],dp[i-1][j-1]) \\
二、当word1[i] == word[j]：dp[i][j] = dp[i][j]
$$
$dp[i-1][j]$表示我们可以在word1的第i个位置添加一个和word2第j处相同的字母，那么$dp[i][j]$最小可以为$dp[i-1][j]+1$；这种操作为插入；同理$dp[i][j-1]$对应的是删除，$dp[i-1][j-1]$对应的是替换i，j使得相同

```python
class Solution:
    def minDistance(self, word1: str, word2: str) -> int:
        m = len(word1)
        n = len(word2)

        dp = [[0 for _ in range(n+1)] for _ in range(m+1)]
        
        for i in range(m+1):
            for j in range(n+1):
                if i == 0:
                    dp[i][j] = j
                elif j == 0:
                    dp[i][j] = i
                else:
                    if word1[i-1] == word2[j-1]:
                        dp[i][j] = dp[i-1][j-1]
                    else:
                        dp[i][j] = min(dp[i-1][j],dp[i][j-1],dp[i-1][j-1]) + 1
        return dp[m][n]
```





### 75.颜色分类：只有012的数字排序

思路

1、单指针，两次遍历；第一次从0开始，把所有的0放在最前边，第二次接着把1放在0的后边

2、双指针，不断调整0、1的位置

```python
class Solution:
    def sortColors(self, nums: List[int]) -> None:
        """
        Do not return anything, modify nums in-place instead.
        """
        p0 = 0
        p1 = 0
        n = len(nums)
        for i in range(n):
            print(p1)
            if nums[i] == 0:
                nums[i],nums[p0]=nums[p0],nums[i]
               
                if p0 >= p1:
                    p1 = p1 + 1
                p0 = p0 + 1
            if nums[i] == 1:
                nums[i],nums[p1]=nums[p1],nums[i]
               
                p1 = p1 + 1
        print(nums)
                
```



### 78.子集：回溯算法

递归函数只需要设置成数据长度的层数即可，每一层只有两种情况，选择这个数字，和不选择

```
输入：nums = [1,2,3]
输出：[[],[1],[2],[1,2],[3],[1,3],[2,3],[1,2,3]]
```

```python
class Solution:
    def subsets(self, nums: List[int]) -> List[List[int]]:
        def dfs(index):
            if index == len(nums):
                tmp = sub_res[::]
                res.append(tmp)
                return
            sub_res.append(nums[index])
            dfs(index+1)
            sub_res.remove(nums[index])
            dfs(index+1)

        res = []
        sub_res = []
        dfs(0)
        return res
```





### 79.单词搜索：回溯算法

思路：回溯算法，需要记录状态，当时做的时候忘记了记录第一步的状态，遍历4个方向的时候完全可以提前写好方向，做一个遍历，避免重复代码

```python
class Solution:
    def exist(self, board: List[List[str]], word: str) -> bool:
        def dfs(row,col,max_row,max_col,word,index):
            if board[row][col] != word[index]:
                return False
            if board[row][col] == word[index] and index == len(word)-1:
                return True
            
            for i,j in directions:
                new_row,new_col = row + i,col+j
                if new_row >= 0 and new_row <= max_row and new_col >= 0 and new_col <= max_col and state[new_row][new_col]:
                    state[new_row][new_col] = False
                    if dfs(new_row,new_col,max_row,max_col,word,index+1):
                        return True
                    else:
                        state[new_row][new_col] = True

            return False

        # board = [["A","B","C","E"],["S","F","C","S"],["A","D","E","E"]]
        directions = [(0, 1), (0, -1), (1, 0), (-1, 0)]
        # word = "ABCCED"
        m = len(board)
        n = len(board[0])

        for i in range(m):
            for j in range(n):
                state = [[True for _ in range(n)] for _ in range(m)]
                state[i][j] = False
                if dfs(i,j,m-1,n-1,word,0):
                    return True
                    print(True)
                    break
        return False
        print(False)
```



### 84.柱状图中最大的矩形：单调队列认真看

思路：考虑一个单调增的形状，只有当遇到一个矮的形状a，那么前边以比a高的图形为高，的最大面积才能确定，宽就是这个图形左右两边的距离，比如 a<b<c<d,d>e,那么 以d为高的最大面积 = $d*((e_{index}-d_{index})+(d_{index}-c_{index}))$，因为d，c之间我们完全可以认为全是比d高的图形，在之前被消除了。所以有以下代码

这里记得考虑边界条件，加上哨兵

```python
class Solution:
    def largestRectangleArea(self, heights: List[int]) -> int:
        ans = 0
        n = len(heights)
        heights = [0] + heights + [0]
        stack = [0]
        for i in range(1,n+2):
            while heights[i] < heights[stack[-1]]:
                pos = stack.pop()
                high = heights[pos]
                wide = i - stack[-1] -1 #重点思考这行代码为什么
                print(ans)
                ans = max(high*wide,ans)
                continue
            
            stack.append(i)
        return ans
```



### 121.买卖股票的最佳时机：动态规划

dp[i]代表前i天的最大利润, 状态转移方程为 dp[i] = max(dp[i-1],prices[i]-minp)



```python
class Solution:
    def maxProfit(self, prices: List[int]) -> int:
        n = len(prices)
        dp = [0]*n
        # dp[i]代表前i天的最大利润
        minp = prices[0]
        for i in range(1,n):
            minp = min(minp,prices[i])
            dp[i] = max(dp[i-1],prices[i]-minp)
        return dp[-1]
```



#### 309.买卖股票的最佳时机（含有冷冻期）：动态规划

$dp[i][0]$：表示第i天结束，持有股票的最大收益

$dp[i][1]$：表示第i天结束，不持有股票，且处于冷冻期的最大收益（就是当天卖出股票了）

$dp[i][2]$：表示第i天结束，不持有股票，且不处于冷冻期的最大收益（当天是冷冻或者当天可以买）

那么状态转移方程为

$dp[i][0]$：这时候就是i-1持有天股票，第i天部操作，或者i-1天不持有股票，第i天买入，那么状态转移为
$$
dp[i][0] = max(dp[i-1][0],dp[i-1][2]-prices[i])
$$


$dp[i][1]$：这时候就是i-1持有天股票，第i天卖出，那么状态转移为
$$
dp[i][0] = dp[i-1][0]+prices[i]
$$
$dp[i][2]$：这时候就是i-1持处于冷冻期或者i-1也不是冷冻期，那么状态转移为
$$
dp[i][0] = max(dp[i-1][0],dp[i-1][2]-prices[i])
$$
这里可以多理解一下。空间优化后的代码

```python
class Solution:
    def maxProfit(self, prices: List[int]) -> int:
        n = len(prices)
        if not prices or n == 1:
            return 0
        p1,p2,p3 = -prices[0],0,0
        for i in range(1,n):
            cur1 = max(p1,p3-prices[i])
            cur2 = p1 + prices[i]
            cur3 = max(p3,p2)
            p1,p2,p3 = cur1,cur2,cur3
        return max(cur2,cur3)
```







### 128.最长连续序列：hashset

问题：在数组中找出连续的数字，使得这些数字长度最长；

思路：

1. 先将数组去重
2. 遍历每一个数x，找x+1，x+2....直到不存在；但是这样也有很坏的情况；所以我们在开始寻找之前，先判断x-1，不在数组中

代码如下：

```python
class Solution:
    def longestConsecutive(self, nums: List[int]) -> int:
        nums_set = set(nums)
        max_len = 0

        for i in nums_set:
            if i-1 not in nums_set:
                tmp = i
                cur_long = 0
                while tmp in nums_set:
                    tmp = tmp + 1
                    cur_long = cur_long + 1
                max_len = max(max_len,cur_long)
        return max_len
```







### 136.只出现一次的数字：异或运算

其他数据都出现了两次，那么异或之后就是0，结果就是最后只出现一次的数字了



```python
class Solution:
    def singleNumber(self, nums: List[int]) -> int:
        single = 0
        for i in nums:
            single = single^i
        
        return single
```



### 139.单词拆分：动态规划



dp[i]表示字符串前i个字符能否被分割成单词，所以状态转移公式就是 dp[i] = dp[j] and s[j:i] 是否在单词列表。所有需要两次遍历

注意边界条件的设置

```python
class Solution:
    def wordBreak(self, s: str, wordDict: List[str]) -> bool:
        n = len(s)
        dp = [False] * (n+1) 
        dp[0] = True
        for i in range(1,n+1):
            for j in range(i):
                if(dp[j] and s[j:i] in wordDict):
                    dp[i] = True 
                    break 
        
        return dp[-1]
```



### 152.乘积最大字数组：动态规划

还是数组问题，最长子序列、最大子数组和类似；但是乘积需要考虑负数，所以状态方程需要维护一个最小乘积

```python
class Solution:
    def maxProduct(self, nums: List[int]) -> int:
        maxF = nums[0]
        minF = nums[0]
        ans = nums[0]
        n = len(nums)
        for i in range(1,n):
            maxtmp = max(maxF*nums[i],minF*nums[i],nums[i])
            mintmp = min(maxF*nums[i],minF*nums[i],nums[i])
            maxF = maxtmp
            
            minF = mintmp
            ans = max(ans,maxF)
            print(ans)
        return ans
```







### 198.打家劫舍

每天晚上不能偷相邻的两间房子，那么dp[i]表示前i间房子能偷到的最大金额，那么状态转移方程为
$$
dp[i] = max(dp[i-2]+nums[i],dp[i-1])
$$


```python
class Solution:
    def rob(self, nums: List[int]) -> int:
        if not nums:
            return 0
        n = len(nums)
        if n==1 :
            return nums[0]
        dp = [0] * n 
        dp[0] = nums[0]
        dp[1] = max(nums[0],nums[1])
        for i in range(2,n):
            dp[i] = max(dp[i-2]+nums[i],dp[i-1])
        return dp[-1]
```



### 200.岛屿数量：dfs

```python
class Solution:
    def numIslands(self, grid: List[List[str]]) -> int:


        def dfs(grid,row,col):

            for i,j in dis:
                cur_row,cur_col=row+i,col+j 
                if cur_row >= 0 and cur_row <= len(grid)-1 and cur_col >= 0 and cur_col <= len(grid[0])-1 and grid[cur_row][cur_col] == "1":
                    grid[cur_row][cur_col] = "0"
                    dfs(grid,cur_row,cur_col)
    



        dis = [(1,0),(-1,0),(0,1),(0,-1)]
        m,n = len(grid),len(grid[0])
        ans = 0
        for i in range(m):
            for j in range(n):
                if grid[i][j] == "1":
                    dfs(grid,i,j)
                    ans = ans + 1 
        return ans
```



### 215.数组中第K个最大的元素：堆和快排

```python
class Solution:
    def findKthLargest(self, nums: List[int], k: int) -> int:

        def quick_sort(lists,i,j,taget):
            if i >= j:
                return lists[len(lists)-taget]
            pivot = lists[i]
            low = i
            high = j
            while i < j:
                while i < j and lists[j] >= pivot:
                    
                    j -= 1
                lists[i]=lists[j]
                while i < j and lists[i] <=pivot:
                    i += 1
                lists[j]=lists[i]
            lists[j] = pivot
            print(j)
            if j == len(lists)-taget:
                print(lists)
                return lists[len(lists)-taget]
            if j > len(lists)-taget:
                return quick_sort(lists,low,i-1,taget)
            else:
                return quick_sort(lists,i+1,high,taget)
        
        return quick_sort(nums,0,len(nums)-1,k)
```



### 221.最大正方形：动态规划

$dp[i][j]$以i，j为右下角的全为1的正方形的最大，状态转移方程为：
$$
dp[i][j] = min(dp[i-1][j],dp[i][j-1],dp[i-1][j-1])+1
$$
具体的证明过程可以看leetcode1277题的证明：https://leetcode.cn/problems/count-square-submatrices-with-all-ones/solution/tong-ji-quan-wei-1-de-zheng-fang-xing-zi-ju-zhen-2/

```python
class Solution:
    def maximalSquare(self, matrix: List[List[str]]) -> int:
        m,n = len(matrix),len(matrix[0])
        ans = 0
        dp = [[0 for _ in range(n)] for _ in range(m)]
        for i in range(m):
            for j in range(n):
                if i==0 or j==0:
                    dp[i][j] = int(matrix[i][j])
                elif matrix[i][j] == "0":
                    dp[i][j] = 0
                else:
                    dp[i][j] = min(dp[i-1][j],dp[i][j-1],dp[i-1][j-1]) + 1
                ans = max(ans,dp[i][j])
                
        # print(dp)
        return ans*ans
```





#### 1277统计全为1正方形子矩阵

假如$dp[i][j] = x$那么以i，j为右下角的正方形就有x个，分别是1、2、3、、、x

```python
class Solution:
    def countSquares(self, matrix: List[List[int]]) -> int:
        m,n = len(matrix),len(matrix[0])
        ans = 0
        dp = [[0 for _ in range(n)] for _ in range(m)]
        for i in range(m):
            for j in range(n):
                if i==0 or j==0:
                    dp[i][j] = matrix[i][j]
                elif matrix[i][j] == 0:
                    dp[i][j] = 0
                else:
                    dp[i][j] = min(dp[i-1][j],dp[i][j-1],dp[i-1][j-1]) + 1
                ans = ans + dp[i][j]
        return ans
```







### 238.出自身以外数组的乘积

需要引入两个辅助数组，分别是R和L，L[i]代表i左边所有元素的乘积，R[i]代表i右边左右元素的乘积，那么结果就是

ans[i] = L[i] * R[i]

优化：其实可以只用一个数组，另一个数组在遍历的时候就可以计算得到

```python
class Solution:
    def productExceptSelf(self, nums: List[int]) -> List[int]:
        n = len(nums)
        l = [1] * n 
        r = [1] * n 
        ans = [1] * n
        for i in range(1,n):
            l[i] = l[i-1] * nums[i-1]
        for i in range(1,n):
            r[n-1-i] = r[n-i] * nums[n-i]
        for i in range(n):
            ans[i] = r[i] * l[i]
        # print(l)
        # print(r)
        return ans
        
```



### 239.滑动窗口的最大值：hive源码中max窗口函数的实现方式

思路：维护一个maxChain<i,nums[i]>，每来一条数据，删除所有队尾小于nums[i]的数据，然后将数据添加到尾部；判断当前的队头的元素是否超出窗口范围，超出就删除，然后取队头的元素为窗口的最大值。

```python
class Solution:
    def maxSlidingWindow(self, nums: List[int], k: int) -> List[int]:
        maxChain = list()
        n = len(nums)
        step = 0
        res = []
        for i in range(n):
            while maxChain and maxChain[-1][1] < nums[i]:
                maxChain.pop()
            maxChain.append([i,nums[i]])
            if i - maxChain[0][0] >= k:
                # maxChain.remove(maxChain[0])
                maxChain.pop(0)
                # maxChain = maxChain[1:]
            step = step + 1
            if step >= k:
                res.append(maxChain[0][1])
                # print(maxChain[0][1])
        return res
```



### 287.寻找重复的数：快慢指针

题目：找出数组中唯一的重复数字

注意：区别于找出数组唯一不重复的数字，用异或来做

思路：快慢指针，先找到相遇的位置，再找到环的入口，建议直接看解析

https://leetcode.cn/problems/find-the-duplicate-number/solution/xun-zhao-zhong-fu-shu-by-leetcode-solution/

```python
class Solution:
    def findDuplicate(self, nums: List[int]) -> int:
        slow = 0
        fast = 0 
        slow = nums[slow]
        fast= nums[nums[fast]]
        while slow != fast:
            slow = nums[slow]
            fast = nums[nums[fast]]
        
        res = 0
        while res != slow:
            res = nums[res]
            slow = nums[slow]
        
        return res

```







### 300.最长递增子序列：动态规划

$dp[i]$表示以$nums[i]$结尾的最长递增子序列的长度，要包含nums[i]，那么状态转移方程为
$$
dp[i]=max(dp[j])+1,其中0<j<i,且nums[j]<num[i]
$$


```python
class Solution:
    def lengthOfLIS(self, nums: List[int]) -> int:
        # 动态规划，其中dp[i] 表示以nums[i]为结尾的最长子序列长度，要包含nums[i]
        n = len(nums)
        dp = [1] * n 
        tmp_ans = 1 
        ans = 1
        for i in range(1,n):
            tmp_ans = 1
            for j in range(i):
                if nums[j] < nums[i] and tmp_ans < dp[j]+1:
                    tmp_ans = dp[j] + 1 
            dp[i] = tmp_ans
            ans = max(ans,dp[i])
        return ans
```



### 312.戳气球

题目表述

有 n 个气球，编号为0 到 n - 1，每个气球上都标有一个数字，这些数字存在数组 nums 中。

现在要求你戳破所有的气球。戳破第 i 个气球，你可以获得 nums[i - 1] * nums[i] * nums[i + 1] 枚硬币。 这里的 i - 1 和 i + 1 代表和 i 相邻的两个气球的序号。如果 i - 1或 i + 1 超出了数组的边界，那么就当它是一个数字为 1 的气球。

求所能获得硬币的最大数量。

来源：力扣（LeetCode）
链接：https://leetcode.cn/problems/burst-balloons

解题思路：

$dp[i][j]$表示i，j开区间内可以获取到的最大金币，那么如果这个区间你选择最后一个戳破k，那么状态转移方程
$$
dp[i][j] = dp[i][k] + nums[i]*nums[k]*nums[j] + dp[k][j]
$$
**注：**但是要注意的是，需要在nums开头和结尾都加上边界条件，写代码的时候要注意，两层遍历的时候，外成遍历区间的程度，内部遍历这个长度的每一个区间。和第五题回文子串有点像，代码如下

```python
class Solution:
    def maxCoins(self, nums: List[int]) -> int:
        nums = [1] + nums + [1]
        n = len(nums)
        dp = [[0]*n for _ in range(n)]
        def dp_ij(i,j):
            tmp_res = 0
            for k in range(i+1,j):
                tmp_res = dp[i][k] + nums[i]*nums[k]*nums[j] + dp[k][j]
                dp[i][j] = max(tmp_res,dp[i][j])


        for L in range(2,n):

            for i in range(0,n-L):
                dp_ij(i,i+L)
        
        print(dp[0][n-1])
        return dp[0][n-1]

```







### 322.零钱兑换：动态规划

题目描述：给你一个整数数组 coins ，表示不同面额的硬币；以及一个整数 amount ，表示总金额。

计算并返回可以凑成总金额所需的 最少的硬币个数 。如果没有任何一种硬币组合能组成总金额，返回 -1 。

来源：力扣（LeetCode）
链接：https://leetcode.cn/problems/coin-change

$dp[i]$表示兑换i金额需要的最少金币，那么动态转移方程为:
$$
dp[i] = min(dp[i-coin_j]) +1
$$
其中$coin_j$表示第j个硬币，代码如下，但是考虑先循环金币和先循环金额

```python
class Solution:
    def coinChange(self, coins: List[int], amount: int) -> int:
        n = len(coins)
        dp = [0] + [float('inf')] * (amount)

        # for i in range(1,amount+1):
        #     for j in range(len(coins)):
        #         if i - coins[j] >= 0:
        #             dp[i] = min(dp[i-coins[j]]+1,dp[i])
        for i in range(len(coins)):
            for j in range(coins[i],amount+1):
                dp[j] = min(dp[j-coins[i]]+1,dp[j])

        return dp[amount] if dp[amount] != float('inf') else -1 

```



### 337.打家劫舍3

题目表述

小偷又发现了一个新的可行窃的地区。这个地区只有一个入口，我们称之为 root 。

除了 root 之外，每栋房子有且只有一个“父“房子与之相连。一番侦察之后，聪明的小偷意识到“这个地方的所有房屋的排列类似于一棵二叉树”。 如果 两个直接相连的房子在同一天晚上被打劫 ，房屋将自动报警。

给定二叉树的 root 。返回 在不触动警报的情况下 ，小偷能够盗取的最高金额 。

来源：力扣（LeetCode）
链接：https://leetcode.cn/problems/house-robber-iii

解析：因为不能同时偷父子，所以只需要比较，爷爷和四个孙子钱的总和，两个爸爸的总和，取大的就好；第二点需要注意的是，因为涉及到递归，所以加了hashmap记忆化



```python
# Definition for a binary tree node.
# class TreeNode:
#     def __init__(self, val=0, left=None, right=None):
#         self.val = val
#         self.left = left
#         self.right = right
class Solution:
    def rob(self, root: Optional[TreeNode]) -> int:
        
        def start(root):
            if not root:
                return 0
            if root in hashmap:
                return hashmap[root]
            money = root.val
            if root.left:
                money = money + start(root.left.left) + start(root.left.right)
            if root.right:
                money = money + start(root.right.left) + start(root.right.right)
            
            res = max(money,start(root.left)+start(root.right))
            hashmap[root] = res
            return res
        hashmap = {}
        return start(root)


```



### 347.前k个高频元素

给你一个整数数组 `nums` 和一个整数 `k` ，请你返回其中出现频率前 `k` 高的元素。你可以按 **任意顺序** 返回答案。

```
输入: nums = [1,1,1,2,2,3], k = 2
输出: [1,2]
```



思路：

1. 遍历统计频率保存为hashmap
2. 创建一个arr，长度为数组长度，将map中key对应的频率放在对应数组的索引位置上，（最大频率肯定不会超过数组最大索引）那么arr从后往前遍历，就是出现频次高的到低的元素，直到统计到n个



```python
class Solution:
    def topKFrequent(self, nums: List[int], k: int) -> List[int]:
        hashmap = {}
        n = len(nums)
        for i in range(n):
            if nums[i] in hashmap:
                hashmap[nums[i]] = hashmap[nums[i]] + 1
            else:
                hashmap[nums[i]] = 1
        size = len(hashmap)
        arr = [[] for i in range(n+1)]
       
        # 将统计好的key放到对应的索引位置
        for key in hashmap.keys():
            arr[hashmap[key]].append(key)
            
        res = []
        
       # 倒排去除频率最高的k个
        for i in range(n+1):
            if arr[n-i]:
                res = res + arr[n-i]
           
            if len(res) >= k :
                break 
        return res
```







### 448.找到所有数组中消失的数字

给你一个含 n 个整数的数组 nums ，其中 nums[i] 在区间 [1, n] 内。请你找出所有在 [1, n] 范围内但没有出现在 nums 中的数字，并以数组的形式返回结果。

来源：力扣（LeetCode）
链接：https://leetcode.cn/problems/find-all-numbers-disappeared-in-an-array



思路：让出现的数据，对应的下标都加上n，最后遍历数组，如果对应索引i的数字小于n那么，i就是没有出现的数字

注意：可能存在遍历的时候对应的数据已经加n了，需要nums[i]%n得到原始的数字



```python
class Solution:
    def findDisappearedNumbers(self, nums: List[int]) -> List[int]:

        n = len(nums)
        res = []
        for i in range(n):
           
            x = nums[i] % n
            nums[x-1] = nums[x-1] + n 
        for i in range(n):
            if nums[i] <= n:
                res.append(i+1)
        return res

```



### 560.和为k的子数组

给你一个整数数组 `nums` 和一个整数 `k` ，请你统计并返回 *该数组中和为 `k` 的连续子数组的个数* 。

**示例 1：**

```
输入：nums = [1,1,1], k = 2
输出：2
```



思路很巧妙，sum_pre[i] 表示的nums前i项和的话，那么j到i的子数组和就可以表示为 sum_pre[i] - sum_pre[j]，如果sum_pre[i] - sum_pre[j] == k，那么j到i的自数组就满足条件，所以现在就有两个思路

1、每次求得sum_pre[i]就从0到i遍历，找到满足sum_pre[i] - sum_pre[j] == k的情况

2、将sum_pre[i]存在hashmap中，key就是和，value就是出现的次数，每次求得sum_pre[i]，只需要找 sum_pre[j] = sum_pre[i] - k 在hashmap中的值就好了，也就是和为sum_pre[j]，出现的次数。

**注意**：先将hashmap[0] = 1 ,加入数组，因为可能出现前n个数字刚好和为k，那么这时候也是一种情况，就不会被统计到



```python
class Solution:
    def subarraySum(self, nums: List[int], k: int) -> int:
        hashmap = {}
        sum_pre = 0
        res = 0 
        n = len(nums)
        hashmap[0] = 1
        for i in range(n):
            sum_pre = sum_pre + nums[i]
            # print(sum_pre)
            if (sum_pre - k) in hashmap:
                res = res + hashmap[(sum_pre-k)]
            if sum_pre in hashmap:
                hashmap[sum_pre] = hashmap[sum_pre] + 1
            else:
                hashmap[sum_pre] = 1 
            # print(hashmap)
        return res
```



### 581.最短连续无序子数组

给你一个整数数组 nums ，你需要找出一个 连续子数组 ，如果对这个子数组进行升序排序，那么整个数组都会变为升序排序。

请你找出符合题意的 最短 子数组，并输出它的长度。

来源：力扣（LeetCode）
链接：https://leetcode.cn/problems/shortest-unsorted-continuous-subarray

输入：nums = [2,6,4,8,10,9,15]
输出：5
解释：你只需要对 [6, 4, 8, 10, 9] 进行升序排序，那么整个表都会变为升序排序。



思路：找子数组右边界，维护一个最大值，遍历数组，当nums[i] < 最大值的时候，这是时候肯定是还没有达到右边界，所以右边界 = i，当当nums[i] > 最大值的时候，这时候只需要更新最大值就好了，其他不需要动，因为这时候可能正是在遍历一个有序的数组

同理左边界也是这么找的，代码如下：注意begin和end的边界条件处理。

```python
class Solution:
    def findUnsortedSubarray(self, nums: List[int]) -> int:
        n = len(nums)
        begin,end = 0,-1
        max_num,min_num = nums[0],nums[n-1]
        for i in range(n):
            if nums[i] < max_num:
                end = i
            else:
                max_num = nums[i]
        
        for i in range(n):
            if nums[n-1-i] > min_num:
                begin = n-1-i
            else:
                min_num = nums[n-1-i]
        # if max_num == min_num:
        #     return 0
        return end - begin + 1
```







### 739.每日温度

给定一个整数数组 temperatures ，表示每天的温度，返回一个数组 answer ，其中 answer[i] 是指对于第 i 天，下一个更高温度出现在几天后。如果气温在这之后都不会升高，请在该位置用 0 来代替。

来源：力扣（LeetCode）
链接：https://leetcode.cn/problems/daily-temperatures

**示例 1:**

```
输入: temperatures = [73,74,75,71,69,72,76,73]
输出: [1,1,4,2,1,1,0,0]
```

**单挑栈：类似与柱状图的最大面积84和滑动窗口的最大值239 **



思路：维护一个单调减的队列，当遇到比队尾大的元素就可以，从队尾往前遍历，那么每一次，都是队尾元素第一次遇到的最大值，这里不懂的话可以直接看leetcode的精讲



```python
class Solution:
    def dailyTemperatures(self, temperatures: List[int]) -> List[int]:
        maxchain = []
        n = len(temperatures)
        if n == 1:
            return [1]
        # maxchain.append([0,temperatures[0]])
        res = [0 for i in range(n)]
        for i in range(n):
            while maxchain and temperatures[i] > maxchain[-1][1]:
                res[maxchain[-1][0]] = i - maxchain[-1][0]
                maxchain.pop()
            maxchain.append([i,temperatures[i]])
        return res

```



### 1143.最长公共子序列

给定两个符串 text1 和 text2，返回这两个字符串的最长 公共子序列 的长度。如果不存在 公共子序列 ，返回 0 。

一个字符串的 子序列 是指这样一个新的字符串：它是由原字符串在不改变字符的相对顺序的情况下删除某些字符（也可以不删除任何字符）后组成的新字符串。

**示例 1：**

```
输入：text1 = "abcde", text2 = "ace" 
输出：3  
解释：最长公共子序列是 "ace" ，它的长度为 3 。
```



思路

$dp[i][j]$表示s[i]和s[j]的最长公共子序列，那么状态转移方程为$dp[i][j] = max(dp[i-1][j],dp[i][j-1])$



```python
class Solution:
    def longestCommonSubsequence(self, text1: str, text2: str) -> int:
        n1 = len(text1)
        n2 = len(text2)
        maxnum = max(n1,n2)
        dp = [[0 for i in range(n2+1)] for j in range(n1+1)]
        
        res = 0
        for i in range(1,n1+1):
            for j in range(1,n2+1):
                if text1[i-1] == text2[j-1]:
                    dp[i][j] = dp[i-1][j-1] + 1
                else:
                    dp[i][j] = max(dp[i-1][j],dp[i][j-1])
                
        print(dp[n1][n2])
        return dp[n1][n2]
        # print(res)
```



这是一个递归版的算法

```python

def longestCommonSubsequence(str1, str2) -> int:
    def dp(i, j):
        # 空串的 base case
        if i == -1 or j == -1:
            return 0
        if str1[i] == str2[j]:
            # 这边找到一个 lcs 的元素，继续往前找
            return dp(i - 1, j - 1) + 1
        else:
            # 谁能让 lcs 最长，就听谁的
            return max(dp(i-1, j), dp(i, j-1))
        
    # i 和 j 初始化为最后一个索引
    return dp(len(str1)-1, len(str2)-1)
```



#### 最长公共子串

与上述不一样的是

$dp[i][j]$表示以s[i]和s[j]的结尾的最长公共子串，如果s[i]和s[j]不相等，那么状态转移方程为$dp[i][j] = 0$

相等的话$dp[i][j] = dp[i-1][j-1]+1$

这里需要维护一个 最大值res，想知道内容的话，可以记录下i或者j的位置



### 316.去除重复字母

给你一个字符串 s ，请你去除字符串中重复的字母，使得每个字母只出现一次。需保证 返回结果的字典序最小（要求不能打乱其他字符的相对位置）。

 

示例 1：

输入：s = "bcabc"
输出："abc"
示例 2：

输入：s = "cbacdcbc"
输出："acdb"



思路：用单调栈，找到所有s[i] > s[i+1],并去除第i个位置的元素，为了保证每个元素都出现一次，那么必须满足一下两个条件

1. 单调栈中存在的元素直接跳过，不能再次添加到单调栈中
2. 在弹出栈顶字符时，如果字符串在后面的位置上再也没有这一字符，则不能弹出栈顶字符。为此，需要记录每个字符的剩余数量，当这个值为 0 时，就不能弹出栈顶字符了。

不能回忆出来的话，建议看解析：https://leetcode.cn/problems/remove-duplicate-letters/solution/yi-zhao-chi-bian-li-kou-si-dao-ti-ma-ma-zai-ye-b-4/

```python
class Solution:
    def removeDuplicateLetters(self, s: str) -> str:
        curstackmap = {}
        lastmap = {}
        charstack = []
        for i in s:
            if i in lastmap:
                lastmap[i] = lastmap[i] + 1 
            else:
                lastmap[i] = 1 
            curstackmap[i] = 0
        for i in s:
            while charstack and curstackmap[i] == 0 and i<charstack[-1] and lastmap[charstack[-1]] > 0:
                tmp = charstack.pop()
                curstackmap[tmp] = 0
            if curstackmap[i] == 0:
                charstack.append(i)
                curstackmap[i] = 1
            lastmap[i] = lastmap[i] - 1 
        return ''.join(charstack)
            
```

#### 相似题：

- [316. 去除重复字母](https://leetcode-cn.com/problems/remove-duplicate-letters/)(困难)
- [321. 拼接最大数](https://leetcode-cn.com/problems/create-maximum-number/)(困难)
- [402. 移掉 K 位数字](https://leetcode-cn.com/problems/remove-k-digits/)(中等)
- [1081. 不同字符的最小子序列](https://leetcode-cn.com/problems/smallest-subsequence-of-distinct-characters/)（中等）
