# Linux Security Investigation

## Project Overview

This project documents a structured Linux security investigation performed against a user-controlled Ubuntu 24.04.3 LTS ARM64 virtual machine running under Parallels.

The investigation focused on identifying:

* Users and administrative privileges
* Authentication activity
* Processes and services
* Network exposure
* Persistence mechanisms
* VPN and remote-access configuration
* Security controls
* System errors and potentially suspicious activity

The investigation was performed using native Linux command-line tools and systemd/journal interfaces.

---

# Lab Environment

| Item               | Value                             |
| ------------------ | --------------------------------- |
| Operating System   | Ubuntu 24.04.3 LTS                |
| Architecture       | ARM64                             |
| Kernel             | Linux 6.14.0-27-generic           |
| Virtualization     | Parallels                         |
| Hostname           | `ubuntu-gnu-linux-24-04-3`        |
| Primary User       | `parallels`                       |
| User UID           | `1000`                            |
| Network Interface  | `enp0s5`                          |
| IP Address         | `10.211.55.4/24`                  |
| Loopback           | `127.0.0.1`                       |
| Investigation Type | Authorized home-lab investigation |

---

# Investigation Methodology

The investigation followed seven stages:

1. System Identification and Baseline
2. Users and Privileges
3. Authentication Activity
4. Processes and Services
5. Network Connections
6. Persistence Mechanisms
7. Security Control Assessment

The goal was to collect evidence first and make security conclusions only after correlating multiple observations.

---

# Step 1 — System Identification and Baseline

## Objective

Establish the identity, operating system, network configuration, current user, privileges, and system uptime before investigating security activity.


## Command 1 — Host and Operating System Information

```bash
hostnamectl
```

### What the command does

`hostnamectl` displays information about the Linux system, including:

* Hostname
* Operating system
* Kernel
* Architecture
* Virtualization platform
* Hardware information

### Output

The system reported:

* Hostname: `ubuntu-gnu-linux-24-04-3`
* Ubuntu 24.04.3 LTS
* Linux kernel 6.14.0-27-generic
* ARM64 architecture
* Virtualization: Parallels
* Hardware vendor: Parallels International GmbH

### Assessment

The system was confirmed to be the expected Ubuntu ARM64 virtual machine.

## Command 2 — Network Configuration

```bash
ip addr
```

### What the command does

`ip addr` displays network interfaces, IP addresses, subnet information, and interface status.

### Output

The primary interface was:

```text
enp0s5
```

with:

```text
10.211.55.4/24
```

The system also had the standard loopback interface:

```text
127.0.0.1
```

### Assessment

The expected Parallels private-network address was confirmed.

The system was operating on the `10.211.55.0/24` lab network.

## Command 3 — Current User

```bash
whoami
```

### What the command does

`whoami` displays the username associated with the current shell session.

### Output

```text
parallels
```

### Assessment

The investigation was being performed from the expected `parallels` account.


## Command 4 — User Identity and Groups

```bash
id
```

### What the command does

`id` displays the current user's:

* UID
* Primary GID
* Supplementary groups

### Output

The account was:

```text
uid=1000(parallels)
gid=1000(parallels)
```

Groups included:

```text
sudo
adm
cdrom
dip
plugdev
users
lpadmin
```

### Assessment

The `parallels` account has membership in the `sudo` group and therefore has administrative capabilities.

This is an important baseline privilege finding.


## Command 5 — System Uptime

```bash
uptime
```

### What the command does

`uptime` reports:

* How long the system has been running
* Number of logged-in users
* System load averages

### Output

The system had been running for an extended period with relatively low system load.

### Assessment

No abnormal resource condition was identified from the baseline uptime/load information.


## Step 1 Evidence


### Step 1 Assessment

The host matched the expected lab environment. No baseline anomaly was identified.

---

# Step 2 — Users and Privileges

## Objective

Identify local accounts, administrative groups, sudo privileges, and recent login history.


## Command 1 — Local Accounts

```bash
getent passwd
```

### What the command does

`getent passwd` retrieves account information from the system's configured account database.

The output includes:

* Username
* UID
* GID
* Home directory
* Login shell

### Output

The primary interactive account was:

```text
parallels:x:1000:1000:Parallels:/home/parallels:/bin/bash
```

Many system/service accounts used restricted shells such as:

```text
/usr/sbin/nologin
/bin/false
```

### Assessment

The account structure appeared consistent with a normal Ubuntu installation.

Service accounts using non-login shells reduce the ability to use those accounts for direct interactive login.


## Command 2 — Sudo Group Membership

```bash
getent group sudo
```

### What the command does

This retrieves the membership of the `sudo` group.

### Output

The `sudo` group contained:

```text
parallels
```

### Assessment

The primary account has administrative privileges.

This is expected for a personally controlled virtual machine but is important to document because compromise of the account would provide a path to elevated privileges.


## Command 3 — Sudo Permissions

```bash
sudo -l
```

### What the command does

`sudo -l` lists the commands the current user is authorized to execute with `sudo`.

### Output

The account had:

```text
(ALL : ALL) ALL
```

This means the account can execute commands as other users, including root, through sudo.

Additional sudo configuration included security-related defaults such as:

```text
env_reset
mail_badpass
secure_path
use_pty
```

### Assessment

The account has full administrative capability.

This is expected for the lab administrator account but should be considered a high-value account from a security perspective.


## Command 4 — Recent Login History

```bash
last -n 10
```

### What the command does

`last` reads login records from `wtmp` and displays recent user sessions, reboots, and shutdowns.

### Output

The displayed entries showed expected user sessions and system reboots.

The username appeared as `parallel` in portions of the output, which appears to be formatting/truncation of the `parallels` account rather than evidence of an additional account.

### Assessment

No clearly unknown interactive user was identified in the displayed records.


## Step 2 Evidence



### Step 2 Assessment

The primary account has full administrative privileges. No obvious unauthorized account was identified during the displayed login/account review.

---

# Step 3 — Authentication Activity

## Objective

Investigate failed logins, authentication-related events, sudo activity, SSH activity, and session activity.


## Command 1 — Failed Login Records

```bash
sudo lastb -n 15
```

### What the command does

`lastb` reads the `btmp` database, which records failed login attempts.

### Output

No failed login records were displayed.

The system reported:

```text
btmp begins Fri Sep 11 00:56:43 2026
```

### Assessment

No failed login attempts were observed in the displayed records.

This does not prove that no authentication failures have ever occurred; it means none were returned by this check.


## Command 2 — Authentication and Session Events

```bash
sudo journalctl --since "7 days ago" | grep -Ei "authentication|failed|failure|sudo|session opened|session closed"
```

### What the command does

This searches the system journal from the previous seven days for terms associated with:

* Authentication
* Failed operations
* Sudo
* Session creation
* Session termination

### Output

The output contained numerous normal CRON session events such as:

```text
pam_unix(cron:session): session opened for user root(uid=0) by root(uid=0)
```

and corresponding session closures.

The output also contained repeated:

```text
fwupd-refresh.service
```

failure messages.

### Assessment

The CRON root sessions were consistent with scheduled system activity.

The `fwupd` failures were separated from authentication activity and investigated later as a system-service issue.


## Command 3 — SSH Logs

```bash
sudo journalctl -u ssh --since "7 days ago"
```

### What the command does

This queries the system journal for the SSH service over the previous seven days.

### Output

```text
-- No entries --
```

### Assessment

No SSH service log entries were found during the selected period.

This does not independently prove that SSH is unavailable; it only indicates that the queried journal contained no SSH entries.


## Step 3 Evidence



### Step 3 Assessment

No obvious repeated failed interactive authentication activity was identified.

Normal root CRON sessions were observed.

No SSH journal activity was identified.

---

# Step 4 — Processes and Services

## Objective

Review active processes and running services for unexpected or suspicious execution.


## Command 1 — Highest CPU Processes

```bash
ps aux --sort=-%cpu | head -20
```

### What the command does

`ps aux` displays running processes.

The output is sorted by CPU usage, and:

```text
head -20
```

limits the display to the first 20 entries.

### Output

The highest CPU entry was the `ps` command itself during execution, showing extremely high CPU usage.

Normal desktop processes were also observed, including:

* GNOME Shell
* GNOME Terminal
* GNOME extensions
* Parallels tools
* CUPS
* Avahi

### Assessment

The high CPU value associated with `ps` itself was a measurement artifact caused by the process being sampled while running.

No obviously malicious process was identified.


## Command 2 — Highest Memory Processes

```bash
ps aux --sort=-%mem | head -20
```

### What the command does

This sorts running processes by memory consumption and displays the top 20.

### Output

Expected desktop applications and services were among the highest consumers.

Examples included:

* `gnome-shell`
* GNOME-related processes
* terminal
* `fwupd`
* `snapd`

### Assessment

No obviously suspicious process was identified based on memory usage.


## Command 3 — Running Services

```bash
systemctl --type=service --state=running
```

### What the command does

This lists currently running system-level services managed by systemd.

### Output

Services included:

* `accounts-daemon`
* `avahi-daemon`
* `cron`
* `cups`
* `dbus`
* `fwupd`
* `gdm`
* `gnome-remote-desktop`
* `NetworkManager`
* `polkit`
* `rsyslog`
* `snapd`
* `systemd-journald`
* `systemd-logind`
* `systemd-resolved`
* `unattended-upgrades`

### Assessment

The services were primarily consistent with an Ubuntu desktop environment.

`gnome-remote-desktop` and `fwupd` were investigated further because of their relevance to remote access and recurring service failures.


## Step 4 Evidence


### Step 4 Assessment

No clearly suspicious process or service was identified.

---

# Step 5 — Network Connections

## Objective

Identify listening ports, active connections, and possible remote-access exposure.


## Command 1 — Listening Network Sockets

```bash
sudo ss -tulpen
```

### What the command does

`ss` displays network sockets.

Options:

* `-t` = TCP
* `-u` = UDP
* `-l` = listening
* `-p` = process information
* `-e` = extended information
* `-n` = numeric addresses/ports

### Output

Observed listeners included:

```text
127.0.0.54:53
127.0.0.53:53
127.0.0.1:631
127.0.0.1:30631
::1:631
```

UDP listeners included Avahi/mDNS services.

### Assessment

The observed TCP listeners were primarily bound to loopback addresses rather than all network interfaces.

This reduced the observed external TCP exposure.


## Command 2 — Active TCP Connections

```bash
sudo ss -tpn
```

### What the command does

This displays active TCP connections and associated processes.

### Output

No active TCP connections were returned at the moment of collection.

### Assessment

No active TCP session was observed at the exact collection time.

This is a point-in-time observation and does not prove that no connections occur at other times.


## Command 3 — GNOME Remote Desktop Status

```bash
systemctl status gnome-remote-desktop --no-pager
```

### What the command does

This displays the status of the GNOME Remote Desktop service.

### Output

The service was:

```text
active (running)
```

A GNOME remote desktop daemon was running.

### Additional observation

The service reported a TPM credential initialization-related message, but no evidence was identified that this resulted in active external remote access.


## Command 4 — Common Remote-Access Ports

```bash
sudo ss -tulpen | grep -E '3389|22|5900|5901'
```

### What the command does

This searches listening sockets for common:

* RDP — 3389
* SSH — 22
* VNC — 5900/5901

### Output

No results were returned.

### Assessment

No listener was observed on these common remote-access ports.


## Step 5 Evidence


### Step 5 Assessment

No evidence of active external remote access was identified during collection.

The GNOME Remote Desktop daemon was running, but no common RDP/SSH/VNC listener was observed.

---

# Step 6 — Persistence Mechanisms

## Objective

Identify services and user-level mechanisms that automatically execute programs when the system or user session starts.


## Command 1 — Enabled System Services

```bash
systemctl list-unit-files --state=enabled --type=service
```

### What the command does

This lists system services configured to start automatically.

### Output

Enabled services included standard Ubuntu components such as:

* AppArmor
* NetworkManager
* cron
* cups
* accounts-daemon
* avahi-daemon
* unattended-upgrades
* gnome-remote-desktop
* OpenVPN

### Assessment

The majority of enabled services were consistent with a normal Ubuntu desktop installation.

Several services were selected for additional validation rather than automatically classified as suspicious.


## Command 2 — Enabled User Services

```bash
systemctl --user list-unit-files --state=enabled
```

### What the command does

This lists systemd services enabled for the current user's graphical session.

### Output

Services included:

* `pipewire.service`
* `pipewire-pulse.service`
* `filter-chain.service`
* `gnome-keyring-daemon.service`
* `wireplumber.service`
* `tracker-miner-fs-3.service`
* `gcr-ssh-agent.service`
* desktop integration services

### Assessment

The entries were primarily consistent with Ubuntu/GNOME desktop functionality.


## Command 3 — Graphical Autostart Directory

```bash
ls -la ~/.config/autostart/ 2>/dev/null
```

### What the command does

This checks the user's standard graphical-session autostart directory.

### Output

No entries were returned.

### Assessment

No user-level `.desktop` autostart entries were identified in this directory.


# OpenVPN Investigation

OpenVPN was specifically investigated because it appeared as an enabled system service.

## Command 4 — OpenVPN Service Status

```bash
systemctl status openvpn.service --no-pager
```

### What the command does

Displays the systemd status of the OpenVPN service.

### Output

The service reported:

```text
Loaded: loaded
Active: active (exited)
Main PID: 1396
status=0/SUCCESS
```

### Assessment

The generic OpenVPN service unit was enabled and had completed successfully.

However, `active (exited)` does not by itself demonstrate that an active VPN tunnel exists.


## Command 5 — OpenVPN-Related Units

```bash
systemctl list-units --type=service | grep -i openvpn
```

### What the command does

Searches currently loaded services for names containing `openvpn`.

### Output

```text
openvpn.service loaded active exited OpenVPN service
```

### Assessment

Only the generic OpenVPN service was observed.


## Command 6 — OpenVPN Configuration Files

```bash
sudo find /etc/openvpn -maxdepth 2 -type f -ls
```

### What the command does

Searches `/etc/openvpn` for files up to two directory levels deep.

### Output

The only file identified was:

```text
/etc/openvpn/update-resolv-conf
```

### Assessment

No `.ovpn` client configuration or obvious VPN profile was identified in the searched directory.

`update-resolv-conf` is a helper script rather than evidence of an active VPN profile.


## Command 7 — VPN Tunnel Interface

```bash
ip addr | grep -A3 -B2 tun
```

### What the command does

Searches network-interface information for interfaces containing `tun`.

OpenVPN commonly creates a `tun` interface when a routed VPN tunnel is active.

### Output

No output was returned.

### Assessment

No active `tun` interface was observed.

### OpenVPN Conclusion

The investigation identified an enabled OpenVPN service unit, but:

* No OpenVPN configuration profile was identified
* No active `tun` interface was observed
* No evidence of an active OpenVPN tunnel was identified

OpenVPN was therefore **not classified as suspicious persistence**.


# `filter-chain.service` Investigation

The user-level `filter-chain.service` initially appeared unusual because of its name, so it was explicitly validated.

## Command 8 — User Service Status

```bash
systemctl --user status filter-chain.service --no-pager
```

### What the command does

Checks the status of the `filter-chain.service` belonging to the current user's systemd session.

### Output

The service was:

```text
PipeWire filter chain daemon
active (running)
```

Main process:

```text
/usr/bin/pipewire -c filter-chain.conf
```

### Assessment

The service was identified as a normal PipeWire component.


## Command 9 — Service Definition

```bash
systemctl --user cat filter-chain.service
```

### What the command does

Displays the systemd unit definition for the user service.

### Important output

```text
Description=PipeWire filter chain daemon
```

and:

```text
ExecStart=/usr/bin/pipewire -c filter-chain.conf
```

The service also contained security restrictions:

```text
LockPersonality=yes
MemoryDenyWriteExecute=yes
NoNewPrivileges=yes
RestrictNamespaces=yes
SystemCallArchitectures=native
SystemCallFilter=@system-service
```

### Assessment

The service is a legitimate PipeWire component and has multiple systemd sandboxing/security restrictions.


## Command 10 — Service Properties

```bash
systemctl --user show filter-chain.service -p FragmentPath -p ExecStart -p User -p ActiveState
```

### Output

The service definition was located at:

```text
/usr/lib/systemd/user/filter-chain.service
```

The executable was:

```text
/usr/bin/pipewire
```

The service state was:

```text
ActiveState=active
```

### Assessment

The service was confirmed as legitimate PipeWire functionality.


## Step 6 Evidence



### Step 6 Assessment

No clearly malicious persistence mechanism was identified.

The OpenVPN service was enabled but showed no evidence of an active tunnel.

The initially questionable `filter-chain.service` was confirmed to be a legitimate PipeWire service.

---

# Step 7 — Security Control Assessment

## Objective

Evaluate host security controls including AppArmor, firewall configuration, automatic updates, kernel security events, package repositories, and system errors.


# AppArmor

## Command 1

```bash
sudo aa-status
```

### What the command does

`aa-status` reports the current status of the AppArmor security framework and its profiles.

### Output

The system reported:

```text
apparmor module is loaded.
155 profiles are loaded.
58 profiles are in enforce mode.
```

Five processes were reported as actively running under enforced profiles.

The system also reported:

```text
0 processes are in complain mode.
0 processes are in prompt mode.
0 processes are in kill mode.
```

Three processes were listed as unconfined while having a defined profile.

### Assessment

AppArmor is loaded and actively enforcing profiles.

This is a positive host security control.

No AppArmor denial event was identified during the subsequent kernel-log search.


# UFW Firewall

## Command 2

```bash
sudo ufw status verbose
```

### What the command does

Displays the status and configuration of Ubuntu's uncomplicated firewall (`ufw`).

### Output

```text
Status: inactive
```

### Assessment

UFW is not currently enabled.

This represents a **defensive configuration gap**.

However, this finding must be correlated with the network investigation. Earlier socket enumeration showed that the observed TCP listeners were primarily bound to loopback addresses, reducing the observed external TCP exposure.

Therefore, the evidence does not support a conclusion that the system was actively exposed to an attack.


# nftables

## Command 3

```bash
sudo nft list ruleset
```

### What the command does

Displays the currently loaded nftables firewall rules.

### Output

No output was returned.

### Assessment

No nftables ruleset was loaded at the time of collection.

Combined with the inactive UFW status, there was no evidence of an active UFW/nftables host firewall policy.


# Automatic Updates

## Command 4

```bash
systemctl status unattended-upgrades --no-pager
```

### What the command does

Displays the status of Ubuntu's unattended-upgrades service.

### Output

The service was:

```text
Loaded: loaded
enabled
Active: active (running)
```

### Assessment

Automatic upgrade functionality is enabled and running.

This is a positive security control because it supports automated package maintenance.


# Kernel Security Events

## Command 5

```bash
sudo journalctl -k --since "7 days ago" | grep -Ei "apparmor|audit|denied|blocked|security"
```

### What the command does

Searches kernel journal messages from the previous seven days for common security-related terms.

### Output

No output was returned.

### Assessment

No matching kernel security messages were identified by this search.

This does not prove that the system generated no security-related events; it means this particular search returned no matches.

# Package Repository and Update Check

## Command 6

```bash
sudo apt update
```

### What the command does

Refreshes local package metadata from configured Ubuntu repositories.

It does not install or upgrade packages.

### Output

The system successfully contacted:

```text
noble
noble-updates
noble-backports
noble-security
```

and downloaded updated ARM64 package metadata.

The operation completed successfully.

### Assessment

Ubuntu's package repositories were reachable and package metadata could be refreshed successfully.

No repository error was observed.


# fwupd Investigation

Repeated errors were identified in the system journal:

```text
Failed to start fwupd-refresh.service
```

This service was investigated rather than immediately classified as suspicious.

## Command 7

```bash
systemctl status fwupd-refresh.service --no-pager -l
```

### What the command does

Displays the complete status of the `fwupd-refresh.service` unit without truncating long lines.

### Output

The service reported:

```text
Active: inactive (dead)
```

and:

```text
ExecStart=/usr/bin/fwupdmgr refresh
```

The process exited with:

```text
status=2
```

The service was triggered by:

```text
fwupd-refresh.timer
```

### Assessment

The recurring error is associated with the firmware metadata refresh command:

```text
/usr/bin/fwupdmgr refresh
```

There is no evidence from this output that the failure is malicious.

It is best classified as a **system maintenance/software configuration issue requiring follow-up**, rather than a confirmed security incident.


## Command 8 — fwupd Journal

The investigation attempted:

```bash
sudo journalctl -u fwup-refresh.service --since "7 days ago" --no-pager
```

### Output

```text
-- No entries --
```

### Important note

The unit name in this command was mistyped as:

```text
fwup-refresh.service
```

The actual service is:

```text
fwupd-refresh.service
```

Therefore, the "No entries" result should **not** be used as evidence that the correct service generated no logs.

The earlier `systemctl status fwupd-refresh.service` output and the system journal already confirmed repeated failures.

This command error is retained in the investigation record for transparency.


# System Clock Assessment

## Command 9

```bash
timedatectl
```

### What the command does

Displays:

* Local time
* UTC time
* RTC time
* Time zone
* Clock synchronization status
* NTP status

### Output

The system reported:

```text
Local time: Tue 2026-09-22
Universal time: Tue 2026-09-22
Time zone: America/New_York
System clock synchronized: no
NTP service: inactive
```

### Assessment

The system clock was **not synchronized through NTP** at the time of collection.

This is an important security-investigation finding because accurate timestamps are essential for event correlation.

The journal also contained entries dated:

```text
Oct 03
```

even though the current system date reported by `timedatectl` was September 22.

Those future-dated entries should therefore be treated cautiously until their origin and journal timestamp behavior are understood.

# GDM Login-Keyring Events

The error-level journal search also returned:

```text
gkr-pam: the password for the login keyring was invalid.
```

### Assessment

This indicates a GNOME login-keyring password mismatch.

It is not, by itself, evidence of an unauthorized login or compromise.

It should be documented separately from the authentication investigation.

---

# Step 7 Evidence



# Final Findings

## Finding 1 — No obvious compromise identified

The investigation did not identify a confirmed malicious:

* User
* Process
* Service
* Persistence mechanism
* Network listener
* VPN tunnel
* Authentication pattern

The available evidence therefore does not establish a compromise.

---

## Finding 2 — Host firewall is inactive

### Severity

**Medium — Defensive Configuration Gap**

### Evidence

```text
UFW: inactive
```

and:

```text
nft list ruleset
```

returned no rules.

### Context

The absence of a host firewall policy is a security-hardening gap.

However, socket enumeration showed no significant externally bound TCP listeners during the investigation, reducing the observed attack surface.

### Recommendation

For a production-like environment, implement and validate a host firewall policy appropriate for the system's intended services.

---

## Finding 3 — AppArmor is enforcing

### Severity

**Positive Security Control**

### Evidence

```text
155 profiles loaded
58 profiles in enforce mode
```

### Assessment

AppArmor provides application-level mandatory access control and was actively enforcing profiles.

---

## Finding 4 — Automatic updates are enabled

### Severity

**Positive Security Control**

### Evidence

```text
unattended-upgrades.service
Active: active (running)
```

### Assessment

Automatic update functionality is enabled.

---

## Finding 5 — OpenVPN service enabled without active tunnel evidence

### Severity

**Low / Informational**

### Evidence

* OpenVPN service enabled
* No `.ovpn` configuration identified
* No `tun` interface observed
* Generic service reported `active (exited)`

### Assessment

No evidence of an active VPN tunnel was identified.

---

## Finding 6 — Repeated fwupd refresh failures

### Severity

**Low — Maintenance Issue**

### Evidence

Repeated:

```text
Failed to start fwupd-refresh.service
```

The service attempted:

```text
/usr/bin/fwupdmgr refresh
```

and exited with:

```text
status=2
```

### Assessment

The failure appears associated with firmware metadata refresh functionality.

No evidence currently connects the failure to malicious activity.

---

## Finding 7 — System clock is not synchronized

### Severity

**Medium — Investigation/Monitoring Concern**

### Evidence

```text
System clock synchronized: no
NTP service: inactive
```

### Assessment

Unsynchronized time can make security-event correlation less reliable.

This is particularly important for SOC investigations involving authentication, network, and process timelines.

---

## Finding 8 — GNOME keyring password mismatch

### Severity

**Low — User/Configuration Issue**

### Evidence

```text
gkr-pam: the password for the login keyring was invalid.
```

### Assessment

This is consistent with a keyring/password synchronization issue and is not sufficient evidence of malicious authentication activity.

---

# Overall Security Assessment

The Linux VM showed a generally recognizable Ubuntu desktop security baseline.

### Positive controls observed

* AppArmor loaded and enforcing profiles
* Automatic updates enabled
* No suspicious persistence mechanism identified
* No obvious malicious process identified
* No obvious unauthorized account identified
* No active OpenVPN tunnel identified
* No common external SSH/RDP/VNC listener observed
* Package repositories successfully reachable

### Security gaps / follow-up items

* UFW is inactive
* No nftables ruleset was loaded
* System clock synchronization is disabled
* `fwupd-refresh.service` repeatedly fails
* GNOME login-keyring password mismatch events were observed
* Future-dated journal entries require cautious interpretation

---

# Investigation Conclusion

The investigation did not identify sufficient evidence to classify the Ubuntu virtual machine as compromised.

The strongest security-control gap identified was the absence of an active host firewall policy through UFW or nftables.

The most important investigation-quality issue was the lack of NTP/time synchronization, because inaccurate or inconsistent timestamps can affect event correlation.

The repeated `fwupd-refresh.service` failures and GNOME keyring messages were documented as system/configuration issues rather than malicious activity because no supporting evidence linked them to compromise.

The investigation demonstrates a defensive workflow based on:

1. Establishing a baseline
2. Identifying privileged users
3. Reviewing authentication activity
4. Examining processes and services
5. Assessing network exposure
6. Investigating persistence
7. Validating security controls
8. Correlating multiple evidence sources before reaching a conclusion

---

# Portfolio Skills Demonstrated

This project demonstrates practical experience with:

* Linux command-line investigation
* Ubuntu system administration
* User and privilege analysis
* Sudo analysis
* Authentication-log analysis
* systemd service investigation
* Process analysis
* Network socket enumeration
* Remote-access exposure assessment
* Persistence analysis
* OpenVPN investigation
* PipeWire/service validation
* AppArmor
* Firewall assessment
* nftables
* Automatic update mechanisms
* Journal analysis
* Timeline validation
* Evidence preservation
* Security finding classification
* Incident-investigation methodology

---

# Analyst Takeaway

A security investigation should not classify something as malicious simply because it is unfamiliar.

In this investigation:

* `filter-chain.service` initially required validation but was confirmed as legitimate PipeWire functionality.
* OpenVPN was enabled but showed no evidence of an active tunnel.
* `fwupd-refresh.service` repeatedly failed, but there was no evidence connecting those failures to malicious activity.
* GNOME keyring errors were observed but did not establish unauthorized authentication.
* The absence of UFW/nftables rules was identified as a hardening gap rather than evidence of compromise.
* Time synchronization was identified as an important issue because reliable timestamps are essential for security investigations.

The investigation therefore emphasizes **evidence-based analysis, correlation, and careful distinction between anomalies, configuration issues, and confirmed security incidents.**

