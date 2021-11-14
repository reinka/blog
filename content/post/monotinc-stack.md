+++ author = "Andrei Pöhlmann"
title = "Algorithm patterns: monotonic stack"
date = "2021-11-14"
description = "Learn how to use a stack to find the next greater / smaller element in O(n) runtime"
tags = [
"leetcode",
"Python",
]
categories = [
"algorithms",
"stack",
]
+++

Stacks can be utilized to keep track of monotonously increasing (or decereasing) orders. This can be useful when you
are interested in finding the next greater / smaller element in `O(n)` runtime.

# Can you give a concrete example?

> The next **greater element** of some element `x` in an array is the **first greater** element that is **to the right**
> of x in the same array.
> 
> Given an array `nums` consisting of integers, return an array `result` of the same length as `nums` where each index `i` in 
> `result` stores the next greater element for `nums[i]`. If there's no greater element, store `-1`.
> 
> Example: 
> 
> `nums = [2,1,2,4,3]` 
> `result = [4,2,4,-1,-1]`

## Brute force

A straight-forward solution would be to use a double for-loop: iterate through the array and for each number, iterate 
through all of the remaining numbers until you find a greater element. This approach would have a time complexity of 
`O(n^2)`.

```Python
class Solution:
    def solve(self, nums: List[int]) -> List[int]:
        n = len(nums)
        result = [-1] * n
        for idx in range(n):
            for next_greater_idx in range(idx + 1, n):
                if nums[next_greater_idx] > nums[idx]:
                    result[idx] = nums[next_greater_idx]
                    break      

        return result
        
if __name__ == "__main__":
    s = Solution()

    print(s.solve([2,1,2,4,3]))
    print(s.solve([5,4,3,2,1,6]))
```
## Can we do better?

What makes the brute-force solution so inefficient is that it basically has no memory, i.e. it discards
previous information. Consider the extreme case where we have an array where the max element is the last 
element, and all other elements being sorted in descending order, e.g.
`[5,4,3,2,1,6]`. 

We can instantly see that for the first `n-1` numbers, where `n` denotes the length of the array, the very last
element is the next greater one. Because of the descending order we know that the next greater element for 
index `0` must also be the next greater element for index `1` to `n-2`.

How can we make use of this property to improve our runtime? 

## Monotonic stack

A monotonic stack is a stack where the elements are always in monotonously increasing / decreasing order. 
For our problem above, this property can help us to keep track of decreasing numbers.

The idea is the following: store elements in the stack until we encounter a new current element that is greater than the 
top of the stack. Once we arrived at such an element, keep popping from the stack while the top of the stack is smaller 
than the current element. Also, for each pop update the corresponding index in `result` (to achieve this we will push 
indices to our stack and work with those instead of directly pushing the numbers).

More specifically, for each number there are two cases we care about. 

1. The current number is **not** greater than the top of the stack. In this case, there's not much to do: just
push the index of the current number onto the stack.


2. The current number **is** greater than the top of the stack. Now this is where we have to update our `result`:
Due to the monotonicity of our stack, we know this is the **first** greater element that we have encountered so far.
   So we pop from the stack and update the corresponding index in `result`. However, we are not done here yet. 
   We need to keep popping because there could be more previous elements that are smaller. So we keep popping until
   either the stack
   is empty or the top of the stack is greater than the current element.
   
And that's basically it. Here's how an implementation could look like in Python 3:

```Python3 
class SolutionStack:
    def solve(self, nums: List[int]) -> List[int]:
        n = len(nums)
        result = [-1] * n
        stack = []

        for curr_idx, curr_num in enumerate(nums):
            # Pop until the current element is not
            # greater than the top of the stack
            while stack and nums[stack[-1]] < curr_num:
                prev_idx = stack.pop()
                # update result with curr_num
                result[prev_idx] = curr_num
                
            # push the current element onto the stack
            stack.append(curr_idx)

        return result


if __name__ == "__main__":
    s = SolutionStack()

    print(s.solve([2,1,2,4,3]))
    print(s.solve([5,4,3,2,1,6]))
```
   

# A few LeetCode problems

* [Monotonic stack tag](https://leetcode.com/tag/monotonic-stack/)

## Easy
* [496. Next Greater Element I](https://leetcode.com/problems/next-greater-element-i/)
## Medium
* [739. Daily Temperatures](https://leetcode.com/problems/daily-temperatures/)
* [1762. Buildings With an Ocean View](https://leetcode.com/problems/buildings-with-an-ocean-view/)
## Hard
* [84. Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/)



# References 

* [LeetCode Discuss: stack solution with very detailed explanation step by step](https://leetcode.com/problems/sum-of-subarray-minimums/discuss/178876/stack-solution-with-very-detailed-explanation-step-by-step)
* [Special Data Structure: Monotonic Stack](https://labuladong.gitbook.io/algo-en/ii.-data-structure/monotonicstack)