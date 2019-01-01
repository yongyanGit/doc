### Two Sum

Given an array of integers, return **indices** of the two numbers such that they add up to a specific target.

**Example:**

```
Given nums = [2, 7, 11, 15], target = 9,

Because nums[0] + nums[1] = 2 + 7 = 9,
return [0, 1].
```

解答：O(n^2)

```java
int[] arr = new int[]{88,11,555,2,333,11,7,15};
int target = 9;
for (int i = 0; i< arr.length-1;i++){

    for (int j = i+1;j<arr.length;j++){
         if (arr[i]+arr[j] == target){
             System.out.print("["+i+","+j+"]");
             break;
         }
     }
 }
```

最优解：不考虑map集合内部的查询效率，实际复杂度为 O(n)

```java
public int[] twoSum(int[] numbers, int target) {
    int[] result = new int[2];
    Map<Integer, Integer> map = new HashMap<Integer, Integer>();
    for (int i = 0; i < numbers.length; i++) {
        if (map.containsKey(target - numbers[i])) {
            result[1] = i + 1;
            result[0] = map.get(target - numbers[i]);
            return result;
        }
        map.put(numbers[i], i + 1);
    }
    return result;
}
```





