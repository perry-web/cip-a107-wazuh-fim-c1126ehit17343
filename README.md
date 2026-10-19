# CIP-A107 SOC Lab: Wazuh Deployment, Agent Registration and File Integrity Monitoring

Student: Perry Koko, C1126EHIT17343
Course: CIP-A107 Security Operations Centre (SOC) Essentials
Lab: SOC Lab: Introduction to Wazuh for Threat Detection & Monitoring (instructor Aminu Idris)

This repository holds the evidence for the lab: a single-node Wazuh 4.12.0 server on Ubuntu 20.04, a Windows 11 Pro agent, real-time File Integrity Monitoring (FIM) on one folder, and two create / modify / delete test runs with the alerts they produced.

## 1. What is in this package

| Path | Content |
|---|---|
| `01_Report/` | Technical report (DOCX and PDF), 11 pages: architecture, installation, registration, FIM, tests, alerts, answers to the four questions, challenges, recommendations |
| `02_Evidence_Appendix/` | Evidence Appendix PDF (46 pages): exhibits E01 to E47 with captions and text exports T01 to T13 |
| `03_Evidence_Manifest/` | Manifest and findings matrix (DOCX, PDF) plus CSV copies |
| `04_Logs/` | Text exports T01 to T13 and `Evidence_Log.txt` |
| `05_Screenshots/` | The 47 cropped screenshots and `SHA256SUMS.txt` |
| `PRE_PUSH_CHECKLIST.md` | Checks to run before pushing |

Read the report first, then use the manifest to find the evidence behind any statement.

## 2. Honesty notes (read before marking)

- The administrator password (E27) and the agent key (E42, T02) are **redacted**. They are not stored anywhere in this package.
- Two Inventory screenshots (host name, board serial, installed software) were **left out** for privacy.
- Files named `T02` to `T06`, `T08`, `T09`, `T12` and `T13` were **rebuilt** from console text pasted during the lab; their first lines say so. `T01`, `T07`, `T10`, `T11` are unchanged copies of the original files.
- Run 1 of the test could not write its local log (the `Documents` folder does not exist). Its alerts are valid. Run 2 is the run used for the hash comparison.
- The ransomware detection rule in the report (answer 3) is a **proposal**. It was not deployed or tested.
- Installer and package files (`wazuh-install.sh`, `.deb` files, the agent MSI) are third-party downloads and are not included.

## 3. Environment

| Item | Value |
|---|---|
| Server | Ubuntu 20.04 VM `icdfa-server`, VMware Workstation NAT, 192.168.119.128, 8.6 GB RAM, 4 vCPU, 50 GB disk |
| Wazuh | 4.12.0 all-in-one: manager, indexer, dashboard, Filebeat |
| Agent | Windows 11 Pro 10.0.26200, agent 001 `Windows-SOC-Lab`, host adapter 192.168.119.1 |
| Clocks | Server and Wazuh in UTC; Windows host UTC-7 |

## 4. Commands used and what they do

### 4.1 Wazuh server (Ubuntu)

| Command | What it does and why |
|---|---|
| `curl -sO <installer URL from the manual>` | Downloads the official assisted installer script. `-s` hides progress, `-O` keeps the file name. |
| `sudo bash ./wazuh-install.sh -a -i -o 2>&1 \| tee ~/install.log` | Runs the installer. `-a` installs all components on one machine. `-i` ignores the hardware check (not "interactive"). `-o` overwrites an earlier installation. `2>&1 \| tee` shows the output and saves it to a file at once. |
| `free -h`, `nproc`, `df -h` | Show memory, CPU count and disk space, to test whether the VM was big enough. |
| `fallocate -l 4G /swapfile; chmod 600 /swapfile; mkswap /swapfile; swapon /swapfile` | Creates a 4 GiB swap file, makes it private, formats it as swap and turns it on. |
| `ip -br addr`, `ping -c 3 8.8.8.8` | Show interface addresses and test internet reachability. |
| `sudo dhclient ens33` | Asks the DHCP server for an address on interface `ens33`. |
| `sudo sed -i 's/enp0s3/ens33/' /etc/netplan/*.yaml; sudo netplan apply` | Corrects the interface name in the network configuration and applies it. |
| `/etc/apt/apt.conf.d/99wazuh-net` with `Acquire::ForceIPv4 "true"; Acquire::Retries "10"; Acquire::http::Timeout "60"; Acquire::https::Timeout "60";` | Makes apt use IPv4, retry failed downloads and wait longer. |
| `sudo apt-get install --download-only wazuh-indexer wazuh-manager filebeat wazuh-dashboard` | Downloads packages without installing them, so a later installer run does not depend on a slow connection. |
| `gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import` | Imports the Wazuh repository signing key (the `apt-key` command is deprecated). |
| `systemctl is-active wazuh-manager wazuh-indexer wazuh-dashboard filebeat` | Prints the state of each service. |
| `sudo ss -tlnp` | Lists listening TCP ports and the program that owns them (1514, 1515, 55000, 443, 9200). |
| `journalctl -u wazuh-indexer --no-pager \| tail` | Reads the service log; shows the start timeout. |
| `/etc/systemd/system/wazuh-indexer.service.d/timeout.conf` with `[Service]` and `TimeoutStartSec=900` | Gives the indexer up to 15 minutes to start before systemd gives up. |
| `sudo /var/ossec/bin/manage_agents` | Interactive tool: A adds an agent, E extracts its key, L lists agents. |
| `sudo /var/ossec/bin/agent_control -l` | Lists agents and their connection state. |
| `sudo grep -E "Rule: (550\|553\|554) " /var/ossec/logs/alerts/alerts.log` | Finds the FIM alerts in the manager's own alert file. |

### 4.2 Windows agent (Administrator PowerShell)

| Command | What it does and why |
|---|---|
| `Start-BitsTransfer -Source <url> -Destination <path>` | Downloads the agent MSI with the Windows background transfer service. |
| `msiexec /i wazuh-agent-4.12.0-1.msi /q` | Installs the MSI silently (`/i` install, `/q` quiet). |
| `Start-Service WazuhSvc`, `Get-Service WazuhSvc`, `Restart-Service WazuhSvc` | Start, check and restart the agent service. |
| `Test-NetConnection 192.168.119.128 -Port 1514` | Tests whether the manager's event port is reachable. |
| `Select-String -Path ...\ossec.log -Pattern "Connected to the server"` | Searches the agent log for the connection message. |
| `New-Item -ItemType Directory -Force -Path C:\Users\DynastyX-koko\SensitiveFiles` | Creates the folder to be monitored. |
| `Copy-Item $conf "$conf.bak"` | Keeps a backup of `ossec.conf` before editing. |
| `Compare-Object (Get-Content $conf.bak) (Get-Content $conf)` | Lists the lines that differ; proves that exactly one line was added. |
| `Get-FileHash $f -Algorithm SHA256` | Computes the SHA-256 of the test file for comparison with the alert. |
| `Set-Content`, `Add-Content`, `Remove-Item` | Create, append to and delete the test file. |
| `Tee-Object -FilePath $log -Append` | Shows a line on screen and appends it to the record file. |

The one configuration line added to `ossec.conf` inside `<syscheck>`:

```
<directories realtime="yes" check_all="yes" report_changes="yes">C:\Users\DynastyX-koko\SensitiveFiles</directories>
```

`realtime="yes"` sends events at once; `check_all="yes"` records size, permissions, owner, times and hashes; `report_changes="yes"` stores the text difference. The last two go beyond the manual.

## 5. Key results

- Five installation runs; the fifth finished (22:48:01 UTC, 18 Oct).
- Agent 001 Active; real-time FIM started 08:34:55 local on 19 Oct.
- Rules fired: 554 (level 5, added), 550 (level 7, modified), 553 (level 7, deleted).
- Run 2 alerts at 15:58:10.341, 15:58:55.443, 15:59:40.419 UTC, about 0.4 s after each action. Size (31 then 82 bytes) and SHA-256 values equal the local record.

## 6. How to verify the files

On Windows PowerShell, from the repository folder:

```
Get-FileHash .\05_Screenshots\E27_*.jpg -Algorithm SHA256
```

Compare with the line for the same file in `05_Screenshots/SHA256SUMS.txt` or `03_Evidence_Manifest/screenshots_manifest.csv`. On Linux use `sha256sum -c SHA256SUMS.txt` inside `05_Screenshots`.
