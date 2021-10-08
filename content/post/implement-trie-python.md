+++ author = "Andrei Pöhlmann"
title = "Implement a Prefix Tree (aka. Trie)"
date = "2021-10-08"
description = "Python implementation of a Prefix Tree aka. Trie"
tags = [
"leetcode",
"Python",
]
categories = [
"algorithms",
"graphs",
]
+++

This post presents a Python implementation for today's daily LeetCode challenge: Implement a Prefix Tree.
(Link: [208. Implement Trie (Prefix Tree)](https://leetcode.com/problems/implement-trie-prefix-tree/)). 

I'm kinda surprised how simple and straight-forward it kinda is especially if you're already familiar
with graph problems (i.e. DFS, BFS, etc.). 

Because Prefix Trees [can be used for Autocomplete](https://en.wikipedia.org/wiki/Trie#Autocomplete),
I wanna post my solution here as a future reference: I plan on implementing an Autocomplete feature for my 
[Telegram Crypto Bot](https://andrei.poehlmann.dev/post/telegram-bot-1/) in the near
future. So why not use what we just learned? =) 

# How does it work

There are pretty nice explanations of what a Trie is and how it works on YouTube. 
I suggest watching these, because they have pretty nice visualizations which makes
graph problems/explanations a lot easier to understand than just reading abstract 
words on a blog post ;)


### Tech Dose
{{< youtube xqsaAhQC6c8 >}}

### Neetcode
{{< youtube oobqoCJlHA0 >}}

# Show me the code already

```Python
class Node:
    # Our Trie will make use of this kind of nodes
    def __init__(self):
        self.children = {}
        self.end = 0

class Trie:

    def __init__(self):
        self.root = Node()        

    def insert(self, word: str) -> None:
        # for each character in the given word
        # traverse the trie and add it to the
        # trie if it does not already exist
        # also, increment the counter at the end which
        # represents how many times we've added this word
        curr = self.root
        for char in word:
            if char not in curr.children:
                curr.children[char] = Node()
            curr = curr.children[char]
        # mark the end   
        curr.end += 1
    
    def __want(self, string: str) -> (bool, int):
        # helper function to avoid duplicate code:
        # traverse the trie and return True + the 
        # number of words we've seen so far matching
        # the string
        curr = self.root
        for char in string:
            if char not in curr.children:
                return False, 0
            curr = curr.children[char]

        return True, curr.end

    def search(self, word: str) -> bool:
        # use our helper function to determine if 
        # we've already seen this word
        _, have = self.__want(word)
        
        return have

    def startsWith(self, prefix: str) -> bool:
        # user our helper function to determin if
        # we've already seen a word with this prefix
        have, _ = self.__want(prefix)
        return have
```
### Result
![LeetCode performance of the above Trie implementation](/images/trie-solution.png)