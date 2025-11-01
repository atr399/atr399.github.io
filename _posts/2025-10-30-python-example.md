---
layout: post
title: "Python-Day3-Practice"
date: 2025-10-30
---

# Valid Parentheses (LeetCode 20) – Adapted to "Validate Nested Configuration Syntax in Network Device Configs"

## Problem Description (Theory & Real-World Usage):
Given a string config from a network device configuration file (e.g., parsed from Cisco or Juniper CLI output), consisting of brackets '(', ')', '{', '}', '[', ']', check if it's valid: every opening bracket must be closed by the matching type, in correct order, with each closer having a corresponding opener. Return True if valid, False otherwise. In theory, this uses a stack (Python list with append/pop for LIFO) to track openings and a dictionary for matching pairs—efficient O(n) time/space for scanning. Real-world: Network configs often use nested brackets (e.g., {} for policy blocks in Juniper, [] for arrays in JSON API responses from NETCONF/RESTCONF). Invalid nesting causes parse errors or deployment failures in automation pipelines. In SRE roles at xAI or Meta, this validates configs before pushing to devices, preventing outages; tie to AI by cleaning data for ML models analyzing config drifts (e.g., feed valid nested structs to anomaly detection).
### 🧩 Example Input/Output

```python
Input: config = "{interface{speed 1000}}"
Output: True (valid nesting)
Input: config = "{[interface](GigabitEthernet1/0/1}{speed 1000]}"
Output: False (mismatched closers)
```

Solution Code:

```python
def is_valid_config(config):
    stack = []
    matching = {')': '(', ']': '[', '}': '{'}
    for char in config:
        if char in matching.values():  # Opening bracket
            stack.append(char)
        elif char in matching:  # Closing bracket
            if not stack or stack.pop() != matching[char]:
                return False
    return not stack  # Stack should be empty if all matched
```

### 🧩 Example usage

```python
config_str = "{[interface](GigabitEthernet1/0/1){speed 1000}}"
print(is_valid_config(config_str))  # Output: True
```


## Min Stack (LeetCode 155) – Adapted to "MinMetricStack for Tracking Network Latency Metrics"

### Problem Description (Theory & Real-World Usage):
Design a class MinMetricStack for tracking latency metrics (e.g., ping RTTs) from network devices, supporting O(1) push (add new latency), pop (remove latest), top (get latest), and getMin (retrieve current minimum). Initialize empty. Theory: Use two stacks (Python lists)—one for values, one for mins (push min of new val and current min). This ensures O(1) access/pop without scanning (O(n) naive min() bad for real-time). Real-world: In SRE at Google or Tesla, monitor latencies for AI workloads (e.g., self-driving data pipes); getMin flags anomalies quickly for auto-remediation. At xAI, stack tracks inference latencies, min for baseline in ML ops.
Input/Output Example:
### 🧩 Example Input/Output

```python
Operations: MinMetricStack(), push(50), push(30), getMin() → 30, top() → 30, pop(), getMin() → 50
```

Solution Code:

```python
class MinMetricStack:
    def __init__(self):
        self.stack = []
        self.min_stack = []

    def push(self, val):
        self.stack.append(val)
        val = min(val, self.min_stack[-1] if self.min_stack else val)
        self.min_stack.append(val)

    def pop(self):
        if self.stack:
            self.stack.pop()
            self.min_stack.pop()

    def top(self):
        if self.stack:
            return self.stack[-1]

    def getMin(self):
        if self.min_stack:
            return self.min_stack[-1]
```

### 🧩 Example usage:

```python
metrics = MinMetricStack()
metrics.push(50)  # Latency 50ms
metrics.push(30)  # Latency 30ms
print(metrics.getMin())  # 30
print(metrics.top())     # 30
metrics.pop()
print(metrics.getMin())  # 50
```
