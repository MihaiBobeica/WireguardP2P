 # Reliability


## IP of remote device


For this setup to work the public IP of the remote device (i.e. "what is my IP") and the public server should both be static and put in the client's WireGuard configs. If that's not the case, use a DDNS service like [DuckDNS](https://www.duckdns.org/) or host your own and use that domain name instead of IPs. DDNS is generally used when you have inbound connections, but in this case it's also useful for the remote device.


If you use DDNS, ensure a daemon is launched on the remote device (and optionally the public server if its IP is also dynamic) to keep the DNS record fresh whenever a network reset or reboot occurs that would change the IP.


## Roaming


If the client moves between networks (i.e. its IP changes), the P2P connection will be temporarily lost. To ensure a stable connection, keeping the keepalive intervals short (10 seconds) is crucial on both ends. If a client moves, the endpoint to the public server will update due to WireGuard's roaming feature. The public server will record this new endpoint and send it to the remote device which will re‑send a WireGuard handshake packet to the new endpoint. As the client also continuously sends out packets every few seconds, a P2P connection will be re‑established automatically.


The time it takes to re‑establish a connection depends on the poll interval of the Python script and the KeepAlive interval of the WireGuard configs (both for the public server and remote device, on the client config and as set by the Python script). Whichever is the longest is the maximum time it can take to re‑establish a connection, so it's important to keep them all under 10 seconds.


## NAT types

This setup provides high performance and reliability on networks where ordinary outbound UDP Internet connectivity works, but it does not guarantee peer-to-peer connectivity from every kind of network. That is because NAT and firewall behavior is not strictly standardized and multiple implementations exist.


Peer-to-peer connectivity relies on UDP hole punching and requires the following conditions to be met:


1. *Baseline connectivity*

All peers must be able to make outbound UDP connection attempts to arbitrary Internet hosts. Inbound UDP packets are accepted only as replies to previously established outbound traffic, as is typical for stateful firewalls.


2. *Endpoint-independent mapping on the client*

The external (IP, port) tuple observed by the public server and remote peers for the client must remain consistent across different destinations. The external port does not need to match the client’s configured WireGuard ListenPort.


3. *Consistent listen port on the remote peer*

The destination port observed by the client for the remote peer must be the WireGuard ListenPort configured on that peer.


When these conditions are satisfied, direct peer-to-peer connectivity will be established.


Almost all modern networks use stateful firewalls with default-deny inbound policies, meaning that inbound UDP packets are only accepted if they match the full connection tuple of a previously established outbound flow (source IP, source port, destination IP, destination port).


Under the STUN classification model, such filtering behavior corresponds to port-restricted NATs, symmetric NATs, or symmetric firewalls. Less restrictive NAT types (full-cone or restricted-cone NATs) are increasingly rare and mainly found in legacy networks.


Among these modern network types, symmetric NAT is the only common case that prevents reliable peer-to-peer connectivity, because it uses endpoint-dependent mapping and assigns different external source ports for different destinations.


## IPv6


If IPv6 traffic is tunneled inside the WireGuard tunnel, it has no effect on the setup. If IPv6 is used as WireGuard endpoints, this project can still be relevant. Even though IPv6 usually does not use NAT, many networks still block inbound connections; these can be punched through to establish a P2P connection. Because IPv6 does not require NAT, it's better suited for P2P. The project has not been tested with IPv6.


## Hardcoding listening port


It is recommended to hardcode a listening port on the client. This ensures that the client uses the same source port when connecting to both peers (public server and remote device) so the client endpoint matches. Although most implementations keep the randomized source port consistent across multiple peers when `ListenPort` is not specified, it's better to fix it as the WireGuard protocol itself does not explicitly mandate consistency when `ListenPort` is undefined.


The port should be unique across all your clients to prevent port collision resolution if two clients are behind the same NAT/firewall; collisions will prevent P2P from succeeding on both clients simultaneously. Set the `ListenPort` on clients for example to `3600 + peer_index`, so `3601`, `3602`, etc.


## Don't let firewalls you control be an obstacle


First ensure that the Wireguard port is open on the remote device for inbound connections, you do not want an unnecessary firewall here.


In many setups you have a second router (like a cellular modem or a home/tenant router) connected to the ISP/building's CG‑NAT infrastructure — meaning you have double NAT: your router and the ISP's router. In that case it is still useful to add a port forward rule so your router does not block the inbound connection, and in case the client is already on the internal network (see Shared router below) it can directly reach the remote device's WireGuard.


If your local router is doing port randomization like on pfSense, then DISABLE that for all inbound/outbound wireguard connections, as this will break the P2P connectivity. The general idea is to apply the least restrictive firewall as possible for the Wireguard port.


The Wireguard protocol is heavily battle tested and very secure against the public internet and other hostile networks. This means that you do should not care about the security or firewall implementation of your untrusted ISP for the VPN, even if the ISP's firewall gets compromised or abruptly lifted in the future. Remember, an open ISP firewall is the best for your wireguard connectivity.


## Shared (CG-NAT) router


If the client and the remote device share the same NAT/firewall router (for example connected to the same company infrastructure, Wi‑Fi, cellular network, or behind the same CG‑NAT), and this router blocks connections between internal networks, then P2P may fail. It is important to check if direct connections using the local IP of the remote device on the shared network are possible, as many routers do not firewall connections between internal networks explicitly. If this fails, then a regular P2P connection may still succeed even if both the client and remote device share a public IP, because the remote device may use a different listening port than the client and hairpin NAT may allow the port punching to work.


If your client roams to a local network of the remote device (for example the Wi‑Fi of a cellular modem you manage but that modem is behind CG‑NAT of the provider), you can route your tunnel directly to the device without using the ISP infrastructure. This results in better reliability and performance. Two main options exist if you control the local network's router; each has advantages and disadvantages:


1. If you use DDNS as explained above, you can put a static local DNS record in the local router. The client will resolve to the local IP of the device inside the network and connect directly. If the client roams to an external network, it resolves to the public IP and uses P2P. The drawback is that DNS resolution to the IP only takes place when you click *connect* or run `wg-quick up` as the WireGuard kernel module or userspace libraries do not understand DNS in endpoints and only accept IPs; this makes roaming not seamless.


2. A more elegant option is to add a [DNAT](https://en.wikipedia.org/wiki/Network_address_translation#DNAT) rule (advanced port forwarding) in the local router so that connections to the remote device's public IP\:WireGuard port are routed directly to the device instead of being sent out to the cellular network. The problem is that the remote device's public IP may change, which is hard to keep in a DNAT rule; however many router implementations support a hostname in the DNAT rule, so you can still use your DDNS name. Often the IP assigned to your modem only changes when the network resets or the router reboots. Examples: [pfSense port forwards](https://docs.netgate.com/pfsense/en/latest/nat/port-forwards.html) or [Ubiquiti DNAT/SNAT docs](https://help.ui.com/hc/en-us/articles/16437942532759-DNAT-SNAT-and-Masquerading-in-UniFi).


3. Sometimes neither option is needed. If your local router has a public IPv4 directly assigned, hairpin NAT (reaching your external IP from within the network), which is usually enabled by default, will keep it working on the local network with public IPs in the configs. Still ensure that a port forward in your router is added as explained in "Don't let your firewall be the second problem".


In some cases with a shared router that blocks internal connections, the proxied config is the only solution.


## Double persistent keepalive and stale peers


In regular WireGuard to a server with inbound connections, PersistentKeepalive of the recommended 25 seconds is only used if a client needs to receive packets after the tunnel went silent for more than 30 seconds (default UDP timeout). Usually only the client has to send these keepalive packets since the server should not send packets if the client disconnects.


In P2P both outbound connection states are needed on the firewalls of the client and remote device. If the state is purged then all connectivity is lost. If a client roams between networks both peers need to send a packet to the other endpoint to punch a new state, so a PersistentKeepalive is also added by the Python script on the remote device.


To prevent sending packets to clients that have been disconnected for a while, a timer (default 120 seconds since the last WireGuard handshake) will purge the PersistentKeepalive, as WireGuard handshakes are usually every 2 minutes.


## Most important: keep the relayed/proxied fallback


Always keep the relayed/proxied config to the public server ready as a fallback because you know for sure that this will work. This means you never lose access to your remote device, though it might impact performance and bandwidth if traffic is routed through a third server.


## Idea: Symmetric NAT P2P using ICMP hack

This setup does not work with symmetric NAT but an idea based on [pwnat](https://github.com/samyk/pwnat) may work on some networks.


The problem with Symmetric NAT is that the source port on WAN is not consistent between two UDP flows, so the remote device cannot use the discovered source port by the public helper server to punch a hole and brute forcing the ports is impractical.


An idea to discover this port is to send as client a ICMP TTL packet that embeds the attempted IP packet send to the remote device to the public server in the ICMP payload. NAT routers will rewrite the embedded UDP datagram packet of a TTL to the WAN source port and usually do not check the destination IP the TTL packet is sent to. This makes it possible for the server possibly to discover the correct source port of the original UDP flow between the two devices. This does obviously require the client to have a small script to send such packet, but such script would then only be necessary on Symmetric NAT networks before attempting the relay backup.


---


# Security


All traffic is end‑to‑end encrypted with WireGuard between clients and both the public server and the remote device. UDP punch‑hole packets are only sent when authorized WireGuard clients connect. The Python API server used to discover endpoints should **only** listen on the WireGuard interface and **only** accept connections over the WireGuard tunnel from authorized peers; it should not be exposed to the internet. It runs over HTTP, but this is acceptable if the connection between the public server and remote device is already encrypted by WireGuard. To further protect access, a `token.txt` is used to guard the API.


Obviously you should use a trusted hosting provider for the public server, as the public server can access your local network.

