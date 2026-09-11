# WireGuard VPN — Build Documentation

This document covers what I did to set up a self-hosted WireGuard VPN server on my Proxmox homelab, including the decisions I made, issues I ran into, and how I resolved them.

---

## Environment

- **Host:** Proxmox VE 
- **Hardware:** Intel 4th Gen Dell Optiplex Business Workstation
- **Container OS:** Debian 12
- **Client:** Windows PC (WireGuard for Windows)

---

## Container Creation

I created a privileged LXC container in Proxmox using the web UI. The container had to be **privileged** because WireGuard needs direct access to the host's TUN kernel device (`/dev/net/tun`) to create the encrypted tunnel interface. Unprivileged containers can't access kernel devices by default.


- Hostname | wireguard 
- OS | Debian 12 
- Disk | 8GB on local-lvm 
- CPU | 1 core 
- RAM | 512MB 
- Swap | 512MB 
- Network | vmbr0 
- Privileged | Yes

<img width="851" height="558" alt="Wireguard Container" src="https://github.com/user-attachments/assets/78ffb90f-69b2-4c9f-8d3f-b23e5dbd35c4" />

---

## Enabling TUN Device Access

Before starting the container I added two lines to the container's config on the Proxmox host to give it access to the TUN device:

```bash
echo "lxc.cgroup2.devices.allow: c 10:200 rwm" >> /etc/pve/lxc/101.conf
echo "lxc.mount.entry: /dev/net/tun dev/net/tun none bind,create=file" >> /etc/pve/lxc/101.conf
```

The first line tells the cgroup to allow the container to access the TUN device. The second line mounts it into the container so it can actually use it. Without these two lines WireGuard fails to start.

---

## Static IP Assignment

I set a DHCP reservation on my router using the container's MAC address so it always receives the same local IP. This is important because WireGuard clients need a fixed endpoint to connect to — if the container's IP changed after a router restart the tunnel would break.

To find the MAC address:

```bash
ip link show eth0
```

---

## Installing WireGuard

Inside the container I updated the package list and installed WireGuard and iptables:

```bash
apt update && apt upgrade -y
apt install wireguard iptables -y
```
<img width="429" height="20" alt="Installing iptables" src="https://github.com/user-attachments/assets/d0045105-b595-4f5e-b74c-da780fe2c3b1" />

<img width="387" height="22" alt="Install Wireguard" src="https://github.com/user-attachments/assets/f9d31584-e12c-4aa4-ac3e-9f1ab8ce3455" />

---

## Generating Server Keys

I generated the server's public/private key pair:

```bash
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
chmod 600 /etc/wireguard/privatekey
```

The public key is safe to share and gets distributed to client devices so they can encrypt traffic destined for the server.

---

## Server Configuration

I created the WireGuard interface config at `/etc/wireguard/wg0.conf`:

```ini
[Interface]
PrivateKey = YOUR_PRIVATE_KEY
Address = 10.0.0.1/24
ListenPort = 51820
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```

The PostUp and PostDown rules tell iptables to allow forwarded traffic through the VPN interface and apply NAT so that traffic from connected clients can reach the rest of the home network. These rules are added when WireGuard starts and removed when it stops.

---

## Enabling IP Forwarding

By default Linux doesn't forward packets between network interfaces. I enabled it permanently by adding it to sysctl:

```bash
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sysctl -p
```

Without this, traffic arriving through the VPN tunnel would have no way to reach the rest of the home network.

---

## Starting WireGuard

```bash
systemctl enable wg-quick@wg0
systemctl start wg-quick@wg0
```

I confirmed it was running with:

```bash
wg show
```

Output showed the interface was up, listening on port 51820, with the correct public key loaded.

<img width="568" height="87" alt="Viewing Wireguard" src="https://github.com/user-attachments/assets/ac653612-3db1-4fae-a66f-d0581d35b1dd" />

<img width="765" height="227" alt="Wireguard Active" src="https://github.com/user-attachments/assets/19261c81-9cfa-410f-8315-ae2344fe8c19" />


---

## Adding a Windows PC as a Client

I generated a separate key pair for the Windows PC client:

```bash
mkdir -p /etc/wireguard/clients
wg genkey | tee /etc/wireguard/clients/pc_privatekey | wg pubkey > /etc/wireguard/clients/pc_publickey
chmod 600 /etc/wireguard/clients/pc_privatekey
```

Each client device gets its own key pair. This means if one device is ever compromised I can remove just that peer without affecting others.

I added the PC as a peer in `wg0.conf`:

```ini
[Peer]
PublicKey = PC_PUBLIC_KEY
AllowedIPs = 10.0.0.2/32
```

On the Windows side I installed WireGuard for Windows and configured a tunnel with:

```ini
[Interface]
PrivateKey = PC_PRIVATE_KEY
Address = 10.0.0.2/24
DNS = 8.8.8.8

[Peer]
PublicKey = SERVER_PUBLIC_KEY
Endpoint = SERVER_LOCAL_IP:51820
AllowedIPs = 10.0.0.0/24
```
<img width="494" height="102" alt="Wireguard App Connected with PC" src="https://github.com/user-attachments/assets/0ca743b0-1322-49d3-a447-2c106499b257" />

---

## Issues Encountered

**iptables not found on first start:**
WireGuard failed immediately on first start because iptables wasn't installed in the base Debian image. Fixed by running `apt install iptables -y` and restarting the service.

**Key format error:**
WireGuard failed with `Key is not the correct length or format` after adding the peer section. The peer's public key placeholder hadn't been replaced with the actual key. Fixed by editing `wg0.conf` and pasting the correct key.

---

## Current Limitations

WireGuard is currently configured for **local network access only**. To enable remote access from outside the home network the following additional steps are needed:

- Configure port forwarding on the router 
- Set up DuckDNS or similar dynamic DNS service to handle the changing home IP
- Update the Windows client endpoint from the local IP to the DuckDNS hostname
