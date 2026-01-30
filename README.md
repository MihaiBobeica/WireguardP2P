# WireGuardP2P — Seamless Peer-to-Peer WireGuard Connections Behind NAT


Setting up a WireGuard VPN can be a headache when your devices are behind NATs or firewalls that block incoming connections — a common scenario with networks you don’t control, such as those using CG-NAT (typical for many mobile or ISP setups).


**WireGuardP2P** solves that problem with a simple Python script that enables direct, peer-to-peer WireGuard connections using [UDP hole punching](https://en.wikipedia.org/wiki/UDP_hole_punching).

No complex relay setups, no third-party signaling servers — just a lightweight and reliable way to make your devices talk directly.


In an ideal world where the client has a static IP you can establish a simple direct P2P by hardcoding it [these manual steps](https://github.com/pirate/wireguard-docs?tab=readme-ov-file#NAT-to-NAT-Connections) to establish a P2P connection. But that obvioously breaks down in the real world where clients constantly move between networks — like switching between home Wi-Fi, work Wi-Fi, and cellular — so WireGuard endpoints aren’t stable

**WireGuardP2P automates the P2P:** it dynamically updates the WireGuard configuration on your remote device so you only have to click *Connect* on your client and will dynamically establish a P2P to the correct client IP. You only need a small public server to help bootstrap the conection.


### **How it Works**

1. **Public Server:** Acts as a bootstrap/relay and records the public IP/port of connecting clients.
2. **Remote Device:** Polls the public server for client endpoints and updates its own WireGuard configuration to "punch" a hole back to the client.
3. **Client:** Connects to both the public server (for signaling) and the remote device (for P2P).

---

### **Setup Guide**

#### **1. Public Server (VPS) Config**

This server must have a public IP and port `51820` open.

```ini
[Interface]
Address = 192.168.20.2/24, 192.168.19.2/24
ListenPort = 51820

[Peer] # Remote Device
PublicKey = <REMOTE_DEVICE_PUBLIC_KEY>
AllowedIPs = 0.0.0.0/0
Endpoint = <REMOTE_DEVICE_PUBLIC_IP>:51820
PersistentKeepalive = 10

[Peer] # Client
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = 192.168.20.3/32, 192.168.19.3/32

```

#### **2. Remote Device Config**

The device you want to access, located behind a NAT.

```ini
[Interface]
Address = 192.168.20.1/24
ListenPort = 51820

[Peer] # Public Server
PublicKey = <PUBLIC_SERVER_PUBLIC_KEY>
Endpoint = <PUBLIC_SERVER_IP>:51820
AllowedIPs = 192.168.20.2/32, 192.168.19.0/24
PersistentKeepalive = 10

[Peer] # Client
PublicKey = <CLIENT_PUBLIC_KEY>
AllowedIPs = 192.168.20.3/32

```

#### **3. Client Configs**

Prepare two configs for the client: one for P2P and one for a proxied fallback.

**P2P Config (Primary):**

```ini
[Interface]
Address = 192.168.20.3/24
ListenPort = 36906 # Hardcode a unique port for stability

[Peer] # Public Server
PublicKey = <PUBLIC_SERVER_PUBLIC_KEY>
Endpoint = <PUBLIC_SERVER_IP>:51820
AllowedIPs = 192.168.20.2/32
PersistentKeepalive = 10

[Peer] # Remote Device
PublicKey = <REMOTE_DEVICE_PUBLIC_KEY>
Endpoint = <REMOTE_DEVICE_PUBLIC_IP>:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 10

```

**Proxied Config (Fallback):**
Use this if P2P fails (e.g., on symmetric NAT networks).

```ini
[Interface]
Address = 192.168.19.3/24

[Peer] # Public Server
PublicKey = <PUBLIC_SERVER_PUBLIC_KEY>
Endpoint = <PUBLIC_SERVER_IP>:51820
AllowedIPs = 0.0.0.0/0

```

---

### **Running the Python Scripts**

1. **Generate a Token:** Create a shared secret in `/etc/wg-publisher/token.txt` and `/etc/wg-subscriber/token.txt`.
2. **Public Server:** Run `server_wg_publisher.py`. It serves a small API over the WireGuard interface to share client endpoints.
3. **Remote Device:** Run `device_wg_subscriber.py`. It polls the server API and updates the local WireGuard peer endpoints dynamically.

### **Core Reliability Tips**

* **Persistent Keepalive:** Keep this at 10 seconds to ensure the NAT hole stays open and to support fast roaming.
* **Symmetric NAT:** This setup may fail on Symmetric NAT (common in some corporate networks). Always keep the **Proxied Fallback** config ready.
* **Security:** The API only listens on the internal WireGuard IP and requires a token. All data traffic remains end-to-end encrypted by WireGuard.
