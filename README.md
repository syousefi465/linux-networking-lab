# 🖧 Linux & Networking Lab Environment
![YouTube]([https://youtube.com](https://youtu.be/3nYW67MnOsA?si=lRPSBYC3IDvte8Lg))

A hands-on home lab built with VirtualBox on Windows, running two Debian VMs configured to simulate a real server/client infrastructure. Covers SSH hardening, firewall configuration, user privilege separation, and internal network setup.

---

## 📋 Lab Overview

| Detail | Value |
|---|---|
| Host OS | Windows |
| Hypervisor | VirtualBox |
| Guest OS | Debian (64-bit) |
| Number of VMs | 2 (server + client) |
| Internal Network | `labnet` |

---

## 🗺️ Network Topology

```
┌─────────────────────────┐        Internal Network (labnet)         ┌─────────────────────────┐
│      debian-server      │ ◄──────────────────────────────────────► │      debian-client      │
│   192.168.56.10         │                                          │   192.168.56.20         │
│                         │                                          │                         │
│  Adapter 1: NAT         │                                          │  Adapter 1: NAT         │
│  Adapter 2: labnet      │                                          │  Adapter 2: labnet      │
└─────────────────────────┘                                          └─────────────────────────┘
```

---

## 💻 VM Specifications

### VM 1 — Server (`debian-server`)

| Setting | Value |
|---|---|
| Hostname | `server` |
| IP Address | `192.168.56.10` |
| RAM | 2048 MB |
| Disk | 20 GB (VDI, dynamically allocated) |
| Network Adapter 1 | NAT |
| Network Adapter 2 | Internal Network — `labnet` |
| Installed packages | `openssh-server`, `ufw`, `sudo` |

### VM 2 — Client (`debian-client`)

| Setting | Value |
|---|---|
| Hostname | `client` |
| IP Address | `192.168.56.20` |
| RAM | 2048 MB |
| Disk | 20 GB (VDI, dynamically allocated) |
| Network Adapter 1 | NAT |
| Network Adapter 2 | Internal Network — `labnet` |

---

## 🔐 SSH Hardening

SSH is configured on the server with the following security measures applied to `/etc/ssh/sshd_config`:

| Setting | Value | Reason |
|---|---|---|
| `Port` | `2222` | Avoids default port 22, reduces automated scanning |
| `PermitRootLogin` | `no` | Prevents direct root brute-force attacks |
| `MaxAuthTries` | `3` | Limits failed login attempts before disconnect |
| `PasswordAuthentication` | `no` | Enforces key-based auth only |

### Key-Based Authentication

An ED25519 key pair was generated on the client and the public key was deployed to the server:

```bash
# On client — generate key pair
ssh-keygen -t ed25519 -C "labuser@client"

# On client — copy public key to server
ssh-copy-id -p 2222 labuser@192.168.56.10
```

SSH access from client to server:

```bash
ssh -p 2222 labuser@192.168.56.10
```

---

## 🔥 Firewall Configuration (UFW)

UFW (Uncomplicated Firewall) is configured on the server with a default-deny inbound policy.

```bash
ufw default deny incoming
ufw default allow outgoing
ufw allow 2222/tcp     # Allow SSH on custom port
ufw enable
```

### Active Rules

```
Status: active

To                         Action      From
--                         ------      ----
2222/tcp                   ALLOW       Anywhere
2222/tcp (v6)              ALLOW       Anywhere (v6)
```

---

## 👥 User Accounts & Privilege Separation

| Username | Role | sudo Access |
|---|---|---|
| `root` | System superuser | Full (direct) |
| `labuser` | Admin user | Yes — via `sudo` group |
| `viewer` | Restricted user | No |

### How it was configured

```bash
# Install sudo
apt install sudo -y

# Grant labuser admin privileges
usermod -aG sudo labuser

# Create restricted account
useradd -m -s /bin/bash viewer
passwd viewer
```

**SSH remote login is only permitted for `labuser`** — root login is disabled via `PermitRootLogin no`.

---

## 🌐 Network Configuration

Both VMs use static IPs on the internal network, configured in `/etc/network/interfaces`:

**Server (`debian-server`)**
```
auto enp0s8
iface enp0s8 inet static
  address 192.168.56.10
  netmask 255.255.255.0
```

**Client (`debian-client`)**
```
auto enp0s8
iface enp0s8 inet static
  address 192.168.56.20
  netmask 255.255.255.0
```

Connectivity verified with:
```bash
# From client
ping 192.168.56.10
```

---

## 🧠 Key Concepts Practiced

- **Virtualization** — spinning up isolated Linux environments on a Windows host using VirtualBox
- **Internal networking** — configuring a private network between VMs without exposing to the host or internet
- **SSH hardening** — changing default ports, disabling root login, enforcing key-based authentication
- **Firewall rules** — default-deny policy with targeted allow rules using UFW
- **User privilege separation** — principle of least privilege: users only have the access they need
- **Infrastructure as documentation** — documenting every config decision for reproducibility

---

## 📁 Project Structure

```
linux-networking-lab/
└── README.md       ← this file
```

---

## 🚀 How to Reproduce This Lab

1. Install [VirtualBox](https://www.virtualbox.org/wiki/Downloads) + Extension Pack on Windows
2. Download the [Debian amd64 netinst ISO](https://www.debian.org/download)
3. Create `debian-server` VM (2 GB RAM, 20 GB disk, NAT + Internal Network `labnet`)
4. Install Debian — select only **SSH server** and **standard system utilities**
5. Clone the VM → `debian-client`, reinitialize MAC address
6. Configure static IPs on both VMs via `/etc/network/interfaces`
7. Apply SSH hardening settings to `/etc/ssh/sshd_config` on server
8. Generate and deploy SSH key pair from client to server
9. Configure UFW on server with default-deny + allow port 2222
10. Create user accounts and assign sudo privileges

---

## 📝 Lessons Learned

- Always reinitialize MAC addresses when cloning VMs — duplicate MACs cause silent network failures
- Check your actual interface name with `ip link show` before editing network config — it may not be `enp0s8`
- Apply firewall rules **before** disabling password auth, or you can lock yourself out
- The order of operations matters: network → SSH → firewall → users

---

*Built as part of a self-directed Linux & networking lab curriculum.*
