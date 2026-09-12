# Current Log Coverage & Possible Additions

## What you'll actually receive right now, with nothing else added

1. **Syslog** — any device you point at `192.168.1.130` with
   `logging host 192.168.1.130 transport udp port 1514` (or TCP 1515, or
   plain old port 514) will show up as log lines: link up/down events,
   config changes, ACL denies, reboots, whatever severity level you set
   with `logging trap`.
2. **NetFlow + IPFIX** (and sFlow, since goflow2 is listening on 6343 too,
   though you haven't mentioned using it) — one combined stream of decoded
   flow records: source/dest IP, ports, protocol, bytes, packet counts. All
   land in the same `job="netflow"` log stream in Loki.

That's it for actual logs. SNMP, ICMP/TCP checks, host stats, and NTP sync
status are all metrics going to Prometheus, not logs — different pipeline,
shown as graphs/numbers in Grafana rather than log lines.

## What you could add at this point, without touching the network side at all

- **The monitoring stack's own container logs** (Grafana/Prometheus/Loki/
  Alloy/exporters' stdout) — right now if one of them errors, you only see
  it via `docker compose logs`. Adding Alloy's `loki.source.docker`
  component would pull all of that into Grafana too, so you'd see stack
  problems in the same place as everything else.
- **The Xubuntu host's own system logs** (`/var/log/syslog`, journald —
  auth attempts, kernel messages, cron, etc.) — currently the syslog
  listener only receives inbound logs from other devices; the host's own OS
  logs aren't being shipped anywhere. Adding `loki.source.journal` would
  fix that.
- **SNMP traps** — different from what you have now. `snmp-exporter` polls
  devices for stats; it doesn't receive traps devices push out on their own
  (like a sudden interface failure). That needs a separate trap receiver if
  you want it.
- **NTP sync status from the devices themselves** — the gap mentioned
  earlier. Needs a custom SNMP module against Cisco's NTP MIB.
- **Firewall logs**, once you actually add a Fortinet device to the lab —
  same syslog pipeline, just another `logging`/`log-config` pointed at
  `192.168.1.130`.
