# singbox-configs

Production-grade secure tunneling architecture using VLESS + XTLS-Vision + Reality, TUN routing, FakeIP DNS, and Caddy-based TLS-consistent fallback service.

Designed for restrictive network environments where TLS consistency, DNS integrity, routing reliability, and leak prevention matter.

---

## Architecture
```
[Client]
│
│  TUN interface ; all traffic captured
│  FakeIP DNS ; no real DNS leaks
│  IPv6 fully rejected
│
[Sing-box client]
│
│  VLESS + XTLS-Vision + Reality
│  uTLS fingerprint: Chrome
│  Port 443
│
│  Looks like: standard HTTPS to your-domain.com
│
[DPI / Censorship layer]
│
│  Sees: TLS 1.3 handshake to a known domain
│  Reality prevents TLS-in-TLS fingerprinting
│
[Sing-box server ; residential IP]
│  Port 443
│
├─ Legitimate browsers → Caddy (127.0.0.1:8443)
│                         Real website, H2, TLS 1.3
│
└─ Proxy clients → direct outbound
```

## What makes this hard to detect

- **Residential IP** ; no datacenter ASN fingerprint
- **Reality protocol** ; no TLS-in-TLS, genuine TLS 1.3 handshake
- **Steal-oneself** ; masquerade target is your own Caddy server, 
  not a third-party site. SNI and certificate are fully consistent.
- **uTLS Chrome fingerprint** ; TLS ClientHello identical to Chrome
- **XTLS-Vision** ; traffic pattern matches real HTTPS

## Alternative: Nginx as masquerade server

Caddy is used here for its simplicity with automatic TLS 
and native Proxy Protocol support. Nginx works equally well 
but requires a more verbose configuration ; separate stream 
block for SNI routing, manual SSL certificate management 
(e.g. via certbot), and explicit proxy_protocol directives.

## Components

| File | Role |
|---|---|
| `server/config.json` | Sing-box server (inbound VLESS + Reality) |
| `server/Caddyfile` | Caddy masquerade site on 127.0.0.1:8443 |
| `client/config.json` | Sing-box client with TUN + FakeIP |
| `docs/windows-autostart.md` | NSSM service setup for Windows |

## Client routing logic

Traffic is split into three paths:

1. **Direct** ; private IPs, server IP itself, selected 
   game launchers (Steam, Epic, Battle.net, GeForce)
2. **Through proxy** ; antizapret ruleset 
   (domains/IPs)
3. **Default** ; everything else goes through proxy

DNS is handled entirely through the tunnel via DoH to 1.1.1.1.
FakeIP resolves A records locally, real resolution happens 
server-side. PTR and AAAA queries are rejected to prevent leaks.

> **Note on antizapret ruleset:** Omitted intentionally. Since 
> the default route is the proxy, all traffic is tunneled 
> regardless. The ruleset added complexity without benefit 
> in a full-tunnel configuration.

## Requirements

**Server (Windows)**
- Sing-box v1.12+
- Caddy v2.8+
- NSSM (service management)
- Domain with DNS pointing to server IP
- Port 443 open (TCP)
- Port 80 open (TCP, for HTTP redirect)

**Client**
- Sing-box v1.12+
- Root / admin privileges (TUN requires it)

## Setup

1. Generate Reality keypair:
```bash
sing-box generate reality-keypair
```

2. Generate UUID:
```bash
sing-box generate uuid
```

3. Replace all placeholders in configs:
   - `YOUR_UUID`
   - `YOUR_SERVER_IP`
   - `YOUR_PRIVATE_KEY` / `YOUR_PUBLIC_KEY`
   - `YOUR_SHORT_ID`
   - `your-domain.com`

4. Start Caddy first, then Sing-box (see `docs/windows-autostart.md`)

## Security notes

- Server listens on `0.0.0.0:443` ; ensure firewall blocks 
  all other ports
- Caddy binds only to `127.0.0.1` ; never exposed directly
- Proxy Protocol restricted to `127.0.0.1/32`
- No logging of client IPs by design

## License

MIT
