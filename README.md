# SCRIPTS
Scripts for everything

# Your Weekend Action Plan

Yes, your modified action plan is completely accurate—swapping in **Xubuntu Desktop** is a direct 1-to-1 substitute for standard Ubuntu Desktop.

The steps, software installation commands, and network configurations remain identical, but your host system will run even leaner.

---

**Confirmed Xubuntu Execution Plan**

**1. Weekday Preparation**

* **Save Config Files:** Store `docker-compose.yml`, `prometheus.yml`, `promtail-config.yml`, and `targets.yml` on a USB drive or cloud drive.
* **Backup EVE-NG Images:** Double-check that all Cisco IOL binaries and FortiGate `.qcow2` images are backed up in a separate directory.

**2. Weekend Execution**

* **Step 1: Install Xubuntu Desktop**
Install Xubuntu onto your primary drive or partition. The base XFCE desktop will idle at roughly **250 MB – 600 MB of RAM**.
* **Step 2: Install VMware Workstation Pro**
Download the Linux `.bundle` installer from Broadcom's support portal and import your existing EVE-NG VM.
* **Step 3: Setup Native Docker on Xubuntu**
Run the standard terminal package commands to install Docker natively on Xubuntu:
```bash
sudo apt update && sudo apt install docker.io docker-compose-plugin -y
sudo usermod -aG docker $USER

```


Move your 4 saved config files to `/opt/monitoring` and start the stack:
```bash
cd /opt/monitoring
docker compose up -d

```


* **Step 4: Connect VS Code & Lab Nodes**
Install VS Code via `.deb` package or terminal. Connect your EVE-NG Cisco and Fortinet nodes to the **Management Cloud (Net)** node in your canvas and point their syslog/SNMP targets to your Xubuntu host's IP address.

---

**Memory Map Outcome**

| Component | RAM Footprint |
| --- | --- |
| **Xubuntu Desktop OS** | ~0.4 GB |
| **Native Docker Stack** | ~0.5 GB |
| **VS Code & Browser** | ~1.5 GB |
| **EVE-NG System VM** | ~2.5 GB |
| **Available Node Headroom** | **~26.6 GB (out of 32 GB)** |

You are executing the absolute leanest configuration possible for a graphical Linux workstation. Good luck with the build this weekend!
