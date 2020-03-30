# 链表环问题

## 141. 环形链表

给定一个链表，判断链表中是否有环。

为了表示给定链表中的环，我们使用整数 `pos` 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 `pos` 是 `-1`，则在该链表中没有环。

 

**示例 1：**

```
输入：head = [3,2,0,-4], pos = 1
输出：true
解释：链表中有一个环，其尾部连接到第二个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist.png)

**示例 2：**

```
输入：head = [1,2], pos = 0
输出：true
解释：链表中有一个环，其尾部连接到第一个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test2.png)

**示例 3：**

```
输入：head = [1], pos = -1
输出：false
解释：链表中没有环。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test3.png)

 

**进阶：**

你能用 *O(1)*（即，常量）内存解决此问题吗？



```
/**
 * Definition for singly-linked list.
 * class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public class Solution {
    public boolean hasCycle(ListNode head) {
        // while(head!=null && head.next!=null){
        //     if (head.next.val==9999){
        //         return true;
        //     }
        //     head.val = 9999;
        //     head = head.next;
        // }
        // return false;
        //转变问题为找相同的数,通过快慢指针+循环来解决
        if (head == null || head.next == null) {
            return false;
        }
        
        ListNode slow = head;
        ListNode fast = head.next;
        
        while (fast != slow) {
            if (fast == null || fast.next == null) {
                return false;
            }
            fast = fast.next.next;
            slow = slow.next;
        }
        return true;
    }
}
```



## 142. 环形链表 II

给定一个链表，返回链表开始入环的第一个节点。 如果链表无环，则返回 `null`。

为了表示给定链表中的环，我们使用整数 `pos` 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 `pos` 是 `-1`，则在该链表中没有环。

**说明：**不允许修改给定的链表。

 

**示例 1：**

```
输入：head = [3,2,0,-4], pos = 1
输出：tail connects to node index 1
解释：链表中有一个环，其尾部连接到第二个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist.png)

**示例 2：**

```
输入：head = [1,2], pos = 0
输出：tail connects to node index 0
解释：链表中有一个环，其尾部连接到第一个节点。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test2.png)

**示例 3：**

```
输入：head = [1], pos = -1
输出：no cycle
解释：链表中没有环。
```

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/07/circularlinkedlist_test3.png)

 

**进阶：**
你是否可以不用额外空间解决此题？



```
/**
 * Definition for singly-linked list.
 * class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) {
 *         val = x;
 *         next = null;
 *     }
 * }
 */
public class Solution {
    public ListNode detectCycle(ListNode head) {
        //LinkedHashSet是找相同数,甚至相同位置的问题的板砖
        // LinkedHashSet lhs = new LinkedHashSet();
        // while(head!=null){
        //     lhs.add(head);
        //     if (head.next!=null && lhs.contains(head.next)){
        //         return head.next;
        //     }
        //     head = head.next;
        // }
        // return null;
        
//         很多题解没有讲清楚非环部分的长度与相遇点到环起点那部分环之间为何是相等的这个数学关系。这里我就补充下为何他们是相等的。
// 假设非环部分的长度是x，从环起点到相遇点的长度是y。环的长度是c。
// 现在走的慢的那个指针走过的长度肯定是x+n1*c+y，走的快的那个指针的速度是走的慢的那个指针速度的两倍。这意味着走的快的那个指针走的长度是2(x+n1*c+y)。

// 还有一个约束就是走的快的那个指针比走的慢的那个指针多走的路程一定是环长度的整数倍。根据上面那个式子可以知道2(x+n1*c+y)-x+n1*c+y=x+n1*c+y=n2*c。

// 所以有x+y=(n2-n1)*c,这意味着什么？我们解读下这个数学公式：非环部分的长度+环起点到相遇点之间的长度就是环的整数倍。这在数据结构上的意义是什么？现在我们知道两个指针都在离环起点距离是y的那个相遇点，而现在x+y是环长度的整数倍，这意味着他们从相遇点再走x距离就刚刚走了很多圈，这意味着他们如果从相遇点再走x就到了起点。
// 那怎么才能再走x步呢？答：让一个指针从头部开始走，另一个指针从相遇点走，等这两个指针相遇那就走了x步此时就是环的起点。
     ListNode fast = head, slow = head;
        while (true) {
            if (fast == null || fast.next == null) return null;
            fast = fast.next.next;
            slow = slow.next;
            if (fast == slow) break;
        }
        fast = head;
        while (slow != fast) {
            slow = slow.next;
            fast = fast.next;
        }
        return fast;

    }
}
```

