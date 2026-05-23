# 🖥️ Linux & Networking Lab Environment

![Platform](https://img.shields.io/badge/Platform-VirtualBox-183A61?style=flat-square&logo=virtualbox)
![OS](https://img.shields.io/badge/OS-Debian-A81D33?style=flat-square&logo=debian)
![Firewall](https://img.shields.io/badge/Firewall-ufw-orange?style=flat-square)
![SSH](https://img.shields.io/badge/SSH-Hardened-green?style=flat-square&logo=openssh)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)

A personal homelab built from scratch in VirtualBox using two Debian VMs — a hardened server and a client — connected via a dual-adapter NAT + Host-only network. This project documents SSH hardening, ufw firewall configuration, and user privilege separation end-to-end.

> **"Infrastructure as documentation"** — every change made to this lab is recorded here.

---

## 📑 Table of Contents

- [Lab Topology](#-lab-topology)
- [VM Specifications](#-vm-specifications)
- [Network Configuration](#-network-configuration)
- [SSH Hardening](#-ssh-hardening)
- [Firewall Rules (ufw)](#-firewall-rules-ufw)
- [User Accounts & Privilege Separation](#-user-accounts--privilege-separation)
- [Step-by-Step Setup Guide](#-step-by-step-setup-guide)
- [Prerequisites](#-prerequisites)

---

## 🗺️ Lab Topology

```
  ┌────────────────────────────────────────────────────┐
  │                   HOST MACHINE                     │
  │                                                    │
  │   ┌─────────────────┐     ┌─────────────────┐      │
  │   │   VM-1: SERVER  │     │   VM-2: CLIENT  │      │
  │   │   (Debian)      │     │   (Debian)      │      │
  │   │                 │     │                 │      │
  │   │ Adapter 1: NAT  │     │ Adapter 1: NAT  │      │
  │   │ (internet)      │     │ (internet)      │      │
  │   │                 │     │                 │      │
  │   │ Adapter 2:      │─────│ Adapter 2:      │      │
  │   │ Host-only       │     │ Host-only       │      │
  │   │ 192.168.56.10   │     │ 192.168.56.20   │      │
  │   └─────────────────┘     └─────────────────┘      │
  │              Host-only Network: vboxnet0           │
  │              Subnet: 192.168.56.0/24               │
  └────────────────────────────────────────────────────┘
```

> ⚠️ Replace IPs above with your actual assigned addresses. Run `ip a` inside each VM to confirm.

---

## 🖥️ VM Specifications

| Property         | VM-1 · Server                  | VM-2 · Client                  |
|------------------|-------------------------------|-------------------------------|
| **Hostname**     | `debian-server`               | `debian-client`               |
| **OS**           | Debian 12 (Bookworm)          | Debian 12 (Bookworm)          |
| **RAM**          | 1024 MB (2048 MB recommended) | 1024 MB (2048 MB recommended) |
| **Storage**      | 20 GB (VDI, dynamically alloc)| 20 GB (VDI, dynamically alloc)|
| **Adapters**     | NAT + Host-only               | NAT + Host-only               |
| **NAT IP**       | 10.0.2.15 (DHCP)              | 10.0.2.15 (DHCP)              |
| **Host-only IP** | 192.168.56.10 (static)        | 192.168.56.20 (static)        |
| **SSH Port**     | `2222`                        | N/A                           |


---

## 🌐 Network Configuration

Two VirtualBox network adapters are attached to each VM:

### Adapter 1 — NAT
- Provides outbound internet access (package installs, updates)
- VMs are **not** directly reachable from host on this adapter
- DHCP-assigned by VirtualBox (`10.0.2.x`)

### Adapter 2 — Host-only (`vboxnet0`)
- Private isolated network between VMs and host
- Used for all SSH sessions and inter-VM communication
- Statically assigned IPs for reliability

### Static IP Setup (Server — `/etc/network/interfaces`)

```bash
# Adapter 1 — NAT (DHCP)
auto enp0s3
iface enp0s3 inet dhcp

# Adapter 2 — Host-only (Static)
auto enp0s8
iface enp0s8 inet static
    address 192.168.56.10
    netmask 255.255.255.0
```

Apply with:
```bash
sudo systemctl restart networking
```

Repeat for the client VM, substituting `192.168.56.20`.

---

## 🔐 SSH Hardening

All hardening is applied to the **server VM** via `/etc/ssh/sshd_config`.

### Changes Made

| Directive              | Default Value | Hardened Value        | Why                                      |
|------------------------|---------------|-----------------------|------------------------------------------|
| `PermitRootLogin`      | `yes`         | `no`                  | Blocks direct root login over SSH        |
| `PasswordAuthentication` `yes`         | `no`                  | Forces key-based auth only               |
| `Port`                 | `22`          | `22222`               | Reduces automated scan noise             |
| `AllowUsers`           | *(not set)*   | `adminuser`           | Whitelists only authorised users         |
| `MaxAuthTries`         | `6`           | `3`                   | Limits brute-force attempts per session  |

### Applying the Config

```bash
# 1. Edit the SSH daemon config
sudo nano /etc/ssh/sshd_config

# 2. Make the changes listed in the table above, then save

# 3. Test config before restarting (prevents lockout)
sudo sshd -t

# 4. Restart SSH service
sudo systemctl restart ssh

# 5. Verify it's listening on the new port
ss -tlnp | grep ssh
```

### Key-Based Authentication Setup

Run the following on the **client VM**:

```bash
# Generate an ED25519 keypair
ssh-keygen -t ed25519 -C "lab-client"

# Copy the public key to the server
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 adminuser@192.168.56.10
```

Test the connection:
```bash
ssh -p 2222 adminuser@192.168.56.10
```

> ✅ If you connect without a password prompt, key-based auth is working.

---

## 🔥 Firewall Rules (ufw)

Configured on the **server VM**. Policy: **deny all inbound, allow all outbound.**

### Rules Applied

```bash
# Set default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH on the custom port only
sudo ufw allow 2222/tcp comment 'SSH hardened port'

# Enable the firewall
sudo ufw enable

# Verify status
sudo ufw status verbose
```

### Expected Output

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)

To                         Action      From
--                         ------      ----
2222/tcp                  ALLOW IN    Anywhere
```

> ⚠️ Always allow your SSH port **before** enabling ufw to avoid locking yourself out.

---

## 👥 User Accounts & Privilege Separation

Two accounts are maintained on the server VM:

| Username      | Role          | sudo Access | SSH Access | Notes                              |
|---------------|---------------|-------------|------------|------------------------------------|
| `adminuser`   | Administrator | Full(`sudo`)| ✅ Yes     | Whitelisted in `AllowUsers`        |
| `stduser`     | Standard      | None        | ❌ No      | Day-to-day unprivileged operations |

### Creating the Accounts

```bash
# Create the admin user and add to sudo group
sudo adduser adminuser
sudo usermod -aG sudo adminuser

# Create the standard user (no sudo)
sudo adduser stduser

# Verify group memberships
groups adminuser
groups stduser
```

### Enforcing SSH Access to Admin Only

In `/etc/ssh/sshd_config`:
```
AllowUsers adminuser
```

This ensures `stduser` cannot SSH into the server even with valid credentials.

---

## 🧭 Step-by-Step Setup Guide

Follow these steps in order to reproduce this lab from scratch.

### Phase 1 — VirtualBox & VM Creation

```
1. Download VirtualBox: https://www.virtualbox.org/
2. Download Debian ISO: https://www.debian.org/download
3. Create VM-1 (Server):
   - Name: debian-server | Type: Linux | Version: Debian (64-bit)
   - RAM: 1024 MB | Storage: 20 GB (VDI, dynamic)
   - Adapter 1: NAT
   - Adapter 2: Host-only Adapter → vboxnet0
4. Install Debian (minimal, no desktop environment)
5. Clone or repeat for VM-2 (Client) → rename to debian-client
```

### Phase 2 — Network Setup

```bash
# On each VM, identify adapter names
ip link show

# Edit /etc/network/interfaces for static host-only IP
sudo nano /etc/network/interfaces

# Restart networking
sudo systemctl restart networking

# Confirm IPs
ip a
```

### Phase 3 — User Accounts

```bash
sudo adduser adminuser && sudo usermod -aG sudo adminuser
sudo adduser stduser
```

### Phase 4 — SSH Hardening

```bash
sudo nano /etc/ssh/sshd_config   # Apply all hardening directives
sudo sshd -t                     # Test config
sudo systemctl restart ssh
```

### Phase 5 — Key-Based Auth

```bash
# On client
ssh-keygen -t ed25519 -C "lab-client"
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 adminuser@192.168.56.10
```

### Phase 6 — Firewall

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 2222/tcp comment 'SSH'
sudo ufw enable
sudo ufw status verbose
```

### Phase 7 — Verify Everything

```bash
# From client — SSH into server
ssh -p 2222 adminuser@192.168.56.10

# From server — check firewall
sudo ufw status verbose

# Check SSH service
sudo systemctl status ssh

# Check listening ports
ss -tlnp
```

---


## ✅ Prerequisites

| Tool / Resource | Version / Link |
|-----------------|---------------|
| VirtualBox | 7.x — [virtualbox.org](https://www.virtualbox.org/) |
| Debian ISO | 13  — [debian.org](https://www.debian.org/download) |
| Host OS | Windows / macOS / Linux |
| Terminal (client) | Any SSH-capable terminal |

---

*Built and documented as part of a hands-on Linux & Networking homelab. Infrastructure as documentation.*
