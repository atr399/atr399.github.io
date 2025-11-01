---
layout: post
title: "Python-Day1-Practice"
date: 2025-10-26
categories: python
---

# Remove Duplicates from Sorted Array (LeetCode 26) – Adapted to "Remove Duplicate IPs from Sorted ACL List"

## Problem Description (Theory & Real-World Usage):
Given a sorted list of IP addresses from an Access Control List (ACL) on a router (e.g., from show access-list output parsed into a list), remove duplicates in-place without using extra space. Keep the relative order, and return the new length. Duplicates happen in real-world due to config errors or merges; removing them prevents bloated ACLs that slow hardware lookups (e.g., in Cisco ASICs). In SRE roles at Tesla or xAI, this could be part of an AI pipeline to clean data before feeding to models for threat detection.
### 🧩 Example Input/Output

```python
Input: ips = ["10.0.0.1", "10.0.0.1", "10.0.0.2", "10.0.0.3", "10.0.0.3"]
Output: 3 (modified list: ["10.0.0.1", "10.0.0.2", "10.0.0.3", ...])
```

Solution Code:

```python
def remove_duplicate(ips):
    if not ips:
        return 0
    l = 1
    for r in range(1, len(ips)):
        if ips[r] != ips[r - 1]:
            ips[l] = ips[r]
            l += 1
    return l
```

### 🧩 Example usage

```python
acl_ips = ["10.0.0.1", "10.0.0.2", "10.0.0.3", "10.0.0.3"]
No_dupl_length = remove_duplicate(acl_ips):
print(acl_ips[:No_dupl_lenght])
```


## Remove Element (LeetCode 27) – Adapted to "Remove Specific VLAN from Interface List"

### Problem Description (Theory & Real-World Usage):
Given a list of VLAN IDs assigned to interfaces on a switch (e.g., from show interfaces switchport parsed output), remove all occurrences of a specific VLAN ID (e.g., a deprecated one) in-place and return the new length. In operations at Microsoft or Google, this cleans configs before deployment, preventing errors like VLAN mismatches in data centers. AI tie-in: Use in scripts that auto-remediate based on ML-detected anomalies.
### 🧩 Example Input/Output

```python
Input: vlans = [10, 20, 10, 30, 10, 40], target_vlan = 10
Output: 3 (modified list: [20, 30, 40, ...])
```

Solution Code:

```python
def remove_elements(vlans, target_vlan):
    k = 0
    for i in range(len(vlans)):
        if vlans[r] != target_vlan:
            vlans[k] = vlans[r]
            k += 1

    return k
```

### 🧩 Example usage:

```python
interface_vlans = [10, 20, 10, 30, 10, 40]
new_length = remove_elements(interface_vlans, target_vlan)
print(interface_vlans[:new_length])
```
