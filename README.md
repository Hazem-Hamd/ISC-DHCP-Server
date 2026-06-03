# ISC DHCP Server — Setup & Configuration Guide

> **Environment:** Ubuntu Linux (tested on Ubuntu 22.04+)
> **Interface:** `enp0s3`
> **Network:** `192.168.1.0/24`
> **DHCP Server IP:** `192.168.1.100`

---

## Table of Contents

1. [What is DHCP?](#1-what-is-dhcp)
2. [Installation](#2-installation)
3. [Service Management](#3-service-management)
4. [Configure the Listening Interface](#4-configure-the-listening-interface)
5. [Configure dhcpd.conf](#5-configure-dhcpdconf)
6. [Static IP Reservation (Fixed Address)](#6-static-ip-reservation-fixed-address)
7. [Verify the Configuration](#7-verify-the-configuration)
8. [Monitoring & Logs](#8-monitoring--logs)
9. [Windows Client — Check & Renew IP](#9-windows-client--check--renew-ip)
10. [Troubleshooting](#10-troubleshooting)
11. [Quick Reference — Command Cheatsheet](#11-quick-reference--command-cheatsheet)

---

## 1. What is DHCP?

**DHCP (Dynamic Host Configuration Protocol)** automatically assigns IP addresses and other network parameters (gateway, DNS, subnet mask, lease time) to client devices on a network.

**ISC DHCP Server** (`isc-dhcp-server`) is the most widely used open-source DHCP server for Linux.

**Key concepts:**
- **Dynamic lease** — IP is picked from a pool and assigned temporarily.
- **Static reservation** — A specific IP is always assigned to a device based on its MAC address.
- **Lease time** — How long a client holds its assigned IP before needing to renew.
- **Authoritative** — Declares this server as the official DHCP authority for the subnet.

---

## 2. Installation

```bash
# Update package lists
sudo apt update

# Install ISC DHCP Server
sudo apt install isc-dhcp-server -y
```

---

## 3. Service Management

```bash
# Start the service
sudo systemctl start isc-dhcp-server

# Enable auto-start on boot
sudo systemctl enable isc-dhcp-server

# Restart the service (after config changes)
sudo systemctl restart isc-dhcp-server

# Check service status
sudo systemctl status isc-dhcp-server
```

**Expected status output (healthy service):**

```
● isc-dhcp-server.service - ISC DHCP IPv4 server
     Loaded: loaded (/lib/systemd/system/isc-dhcp-server.service; enabled; preset: enabled)
     Active: active (running) since Mon 2026-06-01 12:41:19 UTC; 5s ago
    Main PID: 46580 (dhcpd)
...
Jun 01 12:41:19 dhcpd[46580]: Listening on LPF/enp0s3/08:00:27:e5:86:da/192.168.1.0/24
Jun 01 12:41:19 dhcpd[46580]: Server starting service.
```

---

## 4. Configure the Listening Interface

Edit the interface defaults file to tell the DHCP server which network interface to listen on:

```bash
sudo nano /etc/default/isc-dhcp-server
```

Set the following:

```bash
# Specify the interface for IPv4 DHCP
INTERFACESv4="enp0s3"

# Leave IPv6 empty unless needed
INTERFACESv6=""
```

> **Note:** Replace `enp0s3` with your actual interface name. Run `ip a` to list your interfaces.
> <img width="704" height="367" alt="Screenshot 2026-06-01 153435" src="https://github.com/user-attachments/assets/06e1b8dd-5c6d-4ca4-a3d4-03ecdfe2eca6" />


---

## 5. Configure dhcpd.conf

Edit the main DHCP configuration file:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

> **Tip:** Use `Ctrl+N` in nano to move to the next line quickly.

### Full Example Configuration

```conf
# /etc/dhcp/dhcpd.conf
# Sample configuration for ISC dhcpd

# Attention: If /etc/ltsp/dhcpd.conf exists, it will be used instead.

# -------------------------------------------------------
# Global Options (apply to all subnets unless overridden)
# -------------------------------------------------------

# Uncomment to set a domain name for clients:
# option domain-name "example.org";

# DNS servers assigned to clients
option domain-name-servers 1.1.1.1, 8.8.8.8;

# Subnet mask for all clients
option subnet-mask 255.255.255.0;

# Default gateway
option routers 192.168.1.1;

# Broadcast address
option broadcast-address 192.168.1.255;

# Default lease time: 600 seconds (10 minutes)
default-lease-time 600;

# Maximum lease time: 7200 seconds (2 hours)
max-lease-time 7200;

# Declare this as the authoritative DHCP server for the network.
# Without this, the server will not respond to clients whose
# leases have expired.
authoritative;

# -------------------------------------------------------
# Subnet Declaration
# -------------------------------------------------------

subnet 192.168.1.0 netmask 255.255.255.0 {
    # IP address pool (dynamic range)
    range 192.168.1.10 192.168.1.30;

    option subnet-mask 255.255.255.0;
    option routers 192.168.1.1;
    option broadcast-address 192.168.1.255;

    default-lease-time 600;
    max-lease-time 7200;
}
```

> **After saving**, always restart the service:
> ```bash
> sudo systemctl restart isc-dhcp-server
> ```

---

## 6. Static IP Reservation (Fixed Address)

To assign a fixed IP address to a specific device based on its **MAC address**, add a `host` block inside `dhcpd.conf`:

```conf
# Static reservation for host "X-space"
host X-space {
    hardware ethernet e0:0a:f6:68:76:a9;
    fixed-address 192.168.1.25;
}
```

**Another example — host named "hazem":**

```conf
host hazem {
    hardware ethernet <MAC_ADDRESS_HERE>;
    fixed-address 172.26.0.22;
}
```

> **How to find a device's MAC address:**
> - **Linux:** `ip link show` or `ifconfig`
> - **Windows:** `ipconfig /all` or open **Network Connection Details** (see Section 9)

> **Important:** If you assign a static IP to a host, make sure the `fixed-address` is **outside** the dynamic `range` to avoid conflicts. The server will warn you if the same IP exists in both static and dynamic pools.

---

## 7. Verify the Configuration

Before restarting, you can test the config file for syntax errors:

```bash
# Test dhcpd.conf for errors (dry run)
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
```

If there are no errors, the command exits silently. Any issues will be printed to the terminal.

---

## 8. Monitoring & Logs

### View Active DHCP Leases

```bash
# View the current lease database
cat /var/lib/dhcp/dhcpd.leases
```

### Watch Real-Time Logs

```bash
# View the last entries in syslog
tail /var/log/syslog

# Follow syslog in real time
tail -f /var/log/syslog

# Use journalctl to follow DHCP server logs
journalctl -u isc-dhcp-server -f
```

### Sample Log Output

```
Jun 01 12:50:36 dhcpd[46580]: DHCPDISCOVER from e0:0a:f6:68:76:a9 (X-space) via enp0s3
Jun 01 12:50:36 dhcpd[46580]: DHCPOFFER on 192.168.1.11 to e0:0a:f6:68:76:a9 (X-space) via enp0s3
Jun 01 12:50:36 dhcpd[46580]: DHCPREQUEST for 192.168.1.11 (192.168.1.100) from e0:0a:f6:68:76:a9 via enp0s3
Jun 01 12:50:36 dhcpd[46580]: DHCPACK on 192.168.1.11 to e0:0a:f6:68:76:a9 (X-space) via enp0s3
```

### DHCP Message Types Explained

| Message       | Direction         | Meaning                                           |
|---------------|-------------------|---------------------------------------------------|
| DHCPDISCOVER  | Client → Broadcast | Client is looking for a DHCP server               |
| DHCPOFFER     | Server → Client   | Server offers an available IP address             |
| DHCPREQUEST   | Client → Server   | Client formally requests the offered IP           |
| DHCPACK       | Server → Client   | Server acknowledges and confirms the assignment   |
| DHCPNACK      | Server → Client   | Server rejects the request (IP unavailable/wrong) |
| DHCPRELEASE   | Client → Server   | Client releases its IP back to the pool           |

---

## 9. Windows Client — Check & Renew IP

### Open Network Settings

```
Windows + R  →  type: ncpa.cpl  →  Enter
```
Right-click your network adapter → **Status** → **Details**

### Sample Client Details (after successful DHCP assignment)

| Property             | Value                        |
|----------------------|------------------------------|
| Description          | Realtek RTL8852AE WiFi 6     |
| Physical (MAC)       | E0-0A-F6-68-76-A9            |
| DHCP Enabled         | Yes                          |
| IPv4 Address         | 192.168.1.25                 |
| Subnet Mask          | 255.255.255.0                |
| Default Gateway      | 192.168.1.1                  |
| DHCP Server          | 192.168.1.100                |
| DNS Servers          | 1.1.1.1 / 8.8.8.8            |
| Lease Obtained       | Mon, June 1, 2026 3:56:43 PM |
| Lease Expires        | Mon, June 1, 2026 4:06:42 PM |

### Release and Renew IP (Command Prompt as Admin)

```cmd
ipconfig /release
ipconfig /renew
```

> This forces the Windows client to drop its current IP and request a new one from the DHCP server — useful for testing static reservations.

---

## 10. Troubleshooting

| Problem                                    | Likely Cause                              | Fix                                                                 |
|--------------------------------------------|-------------------------------------------|---------------------------------------------------------------------|
| Service fails to start                     | Syntax error in `dhcpd.conf`              | Run `sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf` to check             |
| Clients not getting IPs                    | Wrong interface in `isc-dhcp-server`      | Check `INTERFACESv4` in `/etc/default/isc-dhcp-server`             |
| Static reservation not working             | IP is also inside the dynamic `range`     | Move `fixed-address` outside the `range` block                     |
| Duplicate lease warning in logs            | Host has both static and dynamic leases   | Remove the IP from `range` or delete the old dynamic lease          |
| "No subnet declaration for interface"      | Subnet block missing or wrong IP          | Ensure `dhcpd.conf` has a `subnet` block matching your interface IP |
| Windows client not renewing                | Lease still valid or firewall blocking    | Run `ipconfig /release` then `ipconfig /renew`                     |

---

## 11. Quick Reference — Command Cheatsheet

```bash
# ── Installation ──────────────────────────────────────────────
sudo apt update
sudo apt install isc-dhcp-server -y

# ── Service Control ───────────────────────────────────────────
sudo systemctl start   isc-dhcp-server
sudo systemctl stop    isc-dhcp-server
sudo systemctl restart isc-dhcp-server
sudo systemctl enable  isc-dhcp-server
sudo systemctl status  isc-dhcp-server
<img width="1063" height="211" alt="Screenshot 2026-06-01 154137" src="https://github.com/user-attachments/assets/0127612f-8ccd-45db-8f08-2f08f4fdf4cc" />

# ── Config Files ──────────────────────────────────────────────
sudo nano /etc/default/isc-dhcp-server   # Set listening interface
sudo nano /etc/dhcp/dhcpd.conf           # Main DHCP configuration

# ── Syntax Check ──────────────────────────────────────────────
sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf

# ── Logs & Monitoring ─────────────────────────────────────────
tail /var/log/syslog
<img width="1301" height="208" alt="Screenshot 2026-06-01 155815" src="https://github.com/user-attachments/assets/645bb9e0-4b27-48a8-b74f-b6b3f3413d2e" />
tail -f /var/log/syslog
journalctl -u isc-dhcp-server -f
cat /var/lib/dhcp/dhcpd.leases

# ── Windows (CMD as Admin) ────────────────────────────────────
ipconfig /all
ipconfig /release
ipconfig /renew
```

---

## Network Summary
<img width="352" height="430" alt="Screenshot 2026-06-01 155707" src="https://github.com/user-attachments/assets/8fb0a5ec-3986-4e6d-97b0-6bc8bdc25982" />

```
Server IP  :  192.168.1.100
Subnet     :  192.168.1.0 / 255.255.255.0
Gateway    :  192.168.1.1
DNS        :  1.1.1.1  /  8.8.8.8
DHCP Pool  :  192.168.1.10  →  192.168.1.30
Broadcast  :  192.168.1.255
Interface  :  enp0s3

Static Reservation:
  Host     : X-space
  MAC      : e0:0a:f6:68:76:a9
  Fixed IP : 192.168.1.25
```

---

*This guide was created based on a live lab setup of ISC DHCP Server on Ubuntu Linux with a Windows client for validation.*
