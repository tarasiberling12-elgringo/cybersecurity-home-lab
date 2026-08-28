# Windows Endpoint Security Investigation

## Overview

This project documents a hands-on security investigation of a Windows endpoint using native Windows security tools, PowerShell, Windows Event Logs, and system telemetry.

The objective was to establish a security baseline, investigate authentication activity, identify potential persistence mechanisms, validate running services and processes, review network exposure, and determine whether observed indicators represented legitimate Windows activity or potential security concerns.

The investigation followed a SOC-style workflow focused on **evidence collection, process attribution, event correlation, validation, and risk assessment** rather than assuming that unusual activity was malicious.

---

## Objectives

The investigation focused on the following areas:

* Local account and privilege analysis
* Windows authentication investigation
* Security event correlation
* Windows service analysis
* Scheduled task and persistence hunting
* Startup persistence analysis
* Microsoft Defender validation
* Windows Firewall configuration
* SMB security configuration
* NTFS permission analysis
* Process and network connection attribution
* Digital signature verification
* Identification of suspicious or anomalous activity
* Documentation of findings and security recommendations

---

## Environment

| Component               | Configuration                               |
| ----------------------- | ------------------------------------------- |
| Operating System        | Windows                                     |
| Investigation Host      | Windows virtual machine                     |
| Virtualization          | Parallels                                   |
| Primary Account         | `muhammed`                                  |
| Security Tools          | PowerShell, Windows Event Viewer/Event Logs |
| Endpoint Protection     | Microsoft Defender                          |
| Network Analysis        | PowerShell networking cmdlets               |
| Authentication Analysis | Windows Security Event Logs                 |

---

# Investigation Methodology

The investigation followed a structured endpoint triage methodology:

```text
System Baseline
      ↓
Account Investigation
      ↓
Authentication Analysis
      ↓
Privilege Analysis
      ↓
Service Investigation
      ↓
Persistence Hunting
      ↓
Endpoint Protection Validation
      ↓
Network Investigation
      ↓
Process Attribution
      ↓
Digital Signature Verification
      ↓
Event Correlation
      ↓
Risk Assessment
```

---

# 1. Local Account Investigation

Local Windows accounts were enumerated to identify enabled accounts and potential administrative exposure.

The primary user account was:

```text
muhammed
```

The built-in Administrator and Guest accounts were disabled.

The `muhammed` account was found to be a member of:

```text
BUILTIN\Administrators
BUILTIN\Users
```

### Security Assessment

The account configuration was not itself evidence of compromise. However, membership in the local Administrators group significantly increases the potential impact of an account compromise.

**Risk:** Medium

**Recommendation:** Apply least-privilege principles and use a standard account for routine activities whenever possible.

---

# 2. Privilege Investigation

The account’s effective Windows security context was examined using:

```powershell
whoami /user
whoami /groups
whoami /priv
```

The account operated with a high-integrity security context and possessed several administrative privileges.

Notably, the following privileges were enabled:

```text
SeDebugPrivilege
SeImpersonatePrivilege
SeCreateGlobalPrivilege
SeChangeNotifyPrivilege
```

### Security Assessment

These privileges are commonly associated with administrative Windows accounts. They are not independently indicators of malicious activity.

However, privileges such as `SeDebugPrivilege` and `SeImpersonatePrivilege` can significantly increase the impact of a successful local compromise.

**Risk:** Medium

---

# 3. Authentication Investigation

Windows Security Event Logs were analyzed for authentication activity.

Important events identified included:

| Event ID | Meaning                                  |
| -------- | ---------------------------------------- |
| 4624     | Successful logon                         |
| 4625     | Failed logon                             |
| 4648     | Logon attempt using explicit credentials |
| 4723     | Attempt to change an account password    |
| 4738     | User account changed                     |

Multiple authentication events involving the `muhammed` account were observed.

Several 4648 events showed explicit credentials being used against:

```text
localhost
127.0.0.1
```

The activity was associated with Windows processes including:

```text
C:\Windows\System32\svchost.exe
C:\Windows\System32\lsass.exe
```

### Security Assessment

The presence of 4648 events does not automatically indicate credential theft.

The local destination and Windows system processes suggested that the activity could be generated by legitimate local Windows components or account-related operations.

However, repeated 4625 failures warranted investigation and correlation with services, scheduled tasks, and processes.

**Disposition:** Investigated; no confirmed malicious activity identified.

---

# 4. Windows Service Investigation

Windows services were examined to identify processes operating with elevated privileges.

The User Manager service was investigated:

```text
Service: UserManager
Process: svchost.exe
Account: NT AUTHORITY\SYSTEM
State: Running
Startup: Automatic
```

The service executable was located at:

```text
C:\Windows\System32\svchost.exe
```

The executable was subsequently validated through digital signature verification.

### Security Assessment

The service configuration was consistent with a legitimate Windows service.

**Disposition:** Benign / Expected

---

# 5. Scheduled Task Investigation

Scheduled tasks were enumerated to identify potential persistence mechanisms.

Particular attention was given to:

```text
SoftLandingCreativeManagementTask
```

The task was associated with:

```text
User: muhammed
Logon Type: Interactive
Action: COM Handler
```

The task used the following COM Class ID:

```text
{F576B2F9-7850-4226-ADB0-E5993FED4F02}
```

The task contained both time-based and Windows notification triggers.

### Security Assessment

A user-associated COM-handler task is security-relevant because scheduled tasks can be abused for persistence.

However, the existence of the task alone was insufficient to classify it as malicious.

The task was therefore treated as a **suspicious artifact requiring validation**, rather than being incorrectly labeled malware.

**Disposition:** Investigated; no confirmed malicious behavior identified.

---

# 6. Startup Persistence Investigation

Common Windows startup locations were examined, including:

```text
HKLM:\Software\Microsoft\Windows\CurrentVersion\Run
HKLM:\Software\Microsoft\Windows\CurrentVersion\RunOnce
```

The primary startup entries observed included legitimate Microsoft components such as:

```text
SecurityHealthSystray.exe
Microsoft Edge WebView cleanup
```

User and system Startup folders were also examined.

No suspicious executable or script-based startup persistence was identified.

**Disposition:** No obvious malicious startup persistence found.

---

# 7. Microsoft Defender Validation

Microsoft Defender services and configuration were examined.

Observed security controls included:

```text
AntivirusEnabled              : True
AntispywareEnabled            : True
RealTimeProtectionEnabled     : True
BehaviorMonitorEnabled        : True
IoavProtectionEnabled         : True
NISEnabled                    : True
```

The following Defender services were running:

```text
WinDefend
MDCoreSvc
WdNisSvc
```

Executable paths were verified as Microsoft Defender components.

### Security Assessment

Endpoint protection was operational with real-time protection and network inspection enabled.

**Disposition:** Healthy security posture

---

# 8. Windows Firewall Investigation

Firewall profiles were checked for:

```text
Domain
Private
Public
```

All three profiles were enabled.

This established that Windows Firewall was active across the available network profiles.

**Disposition:** Expected security configuration

---

# 9. SMB Security Investigation

SMB configuration was reviewed because exposed file-sharing services can represent a significant attack surface.

Configuration observed:

```text
SMB1: Disabled
SMB2: Enabled
Security Signing: Required
```

Listening ports included:

```text
TCP/139
TCP/445
```

Default administrative shares were present:

```text
ADMIN$
C$
IPC$
```

### Security Assessment

SMB1 being disabled and SMB signing being required represent positive security controls.

The presence of administrative shares such as `C$` and `ADMIN$` is normal on Windows systems, although they should be monitored in environments where lateral movement is a concern.

**Disposition:** No immediate SMB misconfiguration identified.

---

# 10. NTFS Permission Investigation

Permissions on important system locations were reviewed using:

```powershell
icacls C:\
icacls C:\Windows
Get-Acl C:\
```

The root of the system drive was owned by:

```text
NT SERVICE\TrustedInstaller
```

Expected administrative principals such as:

```text
SYSTEM
Administrators
Users
Authenticated Users
```

were identified.

The Windows directory contained expected protection involving:

```text
TrustedInstaller
SYSTEM
Administrators
Users
```

### Security Assessment

The examined permissions were consistent with a standard Windows installation.

**Disposition:** No obvious abnormal ACL configuration identified.

---

# 11. Network Connection Investigation

Active TCP connections were investigated and mapped to owning processes.

Several HTTPS connections were identified, including traffic to:

```text
20.59.87.225:443
20.59.87.227:443
23.202.61.106:443
23.66.206.51:443
```

The associated processes were identified.

For example:

```text
20.59.87.227:443
        ↓
PID 3288
        ↓
svchost.exe
        ↓
WpnService
        ↓
Windows Push Notifications System Service
```

The service was running as:

```text
NT AUTHORITY\SYSTEM
```

The executable was:

```text
C:\Windows\System32\svchost.exe
```

The executable’s Authenticode signature was verified as valid and signed by Microsoft.

### Security Assessment

The network connection was not treated as malicious solely because the remote IP address could not be resolved through local DNS.

Process attribution and executable validation provided important context.

**Disposition:** No confirmed malicious activity identified.

---

# 12. Process Validation

Processes associated with network connections were examined using process IDs.

Examples included:

```text
explorer.exe
svchost.exe
msedgewebview2.exe
Widgets.exe
prl_tools_service.exe
```

Executable locations were checked to determine whether the processes originated from expected Windows or application directories.

Where appropriate, Authenticode signatures were also verified.

Microsoft-signed components included:

```text
C:\Windows\Explorer.EXE
C:\Windows\System32\svchost.exe
Microsoft Edge WebView components
Windows system components
```

### Security Assessment

The investigated processes were located in expected installation directories and had valid Microsoft signatures where signature validation was performed.

---

# Evidence

---

# Key Findings

| Finding                                         |   Severity | Assessment                                                |
| ----------------------------------------------- | ---------: | --------------------------------------------------------- |
| User account has local administrator privileges |     Medium | Increases impact of account compromise                    |
| `SeDebugPrivilege` enabled                      |     Medium | Expected for administrative context but security-relevant |
| `SeImpersonatePrivilege` enabled                |     Medium | Potentially powerful local privilege                      |
| Multiple failed local authentication events     |     Medium | Requires correlation and monitoring                       |
| Explicit credential events                      | Low–Medium | Investigated; no confirmed malicious source               |
| User-associated COM scheduled task              |     Medium | Persistence mechanism requiring validation                |
| SMB1 disabled                                   |   Positive | Reduces legacy SMB exposure                               |
| SMB signing required                            |   Positive | Improves SMB security                                     |
| Windows Firewall enabled                        |   Positive | Host firewall operational                                 |
| Microsoft Defender active                       |   Positive | Real-time endpoint protection enabled                     |
| Microsoft system binaries signed                |   Positive | No signature anomalies observed                           |
| Suspicious network IP investigated              |        Low | Process/service attribution supported benign explanation  |

---

# Overall Assessment

The Windows endpoint investigation **did not identify confirmed malware, unauthorized persistence, or a confirmed compromised account** based on the evidence collected.

Several artifacts were nevertheless security-relevant and demonstrate why endpoint investigation requires correlation rather than relying on individual indicators.

The most notable areas for continued monitoring were:

1. Local administrator privileges
2. Repeated failed authentication events
3. Explicit credential usage
4. User-associated scheduled tasks
5. SYSTEM-level services generating network connections

The investigation demonstrated a complete endpoint triage workflow from **initial enumeration through process attribution and security assessment**.

---

# Detection Opportunities

The investigation also identified several events that could be converted into SOC detections.

Potential detection logic includes:

### Excessive failed logons

```text
Multiple Event ID 4625
+
Same account
+
Short time window
```

Potential alert:

> Multiple failed authentication attempts against a Windows account.

### Explicit credential usage

```text
Event ID 4648
+
Sensitive account
+
Unexpected process
```

Potential alert:

> Explicit credential usage detected from an unusual process.

### Suspicious scheduled task

```text
New scheduled task
+
User-writable execution path
+
COM handler / script / executable
```

Potential alert:

> Potential persistence through scheduled task.

### Suspicious service

```text
New service
+
SYSTEM execution
+
Unusual executable path
```

Potential alert:

> Potential service-based persistence detected.

### Suspicious network connection

```text
Process
    ↓
Remote IP
    ↓
Unexpected destination
    ↓
Unsigned/unusual executable
```

Potential alert:

> Suspicious outbound connection attributed to an anomalous process.

---

# Tools & Commands

Primary tools used:

```text
PowerShell
Get-LocalUser
Get-LocalGroup
Get-LocalGroupMember
Get-CimInstance
Get-Service
Get-Process
Get-ScheduledTask
Get-ScheduledTaskInfo
Get-WinEvent
Get-NetTCPConnection
Get-SmbServerConfiguration
Get-SmbShare
Get-SmbShareAccess
Get-Acl
icacls
whoami
net user
net use
sc.exe
Get-AuthenticodeSignature
```

---

# Skills Demonstrated

### Windows Security

* Windows account security
* Local privilege analysis
* Windows services
* Scheduled task persistence
* Startup persistence
* NTFS permissions
* SMB security
* Windows Firewall
* Microsoft Defender

### SOC / Blue Team

* Endpoint triage
* Security event analysis
* Authentication investigation
* Process attribution
* Network connection attribution
* IOC investigation
* Evidence correlation
* Risk classification
* False-positive reduction
* Security documentation

### Detection Engineering Foundations

* Event-based detection logic
* Authentication anomaly detection
* Persistence detection
* Process/network correlation
* Security alert prioritization

---

# Lessons Learned

The primary lesson from this investigation was that **anomalous does not automatically mean malicious**.

For example, an outbound HTTPS connection to an unfamiliar IP address initially represents an indicator worth investigating. The correct SOC workflow is to determine:

```text
Who initiated it?
      ↓
Which process?
      ↓
Which executable?
      ↓
Which user/service?
      ↓
Where is the executable located?
      ↓
Is it digitally signed?
      ↓
Is the behavior expected?
      ↓
Does other telemetry support compromise?
```

This approach reduces false positives while preserving the ability to identify genuine threats.

---

# Future Improvements

The next phase of this project will expand the investigation from passive endpoint analysis into **active detection engineering**.

Planned improvements include:

* Sysmon deployment
* Enhanced PowerShell logging
* Process creation monitoring
* Network telemetry collection
* Controlled attack simulation
* Windows Event correlation
* Sigma detection rules
* MITRE ATT&CK mapping
* Incident-response documentation
* SIEM ingestion and analysis
