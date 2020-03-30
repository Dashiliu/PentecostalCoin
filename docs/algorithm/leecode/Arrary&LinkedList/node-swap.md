## 206.反转链表

反转一个单链表。

**示例:**

```
输入: 1->2->3->4->5->NULL
输出: 5->4->3->2->1->NULL
```

**进阶:**
你可以迭代或递归地反转链表。你能否用两种方法解决这道题？

```
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    public ListNode reverseList(ListNode head) {
        // if (head==null || head.next == null){
        //     return head;
        // }
        // ListNode currentNode = head;
        // ListNode nextNode = currentNode.next;
        // head.next = null;
        // while (nextNode!=null){
        //     ListNode nextNode1 = nextNode.next;
        //     nextNode.next = currentNode;
        //     //向后传递
        //     currentNode = nextNode;
        //     nextNode = nextNode1;
        // }
        //return currentNode;
        
        // ListNode newl = null;
        // ListNode li = null;
        // while (head != null){
        //     newl = head;
        //     head = head.next;
        //     newl.next = li;
        //     li = newl;
        // }
        // return newl;

        //递归终止条件是当前为空，或者下一个节点为空
		if(head==null || head.next==null) {
			return head;
		}
		//这里的cur就是最后一个节点
		ListNode cur = reverseList(head.next);
		//这里请配合动画演示理解
		//如果链表是 1->2->3->4->5，那么此时的cur就是5
		//而head是4，head的下一个是5，下下一个是空
		//所以head.next.next 就是5->4
		head.next.next = head;
		//防止链表循环，需要将head.next设置为空
		head.next = null;
		//每层递归函数都返回cur，也就是最后一个节点
		return cur;
    }
}
```



## 24.两两交换链表中的节点

给定一个链表，两两交换其中相邻的节点，并返回交换后的链表。

**你不能只是单纯的改变节点内部的值**，而是需要实际的进行节点交换。

 

**示例:**

```
给定 1->2->3->4, 你应该返回 2->1->4->3.
```

```
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
    //类似链表反转,但是有两个需要注意的地方
    //1.返回值是第二个<-需要判空能不能有第二个
    //2.在交换完一次后,是否能交换第二次时,需要判断如果不能交换第二次,交换后的第二个节点需要连3,否则连4
    public static ListNode swapPairs(ListNode head) {         
    //  if (head == null || head.next == null){               
    //      return head;                                      
    //  }                                                     
    //  ListNode newHead = null;                       
    //  ListNode firstNode = null;                     
    //  ListNode secondNode = null;                    
    //  int count = 0;                                 
    //  while(head != null && head.next != null){       
    //      firstNode = head;                              
    //      secondNode = head.next;                           
    //      head =  head.next.next;                           
    //      if (head!=null && head.next!=null){               
    //         firstNode.next=head.next;                      
    //      }else {                                           
    //          firstNode.next=head;                          
    //      }                                                 
    //      secondNode.next=firstNode;                        
    //      if (++count==1){                                  
    //          newHead = secondNode;                         
    //      }                                                 
    //  }                                                     
    //  return newHead;       
        ListNode pre = new ListNode(0);
        pre.next = head;
        ListNode temp = pre;
        while(temp.next != null && temp.next.next != null) {
            ListNode start = temp.next;
            ListNode end = temp.next.next;
            temp.next = end;
            start.next = end.next;
            end.next = start;
            temp = start;
        }
        return pre.next;                                
 }                                                         
}
```



## 25.K 个一组翻转链表

给你一个链表，每 *k* 个节点一组进行翻转，请你返回翻转后的链表。

*k* 是一个正整数，它的值小于或等于链表的长度。

如果节点总数不是 *k* 的整数倍，那么请将最后剩余的节点保持原有顺序。

 

**示例：**

给你这个链表：`1->2->3->4->5`

当 *k* = 2 时，应当返回: `2->1->4->3->5`

当 *k* = 3 时，应当返回: `3->2->1->4->5`

 

**说明：**

- 你的算法只能使用常数的额外空间。
- **你不能只是单纯的改变节点内部的值**，而是需要实际进行节点交换。



```
/**
 * Definition for singly-linked list.
 * public class ListNode {
 *     int val;
 *     ListNode next;
 *     ListNode(int x) { val = x; }
 * }
 */
class Solution {
//     public ListNode reverseKGroup(ListNode head, int k) {
//         //校验值
//         int count = 0;
//         ListNode l2=null;
//         ListNode tmp=null;
//         ListNode returnHead=null;
//         while(++count<=k && head!=null && head.next!=null){
//             l2 = head.next;
//             if (count==k && l2!=null){
//                 tmp = l2.next;
//                 reverse(head);
                
//                 // {
//                 //     ListNode n1=head;
//                 //     ListNode n2=head.next;
//                 //     ListNode n3=head.next.next;
//                 //     while(n1!=l2&&n3!=null){
//                 //         n3=n2.next;
//                 //         n2.next=n1;
//                 //         n1=n2;
//                 //         n2=n3;
//                 //     }
//                 // }

//                 if (returnHead==null){
//                     returnHead=l2;
//                 }
//                 //方法中重新赋值了l1和l2
//                 //l1的是尾,l2是头
//                 head.next=tmp;
//                 count=0;
//             }
//         }
//         return returnHead;
//     }
//     private ListNode reverse(ListNode head) {
//     ListNode pre = null;
//     ListNode curr = head;
//     while (curr != null) {
//         ListNode next = curr.next;
//         curr.next = pre;
//         pre = curr;
//         curr = next;
//     }
//     return pre;
// }
public ListNode reverseKGroup(ListNode head, int k) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;

    ListNode pre = dummy;
    ListNode end = dummy;

    while (end.next != null) {
        for (int i = 0; i < k && end != null; i++) end = end.next;
        if (end == null) break;
        ListNode start = pre.next;
        ListNode next = end.next;
        end.next = null;
        pre.next = reverse(start);
        start.next = next;
        pre = start;

        end = pre;
    }
    return dummy.next;
}

private ListNode reverse(ListNode head) {
    ListNode pre = null;
    ListNode curr = head;
    while (curr != null) {
        ListNode next = curr.next;
        curr.next = pre;
        pre = curr;
        curr = next;
    }
    return pre;
}

}
```

