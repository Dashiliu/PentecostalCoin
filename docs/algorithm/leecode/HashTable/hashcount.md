#### 242. 有效的字母异位词

难度简单184

给定两个字符串 *s* 和 *t* ，编写一个函数来判断 *t* 是否是 *s* 的字母异位词。

**示例 1:**

```
输入: s = "anagram", t = "nagaram"
输出: true
```

**示例 2:**

```
输入: s = "rat", t = "car"
输出: false
```

**说明:**
你可以假设字符串只包含小写字母。

**进阶:**
如果输入字符串包含 unicode 字符怎么办？你能否调整你的解法来应对这种情况？



```
class Solution {
    //1.问题 字符串长度.length() 数组长度 .length map大小.size()
    //2.问题 转char数组.toCharArray() map的keyset .keySet()
    //3.问题 Integer.equals(Integer) ,因为== 会比较地址有问题
    //4.优化问题 如果仅包含小写字母可以手动实现char[] c = new char[26]; 包含了桶排序/HashTable的思想
    public boolean isAnagram(String s, String t) {
        if (s.length()!=t.length()){
            return false;
        }
        if (s.equals(t)){
            return true;
        }
        HashMap<Character,Integer> countMap = new HashMap<>(s.length());
        HashMap<Character,Integer> resultMap = new HashMap<>(t.length());
        char[] sc=s.toCharArray();
        char[] tc=t.toCharArray();
        for (int i = 0; i < sc.length; i++){
            countMap.computeIfPresent(sc[i],(k,v)->v+1);
            countMap.putIfAbsent(sc[i],1);
            resultMap.computeIfPresent(tc[i],(k,v)->v+1);
            resultMap.putIfAbsent(tc[i],1);
        }
        if (countMap.size()!=resultMap.size()){
            return false;
        }
        for (char c : countMap.keySet()){
            if (!countMap.get(c).equals(resultMap.get(c))){
                return false;
            }
        }
        for (char c : resultMap.keySet()){
            if (!countMap.get(c).equals(resultMap.get(c))){
                return false;
            }
        }
        return true;
    }
}
```

