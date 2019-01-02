### Remove Duplicates from Sorted Array

Given a sorted array *nums*, remove the duplicates [**in-place**](https://en.wikipedia.org/wiki/In-place_algorithm) such that each element appear only *once* and return the new length.

Do not allocate extra space for another array, you must do this by **modifying the input array in-place** with O(1) extra memory.

解答：

```java
public static int removeDuplicates(int[] arrays){
        int index = arrays[0];
        for(int i = 1;i<arrays.length;i++){
            if (arrays[i] != arrays[index]){

                if (i -index > 1){
                    index = index +1;
                }else {
                    index = i;
                }
                arrays[index] = arrays[i];

            }
        }
        return arrays.length;
    }
```

