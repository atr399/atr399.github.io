---
title: "Network-Automation-Day1"
date: 2025-11-8 e.g., 2025-11-01 10:00:00 +0800
categories: [netdevops, lab]
tags: [network-automation, netdevops, lab-setup, netmiko, netbox, batfish, netlab]
---

# Day 1: Foundation - Lab Environment Setup

**Module:** 1 - Foundation  
**Duration:** 3 hours  
**Difficulty:** Intermediate  
**Prerequisites:** Ubuntu 24.04, basic Linux commands, networking knowledge

---

## 🎯 Today's Objectives

By the end of this session, you will:
- ✅ Understand the complete NetDevOps architecture
- ✅ Deploy NetBox (Source of Truth) in Docker
- ✅ Deploy Batfish (Pre-deployment testing) in Docker
- ✅ Create a network topology with netlab (3 routers, 1 switch, 3 hosts)
- ✅ Set up Python environment with all required packages
- ✅ Verify every component is working
- ✅ Execute your first network command programmatically

**Success Criteria:**
```
✅ NetBox web UI accessible at http://localhost:8000
✅ Batfish responding on port 9997
✅ Network lab running (can SSH to router)
✅ Python script successfully connects to router
✅ All verification tests pass
```

---

## 🤔 Before We Start: Check Your Current State

Let's understand where you are right now.

**Quick Check:**
```bash
# Run these commands and note the output
cat /etc/os-release | grep VERSION
python3 --version
docker --version
netlab --version
```

**Questions for you:**
1. Have you worked with Docker before? (yes/no/a little)
2. Have you created VMs or containers for network devices? (yes/no)
3. What's your biggest concern about today's setup?

*Write these down - we'll reference them later*

---

## 📖 Why This Matters: The Big Picture

Before diving into commands, let's understand WHY we're building this lab.

### The Problem in Production

Imagine you work at a company with 500 network devices. You need to:
- Add NTP servers to all devices
- Update ACLs for new security policy
- Deploy OSPF changes

**Manual approach:**
- SSH to 500 devices (8 hours)
- Hope you don't make typos
- No testing before deployment
- Fingers crossed it works

**Automated approach (what we're building):**
- Define change in NetBox (5 minutes)
- Generate configs automatically
- Test with Batfish (catches errors BEFORE deployment)
- Deploy with Ansible (10 minutes for all 500)
- Verify with pyATS (automated checks)

**Our lab simulates this at small scale so you understand the concepts.**

### What We're Building Today

```
┌────────────────────────────────────────────────────────┐
│            YOUR LAB ARCHITECTURE                        │
├────────────────────────────────────────────────────────┤
│                                                         │
│  ┌─────────────────┐  ┌─────────────────┐             │
│  │   NetBox        │  │   Batfish       │             │
│  │   (Docker)      │  │   (Docker)      │             │
│  │   Port: 8000    │  │   Port: 9997    │             │
│  └─────────────────┘  └─────────────────┘             │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │     Network Lab (netlab + containerlab)          │  │
│  ├──────────────────────────────────────────────────┤  │
│  │                                                   │  │
│  │   R1 ─────── R2 ─────── R3    (Routers)          │  │
│  │    │         │         │                         │  │
│  │    └─────────┴─────────┘                         │  │
│  │           SW1            (Switch)                │  │
│  │      │    │    │                                 │  │
│  │     H1   H2   H3         (Hosts)                 │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  Working Directory: ~/netdevops-automation/            │
│  ├── scripts/      (Helper scripts)                    │
│  ├── inventory/    (Device data)                       │
│  ├── configs/      (Generated configs)                 │
│  └── .venv/        (Python packages)                   │
└────────────────────────────────────────────────────────┘
```

**Question for reflection:** 
*Why do we need both NetBox AND network devices? Can't we just practice on real/virtual devices?*

<details>
<summary>Click to see the answer after thinking about it</summary>

**Answer:** NetBox is the "source of truth" - it represents what SHOULD be. Devices show what IS. In production:
- NetBox = Design and intent
- Devices = Reality
- Automation = Makes reality match intent

Without NetBox, you have no authoritative source. With only NetBox, you can't practice actual deployment.
</details>

---

## ⏱️ Time Breakdown for Today

This is how we'll spend 3 hours:

**Hour 1: Foundation (60 min)**
- Create project structure (10 min)
- Set up Python environment (15 min)
- Deploy NetBox (20 min)
- Deploy Batfish (15 min)

**Hour 2: Network Lab (60 min)**
- Understand netlab topology (15 min)
- Deploy network devices (20 min)
- Verify connectivity (15 min)
- Troubleshooting practice (10 min)

**Hour 3: First Automation (60 min)**
- Write first Python script (20 min)
- Connect to router programmatically (20 min)
- Verify complete setup (10 min)
- Reflection and next steps (10 min)

---

## 🚀 Hour 1: Foundation Setup

### Step 1: Create Project Structure (10 min)

**What we're doing:** Creating organized workspace for all automation code.

**Why it matters:** Professional projects need structure. This mirrors real enterprise setups.

```bash
# Navigate to home directory
cd ~

# Create main project directory
mkdir -p netdevops-automation
cd netdevops-automation

# Create organized subdirectories
mkdir -p {inventory,templates,playbooks,tests,configs,scripts,docs,docker}
mkdir -p tests/{batfish,pyats,pytest}
mkdir -p inventory/{host_vars,group_vars}
mkdir -p docker/{netbox,batfish}

# Verify structure
tree -L 2 -d .
# If tree not installed: ls -R
```

**Expected output:**
```
.
├── configs
├── docker
│   ├── batfish
│   └── netbox
├── docs
├── inventory
│   ├── group_vars
│   └── host_vars
├── playbooks
├── scripts
├── templates
└── tests
    ├── batfish
    ├── pyats
    └── pytest
```

**Checkpoint:** 
- Do you see all directories?
- What would you use the `templates/` directory for?
- Where would configuration backups go?

<details>
<summary>Check your understanding</summary>

**Templates:** Jinja2 config templates (coming in Day 3)  
**Configs:** Generated configurations ready to deploy  
**Inventory:** Device data and variables (NetBox data, host info)  
**Tests:** All testing code (Batfish, pyATS, Pytest)  
**Scripts:** Helper utilities we'll build  

If you thought differently, that's okay! We'll use them as we progress.
</details>

---

### Step 2: Python Virtual Environment (15 min)

**What we're doing:** Isolated Python environment with all required packages.

**Why it matters:** Prevents version conflicts. Each project gets its own dependencies.

```bash
# Ensure you're in project root
cd ~/netdevops-automation

# Create virtual environment
python3 -m venv .venv

# Activate it
source .venv/bin/activate

# Your prompt should now show (.venv)
# Verify
which python
# Should show: /home/atr399/netdevops-automation/.venv/bin/python
```

**Create requirements file:**

```bash
cat > requirements.txt << 'EOF'
# Core Automation
ansible==9.10.0
netmiko==4.3.0
napalm==4.1.0

# Network Testing
pyats==24.10
genie==24.10
pybatfish==2024.10.0.dev1

# APIs and Data
requests==2.31.0
pynetbox==7.3.3
jinja2==3.1.3
pyyaml==6.0.1
jsonschema==4.21.1

# Testing
pytest==8.0.0
pytest-html==4.1.1

# Utilities
rich==13.7.0
tabulate==0.9.0
textfsm==1.1.3

# Development
ipython==8.21.0
EOF
```

**Install all packages:**
```bash
# Upgrade pip first
pip install --upgrade pip

# Install everything (this takes 3-5 minutes)
pip install -r requirements.txt

# While it installs, read the "Understanding Dependencies" section below
```

**Understanding Dependencies:**

**Question:** Why do we need `pyats` AND `genie`? Aren't they the same?

<details>
<summary>Think about it, then check</summary>

**pyATS** = Framework (testbeds, connections, test structure)  
**Genie** = Parser library (converts "show ip route" to structured JSON)

Think of it like:
- pyATS = The car (framework to drive around)
- Genie = The GPS (helps you parse/understand where you are)

You need both!
</details>

**Verify installation:**
```bash
# Check key packages
python << 'PYTHON'
import sys

packages = {
    'ansible': 'Ansible',
    'netmiko': 'Netmiko',
    'pyats': 'pyATS',
    'genie': 'Genie',
    'pybatfish': 'Batfish',
    'pynetbox': 'PyNetBox',
    'jinja2': 'Jinja2',
    'pytest': 'Pytest'
}

print("Package Verification:")
print("-" * 40)

for module, name in packages.items():
    try:
        mod = __import__(module)
        version = getattr(mod, '__version__', 'unknown')
        print(f"✅ {name:15} {version}")
    except ImportError:
        print(f"❌ {name:15} NOT INSTALLED")
        sys.exit(1)

print("-" * 40)
print("✅ All packages installed successfully!")
PYTHON
```

**Expected output:**
```
Package Verification:
----------------------------------------
✅ Ansible          9.10.0
✅ Netmiko          4.3.0
✅ pyATS            24.10
✅ Genie            24.10
✅ Batfish          2024.10.0.dev1
✅ PyNetBox         7.3.3
✅ Jinja2           3.1.3
✅ Pytest           8.0.0
----------------------------------------
✅ All packages installed successfully!
```

**Checkpoint:**
- Did all packages install?
- Can you name 3 packages and what they do?
- What would happen if you deactivated the venv and ran python?

---

### Step 3: Deploy NetBox (20 min)

**What we're doing:** NetBox = "Source of Truth" database for network.

**Why NetBox specifically:** 
- Purpose-built for network infrastructure
- IPAM (IP Address Management) built-in
- API-first (everything scriptable)
- Free and open source
- Used by thousands of companies

```bash
cd ~/netdevops-automation/docker/netbox

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  netbox:
    image: netboxcommunity/netbox:v4.0
    depends_on:
      - postgres
      - redis
    env_file: netbox.env
    ports:
      - "8000:8080"
    volumes:
      - netbox-media:/opt/netbox/netbox/media
      - netbox-reports:/opt/netbox/netbox/reports
      - netbox-scripts:/opt/netbox/netbox/scripts
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    env_file: netbox.env
    volumes:
      - netbox-postgres:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - netbox-redis:/data
    restart: unless-stopped

  redis-cache:
    image: redis:7-alpine
    command: redis-server
    restart: unless-stopped

volumes:
  netbox-media:
  netbox-reports:
  netbox-scripts:
  netbox-postgres:
  netbox-redis:
EOF
```

**Understanding Docker Compose:**

**Question:** Why 4 services for NetBox? What does each do?

<details>
<summary>Think about it first</summary>

**netbox:** Main application (web UI + API)  
**postgres:** Database (stores all NetBox data)  
**redis:** Cache (makes it fast)  
**redis-cache:** Additional cache for web UI

This is microservices architecture - each component does one thing well.
</details>

**Create environment file:**
```bash
cat > netbox.env << 'EOF'
# PostgreSQL
POSTGRES_DB=netbox
POSTGRES_USER=netbox
POSTGRES_PASSWORD=netbox123
DB_HOST=postgres
DB_NAME=netbox
DB_USER=netbox
DB_PASSWORD=netbox123

# Redis
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
REDIS_CACHE_HOST=redis-cache
REDIS_CACHE_PORT=6379
REDIS_CACHE_PASSWORD=

# NetBox
SECRET_KEY=r8OwDznj!!dci#P9ghmRfdu1Ysxm0AiPeDCQhKE+N_rClfWNj
ALLOWED_HOSTS=*
CORS_ORIGIN_ALLOW_ALL=True
WEBHOOKS_ENABLED=true

# Superuser (for first login)
SKIP_SUPERUSER=false
SUPERUSER_NAME=admin
SUPERUSER_EMAIL=admin@netdevops.lab
SUPERUSER_PASSWORD=admin
SUPERUSER_API_TOKEN=0123456789abcdef0123456789abcdef01234567
EOF
```

**Start NetBox:**
```bash
# Pull images and start (first time takes 2-3 minutes)
docker compose up -d

# Watch the logs (Ctrl+C when you see "Listening at")
docker compose logs -f netbox
```

**What you should see in logs:**
```
netbox-netbox-1  | Listening at: http://0.0.0.0:8080
```

**Test NetBox:**
```bash
# Wait 30 seconds for initialization, then test
sleep 30

# Test web UI (should return HTML)
curl -s http://localhost:8000 | grep -i netbox

# Test API (should return JSON)
curl -s -H "Authorization: Token 0123456789abcdef0123456789abcdef01234567" \
  http://localhost:8000/api/ | python -m json.tool | head -20
```

**Open in browser:**
```bash
# If on Ubuntu desktop
xdg-open http://localhost:8000

# Or manually go to: http://localhost:8000
# Login: admin / admin
```

**Checkpoint Questions:**
1. Can you access NetBox web UI?
2. Try clicking around - what sections do you see?
3. What would you store in the "DCIM" section?

**Troubleshooting:**

<details>
<summary>If NetBox won't start</summary>

```bash
# Check container status
docker compose ps

# All should show "running"
# If not, check logs
docker compose logs netbox --tail 50

# Common issues:
# 1. Port 8000 already in use
#    Fix: sudo lsof -i :8000  (find what's using it)
#    
# 2. Containers keep restarting
#    Fix: docker compose down && docker compose up -d

# Nuclear option (start fresh)
docker compose down -v  # WARNING: deletes all data
docker compose up -d
```
</details>

---

### Step 4: Deploy Batfish (15 min)

**What we're doing:** Batfish = Pre-deployment testing (offline config analysis).

**Why it matters:** Test changes BEFORE touching production. Catches errors that would cause outages.

```bash
cd ~/netdevops-automation/docker/batfish

# Create docker-compose.yml
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  batfish:
    image: batfish/batfish:latest
    ports:
      - "9997:9997"  # Batfish service
      - "9996:9996"  # Batfish coordinator
    volumes:
      - batfish-data:/data
    environment:
      - BATFISH_ACCEPT_TERMS_OF_SERVICE=true
    restart: unless-stopped

volumes:
  batfish-data:
EOF
```

**Start Batfish:**
```bash
docker compose up -d

# Check logs
docker compose logs -f
# Look for: "Listening on port 9997"
```

**Test Batfish:**
```bash
cd ~/netdevops-automation

# Create quick test script
cat > scripts/test_batfish.py << 'EOF'
#!/usr/bin/env python3
"""Test Batfish connection"""

from pybatfish.client.session import Session
from rich.console import Console

console = Console()

try:
    console.print("🔍 Testing Batfish connection...", style="bold blue")
    
    # Connect to Batfish
    bf = Session(host="localhost")
    
    # Try to set a network (this proves connection works)
    bf.set_network("test_network")
    
    console.print("✅ Batfish is ready!", style="bold green")
    console.print(f"   Host: {bf.host}", style="green")
    console.print(f"   Network: {bf.network}", style="green")
    
except Exception as e:
    console.print(f"❌ Batfish connection failed!", style="bold red")
    console.print(f"   Error: {e}", style="red")
    exit(1)
EOF

chmod +x scripts/test_batfish.py

# Run test
python scripts/test_batfish.py
```

**Expected output:**
```
🔍 Testing Batfish connection...
✅ Batfish is ready!
   Host: localhost
   Network: test_network
```

**Understanding Batfish:**

**Question:** If Batfish analyzes configs "offline", how does it know if routing will work?

<details>
<summary>Think about this deeply</summary>

Batfish builds a **network model** from your configs:
1. Reads all configs (routing protocols, ACLs, interfaces)
2. Builds forwarding tables (just like real routers would)
3. Simulates packet forwarding through the model
4. Tests reachability, routing, ACLs

**Key insight:** It's not running the actual routing protocols. It's computing what WOULD happen based on the configs. This is why it's fast and safe - no real traffic!

**Limitation:** Can't test hardware failures, real-world timing, buggy IOS implementations.
</details>

**Checkpoint:**
- ✅ NetBox running on port 8000?
- ✅ Batfish running on port 9997?
- ✅ Both tested successfully?

---

## ⏱️ Hour 2: Network Lab Deployment

### Step 5: Understanding the Topology (15 min)

**Before we create anything, let's understand WHAT we're building and WHY.**

**Our topology:**
```
                 Internet
                    │
         ┌──────────┴──────────┐
         │                     │
      ┌──┴──┐              ┌──┴──┐
      │ R1  │──────────────│ R2  │   Core Layer
      └──┬──┘              └──┬──┘   (Routing)
         │                     │
         │     ┌───────┐       │
         └─────┤  R3   ├───────┘
               └───┬───┘
                   │
            ┌──────┴──────┐
            │     SW1     │          Distribution
            └──┬───┬───┬──┘          (Switching)
               │   │   │
           ┌───┴┐ ┌┴───┐ ┌┴───┐
           │ H1 │ │ H2 │ │ H3 │     Access
           └────┘ └────┘ └────┘     (Hosts)
```

**Design decisions:**

**Question:** Why 3 routers in a triangle? Why not just 2?

<details>
<summary>Think about redundancy</summary>

With 3 routers:
- Any single router can fail → traffic still flows
- Multiple paths = load balancing
- Practice OSPF with multiple neighbors
- Realistic enterprise design

With only 2:
- Single point of failure
- No path diversity
- Too simple for learning
</details>

**Question:** Why separate switch and routers? Why not route everything?

<details>
<summary>Think about network design</summary>

**Routers (R1-R3):**
- Layer 3 (IP routing)
- Run routing protocols (OSPF, BGP)
- Connect different networks
- More expensive, more powerful

**Switch (SW1):**
- Layer 2 (MAC addresses)
- VLANs, trunking
- Connect end hosts
- Cheaper, simpler

This mirrors real enterprise networks:
- Core/Distribution = Routers
- Access = Switches
</details>

**IP Addressing Plan:**

We'll use netlab's automatic assignment, but here's the concept:
```
Loopbacks:     10.0.0.0/24
  - R1: 10.0.0.1/32
  - R2: 10.0.0.2/32
  - R3: 10.0.0.3/32

Point-to-Point Links:  10.1.0.0/16
  - R1-R2: 10.1.0.0/30
  - R2-R3: 10.1.0.4/30
  - R3-R1: 10.1.0.8/30

LAN Segment:   172.16.0.0/16
  - Managed by SW1
  - Hosts get IPs from this range
```

---

### Step 6: Create Topology File (10 min)

```bash
# Create lab directory
mkdir -p ~/netdevops-automation/labs/lab-001-foundation
cd ~/netdevops-automation/labs/lab-001-foundation

# Create topology file
cat > topology.yml << 'EOF'
---
# NetDevOps Foundation Lab - Day 1
# Simple topology for learning automation basics

# Use IOSv by default
defaults:
  device: iosv

# Define our devices
nodes:
  # Core Routers
  r1:
    device: iosv
    module: [ospf]  # Enable OSPF
    
  r2:
    device: iosv
    module: [ospf]
    
  r3:
    device: iosv
    module: [ospf]

  # Access Switch  
  sw1:
    device: iosv

  # End Hosts
  h1:
    device: linux
  h2:
    device: linux
  h3:
    device: linux

# Define connections
links:
  # Core mesh (full connectivity between routers)
  - r1:
    r2:
  - r2:
    r3:
  - r3:
    r1:
    
  # Routers to switch
  - sw1:
    r1:
  - sw1:
    r2:
  - sw1:
    r3:
    
  # Hosts to switch
  - sw1:
    h1:
  - sw1:
    h2:
  - sw1:
    h3:
EOF
```

**Understanding the Topology File:**

This YAML file is declarative. You say WHAT you want, netlab figures out HOW.

**Question:** What would happen if you added `- r1: r3:` under links?

<details>
<summary>Think about it</summary>

You'd create a direct link between R1 and R3 (in addition to the path through R2).

This would:
- Add another interface to R1 and R3
- Create another /30 network
- OSPF would see two paths between R1-R3
- More realistic for redundancy!

Try this later as an exercise!
</details>

---

### Step 7: Deploy the Lab (20 min)

**Activate netlab environment:**
```bash
# Switch to netlab venv
source ~/netlab-venv/bin/activate

# Verify
which netlab
netlab --version
```

**Start the lab:**
```bash
cd ~/netdevops-automation/labs/lab-001-foundation

# This will:
# 1. Create container configs
# 2. Pull container images (first time: 5-10 min)
# 3. Start all devices
# 4. Configure IP addresses
# 5. Configure OSPF
# 6. Run basic tests

netlab up

# Watch the output - it's teaching you what it's doing!
```

**What you'll see:**
```
[CREATED] provider configuration file: Vagrantfile
[CREATED] transformed topology dump in YAML format in netlab.snapshot.yml
[MAPPED] network devices to Ubuntu server containers
[CREATED] lab configuration file: clab.yml
[CREATED] lab inventory: hosts.yml
[CREATED] Ansible configuration file: ansible.cfg

Starting container lab...
[INFO] Containerlab v0.69.3
[INFO] Creating lab
[INFO] Creating docker network 'clab'
[INFO] Creating r1 container
[INFO] Creating r2 container
[INFO] Creating r3 container
[INFO] Creating sw1 container
[INFO] Creating h1 container
[INFO] Creating h2 container
[INFO] Creating h3 container

Configuring devices...
```

**This takes 5-10 minutes first time. While waiting:**

**Activity:** Read the generated files:
```bash
# In another terminal
cd ~/netdevops-automation/labs/lab-001-foundation

# Look at what netlab created
cat clab.yml        # Containerlab config
cat hosts.yml       # Ansible inventory
cat netlab.snapshot.yml  # Full topology with IPs assigned
```

**Question:** Open `hosts.yml`. What's the username and password for routers?

<details>
<summary>Check the file</summary>

Look for `ansible_user` and `ansible_password` in the file.
Typical defaults: admin/admin or cisco/cisco

We'll need these for automation!
</details>

---

### Step 8: Verify Lab (15 min)

**Check lab status:**
```bash
# See all devices
netlab status

# Should show all devices "Up"
```

**Expected output:**
```
┏━━━━━━━┳━━━━━━━━━━┳━━━━━━━━━━━━━━┳━━━━━━━━━┓
┃ Node  ┃ Status   ┃ Image        ┃ Mgmt IP ┃
┡━━━━━━━╇━━━━━━━━━━╇━━━━━━━━━━━━━━╇━━━━━━━━━┩
│ r1    │ Up       │ ciscous/iosv │ ...     │
│ r2    │ Up       │ ciscous/iosv │ ...     │
│ r3    │ Up       │ ciscous/iosv │ ...     │
│ sw1   │ Up       │ ciscous/iosv │ ...     │
│ h1    │ Up       │ alpine       │ ...     │
│ h2    │ Up       │ alpine       │ ...     │
│ h3    │ Up       │ alpine       │ ...     │
└───────┴──────────┴──────────────┴─────────┘
```

**Connect to a router:**
```bash
# Connect to R1
netlab connect r1

# You should see:
# r1#
```

**On the router, try these commands:**
```cisco
# Check interfaces
show ip interface brief

# Check OSPF neighbors (should see R2 and R3)
show ip ospf neighbor

# Check routing table
show ip route

# Exit router
exit
```

**Checkpoint Questions:**
1. How many OSPF neighbors does R1 have? (Should be 2: R2 and R3)
2. Can you ping R2's loopback from R1? Try: `ping 10.0.0.2`
3. What happens if you reload R2? (Don't actually do it, just think)

**Troubleshooting Practice:**

Let's intentionally break something and fix it.

```bash
# Connect to R1
netlab connect r1

# On R1, shut down interface to R2
conf t
interface GigabitEthernet0/1
shutdown
exit
exit

# Check OSPF neighbors
show ip ospf neighbor
# R2 should be gone!

# Check routing
show ip route ospf
# Routes through R2 should disappear, reroute through R3

# Fix it
conf t
interface GigabitEthernet0/1
no shutdown
exit
exit

# Verify
show ip ospf neighbor
# R2 should come back

exit
```

**Question:** What just happened in the network? Draw the packet flow.

<details>
<summary>Explanation</summary>

**Before shutdown:**
- R1 has 2 paths to R2's network
- OSPF uses cost (bandwidth) to pick best path
- Direct link = lower cost

**After shutdown:**
- R1 loses direct path to R2
- OSPF recalculates
- Traffic now goes R1 → R3 → R2
- Slightly slower but still works

**This is why we have redundancy!**
</details>

---

### Step 9: Troubleshooting Practice (10 min)

**Common Issue 1: Can't connect to device**

```bash
# Try to connect
netlab connect r1

# If it fails, check:
netlab status  # Is device up?
docker ps     # Is container running?

# Get container name
docker ps | grep r1

# Connect directly to container
docker exec -it clab-lab-001-foundation-r1 bash

# Inside container, try
vtysh  # Router CLI
```

**Common Issue 2: OSPF neighbors not forming**

```bash
netlab connect r1

# Check OSPF config
show run | section router ospf

# Check interfaces are up
show ip ospf interface brief

# Debug OSPF (careful - lots of output!)
debug ip ospf adj
# Ctrl+C to stop

undebug all
```

**Common Issue 3: Lab won't start**

```bash
# Stop lab
netlab down

# Clean up
docker system prune -f

# Try again
netlab up
```

---

## ⏱️ Hour 3: First Automation

### Step 10: Write Your First Python Script (20 min)

**What we're doing:** Connect to router with Python instead of SSH.

**Why this matters:** This is the foundation of automation. If you can connect programmatically, you can do anything.

```bash
# Go back to main project
cd ~/netdevops-automation

# Activate Python venv
source .venv/bin/activate

# Create first script
cat > scripts/first_connection.py << 'EOF'
#!/usr/bin/env python3
"""
Day 1: First Python Network Automation Script
Connect to router and get version info
"""

from netmiko import ConnectHandler
from rich.console import Console
from rich.table import Table

console = Console()

# Device details
device = {
    'device_type': 'cisco_ios',
    'host': 'clab-lab-001-foundation-r1',
    'username': 'admin',
    'password': 'admin',
    'port': 22,
}

def main():
    console.print("\n🚀 Connecting to router...", style="bold blue")
    
    try:
        # Connect to device
        with ConnectHandler(**device) as conn:
            console.print("✅ Connected successfully!", style="green")
            
            # Get hostname
            hostname = conn.send_command("show run | include hostname")
            console.print(f"📝 Device: {hostname}", style="cyan")
            
            # Get version
            version = conn.send_command("show version | include Version")
            console.print(f"📦 {version}", style="cyan")
            
            # Get interfaces
            console.print("\n📊 Interfaces:", style="bold yellow")
            interfaces = conn.send_command("show ip interface brief")
            console.print(interfaces)
            
            # Get OSPF neighbors
            console.print("\n🔗 OSPF Neighbors:", style="bold yellow")
            neighbors = conn.send_command("show ip ospf neighbor")
            console.print(neighbors)
            
    except Exception as e:
        console.print(f"❌ Connection failed: {e}", style="bold red")
        return False
    
    console.print("\n✅ Script completed successfully!", style="bold green")
    return True

if __name__ == "__main__":
    success = main()
    exit(0 if success else 1)
EOF

chmod +x scripts/first_connection.py
```

**Understanding the Code:**

Let's break down each part:

```python
from netmiko import ConnectHandler
```
**Question:** What is netmiko?

<details>
<summary>Answer</summary>

**Netmiko** = SSH library specifically for network devices
- Handles SSH connection
- Understands network device prompts
- Works with Cisco, Juniper, Arista, etc.
- Multi-vendor support

Alternative: Use `paramiko` (generic SSH) but netmiko is easier for network devices.
</details>

```python
device = {
    'device_type': 'cisco_ios',
    'host': 'clab-lab-001-foundation-r1',
    ...
}
```

**Question:** Why a dictionary instead of separate variables?

<details>
<summary>Think about scalability</summary>

With dictionary:
```python
devices = [device1, device2, device3]  # Easy to loop
for dev in devices:
    connect(**dev)
```

With separate variables:
```python
host1, user1, pass1, host2, user2, pass2...  # Gets messy fast!
```

**This scales.** When you have 500 devices, you'll appreciate this pattern.
</details>

```python
with ConnectHandler(**device) as conn:
```

**Question:** What does `with` do? Why not just `conn = ConnectHandler(...)`?

<details>
<summary>Important concept</summary>

`with` = Context manager
- Automatically connects
- Runs your code
- **Automatically disconnects** (even if error occurs)

Without `with`:
```python
conn = ConnectHandler(**device)
# ... do stuff ...
conn.disconnect()  # MUST remember this!
# If error before this line, connection stays open!
```

**Best practice:** Always use `with` for connections.
</details>

---

### Step 11: Run Your First Automation (10 min)

```bash
# Make sure you're in the right place
cd ~/netdevops-automation
source .venv/bin/activate

# Run the script
python scripts/first_connection.py
```

**Expected output:**
```
🚀 Connecting to router...
✅ Connected successfully!
📝 Device: hostname r1
📦 Cisco IOS Software, IOSv Software...

📊 Interfaces:
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet0/0     10.1.0.1        YES manual up                    up
GigabitEthernet0/1     10.1.0.5        YES manual up                    up
...

🔗 OSPF Neighbors:
Neighbor ID     Pri   State           Dead Time   Address         Interface
10.0.0.2         1   FULL/DR         00:00:34    10.1.0.2        GigabitEthernet0/0
10.0.0.3         1   FULL/BDR        00:00:38    10.1.0.6        GigabitEthernet0/1

✅ Script completed successfully!
```

**If it works - Congratulations!** You just automated your first network task!

**Checkpoint:**
- Did you connect successfully?
- Do you see OSPF neighbors?
- Try modifying the script to get different output

**Challenge Exercise:**

<details>
<summary>Exercise 1: Modify script to show running config</summary>

Add this to the script:
```python
# Get running config (just the interesting parts)
config = conn.send_command("show run | section router ospf")
console.print("\n⚙️ OSPF Configuration:", style="bold yellow")
console.print(config)
```
</details>

<details>
<summary>Exercise 2: Connect to all 3 routers</summary>

```python
devices = [
    {'device_type': 'cisco_ios', 'host': 'clab-lab-001-foundation-r1', 'username': 'admin', 'password': 'admin'},
    {'device_type': 'cisco_ios', 'host': 'clab-lab-001-foundation-r2', 'username': 'admin', 'password': 'admin'},
    {'device_type': 'cisco_ios', 'host': 'clab-lab-001-foundation-r3', 'username': 'admin', 'password': 'admin'},
]

for device in devices:
    console.print(f"\n{'='*50}")
    console.print(f"Connecting to {device['host']}", style="bold blue")
    # ... rest of connection code ...
```
</details>

---

### Step 12: Complete Verification (10 min)

Let's verify EVERYTHING is working:

```bash
cd ~/netdevops-automation

# Create comprehensive verification script
cat > scripts/verify_day1.sh << 'EOF'
#!/bin/bash
# Day 1 Comprehensive Verification

echo "🔍 Day 1 Lab Verification"
echo "=========================="
echo ""

PASS=0
FAIL=0

check() {
    if eval "$2" &>/dev/null; then
        echo "✅ $1"
        ((PASS++))
    else
        echo "❌ $1"
        ((FAIL++))
    fi
}

# Check Docker services
echo "🐳 Docker Services:"
check "NetBox container running" "docker ps | grep -q netbox"
check "Batfish container running" "docker ps | grep -q batfish"

# Check NetBox
echo ""
echo "📦 NetBox:"
check "NetBox web UI accessible" "curl -s http://localhost:8000 | grep -q NetBox"
check "NetBox API accessible" "curl -s -H 'Authorization: Token 0123456789abcdef0123456789abcdef01234567' http://localhost:8000/api/ | grep -q 'circuits'"

# Check Python environment
echo ""
echo "🐍 Python Environment:"
check "Python venv exists" "[ -d .venv ]"
check "Ansible installed" "source .venv/bin/activate && ansible --version"
check "Netmiko installed" "source .venv/bin/activate && python -c 'import netmiko'"
check "pyATS installed" "source .venv/bin/activate && python -c 'import pyats'"

# Check Network Lab
echo ""
echo "🌐 Network Lab:"
check "R1 container running" "docker ps | grep -q clab.*r1"
check "R2 container running" "docker ps | grep -q clab.*r2"
check "R3 container running" "docker ps | grep -q clab.*r3"
check "SW1 container running" "docker ps | grep -q clab.*sw1"

echo ""
echo "=========================="
echo "Results: ✅ $PASS passed | ❌ $FAIL failed"
echo ""

if [ $FAIL -eq 0 ]; then
    echo "🎉 Perfect! Your Day 1 lab is complete!"
    echo ""
    echo "📚 You can now:"
    echo "   • Access NetBox: http://localhost:8000 (admin/admin)"
    echo "   • Connect to routers: netlab connect r1"
    echo "   • Run Python scripts: python scripts/first_connection.py"
    echo ""
    echo "🚀 Ready for Day 2!"
else
    echo "⚠️  Some components need attention."
    echo "Review the failed checks above."
fi
EOF

chmod +x scripts/verify_day1.sh

# Run verification
./scripts/verify_day1.sh
```

---

### Step 13: Reflection (10 min)

**Answer these questions (write them down):**

1. **Conceptual Understanding:**
   - What is the difference between NetBox and the network lab?
   - Why do we use Docker for NetBox/Batfish but containers for routers?
   - What would happen if NetBox went down? Would automation still work?

2. **Technical Skills:**
   - Can you explain what `netmiko.ConnectHandler` does in your own words?
   - Why did we create a virtual environment for Python?
   - What's the advantage of `netlab` over manually creating containers?

3. **Real-World Application:**
   - How would this lab scale to 100 devices? What would change?
   - If you were at work, what would you automate first?
   - What concerns would you have about automating production?

**Compare your answers:**

<details>
<summary>See suggested answers</summary>

1. **Conceptual:**
   - NetBox = Design/intent. Lab = Reality. Automation makes reality match intent.
   - Docker = Easier for simple services. Containerlab = Better for network devices.
   - If NetBox down: Automation couldn't query "what should be", but devices still work.

2. **Technical:**
   - ConnectHandler = SSH client that understands network device CLIs
   - Virtual env = Isolate dependencies, prevent conflicts
   - Netlab = Abstracts complexity, handles IP assignment, protocol config

3. **Real-World:**
   - 100 devices = Same code, larger inventory, parallel execution needed
   - First to automate = Repetitive tasks (backups, audits, simple changes)
   - Concerns = Testing, rollback, change windows, approvals, documentation
</details>

---

## 📝 What You Learned Today

**Infrastructure:**
- ✅ Created organized project structure
- ✅ Set up Python virtual environment with 20+ packages
- ✅ Deployed NetBox (Source of Truth)
- ✅ Deployed Batfish (Pre-deployment testing)
- ✅ Created 7-device network lab with netlab

**Skills:**
- ✅ Navigate Docker containers
- ✅ Read YAML configurations
- ✅ Use netlab commands
- ✅ Connect to devices via SSH and Python
- ✅ Troubleshoot basic network issues

**Concepts:**
- ✅ Source of Truth principle
- ✅ Pre-deployment testing concept
- ✅ Infrastructure as Code basics
- ✅ Network device automation fundamentals

---

## 🎯 Success Checklist

Before moving to Day 2, verify:

- [ ] NetBox accessible at http://localhost:8000
- [ ] Batfish responding (tested with Python)
- [ ] Network lab running (netlab status shows all "Up")
- [ ] Can connect to R1 with: netlab connect r1
- [ ] Can ping between routers
- [ ] OSPF neighbors showing (show ip ospf neighbor)
- [ ] Python script successfully connected to router
- [ ] All verification tests passed
- [ ] You understand WHY we built each component

---

## 🐛 Troubleshooting Guide

**Issue 1: NetBox won't start**
```bash
docker compose -f docker/netbox/docker-compose.yml logs netbox
# Look for errors
# Common: Port 8000 in use
sudo lsof -i :8000
# Kill the process or change NetBox port
```

**Issue 2: Can't connect to router with Python**
```bash
# Check device is running
docker ps | grep r1

# Test SSH manually
ssh admin@clab-lab-001-foundation-r1
# If this fails, Python will too

# Check firewall
sudo ufw status
```

**Issue 3: Netlab fails to start**
```bash
# Check Docker
docker ps

# Check disk space
df -h

# Clean up old containers
docker system prune -a

# Try again
cd ~/netdevops-automation/labs/lab-001-foundation
netlab down
netlab up
```

**Issue 4: OSPF neighbors not forming**
```bash
netlab connect r1

# Check OSPF enabled
show ip ospf

# Check interfaces
show ip ospf interface

# Check network statements
show run | section router ospf

# If missing, netlab should have configured this
# Try: netlab down && netlab up
```

---

## 🚀 Challenge Exercises (Optional)

Want to go deeper? Try these:

**Challenge 1: Add a 4th router**
- Modify topology.yml to add R4
- Connect it to R2 and R3
- Redeploy: netlab down && netlab up
- Verify OSPF with new topology

**Challenge 2: Create a backup script**
```python
# Save running configs from all routers
# Store in ~/netdevops-automation/backups/
# Filename: hostname_YYYYMMDD_HHMMSS.cfg
```

**Challenge 3: Parse structured data**
```python
# Use TextFSM to parse "show ip interface brief"
# Print only interfaces that are "up/up"
# Count how many interfaces each router has
```

**Challenge 4: NetBox API**
```python
# Using pynetbox, add R1, R2, R3 to NetBox
# Create a site called "Lab"
# Bonus: Automatically discover and add devices
```

---

## 📚 What's Next: Day 2 Preview

**Tomorrow you'll learn:**
- Connect to multiple devices efficiently
- Parse command output to structured data
- Make decisions based on device state
- Handle errors gracefully
- Build a more complex automation script

**Preparation:**
- Your Day 1 lab should stay running
- Keep the same Python venv
- Review netmiko documentation: https://pynet.twb-tech.com/blog/netmiko.html

---

## 💾 Save Your Progress

```bash
# Initialize git if not done
cd ~/netdevops-automation
git init

# Add all files
git add .

# Commit
git commit -m "Day 1 Complete: Lab environment setup

- Deployed NetBox and Batfish in Docker
- Created 7-device network lab with netlab
- Set up Python environment with all packages
- Created first automation script
- All verification tests passed

Lab topology:
- 3 routers (R1, R2, R3) in OSPF mesh
- 1 switch (SW1)
- 3 Linux hosts (H1, H2, H3)

Next: Day 2 - Python automation basics
"

# View your commit
git log --oneline
git show HEAD --stat
```

---

## 📊 Day 1 Stats

**Time spent:** ~3 hours  
**Commands executed:** 50+  
**Concepts learned:** 15+  
**Lines of code written:** 100+  
**Containers deployed:** 11  
**Your automation journey:** 1% complete

---

## 🎉 Congratulations!

You've completed Day 1! You now have:

✅ A production-grade lab environment  
✅ All tools installed and verified  
✅ Your first automation script working  
✅ Foundation for building advanced automation  

**This is huge.** Many engineers never get this far. You should be proud.

**Tomorrow:** We build on this foundation with Python automation.

**Rest well. See you on Day 2! 🚀**

---

**Day 1 Complete ✓**

*Last Updated: November 2025*  
*Created for: atr399*  
*Repository: network-automation-journey*
