### Remove Element

Given an array *nums* and a value *val*, remove all instances of that value [**in-place**](https://en.wikipedia.org/wiki/In-place_algorithm) and return the new length.

Do not allocate extra space for another array, you must do this by **modifying the input array in-place** with O(1) extra memory.

**Example**

```
Given nums = [0,1,2,2,3,0,4,2], val = 2,

Your function should return length = 5, with the first five elements of nums containing 0, 1, 3, 0, and 4.

Note that the order of those five elements can be arbitrary.

It doesn't matter what values are set beyond the returned length.
```



解答：

```java
public int removeElement(int[] nums,int val){

        int index = 0;

        for (int i = 0;i<nums.length;i++){

            if (nums[i] == val){
                if (index == 0 && nums[0] != val){
                    index = i;
                }
            }else {
                nums[index] = nums[i];
                index ++;
            }
        }
        return index;
    }
```

高分解答：

```
 public int removeElement(int[] A, int elem) {
        int m = 0;
        for(int i = 0; i < A.length; i++){

            if(A[i] != elem){
                A[m] = A[i];
                m++;
            }
        }

        return m;
    }
```

