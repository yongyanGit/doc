### Remove Duplicates from Sorted Array

Given a sorted array *nums*, remove the duplicates [**in-place**](https://en.wikipedia.org/wiki/In-place_algorithm) such that each element appear only *once* and return the new length.

Do not allocate extra space for another array, you must do this by **modifying the input array in-place** with O(1) extra memory.

**Example**

```
Given nums = [1,1,2],

Your function should return length = 2, with the first two elements of nums being 1 and 2 respectively.

It doesn't matter what you leave beyond the returned length.
```



解答：

```java
public static int removeDuplicates(int[] arrays){

        int index = 0;

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

        return index +1;
    }
```

高分解答：

```java
public int removeDuplicates(int[] A) {
    if (A.length==0) return 0;
    int j=0;
    for (int i=0; i<A.length; i++)
        if (A[i]!=A[j]) A[++j]=A[i];
    return ++j;
}
```

