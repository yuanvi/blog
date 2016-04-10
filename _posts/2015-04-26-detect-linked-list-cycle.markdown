---
published: true
title: 链表成环问题
layout: post
tags: [链表]
categories: [算法]
---
leetcode上见到的几道题，总结记录一下。

### Problem 1

问题：如何判断链表成环？

解法：一快一慢移动两个指针，如果链表成环，那么两个指针必然会相遇。

```c++
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    bool hasCycle(ListNode *head) {
        if (!head || !head->next) return false;

        ListNode* fast = head, *slow = head;
        while(fast && fast->next) {
            fast = fast->next->next;
            slow = slow->next;

            if (fast == slow) return true;
        }
        return false;
    }
};
```

### Problem 2

问题：如果链表成环，找出环开始的节点。

解法：
Definitions: Cycle = length of the cycle, if exists. C is the beginning of Cycle, S is the distance of slow pointer from C when slow pointer meets fast pointer.

Distance(slow) = C + S, Distance(fast) = 2 * Distance(slow) = 2 * (C + S). To let slow poiner meets fast pointer, only if fast pointer run 1 cycle more than slow pointer. Distance(fast) - Distance(slow) = Cycle => 2 * (C + S) - (C + S) = Cycle => C + S = Cycle
=> C = Cycle - S => This means if slow pointer runs (Cycle - S) more, it will reaches C. So at this time, if there's another point2 running from head => After C distance, point2 will meet slow pointer at C, where is the beginning of the cycle.

```c++
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
class Solution {
public:
    ListNode *detectCycle(ListNode *head) {
        ListNode *slow = head, *fast = head;
        while(fast && fast->next) {
            fast = fast->next->next;
            slow = slow->next;
            if (slow == fast) {
                while (head != slow) {
                    head = head->next;
                    slow = slow->next;
                }
            }
        }           
        return NULL;
    }
};
```

### Problem 3

问题：长度为N的数组，只存在1到N-1之间的数，其中有一个数出现了重复，找出这个重复的数。

解法：同Problem 2，若i为数组下标，可理解i为链表节点地址，arr[i]为next。

```python
class Solution(object):
    def findDuplicate(self, nums):
        """
        :type nums: List[int]
        :rtype: int
        """
        slow = fast = finder = 0
        while True:
            slow = nums[slow]
            fast = nums[nums[fast]]
            if slow == fast:
                while finder != slow:
                    finder = nums[finder]
                    slow = nums[slow]
                return finder
```
