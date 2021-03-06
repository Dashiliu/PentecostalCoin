## 20. 有效的括号

给定一个只包括 `'('`，`')'`，`'{'`，`'}'`，`'['`，`']'` 的字符串，判断字符串是否有效。

有效字符串需满足：

1. 左括号必须用相同类型的右括号闭合。
2. 左括号必须以正确的顺序闭合。

注意空字符串可被认为是有效字符串。

**示例 1:**

```
输入: "()"
输出: true
```

**示例 2:**

```
输入: "()[]{}"
输出: true
```

**示例 3:**

```
输入: "(]"
输出: false
```

**示例 4:**

```
输入: "([)]"
输出: false
```

**示例 5:**

```
输入: "{[]}"
输出: true
```



```
class Solution {
    // public boolean isValid(String s) {
        //5个细节
        // 1.stack的创建,泛型是只能是引用对象
        // 2.push <> pop(拿出去)/peek只是看一眼,不拿出去
        // 3.基本类型是通过 == 判断
        // 4.每次判断后面括号需要判断stack是否为空
        // 5.string 循环完后,需要判断stack是否两两清空
        // Stack<Character> stack = new Stack<Character>();
        // for(char c : s.toCharArray()){
        //     if (c == '('||c=='{'||c=='['){
        //         stack.push(c);
        //         continue;
        //     }
        //     if (c==')'){
        //         if(stack.size()<1||'('!=stack.pop()){
        //             return false;
        //         }
                
        //     }
        //     if (c=='}'){
        //         if(stack.size()<1||'{'!=stack.pop()){
        //             return false;
        //         }
            
        //     }
        //     if (c==']'){
        //         if(stack.size()<1||'['!=stack.pop()){
        //             return false;
        //         }
        //     }
        // }
        // return stack.size()==0;
    // }

  // Hash table that takes care of the mappings.
  private HashMap<Character, Character> mappings;

  // Initialize hash map with mappings. This simply makes the code easier to read.
  public Solution() {
    this.mappings = new HashMap<Character, Character>();
    this.mappings.put(')', '(');
    this.mappings.put('}', '{');
    this.mappings.put(']', '[');
  }

  public boolean isValid(String s) {

    // Initialize a stack to be used in the algorithm.
    Stack<Character> stack = new Stack<Character>();

    for (int i = 0; i < s.length(); i++) {
      char c = s.charAt(i);

      // If the current character is a closing bracket.
      if (this.mappings.containsKey(c)) {

        // Get the top element of the stack. If the stack is empty, set a dummy value of '#'
        char topElement = stack.empty() ? '#' : stack.pop();

        // If the mapping for this bracket doesn't match the stack's top element, return false.
        if (topElement != this.mappings.get(c)) {
          return false;
        }
      } else {
        // If it was an opening bracket, push to the stack.
        stack.push(c);
      }
    }

    // If the stack still contains elements, then it is an invalid expression.
    return stack.isEmpty();
  }
}
```

