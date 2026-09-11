# WireGuard VPN Setup on Proxmox

A guide for setting up a self-hosted WireGuard VPN server in an LXC container on Proxmox.

## Overview

WireGuard is a modern, fast, and secure VPN protocol built into the Linux kernel. This guide sets it up in an isolated LXC container on Proxmox, keeping it separate from the hypervisor and other VMs.

## Requirements

- Proxmox VE (this guide uses kernel 6.x)
- Debian 12 LXC template
- A router with DHCP reservation support
- WireGuard app installed on client devices

---

## Step 1 — Create the LXC Container in Proxmox

In the Proxmox web UI, click **Create CT** and use these settings:

| Setting | Value |
|---------|-------|
| CT ID | 101 |
| Hostname | wireguard |
| Template | debian-12-standard |
| Disk | 8GB on local-lvm |
| CPU | 1 core |
| Memory | 512MB |
| Swap | 512MB |
| Network | vmbr0, DHCP |
| Unprivileged | **No (must be privileged)** |

Do **not** start the container yet.

---

## Step 2 — Enable TUN Device Access

WireGuard needs access to the host's TUN network device. Run these commands on the **Proxmox host shell**:

```bash
echo "lxc.cgroup2.devices.allow: c 10:200 rwm" >> /etc/pve/lxc/101.conf
echo "lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file" >> /etc/pve/lxc/101.conf
```

Then start the container:

```bash
pct start 101
pct enter 101
```

---

## Step 3 — Set a Static IP

In your router's admin panel, create a **DHCP reservation** for the container's MAC address so it always gets the same local IP.

To find the MAC address inside the container:

```bash
ip link show eth0
```

---

## Step 4 — Install WireGuard

Inside the container:

```bash
apt update && apt upgrade -y
apt install wireguard iptables -y
```

---

## Step 5 — Generate Server Keys

```bash
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
chmod 600 /etc/wireguard/privatekey
```

Note your public and private keys:

```bash
cat /etc/wireguard/privatekey
cat /etc/wireguard/publickey
```

---

## Step 6 — Create the Server Config

```bash
nano /etc/wireguard/wg0.conf
```

Paste the following, replacing `YOUR_PRIVATE_KEY` with your server private key:

```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

---

## Step 7 — Enable IP Forwarding

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

---

## Step 8 — Start WireGuard

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

Verify it's running:

```bash
wg show
```

---

## Step 9 — Make Auto-Tune Persist on Reboot

Create a systemd service for the CPU governor:

```bash
apt install linux-cpupower -y
cpupower frequency-set -g powersave

cat > /etc/systemd/system/cpupower.service << EOF
[Unit]
Description=Set CPU governor to powersave

[Service]
Type=oneshot
ExecStart=/usr/bin/cpupower frequency-set -g powersave

[Install]
WantedBy=multi-user.target
EOF

systemctl enable cpupower.service
```

---

## Adding a Client Device

### Generate Client Keys

```bash
mkdir -p /etc/wireguard/clients
wg genkey | tee /etc/wireguard/clients/client_privatekey | wg pubkey > /etc/wireguard/clients/client_publickey
chmod 600 /etc/wireguard/clients/client_privatekey
```

### Add Client as a Peer on the Server

Edit `/etc/wireguard/wg0.conf` and add:

```ini
[Peer]
PublicKey = CLIENT_PUBLIC_KEY
AllowedIPs = 10.0.0.2/32
```

Restart WireGuard:

```bash
systemctl restart wg-quick@wg0
```

### Windows Client Config

Install WireGuard from [wireguard.com/install](https://www.wireguard.com/install), open it, click **Add Tunnel → Add empty tunnel**, and paste:

```ini
[Interface]
PrivateKey = CLIENT_PRIVATE_KEY
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = YOUR_SERVER_LOCAL_IP:51820
AllowedIPs = 10.0.0.0/24
```

Click **Save** then **Activate**.

### Verify Connection

On the Windows client, open Command Prompt and run:

```
ping 10.0.0.1
```

You should receive replies with low latency confirming the tunnel is working.

---

## Useful Commands

| Command | Purpose |
|---------|---------|
| `wg show` | Show WireGuard status and peers |
| `systemctl status wg-quick@wg0` | Check service status |
| `systemctl restart wg-quick@wg0` | Restart WireGuard |
| `ip a` | Show network interfaces and IPs |

---

## Notes

- WireGuard listens on UDP port **51820** by default
- Each client device needs its own unique key pair
- The container must be **privileged** for WireGuard to access the TUN device
- For remote access outside your home network, use a Dynamic DNS service like DuckDNS paired with port forwarding on your router
