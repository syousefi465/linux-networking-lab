# 🖥️ Linux & Networking Lab Environment

![Platform](https://img.shields.io/badge/Platform-VirtualBox-183A61?style=flat-square&logo=virtualbox)
![OS](https://img.shields.io/badge/OS-Debian-A81D33?style=flat-square&logo=debian)
![Firewall](https://img.shields.io/badge/Firewall-ufw-orange?style=flat-square)
![SSH](https://img.shields.io/badge/SSH-Hardened-green?style=flat-square&logo=openssh)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)

A personal homelab built from scratch in VirtualBox using two Debian VMs — a hardened server and a client — connected via a dual-adapter NAT + Host-only network. This project documents SSH hardening, network isolation, firewall configuration, and key-based authentication.

> **"Infrastructure as documentation"** — every change made to this lab is recorded here.

---

## 📑 Table of Contents

- [Lab Topology](#-lab-topology)
- [VM Specifications](#-vm-specifications)
- [Network Configuration](#-network-configuration)
- [SSH Hardening](#-ssh-hardening)
- [Firewall Rules (ufw)](#-firewall-rules-ufw)
- [User Accounts & Privilege Separation](#-user-accounts--privilege-separation)
- [SSH Key Setup & Verification](#-ssh-key-setup--verification)
- [Step-by-Step Setup Guide](#-step-by-step-setup-guide)
- [Lessons Learned & Gotchas](#-lessons-learned--gotchas)
- [Prerequisites](#-prerequisites)

---

## 🗺️ Lab Topology

```
  ┌────────────────────────────────────────────────────┐
  │                   HOST MACHINE                     │
  │                                                    │
  │   ┌─────────────────┐     ┌─────────────────┐     │
  │   │   VM-1: SERVER  │     │   VM-2: CLIENT  │     │
  │   │   (Debian)      │     │   (Debian)      │     │
  │   │                 │     │                 │     │
  │   │ Adapter 1: NAT  │     │ Adapter 1: NAT  │     │
  │   │ (auto-assign)   │     │ (auto-assign)   │     │
  │   │                 │     │                 │     │
  │   │ Adapter 2:      │─────│ Adapter 2:      │     │
  │   │ Host-only       │     │ Host-only       │     │
  │   │ 192.168.56.10   │     │ 192.168.56.20   │     │
  │   └─────────────────┘     └─────────────────┘     │
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
| **NAT IP**       | DHCP (auto-assigned)          | DHCP (auto-assigned)          |
| **Host-only IP** | 192.168.56.10 (static)        | 192.168.56.20 (static)        |
| **SSH Port**     | `2222` (hardened)             | N/A                           |

> 📝 **NAT IPs**: Both VMs use DHCP on the NAT adapter. VirtualBox will automatically assign unique IPs (typically 10.0.2.15 and 10.0.2.16). Verify with `ip a` after booting.

---

## 🌐 Network Configuration

Two VirtualBox network adapters are attached to each VM:

### Adapter 1 — NAT
- Provides outbound internet access (package installs, updates)
- VMs are **not** directly reachable from host on this adapter
- DHCP-assigned by VirtualBox (typically `10.0.2.x`)
- **Each VM gets a unique IP automatically**

### Adapter 2 — Host-only (`vboxnet0`)
- Private isolated network between VMs and host
- Used for all SSH sessions and inter-VM communication
- Statically assigned IPs for reliability
- No internet access (intentional isolation)

### Static IP Setup (Server — `/etc/network/interfaces`)

```bash
# Adapter 1 — NAT (DHCP, auto-assigned)
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

### Verify Network Setup

After applying configuration, run on each VM:
```bash
ip a
# Expected output:
# enp0s3: inet 10.0.2.X/24 (DHCP-assigned, varies)
# enp0s8: inet 192.168.56.10 or 192.168.56.20 (static)
```

---

## 🔐 SSH Hardening

All hardening is applied to the **server VM** via `/etc/ssh/sshd_config`.

### Changes Made

| Directive              | Default Value | Hardened Value | Why                                      |
|------------------------|---------------|----------------|------------------------------------------|
| `PermitRootLogin`      | `yes`         | `no`           | Blocks direct root login over SSH        |
| `PasswordAuthentication` | `yes`       | `no`           | Forces key-based auth only               |
| `Port`                 | `22`          | `2222`         | Reduces automated scan noise             |
| `AllowUsers`           | *(not set)*   | `adminuser`    | Whitelists only authorised users         |
| `MaxAuthTries`         | `6`           | `3`            | Limits brute-force attempts per session  |
| `X11Forwarding`        | `yes`         | `no`           | Disables X11 (not needed in lab)         |

### Applying the Config

```bash
# 1. Edit the SSH daemon config
sudo nano /etc/ssh/sshd_config

# 2. Make the changes listed in the table above, then save

# 3. Test config BEFORE restarting (prevents lockout)
sudo sshd -t
# Should output: no errors

# 4. Restart SSH service
sudo systemctl restart ssh

# 5. Verify it's listening on the new port
ss -tlnp | grep ssh
# Expected: tcp  LISTEN  0  128  0.0.0.0:2222  0.0.0.0:*
```

> ⚠️ **Critical**: Always test config with `sudo sshd -t` before restarting. A bad config will lock you out.

---

## 👥 User Accounts & Privilege Separation

Two accounts are maintained on the server VM:

| Username      | Role          | sudo Access | SSH Access | Notes                              |
|---------------|---------------|-------------|------------|------------------------------------|
| `adminuser`   | Administrator | Full (`sudo`)| ✅ Yes    | Whitelisted in `AllowUsers`        |
| `stduser`     | Standard      | None        | ❌ No      | Day-to-day unprivileged operations |

### Creating the Accounts

```bash
# Create the admin user and add to sudo group
sudo adduser adminuser
sudo usermod -aG sudo adminuser

# Verify admin user was created
id adminuser
# Expected: uid=1001(adminuser) gid=1001(adminuser) groups=1001(adminuser),27(sudo)

# Create the standard user (no sudo)
sudo adduser stduser

# Verify group memberships
groups adminuser
# Expected: adminuser : adminuser sudo
groups stduser
# Expected: stduser : stduser
```

### Enforcing SSH Access to Admin Only

In `/etc/ssh/sshd_config`, ensure this line exists:
```
AllowUsers adminuser
```

Then restart SSH:
```bash
sudo systemctl restart ssh
```

This ensures `stduser` cannot SSH into the server, even with a valid password or key.

---

## 🔐 SSH Key Setup & Verification

### Generating Keys (Client VM)

```bash
# Generate an ED25519 keypair (modern, secure algorithm)
ssh-keygen -t ed25519 -C "lab-client"

# When prompted:
# - Press Enter to save in default location (~/.ssh/id_ed25519)
# - Enter a passphrase (optional but recommended for lab safety)

# Verify keys were created
ls -la ~/.ssh/
# Expected: id_ed25519 (private key) and id_ed25519.pub (public key)
```

### Setting Correct Permissions (Critical)

```bash
# On CLIENT VM — private key permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
ls -la ~/.ssh/id_ed25519
# Expected: -rw------- 1 user user (600 permissions)

# On SERVER VM — authorized_keys permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
ls -la ~/.ssh/authorized_keys
# Expected: -rw------- 1 adminuser adminuser (600 permissions)
```

> ⚠️ SSH will refuse to use keys with incorrect permissions. This is a common issue!

### Copying Public Key to Server (Client VM)

```bash
# Copy the public key to the server's authorized_keys
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 adminuser@192.168.56.10

# When prompted, enter the adminuser password
# (This is the last time you'll need to use the password)

# Verify the public key was installed
cat ~/.ssh/id_ed25519.pub
# The public key content should start with "ssh-ed25519"
```

### Testing Key-Based Authentication

```bash
# From CLIENT VM, connect to server using the key
ssh -p 2222 adminuser@192.168.56.10

# First connection: you may be prompted about the host key fingerprint
# The authenticity of host '[192.168.56.10]:2222' can't be established.
# ED25519 key fingerprint is SHA256:...
# Are you sure you want to continue connecting (yes/no/[fingerprint])?
# Type 'yes' and press Enter

# Expected result: You connect WITHOUT being prompted for a password
# If successful, you're now logged into the server
```

### Verifying Key is Working

```bash
# From client VM
ssh -v -p 2222 adminuser@192.168.56.10 2>&1 | grep "Offering"
# Expected: "Offering public key"

# If you see "Authentications: password" instead, the key wasn't accepted
# Check permissions on ~/.ssh/id_ed25519 and server's ~/.ssh/authorized_keys
```

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

# ⚠️ CRITICAL: Allow SSH BEFORE enabling ufw, or you'll lock yourself out!
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
2222/tcp                   ALLOW IN    Anywhere
```

### Troubleshooting Firewall Issues

If you accidentally lock yourself out:
```bash
# From VirtualBox console (not SSH):
sudo ufw disable
sudo ufw default deny incoming
sudo ufw allow 2222/tcp
sudo ufw enable
```

---

## 🧭 Step-by-Step Setup Guide

Follow these steps in order to reproduce this lab from scratch.

### Phase 1 — VirtualBox & VM Creation

```
1. Download VirtualBox: https://www.virtualbox.org/
2. Download Debian ISO: https://www.debian.org/download
3. Create VM-1 (Server):
   - Name: debian-server | Type: Linux | Version: Debian (64-bit)
   - RAM: 1024 MB (2048 MB+ recommended for smooth operation)
   - Storage: 20 GB (VDI, dynamic allocation)
   - Adapter 1: NAT (for internet access)
   - Adapter 2: Host-only Adapter → vboxnet0
4. Install Debian (minimal, no desktop environment)
5. Clone or repeat for VM-2 (Client) → rename to debian-client
```

### Phase 2 — Network Setup

**On each VM:**

```bash
# Identify adapter names
ip link show
# Expected output lists enp0s3 and enp0s8 (names may vary)

# Edit network interfaces for static host-only IP
sudo nano /etc/network/interfaces
# Add the static config from "Network Configuration" section above

# Restart networking
sudo systemctl restart networking

# Confirm IPs are assigned
ip a
# Should show:
#   enp0s3: inet 10.0.2.X/24 (DHCP from NAT)
#   enp0s8: inet 192.168.56.10 or 192.168.56.20 (static)
```

### Phase 3 — User Accounts

**On server VM:**

```bash
sudo adduser adminuser && sudo usermod -aG sudo adminuser
sudo adduser stduser
groups adminuser
groups stduser
```

### Phase 4 — SSH Hardening

**On server VM:**

```bash
sudo nano /etc/ssh/sshd_config   # Apply all hardening directives from table above
sudo sshd -t                     # Test config (must show no errors)
sudo systemctl restart ssh       # Restart SSH daemon
ss -tlnp | grep ssh              # Verify listening on port 2222
```

### Phase 5 — Key-Based Auth

**On client VM:**

```bash
# Generate ED25519 keypair
ssh-keygen -t ed25519 -C "lab-client"
# Press Enter twice (no passphrase for this lab, or add one if you prefer)

# Set correct permissions
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519

# Copy public key to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 adminuser@192.168.56.10
# Enter adminuser's password when prompted

# Test connection
ssh -p 2222 adminuser@192.168.56.10
# Should connect WITHOUT asking for password
```

### Phase 6 — Firewall

**On server VM:**

```bash
# Allow SSH BEFORE enabling firewall!
sudo ufw allow 2222/tcp comment 'SSH hardened port'

# Set policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Enable firewall
sudo ufw enable

# Verify
sudo ufw status verbose
```

### Phase 7 — Verify Everything

**From client VM:**

```bash
# SSH into server (should work without password)
ssh -p 2222 adminuser@192.168.56.10

# Once connected to server, verify:
sudo ufw status verbose
sudo systemctl status ssh
ss -tlnp | grep ssh

# Test that stduser cannot SSH in
exit  # Return to client
ssh -p 2222 stduser@192.168.56.10
# Expected: Connection refused (permission denied)
```

---

## 💡 Lessons Learned & Gotchas

| # | Gotcha | What Happened | Fix |
|---|--------|---------------|-----|
| 1 | **ufw locked me out** | Enabled ufw before allowing SSH port | Always run `ufw allow <PORT>` **before** `ufw enable` |
| 2 | **SSH still on port 22** | Edited wrong config file | Confirm you're editing `/etc/ssh/sshd_config`, not `~/.ssh/config` |
| 3 | **Key copy failed** | `ssh-copy-id` used default port 22 | Always pass `-p 2222` with custom SSH port |
| 4 | **Host-only adapter had no IP** | Interface not configured in `/etc/network/interfaces` | Manually add static IP block for `enp0s8`, then `sudo systemctl restart networking` |
| 5 | **stduser could still SSH in** | `AllowUsers` not set in sshd_config | Add `AllowUsers adminuser` and `sudo systemctl restart ssh` |
| 6 | **Permission denied (publickey)** | SSH key permissions wrong (not 600) or `.ssh` not 700 | Run `chmod 700 ~/.ssh && chmod 600 ~/.ssh/id_ed25519` on client, `chmod 600 authorized_keys` on server |
| 7 | **Host key verification prompt** | First SSH connection to new host | Type `yes` when prompted "Are you sure you want to continue connecting". This is normal and secure. |
| 8 | **Both VMs have same IP** | Misunderstanding DHCP on NAT adapter | DHCP automatically assigns unique IPs. Verify with `ip a`. Only static IPs (192.168.56.x) should match your config. |

> 📝 Add your own gotchas here as you discover them — this section grows with your experience.

---

## ✅ Prerequisites

| Tool / Resource | Version / Link |
|-----------------|---------------|
| VirtualBox | 7.x — [virtualbox.org](https://www.virtualbox.org/) |
| Debian ISO | 12 (Bookworm) — [debian.org](https://www.debian.org/download) |
| Host OS | Windows / macOS / Linux |
| Terminal (client) | Any SSH-capable terminal (ssh command-line tool) |
| Text Editor | nano, vi, or similar (for sshd_config editing) |

---

## 📁 Repository Structure

```
linux-networking-lab/
├── README.md          ← This file — comprehensive lab documentation
└── configs/
    ├── sshd_config    ← Hardened SSH daemon config (reference)
    └── ufw-rules.txt  ← ufw rule export (reference)
```

---

## 🔗 Related Resources

- [SSH Best Practices](https://man7.org/linux/man-pages/man5/sshd_config.5.html)
- [ufw Firewall Guide](https://wiki.ubuntu.com/UncomplicatedFirewall)
- [VirtualBox Networking](https://www.virtualbox.org/manual/ch06.html)
- [Debian Network Configuration](https://wiki.debian.org/NetworkConfiguration)

---

*Built and documented as part of a hands-on Linux & Networking homelab. Infrastructure as documentation.*
