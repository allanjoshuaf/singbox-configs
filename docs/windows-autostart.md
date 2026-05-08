# Windows Auto-start with NSSM

Both `sing-box` and `caddy` run as Windows services using 
[NSSM](https://nssm.cc) (Non-Sucking Service Manager).

## Prerequisites

- [NSSM](https://nssm.cc/download) extracted somewhere on PATH
- Sing-box binary at `C:\singbox\sing-box.exe`
- Caddy binary at `C:\singbox\caddy\caddy.exe`
- Configs in place (see `server/` directory)

## Install Caddy as a service

```powershell
nssm install caddy "C:\singbox\caddy\caddy.exe"
nssm set caddy AppParameters "run --config C:\singbox\caddy\Caddyfile"
nssm set caddy AppDirectory "C:\singbox\caddy"
nssm set caddy DisplayName "Caddy Web Server"
nssm set caddy Start SERVICE_AUTO_START
nssm start caddy
```

## Install Sing-box as a service

```powershell
nssm install sing-box "C:\singbox\sing-box.exe"
nssm set sing-box AppParameters "run -c C:\singbox\config.json"
nssm set sing-box AppDirectory "C:\singbox"
nssm set sing-box DisplayName "Sing-box Proxy"
nssm set sing-box Start SERVICE_AUTO_START
nssm set sing-box AppDependencies caddy
nssm start sing-box
```

## Verify services

```powershell
nssm status caddy
nssm status sing-box
```

## Stop / restart

```powershell
nssm restart sing-box
nssm stop caddy
```

## Start order

Caddy must start before Sing-box (Reality handshake depends on 
Caddy listening on 127.0.0.1:8443). The `AppDependencies` 
directive handles this automatically.