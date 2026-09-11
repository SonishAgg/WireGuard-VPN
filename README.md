# WireGuard-VPN
Self-hosted WireGuard VPN on Proxmox with Windows client configuration and encrypted tunnel verification

# About This Project
This project documents the process of building a self-hosted VPN server from scratch on a Proxmox homelab using WireGuard. The goal was to understand how VPN tunnels work at a fundamental level-including public/private key cryptography, kernel network devices, IP forwarding, and encrypted peer-to-peer communication

WireGuard was chosen over alternatives like OpenVPN because it is build directly into the Linux kernel, significantly lighter weight, and uses modern cryptographic principles.

# What I Learned
- How public/private key pairs work in practice
- How encrypted tunnels are established between peers
- How LXC container isolation works in Proxmox and when privileged access is needed
- How the Linux TUN kernel device enables virtual networking

# Tech Stack
- Proxmox VE - Hypervisor host
- Debian 12 LXC - Container OS
- WireGuard - VPN Protocol

# Container Specs
- OS - Debian 12
- CPU - 1 core
- RAM - 512MB
- Disk - 8GB

# How It Works
WireGuard uses a public/private key pair for each device:
- Every device generates its own key pair
- The public key is shared with peers and used to encrypt data sent to that device
- The private key never leaves the device as its used to decrypt incoming datat
- Devices exchange public keys so they can encrypt traffic for each other
- The actual data flows through an encrypted tunnel neither side can read without the correct private key

# Planned Improvements 
- Set up router port forwarding for external remote access
- Add mobile client configuration
- Explore WireGuard + Pi-hole integration for encrypted ad-blocking

# Setup Guide
See wireguard-setup.md for full installation and configuration guide
