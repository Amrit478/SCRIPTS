# Xubuntu Monitoring Stack — Setup Runbook

Everything lives in `/opt/monitoring`. Scripts/config files referenced below
were all generated in this conversation — download them again if you don't
have local copies.

## Files used
| File | Goes where |
|---|---|
| `docker-compose.yml` | `/opt/monitoring/docker-compose.yml` |
| `prometheus.yml` | `/opt/monitoring/prometheus.yml` |
| `config.alloy` | `/opt/monitoring/config.alloy` |
| `blackbox.yml` | `/opt/monitoring/blackbox.yml` |
| `cisco.yml`, `fortinet.yml`, `icmp.yml`, `tcp.yml` | `/opt/monitoring/targets/` |
| `monitoring-stack.service` | `/etc/systemd/system/monitoring-stack.service` |
| `static-ip-setup.sh` | run from anywhere (or paste commands directly) |

---

## 1. Lay out the files
```
mkdir -p /opt/monitoring/targets
# copy docker-compose.yml, prometheus.yml, config.alloy, blackbox.yml into /opt/monitoring
# copy cisco.yml, fortinet.yml, icmp.yml, tcp.yml into /opt/monitoring/targets
```

## 2. Validate before starting
```
cd /opt/monitoring
docker compose config
```
This parses everything without starting containers — catches YAML typos early.

## 3. Bring the stack up
```
docker compose up -d
docker compose ps
```
All 8 containers (grafana, prometheus, loki, alloy, snmp-exporter,
blackbox-exporter, node-exporter, goflow2) should show `Up`.

## 4. Bugs hit and fixed along the way
These are already fixed in the current versions of the files, but noted here
in case you rebuild from scratch and hit them again:

- **alloy crash-looped**: `labels = {...}` is not a top-level attribute on
  `loki.source.syslog` / `loki.source.file` in this Alloy version. Fix:
  move `labels` *inside* the `listener { }` block for syslog, and set labels
  as extra keys directly on each `targets` entry for the file source.
- **goflow2 crash-looped (flag error)**: `-metrics.addr` isn't a valid flag
  in this image version — the correct flag is `-addr` (and it already
  defaults to `:8080`, so you can even omit it).
- **goflow2 crash-looped (permission denied)**: the image runs as a
  non-root user by default, but the Docker named volume it writes flow logs
  to was created owned by root. Fix: added `user: "0:0"` to the goflow2
  service in docker-compose.yml so it runs as root and can write to its own
  volume. If you ever hit this fresh, an alternative is
  `docker compose down goflow2 && docker volume rm monitoring_goflow2-logs && docker compose up -d goflow2`
  to force a clean volume.

Diagnostic commands used throughout:
```
docker compose logs <service> --tail=50
docker compose logs <service> | head -20    # for errors hidden above a usage dump
```

## 5. Make the stack survive reboots (systemd)
Create `/etc/systemd/system/monitoring-stack.service`:
```ini
[Unit]
Description=Monitoring stack (Grafana, Prometheus, Loki, Alloy, exporters) via Docker Compose
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/monitoring
ExecStart=/usr/bin/docker compose up -d
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```
(Use `sudo nano <path>` directly — writing to `/etc/systemd/system/` requires
root; a plain `nano` without sudo will fail with "Permission denied".)

```
sudo systemctl daemon-reload
sudo systemctl enable --now monitoring-stack.service
systemctl status monitoring-stack.service    # expect "active (exited)"
```

Verified by an actual reboot: `sudo reboot`, then log back in and run
`docker compose ps` — all 8 containers came up with zero manual intervention.

## 6. Give the host a permanent static IP
Real LAN: `192.168.1.0/24`, AT&T gateway at `192.168.1.254`, host interface
`enp0s31f6`, connection name `netplan-enp0s31f6` (auto-detected).

```bash
CONN_NAME=$(nmcli -g GENERAL.CONNECTION device show enp0s31f6)
echo "Using connection: $CONN_NAME"

sudo nmcli connection modify "$CONN_NAME" ipv4.addresses 192.168.1.130/24 ipv4.gateway 192.168.1.254 ipv4.dns "1.1.1.1 8.8.8.8" ipv4.method manual

sudo nmcli connection up "$CONN_NAME"

ip a show enp0s31f6
ip route
```
Confirmed: `192.168.1.130/24` shows `valid_lft forever` (not dynamic), route
shows `proto static` instead of `proto dhcp`, and it survived a full reboot.

**Note:** if `sudo netplan apply` ever gets run later, it's possible (since
this connection was originally netplan-generated) that it regenerates from
the original YAML and reverts to DHCP. If the IP ever randomly goes dynamic
again, the fix is editing the actual file in `/etc/netplan/` instead of just
the NetworkManager profile.

## 7. Where things stand
- Docker + the whole compose stack: starts on boot automatically via
  `monitoring-stack.service`, no manual `docker compose up` ever needed.
- Xubuntu host: permanently `192.168.1.130` — this is the one IP to put
  into every EVE-NG device (logging host, SNMP server, NTP server, NetFlow/
  IPFIX export destination). Devices themselves can stay on DHCP.
- Still open: bridging EVE-NG's Cloud/pnet network object into the topology,
  configuring the L3 switch/router and L2 switch to point at
  `192.168.1.130`, and adding their real IPs into
  `/opt/monitoring/targets/*.yml` for Prometheus to scrape.
