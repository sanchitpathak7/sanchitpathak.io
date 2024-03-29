+++
title = "Sorting Algorithms Zion"
date = 2024-03-13
+++

In this blog post I am going over different sorting algorithms as I understand them and also compare their efficieny in terms of time complexity.

## Insertion Sort
Insertion sort breaks the arrays into sub-arrays and sorts them individually, which results in a sorted array. This is a stable sorting algorithm since the relative order will remain the same.

### Pseudocode
```
def sortArray(self, nums: List[int]) -> List[int]:
# Insertion sort breaks the array into sub-arrays and sort them individually
# Considering 2 pointers approach
    for i in range(1, len(nums)):
        j = i - 1
        while j >=0 and nums[j + 1] < nums[j]:
            tmp = nums[j + 1]
            nums[j + 1] = nums[j]
            nums[j] = tmp
            j -= 1
    return nums
```

```
Input: nums = [5,1,1,2,0,0]
Output: [0,0,1,1,2,5]
```

### Breakdown
- The i pointer points of the index n−1 where n is the size of the current sub-array.
- The j pointer starts off with being one index behind i and as long as j does not go out of bounds, that is, it is not at a negative index, and the j+1 element is smaller than the jth element, we keep decrementing j. This will ensure that we have sorted all the elements before the ith index before moving to the next sub-array (iteration).
- Above code runs but if submitted on leetcode it gives time limit exceeded error. Reason below.

### Time Complexity
- When using insertion sort on a dataset of size n, if the array is already sorted in the best-case scenario, the operation completes in O(n) time. This is because every element in the array still needs to be checked, but the while loop doesn't need to run.
- However, in the worst-case scenario where all elements are sorted in reverse order, the while loop will run n times within the for loop, leading to a time complexity of O(n^2).
- Thus, the algorithm is efficient for only small input sizes.


---
## Merge Sort
Merge sort is one of the common sorting algorithms as it's efficient. The basic idea is to take the input array and split them into approximately equal halves and then for those halves split them further till size one is reached and then sort and merge them back in a recursive manner. It is a stable algorithm since the relative order will remain the same.

![merge-sort](merge-sort.png)

### Pseudocode
```
def mergeSort(arr, start, end):
    if end - start + 1 <= 1:
        return arr

    # Determine the middle index of the array
    middle = (start + end) // 2

    # Sort the left half
    mergeSort(arr, start, middle)

    # Sort the right half
    mergeSort(arr, middle + 1, end)

    # Merge the sorted halves
    merge(arr, start, middle, end)
    
    return arr

# Merge in-place
def merge(arr, start, middle, end):
    # Copy the sorted left and right halves to temporary arrays
    left_half = arr[start: middle + 1]
    right_half = arr[middle + 1: end + 1]

    # Initialize pointers for left and right arrays
    left_pointer = 0
    right_pointer = 0
    array_pointer = start

    # Merge the two sorted halves into the original array
    while left_pointer < len(left_half) and right_pointer < len(right_half):
        if left_half[left_pointer] <= right_half[right_pointer]:
            arr[array_pointer] = left_half[left_pointer]
            left_pointer += 1
        else:
            arr[array_pointer] = right_half[right_pointer]
            right_pointer += 1
        array_pointer += 1

    # Copy any remaining elements from the left half
    while left_pointer < len(left_half):
        arr[array_pointer] = left_half[left_pointer]
        left_pointer += 1
        array_pointer += 1

    # Copy any remaining elements from the right half
    while right_pointer < len(right_half):
        arr[array_pointer] = right_half[right_pointer]
        right_pointer += 1
        array_pointer += 1
```

```
Input: arr = [5,1,1,2,0,0]
Output: [0,0,1,1,2,5]
```

### Breakdown 
-
-
-

### Time Complexity
- If n is the length of the array, when we go down the levels, we will get n/2 to the power of x, where we divide n by 2 until we get to 1.

- Calculation:
    - Multiply both side by 2^x:
`n = 2^x`
    - Take log on both sides:
`log n = log2^x`
    - Simplify to:
`log n = xlog2`
    - Hence, we get `Olog(n)` for the mergeSort() function.
    - Now, for merge() function, we have n steps at any level of  the tree. Thus, overall time complexity as `O(nlogn)`.
- Merge sort runs in the same time complexity in the worst, average and best case scenarios making it more efficient that insertion sort.

---
## Quick Sort

### Pseudocode
```

```

### Breakdown
-
-
-

### Time Complexity
-
-
-
---
## Bucket Sort

### Pseudocode
```

```

### Breakdown
-
-
-

### Time Complexity
-
-
-
---
## Bubble Sort

### Pseudocode
```

```

### Breakdown
-
-
-

### Time Complexity
-
-
-