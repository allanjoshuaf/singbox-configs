# Sing-box Infrastructure Log Analysis

Three log sources covering the full stack; Sing-box client, Sing-box server
(Canada), and Caddy. Combined span: **March 29 → May 8, 2026**.

---

## The story these logs tell

This infrastructure was built from scratch under real constraints; a restrictive
network environment with active DPI, a residential server, no managed hosting.
The logs document the full arc from initial testing to a stable production tunnel.

---

## Phase 1 — Validating the tunnel (March 29)

The earliest entries show Sing-box running in SOCKS5 proxy mode on
`127.0.0.1:1080`; a deliberate first step to confirm that a VLESS + Reality
tunnel could be established at all between Russia and Canada before committing
to a more complex TUN setup.

```
ERROR connection: report handshake success:
  write tcp 127.0.0.1:1080->127.0.0.1:65157: wsasend:
  An existing connection was forcibly closed by the remote host.
```

These errors are not tunnel failures; they are applications closing their
local SOCKS connections after transferring data. The tunnel itself was
working. SOCKS5 mode confirmed end-to-end connectivity.

---

## Phase 2 — Migration to TUN mode (April 12)

On April 12 at 03:14:56, TUN mode replaces SOCKS entirely:

```
INFO inbound/tun[tun-in]: started at tun0
```

This is a meaningful architectural change. SOCKS mode requires each
application to be configured individually; TUN captures all traffic at
the OS network layer, regardless of application. FakeIP takes over DNS
resolution, and no real DNS queries leave the machine unencrypted.

From this point the error profile changes completely. No more SOCKS
connection noise; clean connection logs, process-level routing working,
FakeIP resolving all A queries to `198.18.0.0/15`.

**8 restarts** are recorded across the 40-day client log; consistent with
iterative config tuning rather than instability.

---

## Caddy — 36 restarts, one real finding

The Caddy log reflects the same iterative phase. 36 restarts over the
observation period; each one corresponding to a config change. Recurring
warning early on:

```
"Caddyfile input is not formatted; run 'caddy fmt --overwrite'"
```

Minor and expected during active development.

More significant: intermittent DNS resolution failures when Caddy attempted
to check Let's Encrypt renewal info:

```
"failed updating renewal info from ACME CA":
  "dial tcp: lookup acme-v02.api.letsencrypt.org: no such host"
```

This happened when the Windows server's DNS was transiently broken; not a
Caddy issue. The certificate itself remained valid throughout. Current cert
expiry: **July 9, 2026**. Renewal window opens approximately June 18.

**Action required before June 18:** confirm DNS resolution is stable on the
server so automatic renewal succeeds.

---

## Server-side view (Canada)

**246,974 lines**, April 20 → May 8. Timezone: EDT (−0400).

The server log shows what the tunnel looks like from the exit point.

**Client connections:**
The dominant inbound IP is `xxx.xxx.xxx` with 4,232 authenticated VLESS
connections. Four additional IPs appear with low counts; these are browsers
hitting the Caddy masquerade site directly over HTTPS. Reality correctly
passes them through to Caddy; they see a real website, not a proxy.

**Top destinations proxied for the client:**

```
optimizationguide-pa.googleapis.com  148
ssl.gstatic.com                       147
www.youtube.com                       117
www.google.com                        106
accounts.google.com                    96
www.linkedin.com                       84
mail.google.com                        39
```

Google services and YouTube dominate; consistent with a Russian client
bypassing blocks on Google infrastructure. LinkedIn confirms active job
searching during the observation period.

**Error profile:** `mux connection closed: read frame header: EOF` entries
are clean client disconnections; the connection multiplexer closes when the
client stops. Not an error condition. The i/o timeout errors are P2P peer
probes on unreachable hosts; documented in the client section below.

---

## Client — DNS and routing (stable)

Across all TUN-mode sessions, DNS behavior is consistent and correct:

- All A queries → FakeIP (`198.18.0.0/15`)
- AAAA queries → rejected at rule level
- PTR queries → rejected (`NXDOMAIN`)
- DoH resolution → tunneled through VLESS to `1.1.1.1`

Zero DNS leaks observed across 357,799 client log lines.

Process-based routing functions correctly throughout; Steam, Epic Games,
and Battle.net route direct as configured. Chrome and system services
route through the tunnel.

---

## Known conflicts and fixes

**AdGuard DNS** — when AdGuard's DNS filtering is active simultaneously
with Sing-box TUN mode, DNS resolution conflicts with FakeIP. Known issue;
disable AdGuard DNS interception when Sing-box is running.

**qBittorrent** — was routing through VLESS for the first weeks, generating
high P2P connection volume through the residential server. Added to the
process exclusion list on May 8; confirmed working direct in the same session:

```
# Before
outbound/vless[vless-out]: outbound connection to xxx.xxx.xxx.xxx:xxxx

# After
outbound/direct[direct]: outbound connection to xxx.xxx.xxx.xxx:xxxx
```

---

## Infrastructure summary

| Component | Status | Notes |
|---|---|---|
| VLESS + Reality tunnel | Stable | 40 days production |
| TUN / FakeIP | No leaks observed | All DNS tunneled |
| Caddy masquerade | Operational | Cert valid until July 9 |
| Process routing | Correct | Steam, Epic, Battle.net direct |
| Server (residential CA) | Stable | 4,232 authenticated sessions |
| DNS leak prevention | Confirmed | AAAA + PTR rejected at rule level |

## Log format reference

Each connection line follows this pattern:

```
+TIMEZONE DATE TIME LEVEL [CONNECTION_ID DURATION] component: message
```

- **CONNECTION_ID** — unique identifier per connection (color-coded in terminal)
- **DURATION** — time since connection opened (e.g., `36m17s` = long-lived)
- **0ms** — new connection
- **~300ms** — second entry for same ID = VLESS handshake completed (RTT to Canada)
