# WireGuard VPN — Build Documentation

This document covers what I did to set up a self-hosted WireGuard VPN server on my Proxmox homelab, including the decisions I made, issues I ran into, and how I resolved them.

---

## Environment

- **Host:** Proxmox VE (kernel 6.17.2)
- **Hardware:** Intel 4th Gen (Haswell) business workstation
- **Container OS:** Debian 12
- **Client:** Windows PC (WireGuard for Windows)

---

## Container Creation

I created a privileged LXC container in Proxmox using the web UI. The container had to be **privileged** because WireGuard needs direct access to the host's TUN kernel device (`/dev/net/tun`) to create the encrypted tunnel interface. Unprivileged containers can't access kernel devices by default.

| Setting | Value |
|---------|-------|
| CT ID | 101 |
| Hostname | wireguard |
| OS | Debian 12 |
| Disk | 8GB on local-lvm |
| CPU | 1 core |
| RAM | 512MB |
| Swap | 512MB |
| Network | vmbr0 |
| Privileged | Yes |

> 📸 **Screenshot:** Proxmox UI showing the completed container creation settings before clicking Finish.

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

> 📸 **Screenshot:** Router admin panel showing the DHCP reservation for the WireGuard container.

---

## Installing WireGuard

Inside the container I updated the package list and installed WireGuard and iptables:

```bash
apt update && apt upgrade -y
apt install wireguard iptables -y
```

iptables had to be installed separately — it wasn't included in the base Debian image and WireGuard's PostUp/PostDown rules depend on it for NAT and traffic forwarding. I discovered this when WireGuard failed on first start with `iptables: command not found`.

---

## Generating Server Keys

I generated the server's public/private key pair:

```bash
wg genkey | tee /etc/wireguard/privatekey | wg pubkey > /etc/wireguard/publickey
chmod 600 /etc/wireguard/privatekey
```

The private key was locked down with `chmod 600` so only root can read it. The public key is safe to share and gets distributed to client devices so they can encrypt traffic destined for the server.

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

> 📸 **Screenshot:** Terminal showing `wg show` output with the interface active and listening port confirmed.

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

> 📸 **Screenshot:** WireGuard for Windows showing the tunnel configuration with the interface and peer sections filled in.

---

## Verifying the Connection

After activating the tunnel on Windows I ran a ping test from Command Prompt:

```
ping 10.0.0.1
```

Result:
```
Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
Approximate round trip times: Minimum = 3ms, Maximum = 5ms, Average = 4ms
```

The tunnel was working with 0% packet loss and 3-5ms latency.

> 📸 **Screenshot:** WireGuard Windows app showing the tunnel as Active with transfer data confirming traffic is flowing. Also include the Command Prompt ping result showing 0% packet loss.

---

## Issues Encountered

**iptables not found on first start:**
WireGuard failed immediately on first start because iptables wasn't installed in the base Debian image. Fixed by running `apt install iptables -y` and restarting the service.

**Key format error:**
WireGuard failed with `Key is not the correct length or format` after adding the peer section. The peer's public key placeholder hadn't been replaced with the actual key. Fixed by editing `wg0.conf` and pasting the correct key.

**cpufrequtils not available:**
The `cpufrequtils` package had no installation candidate on Proxmox's repos. Switched to `linux-cpupower` instead which provides the same functionality via `cpupower frequency-set`.

---

## Useful Commands

| Command | Purpose |
|---------|---------|
| `wg show` | Show active WireGuard interfaces and peers |
| `systemctl status wg-quick@wg0` | Check WireGuard service status |
| `systemctl restart wg-quick@wg0` | Restart WireGuard |
| `ip a` | Show all network interfaces and IPs |
| `journalctl -xeu wg-quick@wg0` | View detailed service logs for troubleshooting |

---

## Current Limitations

WireGuard is currently configured for **local network access only**. To enable remote access from outside the home network the following additional steps are needed:

- Configure port forwarding on the router (UDP 51820 → container IP)
- Set up DuckDNS or similar dynamic DNS service to handle the changing home IP
- Update the Windows client endpoint from the local IP to the DuckDNS hostname
