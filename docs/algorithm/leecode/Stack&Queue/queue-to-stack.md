## 225. 用队列实现栈

使用队列实现栈的下列操作：

- push(x) -- 元素 x 入栈
- pop() -- 移除栈顶元素
- top() -- 获取栈顶元素
- empty() -- 返回栈是否为空

**注意:**

- 你只能使用队列的基本操作-- 也就是 `push to back`, `peek/pop from front`, `size`, 和 `is empty` 这些操作是合法的。
- 你所使用的语言也许不支持队列。 你可以使用 list 或者 deque（双端队列）来模拟一个队列 , 只要是标准的队列操作即可。
- 你可以假设所有操作都是有效的（例如, 对一个空的栈不会调用 pop 或者 top 操作）。

```
// class MyStack {
// //注意点
// //1.初始化实现类ArrayDeque<Integer> +方法名 add<tail>offer remove<head>poll element<>peek 有异常<>无异常
// //2.操作需要上锁,不然会冲突
//     private Queue<Integer> queue;
//     /** Initialize your data structure here. */
//     public MyStack() {
//         queue = new ArrayDeque<Integer>();
//     }
    
//     /** Push element x onto stack. */
//     public void push(int x) {
//         synchronized(queue){
//             Queue<Integer> newQueue = new  ArrayDeque<Integer>();
//             newQueue.add(x);
//             while(!queue.isEmpty()){
//                 newQueue.add(queue.remove());     
//             }
//             queue = newQueue;
//         }
//     }
    
//     /** Removes the element on top of the stack and returns that element. */
//     public int pop() {
//         synchronized(queue){
//             return queue.poll();
//         }
//     }
    
//     /** Get the top element. */
//     public int top() {
//         return queue.peek();
//     }
    
//     /** Returns whether the stack is empty. */
//     public boolean empty() {
//         return queue!=null?queue.isEmpty():true;
//     }
// }
class MyStack {
//注意点
//1.初始化实现类ArrayDeque<Integer> +方法名 add<tail>offer尾+ remove<head>poll头- element<>peek头看 有异常<>无异常
//2.问题 怎么插怎么加其实没有倒转,所以需要找加的末尾哪个才是倒转第一个,另外为了两次操作都能进行需要in和out交换并清空out
    private Queue<Integer> in;
    private Queue<Integer> out;
    private Integer top;
    /** Initialize your data structure here. */
    public MyStack() {
        in = new ArrayDeque<Integer>();
        out = new ArrayDeque<Integer>();
    }
    
    /** Push element x onto stack. */
    public void push(int x) {
        in.add(x);
    }
    
    /** Removes the element on top of the stack and returns that element. */
    public int pop() {
        if (out.isEmpty()){
            //判断in是否为空,不为空倒转后从out拿,为空则null
            while(in.size()>1){
                out.add(in.poll());
            }       
            top = in.poll();
        }
        in = out;
        out= new ArrayDeque<Integer>();
        return top;  
    }
    
    /** Get the top element. */
    public int top() {
        if (out.isEmpty()){
            //判断in是否为空,不为空倒转后从out拿,为空则null
            while(!in.isEmpty()){
                top = in.poll();
                out.add(top);
            }       
        }
        in = out;
        out= new ArrayDeque<Integer>();
        return top;  
    }
     
    /** Returns whether the stack is empty. */
    public boolean empty() {
        return in.isEmpty()&&out.isEmpty();
    }
}
/**
 * Your MyStack object will be instantiated and called as such:
 * MyStack obj = new MyStack();
 * obj.push(x);
 * int param_2 = obj.pop();
 * int param_3 = obj.top();
 * boolean param_4 = obj.empty();
 */
```



