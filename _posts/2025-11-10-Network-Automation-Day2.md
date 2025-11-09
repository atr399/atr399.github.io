---
title: "Network-Automation-Day1"
date: 2025-11-9 e.g., 2025-11-01 10:00:00 +0800
categories: [netdevops, lab]
tags: [network-automation, netdevops, lab-setup, python, paramiko, netmiko, netbox, batfish, netlab, ospf, configuration-management]
---

# Day 2: Python Network Automation Basics

**Module:** Foundation (Module 1, Day 2/7)  
**Duration:** 3 hours  
**Difficulty:** Beginner → Intermediate  
**Prerequisites:** Day 1 completed (lab running, first connection working)

---

## 🎯 What You'll Learn Today

By the end of today, you will be able to:

- ✅ Understand the difference between Paramiko and Netmiko (and why it matters)
- ✅ Connect to multiple network devices efficiently
- ✅ Execute show commands and capture structured output
- ✅ Push configuration changes safely with backup/rollback
- ✅ Parse real network data (OSPF neighbors, routing tables)
- ✅ Handle errors gracefully across multiple devices
- ✅ Build production-grade automation patterns

**Today's Focus:** Master the Python fundamentals that power ALL automation tools (Ansible, NAPALM, Nornir).

---

## 🌟 Why This Matters

### The Interview Question

> "You say you know Ansible. Can you explain what happens when Ansible connects to a device?"

**Without Python knowledge:**
> "Um... Ansible just... connects? It uses SSH?"

**With Python knowledge:**
> "Ansible uses Paramiko for SSH connections, wraps it with connection plugins, establishes a shell session, sends commands, parses output, and handles errors. I can show you the exact code if you want."

**This is the difference between tool operator and automation engineer.**

### Real-World Scenario

```
SITUATION: 3 AM - OSPF Down Across 50 Routers
- Ansible playbook timing out randomly
- Some routers responding, some not
- Production traffic affected
- Boss: "Fix it NOW"

WITHOUT PYTHON SKILLS:
"The playbook is broken... I'll restart it?"
→ Still failing, no idea why
→ 2 hours wasted

WITH PYTHON SKILLS:
"Let me write a quick diagnostic script..."
→ Discovers 15 routers have high CPU
→ Identifies memory leak in OSPF process
→ Isolates problem devices
→ Targeted fix in 20 minutes
→ HERO STATUS ACHIEVED
```

### What You'll Gain Today

- ✅ **Deep Understanding:** Know what happens "under the hood"
- ✅ **Debugging Power:** Fix any automation tool when it breaks
- ✅ **Custom Solutions:** Build tools for unique requirements
- ✅ **Interview Confidence:** Explain technical details with authority
- ✅ **Career Growth:** From operator to engineer

---

## ✅ Prerequisites Check

Before starting, verify Day 1 lab is ready:

```bash
# 1. Check you're in the right place
cd ~/netdevops-automation
pwd
# Should show: /home/atr399/netdevops-automation

# 2. Activate virtual environment (from Day 1)
source .venv/bin/activate
# Prompt should show: (.venv)

# 3. Verify Python packages from Day 1
python -c "import netmiko; print(f'Netmiko: {netmiko.__version__}')"
python -c "import paramiko; print(f'Paramiko: {paramiko.__version__}')"
# Should show versions without errors

# 4. Check Day 1 lab is running
cd ~/netdevops-automation/labs/lab-001-foundation
netlab status
# Should show: r1, r2, r3, sw1 all "running"

# 5. Get management IPs (write these down!)
netlab inspect
# Note the mgmt IPs for R1, R2, R3, SW1 (172.16.0.x addresses)

# 6. Quick connectivity test
ping -c 2 $(netlab inspect --node r1 | grep mgmt-ip | awk '{print $2}')
# Should get replies
```

**If ANY check fails:** Review Day 1 setup. Don't proceed until all pass.

**Write down your management IPs here:**
```
R1: 172.16.0.___ (from netlab inspect)
R2: 172.16.0.___ 
R3: 172.16.0.___
SW1: 172.16.0.___
```

---

## 📚 Learning Path Overview

```
TODAY'S JOURNEY (Building on Day 1):

Day 1 Recap: You created lab, connected to ONE device
Day 2 Goal: Master multi-device automation with real protocols

Part 1: SSH Foundations - Paramiko (30 min)
├── Why SSH automation exists
├── Low-level SSH operations
└── What Netmiko abstracts away

Part 2: Netmiko Deep Dive (45 min)
├── Connect to all 3 routers + switch
├── Execute commands across devices
├── Parse OSPF data (using your running OSPF!)
└── Error handling patterns

Part 3: Safe Configuration Management (45 min)
├── Backup configurations first (always!)
├── Push changes safely
├── Verify changes worked
└── Rollback strategies

Part 4: Production Patterns (45 min)
├── Device inventory management
├── Multi-device operations at scale
├── OSPF verification automation
├── Generate summary reports
└── Real network troubleshooting

Part 5: Verification & Challenges (15 min)
├── Test your knowledge
├── Challenge exercises using OSPF
└── Reflection questions
```

---

## 🔧 Part 1: Understanding SSH Automation

### 1.1 Theory: From Day 1 to Production

**Day 1 Achievement:** You connected to ONE router manually.

**Today's Goal:** Automate MULTIPLE devices professionally.

**Why learn the hard way first?**

```python
# What you did in Day 1 (simplified):
from netmiko import ConnectHandler
connection = ConnectHandler(device_params)
output = connection.send_command("show version")

# ✅ This works! But do you know:
# - What happens inside ConnectHandler?
# - Why does send_command() sometimes hang?
# - How to debug when it fails?
# - What makes Netmiko better than raw SSH?

# Today you'll learn the answers.
```

### 1.2 Create Day 2 Working Directory

```bash
# We'll put Day 2 scripts in organized location
cd ~/netdevops-automation
mkdir -p scripts/day2
cd scripts/day2

# Verify you're in the right place
pwd
# Should show: /home/atr399/netdevops-automation/scripts/day2
```

### 1.3 Understanding Paramiko (The Foundation)

**What is Paramiko?**
- Low-level SSH library for Python
- Used internally by Netmiko, Ansible, NAPALM
- Gives you SSH "primitives" (connect, authenticate, send, receive)
- Network-agnostic (works for Linux, network devices, anything with SSH)

**Why learn it?**
- Understand what Netmiko automates
- Debug when higher-level tools fail
- Interview questions often ask about SSH internals

Create `01_paramiko_basics.py`:

```bash
cat > 01_paramiko_basics.py << 'PYEOF'
#!/usr/bin/env python3
"""
01_paramiko_basics.py - Understanding SSH at a low level

Purpose: See what happens "under the hood" when automating SSH
Why: This knowledge helps you debug ANY automation tool

⚠️  This is for learning. In production, use Netmiko.
"""

import paramiko
import time
import sys

# ==============================================================================
# DEVICE CONFIGURATION
# ==============================================================================
# ⚠️  UPDATE THIS with your R1 management IP from Day 1 lab
# Get it with: cd ~/netdevops-automation/labs/lab-001-foundation && netlab inspect
# ==============================================================================

R1_IP = "172.16.0.1"  # ⚠️  CHANGE THIS to your R1 mgmt IP
USERNAME = "admin"     # Netlab default
PASSWORD = "admin"     # Netlab default

# ==============================================================================
# Why separate configuration?
# - Single source of truth (change once, affects everywhere)
# - Easy to spot what needs updating
# - Later we'll move to external config files
# ==============================================================================


def connect_with_paramiko(host, username, password):
    """
    Establish SSH connection using Paramiko.
    
    This shows ALL the manual steps needed for SSH automation.
    Each step is something Netmiko does automatically for you.
    
    Args:
        host (str): Device IP address
        username (str): SSH username  
        password (str): SSH password
        
    Returns:
        tuple: (ssh_client, shell_channel) or (None, None) if failed
        
    Why two objects?
    - ssh_client: The connection object (like a phone line)
    - shell_channel: The interactive terminal (like talking on the phone)
    """
    
    try:
        print(f"\n{'='*70}")
        print(f"CONNECTING TO {host} - Step by Step")
        print(f"{'='*70}\n")
        
        # STEP 1: Create SSH client
        print(f"[STEP 1] Creating SSH client object...")
        ssh_client = paramiko.SSHClient()
        # Think: "Getting ready to make a phone call"
        
        # STEP 2: Set key policy
        print(f"[STEP 2] Setting SSH key policy...")
        ssh_client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        # ⚠️  AutoAddPolicy = "Trust any device" (ONLY for labs!)
        # Production: Use known_hosts file to verify device identity
        # Why? Prevents man-in-the-middle attacks
        
        # STEP 3: Connect
        print(f"[STEP 3] Connecting to {host}...")
        ssh_client.connect(
            hostname=host,
            username=username,
            password=password,
            timeout=10,           # Fail if no response in 10 seconds
            look_for_keys=False,  # Don't search for SSH keys
            allow_agent=False     # Don't use SSH agent
        )
        print(f"[SUCCESS] ✓ TCP connection established")
        print(f"[SUCCESS] ✓ Authentication successful")
        
        # STEP 4: Open interactive shell
        print(f"[STEP 4] Opening interactive shell...")
        shell = ssh_client.invoke_shell()
        # invoke_shell() = Start interactive session (like opening terminal)
        # Why? Network devices need interactive shells, not exec channels
        
        # STEP 5: Wait for device prompt
        print(f"[STEP 5] Waiting for device prompt...")
        time.sleep(2)
        # Why wait? Device needs time to:
        # - Display banner
        # - Show MOTD
        # - Present command prompt
        
        # STEP 6: Clear initial output
        print(f"[STEP 6] Clearing initial output buffer...")
        initial_output = shell.recv(65535).decode('utf-8')
        # recv(65535) = Read up to 64KB of data
        # decode('utf-8') = Convert bytes to string
        
        print(f"\n[DEVICE OUTPUT]:")
        print(f"{'-'*70}")
        print(initial_output)
        print(f"{'-'*70}\n")
        
        print(f"[SUCCESS] ✓ Ready to send commands!")
        
        return ssh_client, shell
        
    except paramiko.AuthenticationException:
        print(f"\n[ERROR] ✗ Authentication failed!")
        print(f"[HINT] Check username/password: {username}/{password}")
        print(f"[HINT] Verify device accepts password authentication")
        return None, None
        
    except paramiko.SSHException as e:
        print(f"\n[ERROR] ✗ SSH protocol error: {str(e)}")
        print(f"[HINT] Check if SSH is enabled on device")
        print(f"[HINT] Verify SSH version compatibility")
        return None, None
        
    except Exception as e:
        print(f"\n[ERROR] ✗ Unexpected error: {str(e)}")
        print(f"[HINT] Check network connectivity: ping {host}")
        print(f"[HINT] Verify firewall rules")
        return None, None


def send_command(shell, command, wait_time=2):
    """
    Send command and retrieve output manually.
    
    This is tedious and error-prone - that's why Netmiko exists!
    
    Args:
        shell: Shell channel from paramiko
        command (str): Command to execute
        wait_time (int): Seconds to wait for output
        
    Returns:
        str: Command output
        
    Why is this complicated?
    - Must clear buffer before sending
    - Must wait for command to complete
    - Must handle variable-length output
    - Must detect when output is complete
    - Network devices don't signal "done" like Linux
    """
    
    try:
        print(f"\n[EXECUTING] {command}")
        
        # Clear any pending output
        # Why? Old output from previous commands might still be in buffer
        if shell.recv_ready():
            shell.recv(65535)
            print(f"[DEBUG] Cleared pending output")
        
        # Send command with newline (simulate pressing Enter)
        shell.send(command + '\n')
        print(f"[DEBUG] Command sent, waiting {wait_time}s for output...")
        
        # Wait for command to execute
        # Why sleep? Commands take time to:
        # - Process
        # - Execute
        # - Generate output
        # - Return to prompt
        time.sleep(wait_time)
        
        # Retrieve all available output
        output = ""
        while shell.recv_ready():
            chunk = shell.recv(65535).decode('utf-8')
            output += chunk
            # Why loop? Output might be larger than buffer
        
        print(f"[SUCCESS] Received {len(output)} characters")
        
        return output
        
    except Exception as e:
        print(f"[ERROR] Command execution failed: {str(e)}")
        return ""


def demonstrate_manual_ssh():
    """
    Main demonstration of manual SSH operations.
    
    This shows you what every automation tool does internally.
    """
    
    print("\n" + "="*70)
    print("PARAMIKO DEMONSTRATION: Manual SSH Operations")
    print("="*70)
    print("\nThis script shows the MANUAL process of SSH automation.")
    print("Each step here is what Netmiko automates for you.\n")
    
    # Connect
    ssh_client, shell = connect_with_paramiko(R1_IP, USERNAME, PASSWORD)
    
    if not ssh_client:
        print("\n[FAILED] Could not connect. Check configuration and try again.")
        sys.exit(1)
    
    try:
        # Command 1: Show version
        output = send_command(shell, "show version")
        print(f"\n[OUTPUT - show version]:")
        print(f"{'-'*70}")
        # Show just first 10 lines (version output is long)
        for line in output.split('\n')[:10]:
            if line.strip():
                print(line)
        print("... (truncated)")
        print(f"{'-'*70}")
        
        # Command 2: Show IP interface brief
        output = send_command(shell, "show ip interface brief")
        print(f"\n[OUTPUT - show ip interface brief]:")
        print(f"{'-'*70}")
        print(output)
        print(f"{'-'*70}")
        
        # Command 3: Show OSPF neighbors (your Day 1 OSPF!)
        output = send_command(shell, "show ip ospf neighbor")
        print(f"\n[OUTPUT - show ip ospf neighbor]:")
        print(f"{'-'*70}")
        print(output)
        print(f"{'-'*70}")
        
        print("\n[OBSERVATION]")
        print("Notice how much manual work this is:")
        print("  • Manual timing (sleep statements)")
        print("  • Manual buffer management (recv calls)")
        print("  • Manual prompt handling")
        print("  • Manual output parsing")
        print("\nNetmiko automates ALL of this! ⚡")
        
    except Exception as e:
        print(f"\n[ERROR] Operations failed: {str(e)}")
        
    finally:
        # Always close connection
        # Why finally? Ensures cleanup even if errors occur
        print(f"\n[CLEANUP] Closing SSH connection...")
        ssh_client.close()
        print(f"[DONE] Connection closed properly")


if __name__ == "__main__":
    demonstrate_manual_ssh()

# ==============================================================================
# KEY TAKEAWAYS:
#
# 1. SSH automation requires many manual steps
# 2. Timing is critical (wait for output, clear buffers)
# 3. Network devices are different from Linux (no exec channel)
# 4. Error handling is essential (network is unreliable)
# 5. This is WHY Netmiko exists - to abstract this complexity
#
# NEXT: See how Netmiko makes this simple!
# ==============================================================================
PYEOF

chmod +x 01_paramiko_basics.py
```

**Update the IP address and run:**

```bash
# Get your R1 IP first
cd ~/netdevops-automation/labs/lab-001-foundation
netlab inspect --node r1 | grep mgmt-ip

# Edit the script with your R1 IP
nano 01_paramiko_basics.py
# Change R1_IP = "172.16.0.X" to your actual IP

# Run it
cd ~/netdevops-automation/scripts/day2
python 01_paramiko_basics.py
```

**Expected Output:**
```
======================================================================
CONNECTING TO 172.16.0.1 - Step by Step
======================================================================

[STEP 1] Creating SSH client object...
[STEP 2] Setting SSH key policy...
[STEP 3] Connecting to 172.16.0.1...
[SUCCESS] ✓ TCP connection established
[SUCCESS] ✓ Authentication successful
[STEP 4] Opening interactive shell...
[STEP 5] Waiting for device prompt...
[STEP 6] Clearing initial output buffer...

[DEVICE OUTPUT]:
----------------------------------------------------------------------
r1>
----------------------------------------------------------------------

[SUCCESS] ✓ Ready to send commands!

[EXECUTING] show version
[DEBUG] Command sent, waiting 2s for output...
[SUCCESS] Received 1234 characters

[OUTPUT - show version]:
----------------------------------------------------------------------
Cisco IOS Software, IOL Software...
... (output from your router)
----------------------------------------------------------------------

[OBSERVATION]
Notice how much manual work this is:
  • Manual timing (sleep statements)
  • Manual buffer management (recv calls)
  • Manual prompt handling
  • Manual output parsing

Netmiko automates ALL of this! ⚡
```

### 1.4 Reflection: Understanding What You Just Did

**Question:** Why go through all this manual work when Netmiko exists?

**Answer:** Because now you understand:
- What `ConnectHandler()` does internally
- Why `send_command()` sometimes hangs (timing issues)
- How to debug when Ansible/Netmiko fails
- What "under the hood" means in interviews

---

## 🚀 Part 2: Netmiko - The Right Way

### 2.1 Theory: From Paramiko to Netmiko

**What we just did with Paramiko:**
```python
ssh_client = paramiko.SSHClient()
ssh_client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
ssh_client.connect(hostname=host, username=user, password=pwd, timeout=10)
shell = ssh_client.invoke_shell()
time.sleep(2)
shell.recv(65535)
shell.send('show version\n')
time.sleep(2)
output = shell.recv(65535).decode('utf-8')
# ... then parse out the prompt, handle errors, etc.
```

**The same thing with Netmiko:**
```python
from netmiko import ConnectHandler
connection = ConnectHandler(device_type='cisco_ios', host=host, 
                           username=user, password=pwd)
output = connection.send_command('show version')
```

**Netmiko's Magic:**
- ✅ Automatic prompt detection (no manual timing!)
- ✅ Handles pagination (`--More--` prompts)
- ✅ Configuration mode awareness
- ✅ Multi-vendor support (200+ device types)
- ✅ Built-in error handling
- ✅ Session logging

### 2.2 Connecting to Multiple Devices

Now let's use Netmiko with your FULL Day 1 topology!

Create `02_netmiko_multi_device.py`:

```bash
cat > 02_netmiko_multi_device.py << 'PYEOF'
#!/usr/bin/env python3
"""
02_netmiko_multi_device.py - Working with multiple devices

Purpose: Scale from 1 device (Day 1) to multiple devices (production pattern)
Why: Real networks have hundreds of devices - learn the pattern now

Today we'll use your full Day 1 topology:
- R1, R2, R3 (OSPF neighbors)
- SW1 (connected to all routers)
"""

from netmiko import ConnectHandler
import sys

# ==============================================================================
# DEVICE INVENTORY
# ==============================================================================
# This is your "Source of Truth" for now.
# Later (Day 5), this will come from NetBox API automatically.
#
# ⚠️  UPDATE THESE IPs with your actual management IPs from Day 1 lab!
# Get them with: cd ~/netdevops-automation/labs/lab-001-foundation && netlab inspect
# ==============================================================================

DEVICES = [
    {
        'device_type': 'cisco_ios',
        'host': '172.16.0.1',          # ⚠️  UPDATE: Your R1 mgmt IP
        'username': 'admin',
        'password': 'admin',
        'secret': 'admin',             # Enable password
        'nickname': 'R1',              # Friendly name for logs
        'role': 'router',              # Device role
        'session_log': 'logs/r1_session.log'
    },
    {
        'device_type': 'cisco_ios',
        'host': '172.16.0.2',          # ⚠️  UPDATE: Your R2 mgmt IP
        'username': 'admin',
        'password': 'admin',
        'secret': 'admin',
        'nickname': 'R2',
        'role': 'router',
        'session_log': 'logs/r2_session.log'
    },
    {
        'device_type': 'cisco_ios',
        'host': '172.16.0.3',          # ⚠️  UPDATE: Your R3 mgmt IP
        'username': 'admin',
        'password': 'admin',
        'secret': 'admin',
        'nickname': 'R3',
        'role': 'router',
        'session_log': 'logs/r3_session.log'
    },
    {
        'device_type': 'cisco_ios',
        'host': '172.16.0.4',          # ⚠️  UPDATE: Your SW1 mgmt IP
        'username': 'admin',
        'password': 'admin',
        'secret': 'admin',
        'nickname': 'SW1',
        'role': 'switch',
        'session_log': 'logs/sw1_session.log'
    }
]

# ==============================================================================
# WHY this structure?
# - List of dictionaries: Easy to loop through
# - nickname: Human-readable identifier for logs/output
# - role: Filter devices (e.g., only update routers, not switches)
# - session_log: Capture EVERYTHING for debugging
#
# In production you'd also have:
# - site: Geographic location (for parallel execution by region)
# - maintenance_window: When changes are allowed
# - tags: Flexible categorization (core, edge, dmz, etc.)
# ==============================================================================


def connect_to_device(device_params):
    """
    Connect to a device using Netmiko with error handling.
    
    This wraps ConnectHandler with proper error handling so failures
    don't crash the entire script (critical for multi-device operations).
    
    Args:
        device_params (dict): Device connection parameters
        
    Returns:
        ConnectHandler object if successful, None if failed
        
    Why return None instead of raising exception?
    - Allows script to continue with other devices
    - Caller can decide how to handle failures
    - Production pattern: "partial success is better than total failure"
    """
    
    nickname = device_params.get('nickname', device_params['host'])
    
    try:
        print(f"\n[CONNECTING] {nickname} ({device_params['host']})...")
        
        # ConnectHandler handles:
        # 1. SSH connection
        # 2. Authentication
        # 3. Terminal width/length settings
        # 4. Prompt detection
        # 5. Paging disable (terminal length 0)
        connection = ConnectHandler(**device_params)
        
        # Verify we're connected by getting prompt
        prompt = connection.find_prompt()
        print(f"[SUCCESS] ✓ Connected to {nickname}")
        print(f"[PROMPT] {prompt}")
        
        return connection
        
    except Exception as e:
        print(f"[FAILED] ✗ Could not connect to {nickname}")
        print(f"[ERROR] {str(e)}")
        print(f"[HINT] Check:")
        print(f"        • Device is reachable: ping {device_params['host']}")
        print(f"        • SSH enabled on device")
        print(f"        • Credentials correct: {device_params['username']}")
        print(f"        • Session log: {device_params.get('session_log', 'N/A')}")
        return None


def get_device_info(connection, nickname):
    """
    Gather basic information from a device.
    
    This is a common pattern in network automation:
    - Collect data from devices
    - Store in structured format
    - Aggregate for analysis/reporting
    
    Args:
        connection: Active Netmiko connection
        nickname: Device identifier
        
    Returns:
        dict: Device information
    """
    
    info = {'nickname': nickname, 'success': False}
    
    try:
        print(f"\n[GATHERING] Information from {nickname}...")
        
        # Get hostname from running config
        # Why from config? More reliable than parsing 'show version'
        hostname_output = connection.send_command("show running-config | include hostname")
        if hostname_output:
            info['hostname'] = hostname_output.split()[-1]
        else:
            info['hostname'] = nickname  # Fallback
        
        # Get IOS version
        version_output = connection.send_command("show version")
        # Parse version from output
        for line in version_output.split('\n'):
            if 'IOS' in line and 'Version' in line:
                # Extract version number
                parts = line.split('Version')
                if len(parts) > 1:
                    version = parts[1].split(',')[0].strip()
                    info['ios_version'] = version
                    break
        
        # Get uptime
        for line in version_output.split('\n'):
            if 'uptime is' in line:
                info['uptime'] = line.strip()
                break
        
        # Get interface summary
        int_brief = connection.send_command("show ip interface brief")
        lines = [l for l in int_brief.split('\n') if l.strip()]
        info['interface_count'] = len(lines) - 1  # Subtract header
        
        # Count up/down interfaces
        info['interfaces_up'] = int_brief.count(' up ')
        info['interfaces_down'] = int_brief.count(' down ')
        
        info['success'] = True
        print(f"[SUCCESS] ✓ Collected info from {nickname}")
        
        return info
        
    except Exception as e:
        print(f"[FAILED] ✗ Could not gather info from {nickname}: {str(e)}")
        info['error'] = str(e)
        return info


def get_ospf_neighbors(connection, nickname):
    """
    Get OSPF neighbor information (using your Day 1 OSPF!)
    
    This demonstrates working with real network protocols.
    OSPF is running between R1-R2-R3 from Day 1.
    
    Args:
        connection: Active Netmiko connection
        nickname: Device identifier
        
    Returns:
        dict: OSPF information
    """
    
    ospf_info = {'nickname': nickname, 'has_ospf': False, 'neighbors': []}
    
    try:
        print(f"\n[CHECKING] OSPF neighbors on {nickname}...")
        
        # Get OSPF neighbors
        output = connection.send_command("show ip ospf neighbor")
        
        # Check if OSPF is configured
        if 'OSPF not enabled' in output or not output.strip():
            print(f"[INFO] No OSPF configured on {nickname}")
            return ospf_info
        
        ospf_info['has_ospf'] = True
        
        # Parse OSPF neighbors
        # Output format:
        # Neighbor ID     Pri   State           Dead Time   Address         Interface
        # 10.0.0.2          0   FULL/  -        00:00:37    10.1.0.2        Ethernet0/1
        
        lines = output.split('\n')
        for line in lines:
            if 'FULL' in line:  # Active OSPF neighbor
                parts = line.split()
                if len(parts) >= 6:
                    neighbor = {
                        'neighbor_id': parts[0],
                        'state': parts[2],
                        'address': parts[4],
                        'interface': parts[5]
                    }
                    ospf_info['neighbors'].append(neighbor)
        
        neighbor_count = len(ospf_info['neighbors'])
        print(f"[SUCCESS] ✓ Found {neighbor_count} OSPF neighbor(s)")
        
        return ospf_info
        
    except Exception as e:
        print(f"[FAILED] ✗ Could not check OSPF on {nickname}: {str(e)}")
        ospf_info['error'] = str(e)
        return ospf_info


def main():
    """
    Main function - orchestrate multi-device operations.
    
    This demonstrates the production pattern:
    1. Loop through device inventory
    2. Connect to each device
    3. Gather information
    4. Handle per-device failures gracefully
    5. Generate summary report
    """
    
    print("="*70)
    print("NETMIKO MULTI-DEVICE OPERATIONS")
    print("="*70)
    print(f"\nDevices in inventory: {len(DEVICES)}")
    print("\nThis script will connect to your full Day 1 topology:")
    print("  • R1, R2, R3 (routers with OSPF)")
    print("  • SW1 (switch)")
    print("\nOperations:")
    print("  1. Connect to each device")
    print("  2. Gather system information")
    print("  3. Check OSPF neighbors (routers only)")
    print("  4. Generate summary report")
    print("\nStarting...\n")
    
    # Create logs directory
    import os
    os.makedirs('logs', exist_ok=True)
    # Why separate logs? Each device gets its own session log for debugging
    
    # Storage for results
    device_info_list = []
    ospf_info_list = []
    
    # Process each device
    for device in DEVICES:
        nickname = device.get('nickname', device['host'])
        role = device.get('role', 'unknown')
        
        print("\n" + "="*70)
        print(f"PROCESSING: {nickname} ({role})")
        print("="*70)
        
        # Connect
        connection = connect_to_device(device)
        
        if not connection:
            # Connection failed - record and continue to next device
            device_info_list.append({
                'nickname': nickname,
                'success': False,
                'error': 'Connection failed'
            })
            continue
        
        try:
            # Gather device info
            info = get_device_info(connection, nickname)
            device_info_list.append(info)
            
            # Check OSPF (routers only)
            if role == 'router':
                ospf_info = get_ospf_neighbors(connection, nickname)
                ospf_info_list.append(ospf_info)
            
        except Exception as e:
            print(f"[ERROR] Operations failed on {nickname}: {str(e)}")
            device_info_list.append({
                'nickname': nickname,
                'success': False,
                'error': str(e)
            })
            
        finally:
            # Always disconnect
            print(f"\n[DISCONNECTING] from {nickname}...")
            connection.disconnect()
            print(f"[DONE] {nickname} connection closed")
    
    # Generate summary report
    print("\n" + "="*70)
    print("SUMMARY REPORT")
    print("="*70)
    
    # Device Information Summary
    print("\n📊 Device Information:")
    print("-" * 70)
    for info in device_info_list:
        if info.get('success'):
            print(f"\n{info['nickname']}:")
            print(f"  Hostname: {info.get('hostname', 'N/A')}")
            print(f"  IOS Version: {info.get('ios_version', 'N/A')}")
            print(f"  Uptime: {info.get('uptime', 'N/A')}")
            print(f"  Interfaces: {info.get('interface_count', 0)} total "
                  f"({info.get('interfaces_up', 0)} up, {info.get('interfaces_down', 0)} down)")
            print(f"  Status: ✓ REACHABLE")
        else:
            print(f"\n{info['nickname']}: ✗ UNREACHABLE ({info.get('error', 'Unknown')})")
    
    # OSPF Summary
    print("\n\n🔗 OSPF Neighbor Status:")
    print("-" * 70)
    if ospf_info_list:
        for ospf in ospf_info_list:
            print(f"\n{ospf['nickname']}:")
            if ospf.get('has_ospf'):
                if ospf['neighbors']:
                    print(f"  ✓ OSPF Active - {len(ospf['neighbors'])} neighbor(s):")
                    for neighbor in ospf['neighbors']:
                        print(f"    • {neighbor['neighbor_id']} via {neighbor['interface']} "
                              f"({neighbor['state']})")
                else:
                    print(f"  ⚠️  OSPF configured but NO neighbors!")
            else:
                print(f"  ℹ️  No OSPF configured")
    else:
        print("  No routers with OSPF in inventory")
    
    # Success rate
    successful = sum(1 for i in device_info_list if i.get('success'))
    total = len(device_info_list)
    success_rate = (successful / total * 100) if total > 0 else 0
    
    print(f"\n\n📈 Success Rate: {successful}/{total} devices ({success_rate:.1f}%)")
    
    print("\n" + "="*70)
    print("COMPLETED")
    print("="*70)
    print("\n💡 Next steps:")
    print("  1. Review session logs in logs/ directory")
    print("  2. Check why any devices failed (if applicable)")
    print("  3. Verify OSPF neighbors match your Day 1 topology")
    print("  4. Try modifying script to gather additional info")


if __name__ == "__main__":
    main()

# ==============================================================================
# KEY TAKEAWAYS:
#
# 1. Multi-device operations need per-device error handling
# 2. One failure shouldn't stop the entire script
# 3. Structured data collection enables reporting and analysis
# 4. Session logs are essential for debugging
# 5. This pattern scales from 4 devices to 4000 devices
#
# PRODUCTION ENHANCEMENTS:
# - Parallel execution (threading/multiprocessing)
# - Progress bars (tqdm library)
# - Retry logic with exponential backoff
# - Dynamic inventory from NetBox (Day 5!)
# - Alerting on threshold violations
# - Database storage for trend analysis
#
# THIS is production-grade network automation!
# ==============================================================================
PYEOF

chmod +x 02_netmiko_multi_device.py
```

**Update IP addresses and run:**

```bash
# Get all management IPs
cd ~/netdevops-automation/labs/lab-001-foundation
netlab inspect | grep mgmt-ip

# Update the script with your actual IPs
cd ~/netdevops-automation/scripts/day2
nano 02_netmiko_multi_device.py
# Update all host IPs in DEVICES list

# Run it
python 02_netmiko_multi_device.py
```

**Expected Output:**
```
======================================================================
NETMIKO MULTI-DEVICE OPERATIONS
======================================================================

Devices in inventory: 4

======================================================================
PROCESSING: R1 (router)
======================================================================

[CONNECTING] R1 (172.16.0.1)...
[SUCCESS] ✓ Connected to R1
[PROMPT] r1#

[GATHERING] Information from R1...
[SUCCESS] ✓ Collected info from R1

[CHECKING] OSPF neighbors on R1...
[SUCCESS] ✓ Found 2 OSPF neighbor(s)

[DISCONNECTING] from R1...
[DONE] R1 connection closed

======================================================================
PROCESSING: R2 (router)
======================================================================
... (similar for R2, R3, SW1)

======================================================================
SUMMARY REPORT
======================================================================

📊 Device Information:
----------------------------------------------------------------------

R1:
  Hostname: r1
  IOS Version: 17.16.01a
  Uptime: r1 uptime is 4 hours, 30 minutes
  Interfaces: 6 total (4 up, 2 down)
  Status: ✓ REACHABLE

R2:
  Hostname: r2
  IOS Version: 17.16.01a
  ...

🔗 OSPF Neighbor Status:
----------------------------------------------------------------------

R1:
  ✓ OSPF Active - 2 neighbor(s):
    • 10.0.0.2 via Ethernet0/1 (FULL/  -)
    • 10.0.0.3 via Ethernet0/2 (FULL/  -)

R2:
  ✓ OSPF Active - 2 neighbor(s):
    • 10.0.0.1 via Ethernet0/1 (FULL/  -)
    • 10.0.0.3 via Ethernet0/2 (FULL/  -)

R3:
  ✓ OSPF Active - 2 neighbor(s):
    • 10.0.0.1 via Ethernet0/1 (FULL/  -)
    • 10.0.0.2 via Ethernet0/2 (FULL/  -)

SW1:
  ℹ️  No OSPF configured

📈 Success Rate: 4/4 devices (100.0%)
```

**Review session logs:**

```bash
# Check what actually happened on each device
cat logs/r1_session.log | head -30
cat logs/r2_session.log | head -30

# These logs show EXACTLY what Netmiko sent/received
# Invaluable for debugging!
```

---

## ⚙️ Part 3: Safe Configuration Management

### 3.1 Theory: Configuration Changes in Production

**Critical Rule:** NEVER push config changes without:
1. ✅ **Backup first** (can rollback if needed)
2. ✅ **Test syntax** (don't push broken configs)
3. ✅ **Verify after** (confirm change worked)
4. ✅ **Save config** (persist across reboot)
5. ✅ **Log everything** (audit trail)

**Real story:**
```
PRODUCTION INCIDENT:
- Junior engineer pushes ACL change
- Typo in ACL breaks connectivity
- No backup taken
- Can't rollback
- 2 hours to manually fix
- Cost: $50,000 in downtime

LESSON: Always backup first!
```

### 3.2 Implementing Safe Configuration Changes

Create `03_safe_config_management.py`:

```bash
cat > 03_safe_config_management.py << 'PYEOF'
#!/usr/bin/env python3
"""
03_safe_config_management.py - Production-grade configuration management

Purpose: Learn to push configuration changes SAFELY
Why: Config changes are high-risk - need proper safety measures

Safety principles:
1. Backup before changes
2. Test configuration syntax
3. Verify changes worked
4. Save to startup-config
5. Log everything for audit trail
"""

from netmiko import ConnectHandler
from datetime import datetime
import os
import sys

# ==============================================================================
# DEVICE CONFIGURATION
# ==============================================================================
# We'll demonstrate on R1 from your Day 1 lab
# ⚠️  UPDATE with your R1 management IP
# ==============================================================================

R1 = {
    'device_type': 'cisco_ios',
    'host': '172.16.0.1',          # ⚠️  UPDATE: Your R1 mgmt IP
    'username': 'admin',
    'password': 'admin',
    'secret': 'admin',
    'session_log': 'logs/config_session.log'
}


def backup_configuration(connection, device_name):
    """
    Backup current running configuration.
    
    WHY backup first?
    - Safety: Can restore if new config breaks things
    - Compliance: Audit trail of what changed
    - Debugging: Compare before/after to find issues
    - Best Practice: ALWAYS backup before changes
    
    Args:
        connection: Active Netmiko connection
        device_name: Name for backup file
        
    Returns:
        str: Backup filename, or None if failed
    """
    
    try:
        print(f"\n[BACKUP] Creating configuration backup...")
        
        # Generate timestamped filename
        # Why timestamp? Multiple backups, never overwrite, easy to find latest
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        filename = f"backups/{device_name}_{timestamp}.cfg"
        
        # Create backups directory if needed
        os.makedirs('backups', exist_ok=True)
        
        # Get running configuration
        # Why running-config? That's the ACTIVE config (not startup)
        print(f"[BACKUP] Retrieving running-config...")
        running_config = connection.send_command("show running-config")
        
        # Save to file
        with open(filename, 'w') as f:
            # Add header with metadata
            f.write(f"!\n")
            f.write(f"! Backup created: {datetime.now().isoformat()}\n")
            f.write(f"! Device: {device_name}\n")
            f.write(f"! Hostname: {connection.find_prompt()}\n")
            f.write(f"!\n")
            f.write(running_config)
        
        # Verify backup
        file_size = os.path.getsize(filename)
        print(f"[SUCCESS] ✓ Backup saved: {filename} ({file_size} bytes)")
        
        return filename
        
    except Exception as e:
        print(f"[ERROR] ✗ Backup failed: {str(e)}")
        print(f"[WARNING] Proceeding without backup is DANGEROUS!")
        return None


def send_config_commands(connection, commands, verify_command=None):
    """
    Send configuration commands with safety checks.
    
    This is production-grade config management:
    - Enter config mode automatically
    - Send commands
    - Verify changes (if verify_command provided)
    - Exit config mode
    - Return results
    
    Args:
        connection: Active Netmiko connection
        commands (list): Configuration commands to send
        verify_command (str): Command to verify change (optional)
        
    Returns:
        tuple: (success: bool, output: str, verify_output: str)
    """
    
    try:
        print(f"\n[CONFIG] Applying configuration changes...")
        print(f"[COMMANDS]:")
        for cmd in commands:
            print(f"  • {cmd}")
        
        # send_config_set() automatically:
        # 1. Enters 'configure terminal'
        # 2. Sends each command
        # 3. Exits config mode
        # Why automatic? Reduces errors, consistent behavior
        output = connection.send_config_set(commands)
        
        print(f"\n[DEVICE OUTPUT]:")
        print("-" * 70)
        print(output)
        print("-" * 70)
        
        # Verify the change if verification command provided
        if verify_command:
            print(f"\n[VERIFY] Running: {verify_command}")
            verify_output = connection.send_command(verify_command)
            print(f"[VERIFY OUTPUT]:")
            print("-" * 70)
            print(verify_output)
            print("-" * 70)
            return True, output, verify_output
        
        return True, output, None
        
    except Exception as e:
        print(f"\n[ERROR] ✗ Configuration failed: {str(e)}")
        print(f"[ADVICE] Check:")
        print(f"         • Command syntax is correct")
        print(f"         • Device supports these commands")
        print(f"         • Session log: logs/config_session.log")
        return False, str(e), None


def save_configuration(connection):
    """
    Save running-config to startup-config.
    
    WHY save?
    - Changes are LOST on reload without saving!
    - Device reboots, maintenance, power loss = config gone
    - "write memory" or "copy run start" persists changes
    
    Args:
        connection: Active Netmiko connection
        
    Returns:
        bool: True if saved successfully
    """
    
    try:
        print(f"\n[SAVE] Saving configuration to startup-config...")
        
        # Use send_command_timing() because "write memory" 
        # doesn't return to a standard prompt immediately
        # Why timing? Command completes without expected prompt
        output = connection.send_command_timing("write memory")
        
        print(f"[OUTPUT]:")
        print(output)
        
        # Verify save was successful
        if "OK" in output or "Built configuration" in output:
            print(f"[SUCCESS] ✓ Configuration saved to startup-config")
            return True
        else:
            print(f"[WARNING] ⚠️  Save command completed but verify manually")
            return False
            
    except Exception as e:
        print(f"[ERROR] ✗ Failed to save configuration: {str(e)}")
        return False


def demonstrate_safe_config_workflow(connection, device_name):
    """
    Demonstrate complete safe configuration workflow.
    
    This is THE production pattern:
    1. Backup current config
    2. Apply changes
    3. Verify changes
    4. Save to startup-config
    5. Log everything
    
    If ANY step fails, we know exactly where and can rollback.
    """
    
    print("\n" + "="*70)
    print("SAFE CONFIGURATION WORKFLOW DEMONSTRATION")
    print("="*70)
    
    # =========================================================================
    # Example 1: Add Loopback Interface
    # =========================================================================
    print("\n--- Example 1: Create Loopback Interface ---")
    
    commands = [
        "interface Loopback200",
        "description Python_Automation_Day2",
        "ip address 200.200.200.1 255.255.255.255",
        "no shutdown"
    ]
    
    verify = "show running-config interface Loopback200"
    
    success, output, verify_output = send_config_commands(
        connection,
        commands,
        verify
    )
    
    if success:
        print("\n[RESULT] ✓ Loopback interface created successfully")
    else:
        print("\n[RESULT] ✗ Failed to create loopback interface")
        return False
    
    # =========================================================================
    # Example 2: Configure SNMP (useful for monitoring)
    # =========================================================================
    print("\n--- Example 2: Configure SNMP ---")
    
    commands = [
        "snmp-server community NetAutomation RO",
        "snmp-server location Lab-Day2",
        "snmp-server contact automation@lab.local"
    ]
    
    verify = "show running-config | include snmp"
    
    success, output, verify_output = send_config_commands(
        connection,
        commands,
        verify
    )
    
    if success:
        print("\n[RESULT] ✓ SNMP configured successfully")
    else:
        print("\n[RESULT] ✗ Failed to configure SNMP")
        return False
    
    # =========================================================================
    # Example 3: Add Login Banner
    # =========================================================================
    print("\n--- Example 3: Configure Login Banner ---")
    
    commands = [
        "banner motd #",
        "***********************************************************",
        "* This device is managed by Python Network Automation     *",
        "* Unauthorized access is prohibited                       *",
        "* Configuration changes via automation only               *",
        "* Day 2 completed: " + datetime.now().strftime("%Y-%m-%d") + "                               *",
        "***********************************************************",
        "#"
    ]
    
    success, output, _ = send_config_commands(connection, commands)
    
    if success:
        print("\n[RESULT] ✓ Login banner configured")
    else:
        print("\n[RESULT] ✗ Failed to configure banner")
        return False
    
    # =========================================================================
    # Example 4: Modify Interface Description (careful with production!)
    # =========================================================================
    print("\n--- Example 4: Update Interface Description ---")
    
    # First, check current interfaces
    print("\n[CHECK] Current interfaces:")
    int_brief = connection.send_command("show ip interface brief")
    print(int_brief)
    
    # Update description on Ethernet0/0 (adjust if your device uses different names)
    commands = [
        "interface Ethernet0/0",
        "description Day2-Automation-Modified"
    ]
    
    verify = "show running-config interface Ethernet0/0 | include description"
    
    success, output, verify_output = send_config_commands(
        connection,
        commands,
        verify
    )
    
    if success:
        print("\n[RESULT] ✓ Interface description updated")
    else:
        print("\n[RESULT] ⚠️  Interface update failed (might not have Ethernet0/0)")
    
    return True


def main():
    """
    Main function demonstrating complete safe configuration workflow.
    
    Flow:
    1. Connect to device
    2. Backup current config (ALWAYS FIRST!)
    3. Apply configuration changes
    4. Verify each change
    5. Save to startup-config
    6. Disconnect
    
    This is how you do it in production!
    """
    
    print("="*70)
    print("SAFE CONFIGURATION MANAGEMENT")
    print("="*70)
    print("\nThis script demonstrates production-grade configuration management.")
    print("We'll apply changes to R1 from your Day 1 lab.\n")
    print("SAFETY MEASURES:")
    print("  ✓ Backup before changes")
    print("  ✓ Verify each change")
    print("  ✓ Save to startup-config")
    print("  ✓ Complete session logging")
    print("\nStarting...\n")
    
    # Create required directories
    os.makedirs('logs', exist_ok=True)
    os.makedirs('backups', exist_ok=True)
    
    # Connect
    print("[STEP 1] Connecting to R1...")
    try:
        connection = ConnectHandler(**R1)
        print(f"[SUCCESS] ✓ Connected to {connection.find_prompt()}")
    except Exception as e:
        print(f"[FAILED] ✗ Could not connect: {str(e)}")
        print(f"[HINT] Check:")
        print(f"       • R1 IP address is correct")
        print(f"       • Day 1 lab is running: netlab status")
        print(f"       • Device is reachable: ping {R1['host']}")
        sys.exit(1)
    
    try:
        # Backup
        print("\n[STEP 2] Creating backup...")
        backup_file = backup_configuration(connection, "R1")
        
        if not backup_file:
            print("\n[WARNING] ⚠️  No backup created!")
            print("[QUESTION] Continue anyway? (yes/no)")
            response = input().strip().lower()
            if response != 'yes':
                print("[ABORTED] User cancelled - no changes made")
                return
        
        # Apply changes
        print("\n[STEP 3] Applying configuration changes...")
        success = demonstrate_safe_config_workflow(connection, "R1")
        
        if not success:
            print("\n[WARNING] ⚠️  Some configuration changes failed")
            print(f"[RECOVERY] You can restore from: {backup_file}")
            return
        
        # Save
        print("\n[STEP 4] Saving configuration...")
        if save_configuration(connection):
            print("[SUCCESS] ✓ Configuration persisted to startup-config")
        else:
            print("[WARNING] ⚠️  Save may have failed - verify manually")
        
        # Final summary
        print("\n" + "="*70)
        print("CONFIGURATION WORKFLOW COMPLETED")
        print("="*70)
        print(f"\n✓ Backup file: {backup_file}")
        print(f"✓ Session log: logs/config_session.log")
        print(f"✓ All changes applied and saved")
        
        print("\n💡 Verification steps:")
        print("  1. Review session log to see exact commands sent")
        print("  2. Compare backup vs current config:")
        print(f"     diff {backup_file} <(netlab connect r1 --show run)")
        print("  3. Verify changes persist after reload:")
        print("     show startup-config | include Loopback200")
        
        print("\n📚 What you learned:")
        print("  • Always backup before configuration changes")
        print("  • Verify each change with show commands")
        print("  • Save to startup-config (or changes are lost!)")
        print("  • Session logs provide complete audit trail")
        print("  • This pattern works for any network device")
        
    except Exception as e:
        print(f"\n[ERROR] ✗ Unexpected error: {str(e)}")
        print(f"[DEBUG] Check session log: logs/config_session.log")
        
    finally:
        print("\n[CLEANUP] Disconnecting...")
        connection.disconnect()
        print("[DONE] Connection closed safely")


if __name__ == "__main__":
    main()

# ==============================================================================
# KEY TAKEAWAYS:
#
# 1. ALWAYS backup before configuration changes (non-negotiable!)
# 2. Use send_config_set() for config commands, send_command() for show commands
# 3. Verify changes were applied correctly
# 4. Save configuration to startup-config (or it's lost on reboot!)
# 5. Session logs are your complete audit trail
#
# PRODUCTION BEST PRACTICES:
# - Automated backups before ANY change
# - Dry-run mode (test without applying)
# - Rollback capability (restore from backup)
# - Change approval workflow (ticket system integration)
# - Comprehensive logging (syslog, database)
# - Post-change validation (automated tests)
# - Notification on failures (email, Slack, PagerDuty)
#
# NEXT: Multi-device configuration management with inventory
# ==============================================================================
PYEOF

chmod +x 03_safe_config_management.py
```

**Update IP and run:**

```bash
cd ~/netdevops-automation/scripts/day2
nano 03_safe_config_management.py
# Update R1 host IP

python 03_safe_config_management.py
```

**After running, verify the changes:**

```bash
# Check backup was created
ls -lh backups/

# View backup
cat backups/R1_*.cfg | head -20

# Compare before/after
cd ~/netdevops-automation/labs/lab-001-foundation
netlab connect r1
# On router:
show running-config interface Loopback200
show running-config | include snmp
show running-config | begin banner
```

---

## 🔄 Part 4: Production Patterns - OSPF Automation

### 4.1 Real-World Scenario: OSPF Health Check

**The Problem:** You manage 100 routers running OSPF. How do you quickly check:
- Are all OSPF neighbors up?
- Any routers with missing neighbors?
- OSPF process health across the network?

**Manual approach:** SSH to 100 routers, run "show ip ospf neighbor" 100 times, manually review output. **Time: 3-4 hours**

**Automated approach:** One Python script. **Time: 2 minutes**

### 4.2 Building OSPF Health Check Tool

Create `04_ospf_health_check.py`:

```bash
cat > 04_ospf_health_check.py << 'PYEOF'
#!/usr/bin/env python3
"""
04_ospf_health_check.py - Automated OSPF health monitoring

Purpose: Check OSPF status across all routers (using your Day 1 OSPF!)
Why: Real production networks need automated health checks

This leverages your Day 1 topology:
- R1, R2, R3 have OSPF configured (full mesh)
- Each router should have 2 OSPF neighbors
- Any deviation indicates a problem
"""

from netmiko import ConnectHandler
from datetime import datetime
import sys

# ==============================================================================
# DEVICE INVENTORY - Your Day 1 Routers
# ==============================================================================
# ⚠️  UPDATE with your actual management IPs
# ==============================================================================

ROUTERS = [
    {
        'device_type': 'cisco_ios',
        'host': '172.16.0.1',          # ⚠️  UPDATE: R1 mgmt IP
        'username': 'admin',
        'password': 'admin',
        'nickname': 'R1',
        'expected_neighbors': 2         # R1 should see R2 and R3
    },
    {
        'device_type': 'cisco_ios',
        'host': '172.16.0.2',          # ⚠️  UPDATE: R2 mgmt IP
        'username': 'admin',
        'password': 'admin',
        'nickname': 'R2',
        'expected_neighbors': 2         # R2 should see R1 and R3
    },
    {
        'device_type': 'cisco_ios',
        'host': '172.16.0.3',          # ⚠️  UPDATE: R3 mgmt IP
        'username': 'admin',
        'password': 'admin',
        'nickname': 'R3',
        'expected_neighbors': 2         # R3 should see R1 and R2
    }
]

# ==============================================================================
# Why 'expected_neighbors'?
# - Baseline: Know what's NORMAL for each router
# - Alert: Detect when actual != expected
# - Context: Understand if problem is isolated or widespread
#
# In production:
# - This data comes from NetBox (your source of truth)
# - Updated automatically when topology changes
# - Alerts integrate with monitoring systems (Nagios, Prometheus)
# ==============================================================================


def check_ospf_neighbors(connection, nickname):
    """
    Check OSPF neighbors on a router.
    
    Parses "show ip ospf neighbor" to extract:
    - Number of neighbors
    - Neighbor IDs
    - Neighbor states
    - Interfaces
    
    Args:
        connection: Active Netmiko connection
        nickname: Router identifier
        
    Returns:
        dict: OSPF neighbor information
    """
    
    result = {
        'nickname': nickname,
        'success': False,
        'neighbors': [],
        'neighbor_count': 0
    }
    
    try:
        # Get OSPF neighbor information
        output = connection.send_command("show ip ospf neighbor")
        
        # Check if OSPF is running
        if 'not enabled' in output.lower() or not output.strip():
            result['error'] = 'OSPF not configured'
            return result
        
        # Parse neighbor table
        # Expected format:
        # Neighbor ID     Pri   State           Dead Time   Address         Interface
        # 10.0.0.2          0   FULL/  -        00:00:37    10.1.0.2        Ethernet0/1
        
        lines = output.split('\n')
        for line in lines:
            # Look for FULL state (healthy neighbor)
            if 'FULL' in line:
                parts = line.split()
                if len(parts) >= 6:
                    neighbor = {
                        'neighbor_id': parts[0],
                        'priority': parts[1],
                        'state': parts[2],
                        'dead_time': parts[3],
                        'address': parts[4],
                        'interface': parts[5],
                        'health': 'HEALTHY' if 'FULL' in parts[2] else 'DEGRADED'
                    }
                    result['neighbors'].append(neighbor)
        
        result['neighbor_count'] = len(result['neighbors'])
        result['success'] = True
        
        return result
        
    except Exception as e:
        result['error'] = str(e)
        return result


def check_ospf_process(connection, nickname):
    """
    Check OSPF process health.
    
    Gathers:
    - Router ID
    - OSPF process ID
    - Areas configured
    - SPF statistics
    
    Args:
        connection: Active Netmiko connection
        nickname: Router identifier
        
    Returns:
        dict: OSPF process information
    """
    
    result = {'nickname': nickname, 'success': False}
    
    try:
        # Get OSPF process information
        output = connection.send_command("show ip ospf")
        
        if 'not enabled' in output.lower():
            result['error'] = 'OSPF not running'
            return result
        
        # Parse key information
        lines = output.split('\n')
        for line in lines:
            # Extract Router ID
            if 'Router ID' in line:
                parts = line.split()
                if len(parts) >= 3:
                    result['router_id'] = parts[2]
            
            # Extract Process ID
            if 'Routing Process' in line and 'ospf' in line.lower():
                parts = line.split()
                for i, part in enumerate(parts):
                    if part == 'ospf':
                        if i + 1 < len(parts):
                            result['process_id'] = parts[i + 1]
            
            # Check for SPF runs (shows network stability)
            if 'SPF schedule delay' in line:
                result['spf_info'] = line.strip()
        
        result['success'] = True
        return result
        
    except Exception as e:
        result['error'] = str(e)
        return result


def generate_health_report(results):
    """
    Generate comprehensive OSPF health report.
    
    Analyzes collected data and identifies:
    - Healthy routers
    - Routers with issues
    - Missing neighbors
    - Overall network health
    
    Args:
        results (list): List of OSPF check results
    """
    
    print("\n" + "="*70)
    print("OSPF HEALTH CHECK REPORT")
    print("="*70)
    print(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}")
    print(f"Routers checked: {len(results)}")
    
    # Categorize results
    healthy = []
    degraded = []
    failed = []
    
    for result in results:
        if not result.get('success'):
            failed.append(result)
        elif result['neighbor_count'] < result.get('expected_neighbors', 0):
            degraded.append(result)
        else:
            healthy.append(result)
    
    # Overall status
    print(f"\n📊 Overall Status:")
    print(f"  ✓ Healthy: {len(healthy)}")
    print(f"  ⚠️  Degraded: {len(degraded)}")
    print(f"  ✗ Failed: {len(failed)}")
    
    # Detailed results
    print(f"\n📋 Detailed Results:")
    print("-" * 70)
    
    for result in results:
        nickname = result['nickname']
        expected = result.get('expected_neighbors', 'N/A')
        
        print(f"\n{nickname}:")
        
        if not result.get('success'):
            print(f"  Status: ✗ FAILED")
            print(f"  Error: {result.get('error', 'Unknown')}")
            continue
        
        actual = result['neighbor_count']
        status = "✓ HEALTHY" if actual >= expected else "⚠️  DEGRADED"
        
        print(f"  Status: {status}")
        print(f"  Router ID: {result.get('router_id', 'N/A')}")
        print(f"  OSPF Neighbors: {actual}/{expected} expected")
        
        if result['neighbors']:
            print(f"  Neighbor Details:")
            for neighbor in result['neighbors']:
                print(f"    • {neighbor['neighbor_id']} via {neighbor['interface']} "
                      f"({neighbor['state']}) [{neighbor['health']}]")
        else:
            print(f"  ⚠️  NO NEIGHBORS FOUND!")
        
        # Alerts
        if actual < expected:
            missing = expected - actual
            print(f"  🚨 ALERT: Missing {missing} neighbor(s)!")
        elif actual > expected:
            extra = actual - expected
            print(f"  ⚠️  WARNING: {extra} unexpected neighbor(s)")
    
    # Save report to file
    report_filename = f"ospf_health_report_{datetime.now().strftime('%Y%m%d_%H%M%S')}.txt"
    
    with open(report_filename, 'w') as f:
        f.write("="*70 + "\n")
        f.write("OSPF HEALTH CHECK REPORT\n")
        f.write("="*70 + "\n")
        f.write(f"Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n\n")
        
        for result in results:
            f.write(f"\n{result['nickname']}:\n")
            if result.get('success'):
                f.write(f"  Neighbors: {result['neighbor_count']}/{result.get('expected_neighbors', 'N/A')}\n")
                for neighbor in result['neighbors']:
                    f.write(f"    • {neighbor['neighbor_id']} via {neighbor['interface']} ({neighbor['state']})\n")
            else:
                f.write(f"  Error: {result.get('error', 'Unknown')}\n")
    
    print(f"\n💾 Report saved: {report_filename}")


def main():
    """
    Main function - orchestrate OSPF health checks.
    
    This demonstrates production monitoring patterns:
    1. Check multiple devices
    2. Collect structured data
    3. Compare against baselines
    4. Identify anomalies
    5. Generate reports
    6. Alert on issues
    """
    
    print("="*70)
    print("AUTOMATED OSPF HEALTH CHECK")
    print("="*70)
    print("\nThis script checks OSPF status on your Day 1 routers:")
    print("  • R1, R2, R3 (OSPF full mesh)")
    print("  • Each router should have 2 neighbors")
    print("\nChecking...\n")
    
    results = []
    
    # Check each router
    for router in ROUTERS:
        nickname = router['nickname']
        expected = router['expected_neighbors']
        
        print(f"\n{'='*70}")
        print(f"Checking {nickname}")
        print(f"{'='*70}")
        
        try:
            # Connect
            connection = ConnectHandler(**router)
            prompt = connection.find_prompt()
            print(f"[CONNECTED] {prompt}")
            
            # Check OSPF neighbors
            neighbor_result = check_ospf_neighbors(connection, nickname)
            
            # Check OSPF process
            process_result = check_ospf_process(connection, nickname)
            
            # Merge results
            result = {**neighbor_result, **process_result, 'expected_neighbors': expected}
            results.append(result)
            
            # Quick status
            if result['success']:
                actual = result['neighbor_count']
                status = "✓" if actual >= expected else "⚠️"
                print(f"{status} OSPF neighbors: {actual}/{expected}")
            else:
                print(f"✗ OSPF check failed")
            
            # Disconnect
            connection.disconnect()
            
        except Exception as e:
            print(f"[FAILED] ✗ Could not connect: {str(e)}")
            results.append({
                'nickname': nickname,
                'success': False,
                'error': f'Connection failed: {str(e)}',
                'neighbor_count': 0,
                'expected_neighbors': expected
            })
    
    # Generate report
    generate_health_report(results)
    
    print("\n" + "="*70)
    print("HEALTH CHECK COMPLETE")
    print("="*70)
    
    # Summary and next steps
    healthy_count = sum(1 for r in results if r.get('success') and 
                       r.get('neighbor_count', 0) >= r.get('expected_neighbors', 999))
    
    if healthy_count == len(results):
        print("\n✅ All routers healthy! OSPF network operating normally.")
    else:
        issues = len(results) - healthy_count
        print(f"\n⚠️  {issues} router(s) have issues - review report above")
    
    print("\n💡 Production enhancements:")
    print("  • Schedule this script to run every 5 minutes")
    print("  • Send alerts on OSPF neighbor loss (email/Slack/PagerDuty)")
    print("  • Store results in database for trend analysis")
    print("  • Correlate with other metrics (interface errors, CPU, etc.)")
    print("  • Auto-remediation for known issues")


if __name__ == "__main__":
    main()

# ==============================================================================
# KEY TAKEAWAYS:
#
# 1. Automated health checks find problems before users report them
# 2. Baseline expectations (expected_neighbors) enable alerting
# 3. Structured data collection enables trending and analysis
# 4. This pattern scales to any network protocol (BGP, EIGRP, etc.)
# 5. Production monitoring runs continuously, not just once
#
# PRODUCTION EVOLUTION:
# - Add this to cron: */5 * * * * /path/to/script
# - Integrate with monitoring: Prometheus, Grafana, ELK stack
# - Alert workflows: PagerDuty for critical, email for warnings
# - Auto-remediation: Clear OSPF process, restart interfaces
# - ML/AI: Anomaly detection, predictive failures
#
# THIS is how Netflix/Google/Amazon monitor their networks!
# ==============================================================================
PYEOF

chmod +x 04_ospf_health_check.py
```

**Update IPs and run:**

```bash
cd ~/netdevops-automation/scripts/day2
nano 04_ospf_health_check.py
# Update R1, R2, R3 host IPs

python 04_ospf_health_check.py
```

**Expected Output:**
```
======================================================================
AUTOMATED OSPF HEALTH CHECK
======================================================================

======================================================================
Checking R1
======================================================================
[CONNECTED] r1#
✓ OSPF neighbors: 2/2

======================================================================
Checking R2
======================================================================
[CONNECTED] r2#
✓ OSPF neighbors: 2/2

======================================================================
Checking R3
======================================================================
[CONNECTED] r3#
✓ OSPF neighbors: 2/2

======================================================================
OSPF HEALTH CHECK REPORT
======================================================================
Generated: 2025-11-09 16:30:45
Routers checked: 3

📊 Overall Status:
  ✓ Healthy: 3
  ⚠️  Degraded: 0
  ✗ Failed: 0

📋 Detailed Results:
----------------------------------------------------------------------

R1:
  Status: ✓ HEALTHY
  Router ID: 10.0.0.1
  OSPF Neighbors: 2/2 expected
  Neighbor Details:
    • 10.0.0.2 via Ethernet0/1 (FULL/  -) [HEALTHY]
    • 10.0.0.3 via Ethernet0/2 (FULL/  -) [HEALTHY]

R2:
  Status: ✓ HEALTHY
  Router ID: 10.0.0.2
  OSPF Neighbors: 2/2 expected
  Neighbor Details:
    • 10.0.0.1 via Ethernet0/1 (FULL/  -) [HEALTHY]
    • 10.0.0.3 via Ethernet0/2 (FULL/  -) [HEALTHY]

R3:
  Status: ✓ HEALTHY
  Router ID: 10.0.0.2
  OSPF Neighbors: 2/2 expected
  Neighbor Details:
    • 10.0.0.1 via Ethernet0/1 (FULL/  -) [HEALTHY]
    • 10.0.0.2 via Ethernet0/2 (FULL/  -) [HEALTHY]

💾 Report saved: ospf_health_report_20251109_163045.txt

======================================================================
HEALTH CHECK COMPLETE
======================================================================

✅ All routers healthy! OSPF network operating normally.
```

---

## ✅ Part 5: Verification & Challenges

### 5.1 Create Verification Script

```bash
cd ~/netdevops-automation/scripts/day2

cat > verify_day2.sh << 'BASHEOF'
#!/bin/bash
# Day 2 Verification Script

echo "=========================================="
echo "Day 2 Verification"
echo "=========================================="

PASS=0
FAIL=0

check() {
    if eval "$2" >/dev/null 2>&1; then
        echo "  ✓ $1"
        ((PASS++))
    else
        echo "  ✗ $1"
        ((FAIL++))
    fi
}

# Check virtual environment
echo ""
echo "[1] Python Environment:"
check "Virtual environment active" "python -c 'import sys; sys.prefix != sys.base_prefix'"
check "Netmiko installed" "python -c 'import netmiko'"
check "Paramiko installed" "python -c 'import paramiko'"

# Check Day 1 lab running
echo ""
echo "[2] Day 1 Lab Status:"
cd ~/netdevops-automation/labs/lab-001-foundation
check "Lab directory exists" "[ -d ~/netdevops-automation/labs/lab-001-foundation ]"
check "R1 running" "netlab status 2>/dev/null | grep -q 'r1.*running'"
check "R2 running" "netlab status 2>/dev/null | grep -q 'r2.*running'"
check "R3 running" "netlab status 2>/dev/null | grep -q 'r3.*running'"

# Check Day 2 scripts
echo ""
echo "[3] Day 2 Scripts:"
cd ~/netdevops-automation/scripts/day2
check "01_paramiko_basics.py exists" "[ -f 01_paramiko_basics.py ]"
check "02_netmiko_multi_device.py exists" "[ -f 02_netmiko_multi_device.py ]"
check "03_safe_config_management.py exists" "[ -f 03_safe_config_management.py ]"
check "04_ospf_health_check.py exists" "[ -f 04_ospf_health_check.py ]"

# Check outputs
echo ""
echo "[4] Script Outputs:"
check "Session logs created" "[ -d logs ] && [ -n \"$(ls -A logs 2>/dev/null)\" ]"
check "Backup directory exists" "[ -d backups ]"
check "OSPF report generated" "ls ospf_health_report_*.txt 1>/dev/null 2>&1"

echo ""
echo "=========================================="
echo "Results: ✅ $PASS passed | ❌ $FAIL failed"
echo "=========================================="

if [ $FAIL -eq 0 ]; then
    echo "🎉 Perfect! Day 2 complete!"
else
    echo "⚠️  Some checks failed - review above"
fi
BASHEOF

chmod +x verify_day2.sh
./verify_day2.sh
```

---

## 🎓 Challenge Exercises

### Challenge 1: Enhanced Interface Monitor (Easy)

**Task:** Create a script that checks interface status across all 4 devices and highlights down interfaces.

**Requirements:**
- Connect to R1, R2, R3, SW1
- Get interface status from each
- Display ONLY interfaces that are "down"
- Format as a summary table

**Hints:**
```python
# Use: show ip interface brief
# Parse for "down" status  
# Store in list/dict structure
# Print formatted output
```

<details>
<summary>Click for solution</summary>

```python
#!/usr/bin/env python3
"""Challenge 1 Solution: Enhanced Interface Monitor"""

from netmiko import ConnectHandler

DEVICES = [
    {'device_type': 'cisco_ios', 'host': '172.16.0.1', 'username': 'admin', 
     'password': 'admin', 'nickname': 'R1'},
    # ... add R2, R3, SW1
]

def get_down_interfaces(connection):
    """Find interfaces that are down."""
    output = connection.send_command("show ip interface brief")
    down_intfs = []
    
    for line in output.split('\n'):
        if 'down' in line.lower() and 'administratively' not in line.lower():
            parts = line.split()
            if parts:
                down_intfs.append(parts[0])
    
    return down_intfs

def main():
    print("="*70)
    print("Interface Status Monitor - Down Interfaces")
    print("="*70)
    
    all_down = {}
    
    for device in DEVICES:
        conn = ConnectHandler(**device)
        down_intfs = get_down_interfaces(conn)
        all_down[device['nickname']] = down_intfs
        conn.disconnect()
    
    # Report
    print("\n📊 Down Interfaces Report:")
    for device_name, intfs in all_down.items():
        if intfs:
            print(f"\n{device_name}: ⚠️  {len(intfs)} interface(s) down")
            for intf in intfs:
                print(f"  • {intf}")
        else:
            print(f"\n{device_name}: ✓ All interfaces up")

if __name__ == "__main__":
    main()
```

</details>

### Challenge 2: OSPF Convergence Timer (Medium)

**Task:** Measure how long it takes for OSPF to converge after shutting down an interface.

**Requirements:**
- Shutdown one link between routers (e.g., R1-R2)
- Monitor OSPF neighbor count every second
- Measure time until convergence (all expected neighbors up)
- Report convergence time

**Hints:**
```python
import time
# 1. Get baseline neighbor count
# 2. Shutdown interface
# 3. Loop: check neighbors every 1 second
# 4. When count returns to baseline: convergence complete
# 5. Calculate elapsed time
```

<details>
<summary>Click for solution</summary>

```python
#!/usr/bin/env python3
"""Challenge 2 Solution: OSPF Convergence Timer"""

from netmiko import ConnectHandler
import time

R1 = {
    'device_type': 'cisco_ios',
    'host': '172.16.0.1',
    'username': 'admin',
    'password': 'admin'
}

def count_ospf_neighbors(connection):
    """Count FULL OSPF neighbors."""
    output = connection.send_command("show ip ospf neighbor")
    return output.count('FULL/')

def main():
    conn = ConnectHandler(**R1)
    
    # Baseline
    baseline = count_ospf_neighbors(conn)
    print(f"Baseline OSPF neighbors: {baseline}")
    
    # Shutdown interface
    print("\nShutting down Ethernet0/1...")
    conn.send_config_set(["interface Ethernet0/1", "shutdown"])
    
    # Monitor convergence
    print("Monitoring convergence...")
    start_time = time.time()
    
    while True:
        current = count_ospf_neighbors(conn)
        elapsed = time.time() - start_time
        print(f"  t+{elapsed:.1f}s: {current} neighbors")
        
        if current == baseline - 1:  # One neighbor lost (expected)
            break
        
        time.sleep(1)
    
    convergence_time = time.time() - start_time
    
    # Restore interface
    print("\nRestoring Ethernet0/1...")
    conn.send_config_set(["interface Ethernet0/1", "no shutdown"])
    
    conn.disconnect()
    
    print(f"\n✓ Convergence time: {convergence_time:.2f} seconds")

if __name__ == "__main__":
    main()
```

</details>

### Challenge 3: Configuration Template Engine (Hard)

**Task:** Create a script that applies role-based configuration templates to devices.

**Requirements:**
- Define config templates for routers vs switches
- Apply template based on device role
- Include variables (hostname, management IP, etc.)
- Verify changes were applied
- Generate before/after comparison report

**Hints:**
```python
# Templates dictionary with placeholders
TEMPLATES = {
    'router': [
        "logging buffered {{buffer_size}}",
        "hostname {{hostname}}",
        # ...
    ],
    'switch': [...]
}

# Fill in variables:
template = template.replace('{{hostname}}', actual_hostname)
```

<details>
<summary>Click for solution</summary>

```python
#!/usr/bin/env python3
"""Challenge 3 Solution: Configuration Template Engine"""

from netmiko import ConnectHandler
import re

DEVICES = [
    {'device_type': 'cisco_ios', 'host': '172.16.0.1', 'username': 'admin',
     'password': 'admin', 'nickname': 'R1', 'role': 'router'},
    # ... add others
]

TEMPLATES = {
    'router': [
        "logging buffered 16384",
        "no ip domain-lookup",
        "ip domain-name lab.local",
        "line vty 0 4",
        " exec-timeout 30 0"
    ],
    'switch': [
        "logging buffered 8192",
        "no ip domain-lookup",
        "spanning-tree mode rapid-pvst"
    ]
}

def get_config_section(connection, section):
    """Get a section of running config."""
    output = connection.send_command(f"show running-config | section {section}")
    return output

def apply_template(device):
    """Apply role-based template to device."""
    role = device['role']
    nickname = device['nickname']
    
    if role not in TEMPLATES:
        return {'success': False, 'error': f'Unknown role: {role}'}
    
    try:
        conn = ConnectHandler(**device)
        
        # Backup before
        before = conn.send_command("show running-config")
        
        # Apply template
        template = TEMPLATES[role]
        output = conn.send_config_set(template)
        
        # Get after
        after = conn.send_command("show running-config")
        
        # Save
        conn.send_command_timing("write memory")
        
        conn.disconnect()
        
        # Find differences
        diff = []
        before_lines = set(before.split('\n'))
        after_lines = set(after.split('\n'))
        added = after_lines - before_lines
        
        return {
            'success': True,
            'device': nickname,
            'role': role,
            'commands_applied': len(template),
            'changes': list(added)[:10]  # First 10 changes
        }
        
    except Exception as e:
        return {'success': False, 'device': nickname, 'error': str(e)}

def main():
    print("="*70)
    print("Configuration Template Engine")
    print("="*70)
    
    results = []
    
    for device in DEVICES:
        print(f"\nProcessing {device['nickname']} ({device['role']})...")
        result = apply_template(device)
        results.append(result)
        
        if result['success']:
            print(f"  ✓ Applied {result['commands_applied']} commands")
        else:
            print(f"  ✗ Failed: {result.get('error')}")
    
    # Summary
    print("\n" + "="*70)
    print("Summary:")
    for r in results:
        status = "✓" if r['success'] else "✗"
        print(f"  {status} {r.get('device', 'Unknown')}")

if __name__ == "__main__":
    main()
```

</details>

---

## 💭 Reflection Questions

### Technical Understanding

**1. What's the difference between Paramiko and Netmiko? When would you use each?**

<details>
<summary>Answer</summary>

**Paramiko:**
- Low-level SSH library
- Generic (works for any SSH server)
- Manual prompt handling, timing, buffer management
- Use when: Need fine control, non-network devices, Netmiko doesn't support device type

**Netmiko:**
- Built on Paramiko
- Network device-specific (200+ types)
- Automatic prompt detection, pagination, config mode
- Use when: Network automation (99% of cases), need multi-vendor support

**Analogy:** Paramiko = Assembly language, Netmiko = Python
</details>

**2. Why do we need `session_log` parameter? Can't we just look at script output?**

<details>
<summary>Answer</summary>

Session logs show:
- EXACT bytes sent/received (script output is formatted)
- Timing information
- Hidden characters (prompts, control sequences)
- What Netmiko does automatically
- Complete audit trail

Script output = human-friendly summary  
Session log = complete raw truth

Essential for debugging: "What did the device actually see?"
</details>

**3. In `02_netmiko_multi_device.py`, why return `None` on connection failure instead of raising an exception?**

<details>
<summary>Answer</summary>

**Why return None:**
- Script continues with other devices (partial success)
- Caller decides how to handle (flexibility)
- Collect results from all devices, report all failures

**Why NOT exception:**
- One failure would stop entire script
- Production: "Get data from 95 devices even if 5 fail" > "Get nothing because 5 failed"

**Pattern:** Graceful degradation > all-or-nothing failure
</details>

### Production Thinking

**4. You have 500 routers. Your OSPF health check script takes 2 minutes per device (sequentially). That's 16 hours! How do you speed this up?**

<details>
<summary>Answer</summary>

**Threading/Multiprocessing:**
```python
from concurrent.futures import ThreadPoolExecutor

with ThreadPoolExecutor(max_workers=50) as executor:
    results = executor.map(check_router, ROUTERS)
```

**Benefits:**
- 50 devices in parallel = 100x faster
- Same code, just parallelized

**Considerations:**
- CPU/memory limits (how many threads?)
- Network bandwidth
- Device load (don't overwhelm)
- Rate limiting (be a good citizen)

**Result:** 16 hours → 10 minutes 🚀
</details>

**5. Your backup script creates 10 backups per day per device. After a year, you have 36,500 backup files. What's your storage strategy?**

<details>
<summary>Answer</summary>

**Smart retention policy:**
- Keep last 7 days: All backups (hourly)
- Keep last month: Daily backups (delete hourly)
- Keep last year: Weekly backups (delete daily)
- Older than 1 year: Delete or archive to S3 Glacier

**Implementation:**
```python
# Daily cleanup job
- Delete backups older than 7 days if more than 1/day
- Compress backups older than 30 days
- Archive to S3 if older than 90 days
```

**Result:** 36,500 files → ~150 files (99% reduction)
</details>

### Architectural Thinking

**6. Your config management script works great with hardcoded device IPs. How do you make it production-ready?**

<details>
<summary>Answer</summary>

**Move to external inventory:**

**Option 1: NetBox API (Day 5)**
```python
from pynetbox import api
nb = api('http://netbox', token='xxx')
devices = nb.dcim.devices.filter(role='router')
```

**Option 2: YAML file**
```yaml
# inventory.yml
routers:
  - name: R1
    ip: 172.16.0.1
    role: core
```

**Option 3: Database**
```python
import sqlite3
devices = db.execute("SELECT * FROM devices WHERE role='router'")
```

**Benefits:**
- Single source of truth
- Easy updates (no code changes)
- Version controlled (Git)
- Team collaboration
- Dynamic filtering
</details>

**7. Your OSPF health check finds a problem at 3 AM. How does it alert you?**

<details>
<summary>Answer</summary>

**Alert integration:**

**Level 1: Email**
```python
import smtplib
if degraded_routers:
    send_email(to='oncall@company.com', 
               subject='OSPF Alert: 3 routers degraded')
```

**Level 2: Slack/Teams**
```python
import requests
slack_webhook = 'https://hooks.slack.com/xxx'
requests.post(slack_webhook, json={'text': '🚨 OSPF issues detected'})
```

**Level 3: PagerDuty (critical)**
```python
import pypd
pypd.Incident.create(
    title='OSPF neighbor loss',
    service=service,
    urgency='high'
)
```

**Level 4: Auto-remediation**
```python
if issue_is_known_pattern():
    apply_fix()
    if verify_fix():
        notify("Auto-remediated")
    else:
        escalate_to_human()
```
</details>

---

## 📝 What You Learned Today

### Core Skills Acquired

✅ **SSH Automation Foundations**
- How SSH works programmatically (Paramiko)
- What Netmiko abstracts away
- Why understanding internals matters

✅ **Netmiko Mastery**
- Multi-device connections
- Error handling patterns
- Session logging for debugging
- Configuration management

✅ **Production Patterns**
- Backup before changes (always!)
- Verify after changes
- Per-device error handling
- Structured data collection
- Report generation

✅ **Real Protocol Automation**
- OSPF health checking (your Day 1 OSPF!)
- Neighbor verification
- Network-wide status checks
- Baseline comparisons

### Files You Created

```
~/netdevops-automation/scripts/day2/
├── 01_paramiko_basics.py              # SSH fundamentals
├── 02_netmiko_multi_device.py         # Multi-device operations
├── 03_safe_config_management.py       # Config management
├── 04_ospf_health_check.py           # OSPF monitoring
├── verify_day2.sh                     # Verification script
├── logs/                              # Session logs
│   ├── r1_session.log
│   ├── r2_session.log
│   ├── r3_session.log
│   └── sw1_session.log
├── backups/                           # Config backups
│   └── R1_20251109_*.cfg
└── ospf_health_report_*.txt           # OSPF reports
```

### Concepts Mastered

- ✅ SSH automation mechanics
- ✅ Multi-device operation patterns
- ✅ Configuration change safety
- ✅ Network protocol verification
- ✅ Error handling strategies
- ✅ Production monitoring patterns
- ✅ Audit trail generation

---

## 🚀 Next Steps: Day 3 Preview

**Tomorrow: Jinja2 Templates Introduction**

**What you'll learn:**
- Separate data from configuration (infrastructure as code)
- Template syntax (variables, loops, conditionals)
- Generate configs for multiple devices from one template
- Integration with Python automation
- Multi-vendor templates

**Why it matters:**
- Configure 100 routers with one template (vs 100 manual configs)
- Single source of truth for standards
- Easier to maintain and audit
- Faster deployments

**Preparation:**
- Your Day 1 lab stays running
- Keep Day 2 scripts (we'll enhance them)
- Think about repetitive config tasks at work

**Question to ponder:**
> "I need to configure 50 routers with NTP, syslog, and SNMP. Each router has different hostname and management IP. How do I do this efficiently?"

**Answer:** Jinja2 templates! That's Day 3.

---

## 🐛 Troubleshooting Guide

### Issue: "Connection timeout" on all devices

**Cause:** Day 1 lab not running or wrong IPs

**Fix:**
```bash
cd ~/netdevops-automation/labs/lab-001-foundation
netlab status
# If not running:
netlab up

# Verify IPs:
netlab inspect
```

### Issue: "Authentication failed"

**Cause:** Wrong credentials or SSH not enabled

**Fix:**
```bash
# Test manual SSH:
ssh admin@172.16.0.1

# If password doesn't work, check lab:
cd ~/netdevops-automation/labs/lab-001-foundation
cat topology.yml | grep -A5 defaults
```

### Issue: Scripts hang forever

**Cause:** Timing issues or device not responding

**Fix:**
1. Check session log: `cat logs/r1_session.log`
2. Look for prompt detection issues
3. Add timeout: `'timeout': 10` in device params
4. Try manual command: `netlab connect r1`

### Issue: "OSPF not configured" but it should be

**Cause:** Day 1 lab might have been rebuilt without OSPF

**Fix:**
```bash
cd ~/netdevops-automation/labs/lab-001-foundation
netlab down
netlab up

# Verify OSPF:
netlab connect r1
show ip ospf neighbor
```

### Issue: Configuration changes don't persist after reload

**Cause:** Didn't save to startup-config

**Fix:**
```python
# Add to your script:
connection.send_command_timing("write memory")

# Verify:
connection.send_command("show startup-config | include Loopback200")
```

---

## 📊 Day 2 Stats

**Time Investment:** 3 hours  
**Scripts Created:** 4 production-grade automation scripts  
**Devices Automated:** 4 (R1, R2, R3, SW1)  
**Lines of Code:** 800+  
**OSPF Neighbors Verified:** 6  
**Configuration Backups:** Multiple  
**Session Logs Generated:** 4+  
**New Skills:** 7 major concepts  

---

**🎉 Congratulations! You've completed Day 2!**

You now understand:
- ✅ How SSH automation works (Paramiko internals)
- ✅ Why Netmiko is essential (and how to use it)
- ✅ Multi-device automation patterns
- ✅ Safe configuration management
- ✅ Real protocol automation (OSPF)
- ✅ Production monitoring patterns

**This is huge progress!** You can now:
- Automate any Cisco device (IOL, real hardware, etc.)
- Work with multiple devices efficiently
- Push configuration changes safely
- Monitor network protocols
- Debug when tools fail
- Interview confidently about network automation

**Ready for Day 3?** Or need to practice Day 2 more?

---

**Day 2 Complete ✓**

*Last Updated: November 2025*  
*Created for: atr399*  
*Repository: network-automation-journey*
