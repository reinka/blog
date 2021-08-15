+++ author = "Andrei Pöhlmann"
title = "Algorithm Patterns: Linked Lists"
date = "2021-08-15"
description = "title here"
tags = [
"leetcode",
]
categories = [
"algorithms"
]
series = ["Algorithm Patterns"]
+++

This post lists a bunch of useful patterns when working with linked lists (both singly and doubly linked lists).


# Dummy head and tail pointers

Linked lists problems often ask you to perform some operations on the original linked list and to return
a modified version of it, e.g. Leetcode's [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/).
To always keep track of the head (i.e. the first node) of the linked list, it is therefore useful to create a dummy 
node that always points to the first node of the list. 

In the end, when returning your result, you can
make use of this dummy node and just `return dummy.next`, e.g.

```python
class Solution:
    def reverseList(self, head: Optional[ListNode]) -> Optional[ListNode]:
      ...
      dummy = ListNode(next=head)
      
      # modify your linked list
      ...
      
      return dummy.next
```

A dummy tail can be useful when working with doubly linked lists: both the dummy head and tail mark the
boundaries of the linked list. This can allow you to skip the usual `if node.next is not None` checks during updates. 
Leetcode's [146. LRU Cache](https://leetcode.com/problems/lru-cache/) is an example where this pattern is useful. 

# Slow and fast pointer

Linked lists problems often boil down to manipulating pointers and to keeping track of them while updating them. One 
useful pattern is the slow-and-fast-pointer approach: the trick here is to define two distinct pointers that
traverse the list with two different speeds, e.g. the slow one iterates one node at the time while the fast
one iterates two nodes at a time. This pattern can be used to solve 
[141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/). 

```Python
slow = head
fast = head.next
while slow != fast:
    # do some None checks and other necessary operations here
    ...
    # move on with pointers
    slow = slow.next
    fast = fast.next.next

```

Sometimes there are variations of this pattern, e.g. where the traversal speed of the fast and slow pointer 
is identical, however their starting point is not: they start N-steps apart form each other and move with the 
same speed, e.g. in [19. Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/). 

A combination of both patterns is used in [142. Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii/), 
where you basically need to run two while loops, sequentially: 

* the first while loop detects the cycle: using the slow-and-fast pointer approach with different speeds will make the 
fast pointer 
  * either hit the end of the list (`None`) or
  * reach the first pointer at some intersection node
    
* the second while loop will find the actual entrance node to the cycle

This is also known as "Floyd's cycle detection algorithm" or "Floyd's tortoise and hare".

# Know thy pointers 

As mentioned above, linked list operations are basically just pointer manipulations. I personally find linked list 
problems conceptually easier than most medium array problems, but I can still confuse myself when I try to solve a pointer
manipulation problem that involves multiple pointers merely in my head. 

## Example: swap nodes
### `first.next = second.next; second.next = first; prev.next = second; prev = first;` what the actual f ?

It's often useful to draw an example linked list
to visualize your problem and the solution and then to translate that into code. A nice example would be 
[24. Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs/), where you have to swap each pair of nodes, 
e.g. 

```Python
1->2->3->4
``` 
should become

```Python
2->1->4->3
```

Here, again, we need two pointers that point to the two nodes that need to be swapped. However, this time two pointers 
is not enough: we also need a 3rd pointer, `previous` or short `prev`, because `prev.next` must also be updated. 

Let's take our `1->2->3->4` example for instance. As discussed above we would make use of
a dummy node that points to the head/first node of the linked list, i.e. 

```Python
dummy->1->2->3->4
``` 

Let's say we now swap `1` and `2` only, this would result in something like this:

```Python
dummy
     \
      1 -> 3 -> 4
   2 /
```
So `2->1` seems correct. However, notice that we also need to update the `dummy` node, or in other words the `prev` node,
because its next pointer still points to the `1`. 

Let's call the `slow` and `fast` pointer `first` and `second` here (makes it conceptually simpler to think of it IMO). 
So our code could look something along the lines of:

```Python
class Solution:
    def swapPairs(self, head: Optional[ListNode]) -> Optional[ListNode]:
        # define a dummy which we will use to return the solution
        dummy = ListNode()
        dummy.next = head
        
        # need previous to update prev.next
        # that changes because of the swap
        # of the next 2 nodes
        prev = dummy
        
        while prev.next is not None and prev.next.next is not None:
            # define pointers for the two nodes to swap
            first = prev.next
            second = prev.next.next
            
            # do the swap
            first.next = second.next
            second.next = first
            
            # now update the previous node
            prev.next  = second
            
            # move on in the linked list
            # set prev to first, which after the swap corresponds
            # to the 2nd next node after prev in the linked list 
            prev = first
            
        return dummy.next
```
### Don't be lazy: do the drawing

Even after coming up with the above solution myself, the code still looks slightly confusing to me because of all 
the pointer assignments. The first time, it took me ~45 minutes to write it down even though I knew I had to
use some kind of slow-and-fast-pointer approach. Mostly because at first I was too lazy to do the drawing, 
and I thought maybe I could solve it only with 2 pointers instead of 3 -- making it a "more elegant" solution
because it needs fewer variables, right?

#  Floyd's cycle detection algorithm

We already talked about this one above under "Slow and fast pointer", and the classical problem would be 
[142. Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii/).

Now I don't know how likely it is to face it in a coding interview, however I would still suggest to learn about it. What 
struck me most was another Leetcode problem, [287. Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number/):


> Given an array of integers nums containing n + 1 integers where each integer is in the range [1, n] inclusive. 
> 
> There is only one repeated number in nums, return this repeated number.
> 
> You must solve the problem without modifying the array nums and uses only constant extra space.

It turns out this problem can be actually reduced to a cycle detection problem, allowing you to solve it optimally in 
O(N) time and O(1) space complexity, by using Floyd's cycle detection algorithm.


# Some Leetcode problems

## Easy
* [237. Delete Node in a Linked List](https://leetcode.com/problems/delete-node-in-a-linked-list/)
* [21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/)
* [141. Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/)
* [206. Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/)
## Medium
* [707. Design Linked List](https://leetcode.com/problems/design-linked-list/)
* [19. Remove Nth Node From End of List](https://leetcode.com/problems/remove-nth-node-from-end-of-list/)
* [24. Swap Nodes in Pairs](https://leetcode.com/problems/swap-nodes-in-pairs/)
* [143. Reorder List](https://leetcode.com/problems/reorder-list/)  
* [142. Linked List Cycle II](https://leetcode.com/problems/linked-list-cycle-ii/)
## Hard
* [23. Merge k Sorted Lists](https://leetcode.com/problemset/all/?page=1&search=linked+lists)
* [287. Find the Duplicate Number](https://leetcode.com/problems/find-the-duplicate-number/)