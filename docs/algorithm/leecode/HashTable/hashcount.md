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
//多种解法1.仅小写字母,用快排O(nlogn)后比较2.桶排序思想3.hashtable计数
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



#### 1. 两数之和

难度简单8118

给定一个整数数组 `nums` 和一个目标值 `target`，请你在该数组中找出和为目标值的那 **两个** 整数，并返回他们的数组下标。

你可以假设每种输入只会对应一个答案。但是，数组中同一个元素不能使用两遍。

 

**示例:**

```
给定 nums = [2, 7, 11, 15], target = 9

因为 nums[0] + nums[1] = 2 + 7 = 9
所以返回 [0, 1]
```



```
class Solution {
    //多种方法1.遍历,hash<value,index>
    //1.问题 注意Map的k,v都是Integer需要用对象来取,数组长度 .length
    public int[] twoSum(int[] nums, int target) {
        Map<Integer,Integer> countMap = new HashMap<>(nums.length);
        Integer firstIndex = null;
        for (int i = 0; i<nums.length; i++){
            if ((firstIndex=countMap.get(new Integer(target-nums[i])))!=null){
                return new int[]{firstIndex,i};
            }
            countMap.put(nums[i],i);
        }
        return null;
    }
}
```



#### 15. 三数之和

难度中等2039

给你一个包含 *n* 个整数的数组 `nums`，判断 `nums` 中是否存在三个元素 *a，b，c ，*使得 *a + b + c =* 0 ？请你找出所有满足条件且不重复的三元组。

**注意：**答案中不可以包含重复的三元组。

 

**示例：**

```
给定数组 nums = [-1, 0, 1, 2, -1, -4]，

满足要求的三元组集合为：
[
  [-1, 0, 1],
  [-1, -1, 2]
]
```



```
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        //思路,和二数求和不同,因为2数直接就是已知一个求一个,所以需要构造二数条件
        //通过快排,找到最小非负数,以此为分界点包含分界点向右的计算22求和或者单个放入map中
        //左侧双层循环,单层中查右侧双,双层中查右侧单

        //上面的思路是对二数之和没有真实理解的情况,实际只用 -(a+b)=c就可 O(n2)
        //另外一种方法就是减少空间复杂度的先快排O(nlogn),然后O(n)循环a,剩下的b和c因为是排过序的所以
        //从两边往中间找,O(n)最后的时间复杂度是O(nlogn+n2),但是空间会少一点


    }
}
```

