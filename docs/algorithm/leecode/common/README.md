1.数组遍历框架

```
void traverse(int[] arr) {
  for (int i = 0; i < arr.length; i++) {
    // 迭代访问 arr[i]
  }
}
```

2.链表遍历框架

```
/* 基本的单链表节点 */
class ListNode {
  int val;
  ListNode next;
}
void traverse(ListNode head) {
  for (ListNode p = head; p != null; p = p.next) {
    // 迭代访问 p.val
  }
}
void traverse(ListNode head) {
  // 递归访问 head.val
  traverse(head.next)
}
```

3.⼆叉树遍历框架

```
/* 基本的⼆叉树节点 */
class TreeNode {
  int val;
  TreeNode left, right;
}
void traverse(TreeNode root) {
  //前序遍历
  traverse(root.left)
  //中序遍历
  traverse(root.right)
  //后序遍历
}
```

4.N 叉树的遍历框架

```
/* 基本的 N 叉树节点 */
class TreeNode {
  int val;
  TreeNode[] children;
}
void traverse(TreeNode root) {
  for (TreeNode child : root.children)
  	// 前序遍历需要的操作
  	traverse(child)
  	// 后序遍历需要的操作
}
```

5.动态规划

```
如何发现是动态规划?
1.求最值问题(求最⻓递增⼦序列，最⼩编辑距离)
2.核⼼问题是穷举,存在「重叠⼦问题」,具备「最优⼦结构」即子问题互相独立
3.正确的「状态转移⽅程」(明确「状态」 -> 定义 dp 数组/函数的含义 -> 明确「选择」-> 明确 basecase)
4.备忘录数组或map避免重叠子问题

动态规划套路(找零钱)
1.先确定「状态」，也就是原问题和⼦问题中变化的变量。由于硬币数量⽆限，所以唯⼀的状态就是⽬标⾦额 amount 。
2.然后确定 dp 函数的定义.当前的⽬标⾦额是 n ，⾄少需要 dp(n) 个硬币凑出该⾦额。
3.然后确定「选择」并择优.也就是对于每个状态，可以做出什么选择改变当前状态。具体到这个问题，⽆论当的⽬标⾦额是多少，选择就是从⾯额列表coins 中选择⼀个硬币，然后⽬标⾦额就会减少
4.最后明确 base case，显然⽬标⾦额为 0 时，所需硬币数量为 0；当⽬标⾦额⼩于 0 时，⽆解，返回 -1

示例:
*找零钱
递归
def coinChange(coins: List[int], amount: int):
	# 备忘录
	memo = dict()
	def dp(n):
		# 查备忘录，避免重复计算
		if n in memo: return memo[n]
		if n == 0: return 0
		if n < 0: return -1
		res = float('INF')
		for coin in coins:
				subproblem = dp(n - coin)
				if subproblem == -1: continue
				res = min(res, 1 + subproblem)
		# 记⼊备忘录
		memo[n] = res if res != float('INF') else -1
		return memo[n]
return dp(amount)

迭代 dp[i] = x 表⽰，当⽬标⾦额为 i 时，⾄少需要 x 枚硬币
int coinChange(vector<int>& coins, int amount) {
	// 数组⼤⼩为 amount + 1，初始值也为 amount + 1
	vector<int> dp(amount + 1, amount + 1);
	// base case
	dp[0] = 0;
	for (int i = 0; i < dp.size(); i++) {
		// 内层 for 在求所有⼦问题 + 1 的最⼩值
		for (int coin : coins) {
			// ⼦问题⽆解，跳过
			if (i - coin < 0) continue;
				dp[i] = min(dp[i], 1 + dp[i - coin]);
		}
	}
	return (dp[amount] == amount + 1) ? -1 : dp[amount];
}



*斐波那契数列
递归 自底向上
int fib(int N) {
  if (N < 1) return 0;
  // 备忘录全初始化为 0
  vector<int> memo(N + 1, 0);
  // 初始化最简情况
  return helper(memo, N);
}
int helper(vector<int>& memo, int n) {
  // base case
  if (n == 1 || n == 2) return 1;
  // 已经计算过
  if (memo[n] != 0) return memo[n];
  memo[n] = helper(memo, n - 1) +
  helper(memo, n - 2);
  return memo[n];
}

迭代 自上而下
int fib(int n) {
  if (n == 2 || n == 1)
    return 1;
  int prev = 1, curr = 1;
  for (int i = 3; i <= n; i++) {
    int sum = prev + curr;
    prev = curr;
    curr = sum;
  }
  return curr;
}

```

6.回溯算法框架(DFS深度优先搜索)

```
解决⼀个回溯问题，实际上就是⼀个决策树的遍历过程.和动态规划的区别是子问题不独立即有选择列表
1、路径：也就是已经做出的选择。
2、选择列表：也就是你当前可以做的选择。
3、结束条件：也就是到达决策树底层，⽆法再做选择的条件。

result = []
def backtrack(路径, 选择列表):
	if 满⾜结束条件:
		result.add(路径)
		return
	for 选择 in 选择列表:
		//做选择
		//1.将该选择从选择列表移除
		//2.路径.add(选择)
		backtrack(路径, 选择列表)
		//撤销选择
		//1.路径.remove(选择)
		//2.将该选择再加⼊选择列表

示例:
*全排列问题
List<List<Integer>> res = new LinkedList<>();
/* 主函数，输⼊⼀组不重复的数字，返回它们的全排列 */
List<List<Integer>> permute(int[] nums) {
	// 记录「路径」
	LinkedList<Integer> track = new LinkedList<>();
	backtrack(nums, track);
	return res;
}
// 路径：记录在 track 中
// 选择列表：nums 中不存在于 track 的那些元素
// 结束条件：nums 中的元素全都在 track 中出现
void backtrack(int[] nums, LinkedList<Integer> track) {
	// 触发结束条件
	if (track.size() == nums.length) {
		res.add(new LinkedList(track));
		return;
	}
	for (int i = 0; i < nums.length; i++) {
		// 排除不合法的选择
		if (track.contains(nums[i]))
			continue;
		// 做选择
		track.add(nums[i]);
		// 进⼊下⼀层决策树
		backtrack(nums, track);
		// 取消选择
		track.removeLast();
	}
}
```

7.BFS 广度优先搜索框架

```
BFS 相对 DFS 的最主要的区别是：BFS 找到的路径⼀定是最短的，但代价就是空间复杂度⽐ DFS ⼤很多.
如 「⼆叉树的最⼩⾼度」和「打开密码锁的最少步数」本质上就是⼀幅「图」，让你从⼀个起点，⾛到终点，问最短路径
DFS 实际上是靠递归的堆栈记录⾛过的路径,⽽ BFS 借助队列做到⼀次⼀步「⻬头并进」，是可以在不遍历完整棵树的条件下找到最短距离的.

// 计算从起点 start 到终点 target 的最近距离
int BFS(Node start, Node target) {
	Queue<Node> q; // 核⼼数据结构
	Set<Node> visited; // 避免⾛回头路
	
	q.offer(start); // 将起点加⼊队列
	visited.add(start);
	int step = 0; // 记录扩散的步数
	while (q not empty) {
		int sz = q.size();
		/* 将当前队列中的所有节点向四周扩散 */
		for (int i = 0; i < sz; i++) {
			Node cur = q.poll();
			/* 划重点：这⾥判断是否到达终点 */
			if (cur is target)
				return step;
			/* 将 cur 的相邻节点加⼊队列 */
			for (Node x : cur.adj()){
				if (x not in visited) {
					q.offer(x);
					visited.add(x);
				}
			}
		}
		/* 划重点：更新步数在这⾥ */
		step++;
	}
}
BFS 的核⼼数据结构； cur.adj() 泛指 cur 相邻的节点，⽐如说⼆维数组中， cur 上下左右四⾯的位置就是相邻节点； visited 的主要作⽤是防⽌⾛回头路，⼤部分时候都是必须的，但是像⼀般的⼆叉树结构没有⼦节点到⽗节点的指针，不会⾛回头路就不需要 visited 。

示例:
*⼆叉树的最⼩⾼度
int minDepth(TreeNode root) {
	if (root == null) return 0;
	Queue<TreeNode> q = new LinkedList<>();
	q.offer(root);
	// root 本⾝就是⼀层，depth 初始化为 1
	int depth = 1;
	
	while (!q.isEmpty()) {
		int sz = q.size();
		/* 将当前队列中的所有节点向四周扩散 */
		for (int i = 0; i < sz; i++) {
			TreeNode cur = q.poll();
			/* 判断是否到达终点 */
			if (cur.left == null && cur.right == null)
				return depth;
			/* 将 cur 的相邻节点加⼊队列 */
			if (cur.left != null)
				q.offer(cur.left);
			if (cur.right != null)
				q.offer(cur.right);
		}
		/* 这⾥增加步数 */
		depth++;
	}
	return depth;
}

```

8.双向 BFS 优化

```
传统的 BFS 框架就是从起点开始向四周扩散，遇到终点时停⽌；⽽双向 BFS 则是从起点和终点同时开始扩散，当两边有交集的时候停⽌。必须知道终点在哪⾥.
int openLock(String[] deadends, String target) {
	Set<String> deads = new HashSet<>();
	for (String s : deadends) deads.add(s);
	// ⽤集合不⽤队列，可以快速判断元素是否存在
	Set<String> q1 = new HashSet<>();
	Set<String> q2 = new HashSet<>();
	Set<String> visited = new HashSet<>();
	int step = 0;
	q1.add("0000");
	q2.add(target);
	while (!q1.isEmpty() && !q2.isEmpty()) {
		// 哈希集合在遍历的过程中不能修改，⽤ temp 存储扩散结果
		Set<String> temp = new HashSet<>();
		/* 将 q1 中的所有节点向周围扩散 */
		for (String cur : q1) {
			/* 判断是否到达终点 */
			if (deads.contains(cur))
				continue;
			if (q2.contains(cur))
				return step;
			visited.add(cur);
			/* 将⼀个节点的未遍历相邻节点加⼊集合 */
			for (int j = 0; j < 4; j++) {
				String up = plusOne(cur, j);
				if (!visited.contains(up))
				temp.add(up);
				String down = minusOne(cur, j);
				if (!visited.contains(down))
					temp.add(down);
			}
		}
		/* 在这⾥增加步数 */
		step++;
		// temp 相当于 q1
		// 这⾥交换 q1 q2，下⼀轮 while 就是扩散 q2
		q1 = q2;
		q2 = temp;
	}
	return -1;
}
```

9.二分查找框架

```
int binarySearch(int[] nums, int target) {
	int left = 0, right = ...;
	while(...) {
		int mid = left + (right - left) / 2;
		if (nums[mid] == target) {
			...
		} else if (nums[mid] < target) {
			left = ...
		} else if (nums[mid] > target) {
			right = ...
		}
	}
	return ...;
}

示例:
*查询数组中是否存在一个数
int binarySearch(int[] nums, int target) {
	int left = 0;
	int right = nums.length - 1; // 注意
	while(left <= right) {
		int mid = left + (right - left) / 2;
		if (nums[mid] < target) {
			left = mid + 1;
		} else if (nums[mid] > target) {
			right = mid - 1;
		} else if(nums[mid] == target) {
			// 直接返回
			return mid;
		}
	}
	return -1;
}
```

10.寻找边界的二分搜索框架

```
左侧边界
int left_bound(int[] nums, int target) {
	int left = 0, right = nums.length - 1;
	while (left <= right) {
		int mid = left + (right - left) / 2;
		if (nums[mid] < target) {
			left = mid + 1;
		} else if (nums[mid] > target) {
			right = mid - 1;
		} else if (nums[mid] == target) {
			// 别返回，锁定左侧边界
			right = mid - 1;
		}
	}
	// 最后要检查 left 越界的情况
	if (left >= nums.length || nums[left] != target)
		return -1;
	return left;
}
右侧边界
int right_bound(int[] nums, int target) {
	int left = 0, right = nums.length - 1;
	while (left <= right) {
		int mid = left + (right - left) / 2;
		if (nums[mid] < target) {
			left = mid + 1;
		} else if (nums[mid] > target) {
			right = mid - 1;
		} else if (nums[mid] == target) {
			// 别返回，锁定右侧边界
			left = mid + 1;
		}
	}
	// 最后要检查 right 越界的情况
	if (right < 0 || nums[right] != target)
		return -1;
	return right;
}
```

11.快慢指针 

```
一般问题是在链表上找某个位置
示例:	
*寻找链表中点
while (fast != null && fast.next != null) {
    fast = fast.next.next;
    slow = slow.next;
}
// slow 就在中间位置
return slow;

*寻找链表的倒数第 k 个元素
ListNode slow, fast;
slow = fast = head;
while (k-- > 0) 
    fast = fast.next;

while (fast != null) {
    slow = slow.next;
    fast = fast.next;
}
return slow;
```

12.左右指针

```
左右指针在数组中实际是指两个索引值，一般初始化为 left = 0, right = nums.length - 1 。
一般问题是数组上的操作,有序或者不需要顺序
示例:
*二分查找

*两数之和
int[] twoSum(int[] nums, int target) {
    int left = 0, right = nums.length - 1;
    while (left < right) {
        int sum = nums[left] + nums[right];
        if (sum == target) {
            // 题目要求的索引是从 1 开始的
            return new int[]{left + 1, right + 1};
        } else if (sum < target) {
            left++; // 让 sum 大一点
        } else if (sum > target) {
            right--; // 让 sum 小一点
        }
    }
    return new int[]{-1, -1};
}

*反转数组
void reverse(int[] nums) {
    int left = 0;
    int right = nums.length - 1;
    while (left < right) {
        // swap(nums[left], nums[right])
        int temp = nums[left];
        nums[left] = nums[right];
        nums[right] = temp;
        left++; right--;
    }
}
```

13.滑动窗口

```
int left = 0, right = 0;
while (right < s.size()) {`
	// 增⼤窗⼝
	window.add(s[right]);
	right++;//窗口右边界右移
	while (window 判断是否需要收缩) {
		// 缩⼩窗⼝
		window.remove(s[left]);
		left++;//窗口左边界右移
	}
}
```

