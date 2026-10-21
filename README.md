# 🛡️ Horse Plinko Cyber Challenge Season 4 — Blue Team Incident Response Write-Up

## 🚀 Executive Summary
During the **Horse Plinko Cyber Challenge 4 (Fall 2026)** hosted by Hack@UCF at the UCF FAIRWINDS Alumni Center, I acted as a Lead Blue Team Incident Responder tasked with defending the critical aerospace infrastructure of the **International Horse Plinko League (IHPL)**. The network infrastructure comprised a mixed landscape of Linux and Windows environments running scored automation pipelines essential for launching rockets under *Mission Fortuna*.

Throughout a relentless live-fire engagement against the threat actor group known as the **Horse Liberation Front (Red Team)**, I successfully audited, hardened, and completely purged a massive, multi-tiered malware campaign targeting our Windows Server 2022 build systems (`assembly-t`). My immediate technical responses restored host infrastructure integrity, sealed a critical unauthenticated remote code execution (RCE) vector, and recovered broken user workflows under active exploitation.

---

## 🏗️ Competition Landscape & Environment Topology
Per the official IHPL Onboarding architecture, our team was responsible for protecting the core network subnet (`172.16.t.0/24`) tied back to the central scoring engine via NetBird overlay tunnels. 

### Scored Infrastructure Assigned:
*   **`assembly-t` (172.16.t.30):** Windows Server 2022 hosting Java-based Jenkins automation pipelines (`rocket-build`). *(Primary focus of this incident)*.
*   **`manifest-t` (172.16.t.40):** Windows Server 2019 running a production MariaDB server backing local web frameworks.
*   **`ignition-t` (172.16.t.20):** Fedora Server 38 hosting an internal OpenWebUI AI chat infrastructure.
*   **`guidance-t` (172.16.t.10):** Ubuntu 22.04 running the critical customer-facing *PlinkCom* web console.
*   **`blackbox-t` (172.16.t.5):** Linux Mint 22 OpenObserve SIEM collector. *(Out of scope for Red Team)*.

---

## 🐴 Case Study: Eradicating the "Horse" Malware Campaign
An aggressive, multi-layered Python-compiled malware suite was uncovered executing directly inside system spaces on `assembly-t`. The payload generated persistent, intrusive desktop graphic loops while dropping active command-and-control (C2) network threads back to red team endpoints.

Below is the structured breakdown of the steps I executed to completely eliminate the compromise lifecycle and secure the machine.

### 🔍 1. Forensic Discovery & Triage
*   **Asset Inspection:** Identified that the payload was built as a standalone **PyInstaller compiled executable container** utilizing a bundled **Python 3.13 runtime**. Upon execution, it dynamically unpacked graphic dependencies (`bhorsespin.gif`) into hidden scratch directories (`_MEI*`) within the `C:\Windows\Temp\` and local AppData profiles.
*   **Process Masquerading Uncovered:** Leveraged Sysinternals `Process Explorer` and `TCPView` to capture rogue behavior hiding behind valid administrative naming metrics.
*   **Network Callbacks Traced:** Captured a live thread injection payload within the legitimate user desktop user shell (`explorer.exe` at PID 7744) and a fake copy of the Distributed Transaction Coordinator (`msdtc.exe` at PID 4312) maintaining established socket streams out to the attacker's public IP engine (**`6.106.130.15`** on port **`6000`** and **`215.132.16.116`** on port **`9000`**).

### 🛠️ 2. Dismantling the Multi-Tiered Persistence Loop
The malware implemented a persistent "watchdog" layout designed to instantly re-spawn components if individual execution trees were stopped. I executed a coordinated, zero-downtime eradication pipeline via PowerShell:

1.  **Terminated Active Memory Subversions:** Directly killed infected processes, script wrappers, and background dependencies simultaneously to drop active callbacks before hooks could revive:
    ```powershell
    taskkill /f /im java.exe /im explorer.exe /im mysqld.exe
    taskkill /f /im updater.exe /im speech_agent.exe /im "Windows Cache Telemetry.exe" /im spoolsv.exe
    ```
2.  **Evicted Malicious Windows Services:** The malware had registered custom unverified system services running extensionless or misplaced binaries to monitor the host. I terminated and permanently deleted all four fake entries:
    ```powershell
    Stop-Service -Name "iehandler", "Oracle", "MySQL Server Daemon", "Distributed Transaction Coordinator" -Force
    sc.exe delete "iehandler"
    sc.exe delete "Oracle"
    sc.exe delete "MySQL Server Daemon"
    sc.exe delete "Distributed Transaction Coordinator"
    ```
3.  **Unregistered Automated Scheduled Tasks:** Purged hidden automation tracking tasks triggering loops on intervals:
    ```powershell
    Unregister-ScheduledTask -TaskName "BITS Proxy Host" -Confirm:$false
    Unregister-ScheduledTask -TaskName "MySQL Command-Line Client" -Confirm:$false
    ```
4.  **Purged the Registry Run Keys & Local App Directories:** Wiped out hidden user startup links and localized shortcut items (`Chrome Update Services.exe`) inside the execution profile directory hives:
    ```powershell
    Remove-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" -Name "explorer","jenkins-agent","speech_agent" -Force
    Remove-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Run" -Name "JavaRunner","speech_agent","spoolsv","Windows Cache Telemetry" -Force
    ```

### 🪓 3. Seizing NTFS Ownership & Destroying Master Binaries
The malware stored its massive 33.5 MB compiled parent payloads inside highly protected corporate folders. I reclaimed permission matrices and permanently wiped the objects:
```powershell
# Seize file system permissions from protected Defender directories
takeown /f "C:\Program Files\Windows Defender Advanced Threat Protection\SenseCncProxyMirror.exe" /A
icacls "C:\Program Files\Windows Defender Advanced Threat Protection\SenseCncProxyMirror.exe" /grant Administrators:F /t

# Shred the file contents and purge the binaries from disk
Clear-Content -Path "C:\Program Files\Windows Defender Advanced Threat Protection\SenseCncProxyMirror.exe" -Force
Remove-Item -Path "C:\Program Files\Windows Defender Advanced Threat Protection\SenseCncProxyMirror.exe" -Force
Remove-Item -Path "C:\Windows\Media\msdtc.exe", "C:\Program Files\Windows NT\mysqld.exe", "C:\Program Files (x86)\Internet Explorer\iehandler" -Force
```

---

## 🔒 Jenkins Platform Remediation & Exploit Mitigation
### 🕳️ Root Cause Vulnerability: CVE-2025-64132 (Jenkins MCP Plugin Bypass)
The red team gained continuous access to the environment by exploiting an unauthenticated remote execution vulnerability inside the **Jenkins MCP Server Plugin** (Version `0.84.v50ca_24ef83f2`) running on Jenkins Core `2.568.3`. This allowed attackers to hit the `/mcp-server/mcp` endpoint anonymously and issue commands.

### 🛡️ Hardening Actions Executed:
1.  **Plugin Eviction:** Stopped the service, completely deleted the `.jpi` packages and associated binary folders from the disk workspace, and cleared cache folders (`winstone*`/`jetty*`) to drop memory holds.
2.  **Authorization Matrix Restoration:** Modified global parameters inside `config.xml` to flip the broad, vulnerable `<authorizationStrategy>` from *"Anyone can do anything"* to **"Logged-in users can do anything"**, explicitly stripping anonymous users of read, build, or endpoint execution rights.
3.  **Credential Hardening:** Reclaimed access to the hijacked administrative account **`jmoney`** by generating clean `<passwordHash>` bcrypt strings and repairing local Group Policy settings that were blocking remote desktop logins.
4.  **Configuration Write-Protection:** Used `icacls` to restrict write permissions on `jenkins.xml`, successfully blocking automated malicious parsing scripts (evidenced by the rogue `hpcc_port_shift_state` artifact) from shifting active HTTP listening web ports away from port `8080`.
5.  **Validation Testing:** Executed an unauthenticated network validation query from PowerShell, successfully verifying that access attempts drop with an explicit **`HTTP 403 Forbidden`** response code, while the core deployment pipeline (`rocket-build`) remained completely green.

---

## 🧱 Local OS Perimeter Hardening
To prevent lateral movement and unauthorized network reconnaissance without disrupting critical RDP scoring checks, I customized the **Windows Defender Firewall with Advanced Security** inbound/outbound rules:

### Rules Safely Disabled:
*   **WinRM (HTTP/HTTPS) Inbound:** Closed down remote public PowerShell query accessibility.
*   **AllJoyn Router (TCP/UDP Inbound & Outbound):** Disabled unneeded consumer smart home discovery protocols.
*   **Cast to Device functionality & mDNS:** Stopped multicast local service name and endpoint broadcasting over public and private interfaces.
*   **ICMP (Echo Request Inbound/Outbound):** Blocked external ping scans to conceal the server from public mapping and network enumeration.

---

## 💻 Workstation Troubleshooting & Connectivity Recovery
During the engagement, CEO J-Money's personal laptop (`172.16.t.176`) lost entire web connection layers following a phishing simulation scenario involving an emailed "new browser" setup utility. 

### 🔧 Diagnostic Triage & Repair Action:
*   **Root Cause Analysis:** Verified that `ipconfig /all` returned clean hardware leases and valid public DNS upstreams (`1.1.1.1`/`8.8.8.8`). However, the installer had activated system-wide **Web Proxy Auto-Discovery (WPAD)** rules, routing browser hooks into a dead local proxy loopback.
*   **Remediation Steps:**
1. **Turned OFF** "Automatically detect settings" within Internet Properties (`inetcpl.cpl`).
2. Used administrative command contexts to completely **flush local name containers**:
   ```powershell
   Set-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings" -Name "ProxyAutoDetect" -Value 0
   ipconfig /flushdns
   ```
3. **Cleared local browser application session indices** and hard-refreshed (`Ctrl + F5`), instantly bringing the executive network connection back online safely.
