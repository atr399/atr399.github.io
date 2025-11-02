---
title: "Python-Day5"
date: 2025-11-2 # e.g., 2025-11-01 10:00:00 +0800
categories: [python, programming]
tags: [data-structures, algorithms, guide]
---

# Merge Two Sorted Linked Lists (LeetCode 21) – Adapted to "Merge Two Sorted Linked Lists of Network Log Entries"

## Problem Description (Theory & Real-World Usage):
Given heads of two sorted linked lists representing timestamped log entries from two network devices (e.g., syslog from routers, sorted by time), merge into one sorted list using nodes from both. Return new head. Theory: Linked lists suit sequential logs (dynamic, no resizing); iterative merge uses dummy node/two pointers for O(m+n) time/O(1) space (efficient for large logs). Recursive: Elegant but O(m+n) space (call stack)—risky for deep merges. Constraints (n<=100) fit both; in prod, iterative preferred. Real-world: In SRE at Meta/Google, merge logs from redundant devices for unified analysis (e.g., correlate events); at xAI/Tesla, feed merged timelines to AI for anomaly detection (e.g., ML on sorted events for predictive maintenance).
### 🧩 Example Input/Output

```python
Input: list1 = [ "2025-11-02 10:00:01 Error" → "2025-11-02 10:00:03 Warning" ], list2 = [ "2025-11-02 10:00:02 Info" → "2025-11-02 10:00:04 Error" ]
Output: [ "2025-11-02 10:00:01 Error" → "2025-11-02 10:00:02 Info" → "2025-11-02 10:00:03 Warning" → "2025-11-02 10:00:04 Error" ]
```

Solution Code:

```python
class ListNode:
    def __init__(self, val=None, next=None):
        self.val = val
        self.next = next

def merge_logs_iterative(list1, list2):
    dummy = node = ListNode()

    while list1 and list2:
        if list1.val < list2.val:
            node.next = list1
            list1 = list1.next
        else:
            node.next = list2
            list2 = list2.next
        node = node.next

    if list1:
        node.next = list1
    
    if list2:
        node.next = list2

    return dummy.next
```

### 🧩 Example usage

```python
log1a = ListNode("2025-11-02 10:00:01 Error")
log1b = ListNode("2025-11-02 10:00:03 Warning")
log1a.next = log1b
log2a = ListNode("2025-11-02 10:00:02 Info")
log2b = ListNode("2025-11-02 10:00:04 Error")
log2a.next = log2b
merged_head = merge_logs_iterative(log1a, log2a)
# Print merged: 2025-11-02 10:00:01 Error -> ...
```
