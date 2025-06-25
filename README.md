# 🔐 Building a Real-Time Detection Pipeline with Snort and Splunk (Homelab Edition) - by Silas Binitie

---

## 🧹 Overview

As part of my blue teaming and threat detection journey, I built a fully operational mini-SIEM pipeline in my homelab using **Snort** as the IDS and **Splunk** as the log analysis and visualization engine. This project simulates a real-world SOC (Security Operations Center) setup by detecting live attacks and forwarding the alerts for deeper analysis and dashboarding.

---

## ⚙️ Tools Used

* **Snort (v2)** - Intrusion Detection System
* **Splunk Universal Forwarder** - Log forwarder
* **Splunk Enterprise** - SIEM + Dashboards
* **pfSense** - Virtual firewall/router
* **Ubuntu Server** - Host for Snort + Forwarder
* **Kali Linux** - Red Team VM to simulate attacks
* **VirtualBox** - Lab environment

---

## 🏗️ Architecture

```
[ Kali Linux ] ---> [ pfSense Firewall ] ---> [ Ubuntu Server: Snort + Splunk UF ] ---> [ Splunk Enterprise Server ]
```

* All traffic passes through pfSense LAN.
* Snort monitors the LAN interface for signs of attack.
* Detected events are logged and forwarded to Splunk.

---

## 🛠️ Step-by-Step Implementation

### 1. Setup Networking in VirtualBox

* **pfSense LAN**: `192.168.10.1`
* **Snort Server**: `192.168.10.5`
* **Kali Attacker**: `192.168.10.x`

Configured internal network adapters for proper isolation and routing.

---

### 2. Install Snort on Ubuntu Server

```bash
sudo apt update
sudo apt install snort -y
```

Defined `HOME_NET` as `192.168.10.0/24`

---

### 3. Create Custom Snort Rules

```snort
alert tcp any any -> $HOME_NET 22 (msg:"🚨 SSH Connection Attempt"; sid:1000004; rev:1;)
```

Saved in:

```bash
/etc/snort/rules/local.rules
```

Confirmed inclusion in `snort.conf`:

```conf
include $RULE_PATH/local.rules
```

---

### 4. Run Snort and Generate Alerts

```bash
sudo snort -c /etc/snort/snort.conf -i enp0s8
```

Checked alerts:

```bash
tail -f /var/log/snort/alert
```

---

### 5. Install Splunk Universal Forwarder

Transferred `.deb` to Snort VM:

```bash
sudo dpkg -i splunkforwarder.deb
sudo /opt/splunkforwarder/bin/splunk start --accept-license
```

---

### 6. Configure Forwarder

```bash
sudo /opt/splunkforwarder/bin/splunk add forward-server <splunk_server_ip>:9997
```

Configured monitoring:

```ini
[monitor:///var/log/snort/alert]
disabled = false
sourcetype = snort_alert
index = endpoint
```

Restarted forwarder:

```bash
sudo /opt/splunkforwarder/bin/splunk restart
```

---

### 7. Build Table in Splunk Web

Search:

```spl
index=endpoint sourcetype=snort_alert
| rex field=_raw "sid:(?<sid>\d+);"
| rex field=_raw "msg:\"(?<msg>[^\"]+)\";"
| rex field=_raw "(?<src_ip>\d{1,3}(?:\.\d{1,3}){3}):(?<src_port>\d+) -> (?<dst_ip>\d{1,3}(?:\.\d{1,3}){3}):(?<dst_port>\d+)"
| table _time msg src_ip src_port dst_ip dst_port sid
| sort - _time
```

---

## 🚧 Roadblocks & Fixes

| Problem                          | Solution                                                 |
| -------------------------------- | -------------------------------------------------------- |
| nano not working inside Snort    | Used `vi` instead                                        |
| Snort rules not triggering       | Fixed interface/IP mismatch and re-tested with live data |
| Wrong IP assigned in Snort setup | Corrected `ipvar HOME_NET` to actual subnet              |
| Splunk not receiving alerts      | Fixed `inputs.conf` and restarted Splunk Forwarder       |
| Couldn't install Splunk forwarder via `dpkg` | downloaded  from  splunk.com (the deb file) the shared it throug shared folder to my Ubuntu Server               |
| SSH service missing for test     | Installed OpenSSH + updated pfSense rules                |

---

## 📊 Final Result

* Live SSH alerts triggered from Kali to Snort
* Snort detected and logged connection attempts
* Alerts successfully forwarded to Splunk
* Splunk dashboards show parsed fields like `msg`, `src_ip`, `SID`, etc.

---

## 🎯 Next Goals

* GeoIP dashboards using `iplocation`
* Rules for brute force or port scans
* Simulate RDP, HTTP attacks and log them in Splunk
* Integrate with TheHive or Wazuh

