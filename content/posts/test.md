---
title: "Test"
date: 2020-08-04T17:31:20+08:00
draft: false
---

# LeetCode(Test)

最长回文子串 O(n)

```python
class Solution:
    def longestPalindrome(self, s: str) -> str:
        ss = '#' + '#'.join(s) + '#'
        
        LR = [0 for i in range(len(ss))]
        maxR = 0;   
        center = 0;
        maxlen = 0;
        maxi = 0;
        for i in range(len(ss)):
            if i < maxR:
                LR[i] = min(maxR - i + 1, LR[2 * center - i])
            else:
                LR[i] = 1;
                
            while i - LR[i] >= 0 and i + LR[i] < len(ss) and ss[i-LR[i]] == ss[i+LR[i]]:
                LR[i] += 1
            
            if i + LR[i] - 1 > maxR:
                maxR = i + LR[i] - 1
                center = i
        
            if maxlen < LR[i] - 1:
                maxlen = LR[i] - 1
                maxi = i
        
        return ss[maxi - LR[maxi] + 1 : maxi + LR[maxi]].replace('#', '')
```

