#### 98. 验证二叉搜索树

难度中等516

给定一个二叉树，判断其是否是一个有效的二叉搜索树。

假设一个二叉搜索树具有如下特征：

- 节点的左子树只包含**小于**当前节点的数。
- 节点的右子树只包含**大于**当前节点的数。
- 所有左子树和右子树自身必须也是二叉搜索树。

**示例 1:**

```
输入:
    2
   / \
  1   3
输出: true
```

**示例 2:**

```
输入:
    5
   / \
  1   4
     / \
    3   6
输出: false
解释: 输入为: [5,1,4,null,null,3,6]。
     根节点的值为 5 ，但是其右子节点值为 4 
```

```
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
//重中之重不能初始化,初始化了min和max相当于把树的最左,最右做了限制,其实应该是没限制,即非null才进行区间比较
//     public boolean helper(TreeNode node, Integer lower, Integer upper) {
//         if (node == null) return true;

//         int val = node.val;
//         if (lower != null && val <= lower) return false;
//         if (upper != null && val >= upper) return false;

//         // if (! helper(node.right, val, upper)) return false;
//         // if (! helper(node.left, lower, val)) return false;

//         return helper(node.right, val, upper)&&helper(node.left, lower, val);
//    }

//     public boolean isValidBST(TreeNode root) {
//          return helper(root, null, null);
//     }


    public boolean isValidBST(TreeNode root) {
        //递归思路解决这个问题
        if (root==null){
            return true;
        }
        return valid(root,null,null);
    }
    // private boolean valid(TreeNode node, int min, int max){
    //     if (node != null){
    //         if (node.val<min || node.val>max){
    //             return false;
    //         }
    //         return valid(node.left,min,node.val) && valid(node.right,node.val,max);
    //     }
    //     return true;
    // }

    private boolean valid(TreeNode node, Integer min, Integer max){
        if (node != null){
            if (min!=null&&node.val<=min) return false;
            if (max!=null&&node.val>=max) return false;
           
            return valid(node.left,min,node.val) && valid(node.right,node.val,max);
        }
        return true;
    }
}
```

