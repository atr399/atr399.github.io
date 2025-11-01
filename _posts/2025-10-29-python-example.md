---
layout: post
title: "Python-Day2-Practice"
date: 2025-10-29
---

# Concatenation of Array (LeetCode 1929) – Adapted to "Concatenate BGP Peer Lists for Redundant Configurations"
## Problem Description (Theory & Real-World Usage):
Given a list of BGP peer IP addresses of length n (e.g., parsed from show bgp summary on a router), create a new list ans of length 2n by concatenating the original list with itself. This duplicates the peers for redundant setups, like active-standby routers in a data center. In theory, Python lists are dynamic arrays, and concatenation creates a new list by copying elements (O(n) time/space complexity—efficient for n <= 1000 per constraints). Real-world: At companies like Meta or Oracle, SREs automate BGP configs for high availability; duplicating peers ensures consistent templates across redundant devices, reducing failover errors. In AI ops at xAI or Tesla, this could prepare data for ML models analyzing peering patterns (e.g., concatenate logs for time-series input to detect anomalies).
### 🧩 Input/Output Example:
```python
Input: peers = ["192.168.1.1", "192.168.1.2", "10.0.0.1"]
Output: ["192.168.1.1", "192.168.1.2", "10.0.0.1", "192.168.1.1", "192.168.1.2", "10.0.0.1"]
```

Solution Code:

```python
ans = []

for i in range(2):
    for i in nums:
        ans.append(i)
return ans
```