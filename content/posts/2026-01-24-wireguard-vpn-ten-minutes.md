---
title: "WireGuard VPN in 10 Minutes: Secure Access to Your Homelab"
date: 2026-01-24T09:00:00Z
draft: false
tags:
  - "tutorial"
  - "devops"
  - "security"
---

I've run OpenVPN, IPsec, SSH tunnels, and various cursed iptables hacks to access home services remotely. Then I tried [WireGuard](https://www.wireguard.com/) and felt stupid for not switching sooner. It's faster, simpler, and the config file is like 15 lines. Let me show you.

## Why WireGuard over OpenVPN

Not going to belabor this. Quick comparison:

- **Speed**: WireGuard is significantly faster. It lives in the Linux kernel, uses modern cryptography, and the codebase is ~4,000 lines vs OpenVPN's ~100,000. Less code, fewer bugs, less overhead.
- **Config**: An OpenVPN config file looks like a novel. A WireGuard config is a handful of lines. You'll see.
- **Connection time**: WireGuard connects in under a second. OpenVPN takes 5-15 seconds to negotiate. If you're on mobile and switching between WiFi and cellular, this matters a lot.
- **Battery**: WireGuard is way easier on mobile battery because it only sends packets when there's actual traffic. No keepalive overhead.

OpenVPN isn't bad — it's battle-tested and works everywhere. But for a homelab VPN where you control both ends, WireGuard is the obvious choice now.

## The setup

You need two things: a server (your homelab box or a VPS) and a client (your laptop, phone, whatever). Each side gets a keypair. They exchange public keys. That's the entire trust model.

### Step 1: Install WireGuard

On Ubuntu/Debian (server and client):

```bash
sudo apt update && sudo apt install -y wireguard
```

On macOS (client):

```bash
brew install wireguard-tools
```

Or just grab the WireGuard app from the App Store / Google Play for mobile.

### Step 2: Generate keys

Do this on both the server and the client:

```bash
wg genkey | tee privatekey | wg pubkey > publickey
```

You now have two files: `privatekey` and `publickey`. Guard the private key. Share the public key.

### Step 3: Server config

Create `/etc/wireguard/wg0.conf` on your server:

```ini
[Interface]
# Server's private key
PrivateKey = SERVER_PRIVATE_KEY_HERE
# VPN subnet — pick something that doesn't conflict with your LAN
Address = 10.0.0.1/24
# Port WireGuard listens on
ListenPort = 51820
# NAT: let VPN clients reach your LAN
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# Client's public key
PublicKey = CLIENT_PUBLIC_KEY_HERE
# IP this client gets on the VPN
AllowedIPs = 10.0.0.2/32
```

Replace `eth0` with whatever your server's main network interface is (`ip a` to check).

Enable IP forwarding:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### Step 4: Client config

Create `/etc/wireguard/wg0.conf` on your client (or import into the mobile app):

```ini
[Interface]
# Client's private key
PrivateKey = CLIENT_PRIVATE_KEY_HERE
# Client's IP on the VPN
Address = 10.0.0.2/24
# Use your home DNS (Pi-hole, maybe?)
DNS = 10.0.0.1

[Peer]
# Server's public key
PublicKey = SERVER_PUBLIC_KEY_HERE
# Server's public IP and port
Endpoint = your-server-ip:51820
# Route all traffic through VPN (full tunnel)
# Or use 10.0.0.0/24, 192.168.1.0/24 for split tunnel
AllowedIPs = 0.0.0.0/0
# Keep the connection alive behind NAT
PersistentKeepalive = 25
```

**Split tunnel vs full tunnel**: `AllowedIPs = 0.0.0.0/0` sends ALL traffic through the VPN. If you only want to reach your home network, use something like `AllowedIPs = 10.0.0.0/24, 192.168.1.0/24` instead. Split tunnel is usually what you want for a homelab — you get access to home services without routing your Netflix traffic through your home ISP.

### Step 5: Start it up

On the server:

```bash
sudo systemctl enable wg-quick@wg0
sudo systemctl start wg-quick@wg0
```

On the client:

```bash
sudo wg-quick up wg0
```

Test it:

```bash
ping 10.0.0.1  # Should reach your server
```

That's it. You're connected. The whole thing probably took less than 10 minutes if you weren't reading this article.

### Step 6: Open the firewall

If your server is behind a router (homelab), forward UDP port 51820 to your server's LAN IP. If it's a VPS, make sure your cloud firewall allows UDP 51820 inbound.

```bash
# If using ufw on the server:
sudo ufw allow 51820/udp
```

## Adding more clients

Each new device is just another `[Peer]` block on the server and a new config file on the client. Generate a new keypair, pick the next IP (10.0.0.3, 10.0.0.4, etc.), add the peer, and reload:

```bash
sudo wg syncconf wg0 <(wg-quick strip wg0)
```

No restart needed. The new peer connects immediately.

## The easy button: Tailscale

Okay, real talk. If the above feels like too much, just use [Tailscale](https://tailscale.com/). It's WireGuard underneath, but it handles all the key exchange, NAT traversal, and config for you. Install it, sign in, done.

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Every device on your Tailscale network can reach every other device. No port forwarding. No firewall rules. No config files to manage. Free for up to 100 devices.

So why would you run raw WireGuard instead of Tailscale?

- **You want full control** over the VPN topology and routing
- **You don't want a third party** involved in your key coordination (even though Tailscale's architecture is solid, the coordination server sees your device metadata)
- **You're running a VPN for others** — giving friends/family access to specific services
- **You want a VPN exit node** on a VPS in another country
- **You're learning** — and honestly, this is reason enough

For most homelab users? Tailscale is the right call. It's one of those rare tools that's both easier and better for the common case.

## Hardening tips

A few things to tighten up your raw WireGuard setup:

**Restrict allowed IPs on the server side.** Each peer should only have the specific `/32` IP it's assigned, not a broad subnet. This prevents a compromised client from impersonating other VPN clients.

**Use a non-default port.** Port 51820 is WireGuard's default and gets scanned. Pick something random like 41932. Update `ListenPort` on the server and `Endpoint` on the client.

**Rotate keys occasionally.** There's no built-in key rotation, but regenerating keys every 6-12 months is good hygiene. It's just two commands and a config edit.

**Don't share private keys across devices.** Each device gets its own keypair. If a device is compromised, you revoke only that peer.

## Use case: access your home services from anywhere

Here's my actual setup. I have a mini PC at home running [Nextcloud](https://nextcloud.com/), [Vaultwarden](https://github.com/dani-garcia/vaultwarden), Grafana, and a few other things. All of these listen on localhost or the LAN — nothing is exposed to the public internet. WireGuard lets me reach them from my laptop at a coffee shop or my phone on the train.

My `AllowedIPs` on the client side: `10.0.0.0/24, 192.168.1.0/24`. Split tunnel. Only traffic destined for my home network goes through the VPN. Everything else goes through whatever WiFi I'm on. Fast, secure, and my home IP doesn't appear in random access logs.

## Cheat Sheet

| Item | Description | Link |
|------|-------------|------|
| WireGuard | Fast, modern VPN protocol, lives in the Linux kernel | [wireguard.com](https://www.wireguard.com/) |
| `wg genkey \| tee privatekey \| wg pubkey > publickey` | Generate a WireGuard keypair | — |
| `/etc/wireguard/wg0.conf` | Config file location on Linux | — |
| `wg-quick up wg0` / `wg-quick down wg0` | Start/stop the VPN tunnel | — |
| `systemctl enable wg-quick@wg0` | Auto-start WireGuard on boot | — |
| `wg syncconf wg0 <(wg-quick strip wg0)` | Reload config without restarting (add peers live) | — |
| `AllowedIPs = 0.0.0.0/0` | Full tunnel — all traffic through VPN | — |
| `AllowedIPs = 10.0.0.0/24, 192.168.1.0/24` | Split tunnel — only home traffic through VPN | — |
| `PersistentKeepalive = 25` | Keep connection alive behind NAT (client-side) | — |
| `net.ipv4.ip_forward=1` | Enable IP forwarding on the server for routing | — |
| [Tailscale](https://tailscale.com/) | Managed WireGuard — handles keys, NAT traversal, config | Free for 100 devices |
| **Rule of thumb** | Use Tailscale unless you need full control. Use raw WireGuard when you want to learn or have specific routing needs. | — |
