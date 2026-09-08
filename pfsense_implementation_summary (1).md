# Network Infrastructure & Security Implementation Summary: pfSense, Snort, Active Directory & OpenVPN

---

## 1. Executive Summary & Architecture Overview

This technical documentation compiles the complete configuration, diagnostic troubleshooting, syntax resolutions, and verification procedures executed across the network environment.

### Target Network Architecture
* **WAN Interface:**
  * pfSense WAN IP: `192.168.99.123`
  * Upstream ISP Gateway / DNS: `192.168.99.254`
* **SERVERS / LAN Subnet:** `192.168.2.0/24`
  * pfSense Gateway: `192.168.2.254`
  * Domain Controller & DNS (`WinSRV1`): `192.168.2.10`
  * Enterprise Certificate Authority (`WinSRV3`): `192.168.2.30`
* **DMZ Subnet:** `192.168.1.0/24`
  * Web Server: `192.168.1.10`
* **OpenVPN Tunnel Subnet:** `10.0.8.0/24`
  * Remote Client (`WinClient3`): `10.0.8.2`
* **Domain / Directory Structure:**
  * Root Domain: `DC=manila,DC=com`
  * Organizational Units: `OU=IT,OU=manila,DC=manila,DC=com`
  * Target VPN Security Group: `CN=VPN Users,OU=manila,DC=manila,DC=com`
  * Target Test Account: `C1`

---

## 2. Issues Encountered & Root Cause Analysis

| Component | Symptom / Error | Root Cause |
| :--- | :--- | :--- |
| **DNS Resolution** | `WinSRV1` showed `<Unable to resolve> An unknown error occurred` when forwarding to `192.168.2.254`. | Port 53 binding conflict between **DNS Resolver (Unbound)** and **DNS Forwarder (dnsmasq)** on pfSense, coupled with missing outbound rule permissions on the `SERVERS` interface. |
| **Snort IDS/IPS** | Custom detection rules failed to trigger alerts when browsing restricted sites. | Syntax error in `custom.rules`: `ip` protocol was specified alongside a destination port (`any 53`). Snort requires `udp` or `tcp` when layer 4 ports are defined. |
| **Active Directory LDAP** | pfSense diagnostic test reported `Authentication failed.` | (1) Test query used User Principal Name (`C1@manila.com`) instead of `sAMAccountName` (`C1`).<br>(2) Container was configured to default `CN=Users`, whereas user `C1` resided in `OU=IT,OU=manila`.<br>(3) Extended query referenced incorrect group DN (`CN=VPN Users,CN=Users` instead of `OU=manila`). |
| **Certificate Authority Import** | CA import threw validation error: `Value must be greater than or equal to 1`. | pfSense Certificate Manager required a valid starting integer (`1`) for **Next Certificate Serial** when importing an external CA public key without a private key. |

---

## 3. Step-by-Step Fixes & Configurations Applied

### Phase A: DNS Forwarding & Outbound Routing
1. **Resolved pfSense Port 53 Service Conflicts:**
   * Navigated to **Services > DNS Resolver > General Settings** and ensured **Enable DNS Resolver** was unchecked.
   * Navigated to **Services > DNS Forwarder**, enabled the forwarder, and registered interfaces.
2. **Configured Outbound DNS Firewall Rules:**
   * Navigated to **Firewall > Rules > SERVERS**.
   * Added rule:
     * **Action:** `Pass`
     * **Interface:** `SERVERS`
     * **Address Family:** `IPv4`
     * **Protocol:** `TCP/UDP`
     * **Source:** `SERVERS subnets`
     * **Destination:** `*`
     * **Destination Port:** `53 (DNS)`
3. **Reconfigured WinSRV1 Forwarders:**
   * Opened DNS Manager (`dnsmgmt.msc`) on `WinSRV1`.
   * Under Server Properties > **Forwarders**, deleted `192.168.2.254`.
   * Added the ISP Gateway IP directly: `192.168.99.254`.

### Phase B: Snort Engine & Custom Filtering Rules
1. **Corrected Custom Rule Syntax:**
   * Navigated to **Services > Snort > Interfaces > WAN (vmx0) > WAN Rules**.
   * Selected Category: `custom.rules`.
   * Defined rule definitions:
     ```text
     drop tcp any any <> 192.168.99.0/24 any (flags:FPU; msg:"Possible XMAS scan"; sid:100001; rev:1;)
     alert udp any any -> any 53 (msg:"Starcity DNS Access Attempt"; content:"starcity"; nocase; sid:100002; rev:1;)
     alert tcp any any -> any 80 (msg:"Starcity HTTP Access Attempt"; content:"starcity.com.ph"; nocase; sid:100003; rev:1;)
     alert tcp any any -> any 443 (msg:"Starcity HTTPS Access Attempt"; tls.sni; content:"starcity.com.ph"; nocase; sid:100004; rev:1;)
     ```
2. **Reloaded Interface:**
   * Under **Snort Interfaces**, restarted `WAN (vmx0)`.
   * Verified successful compilation under **Status > System Logs > Packages > Snort**.

### Phase C: Active Directory LDAP Integration
1. **Configured Directory Binding:**
   * Navigated to **System > User Manager > Authentication Servers**.
   * Added / updated server entry:
     * **Descriptive name:** `WinSRV1_AD`
     * **Type:** `LDAP`
     * **Hostname or IP address:** `192.168.2.10`
     * **Port value:** `389`
     * **Transport:** `Standard TCP`
     * **Peer Certificate Authority:** `WinSRV3_CA` (or `Global Root CA List`)
     * **Search scope:** `Entire Subtree`
     * **Base DN:** `DC=manila,DC=com`
     * **Authentication containers:** `OU=IT,OU=manila,DC=manila,DC=com`
     * **Bind user DN:** `CN=Administrator,CN=Users,DC=manila,DC=com`
     * **Bind password:** `P@ssw0rd`
     * **Initial Template:** `Microsoft Active Directory`
     * **User naming attribute:** `sAMAccountName`
     * **Group member attribute:** `member`
     * **Extended query:** Checked
     * **Query Filter:** `&(objectClass=user)(memberOf=CN=VPN Users,OU=manila,DC=manila,DC=com)`

### Phase D: Certificate Management & OpenVPN Remote Access
1. **WinSRV3 CA Public Key Import:**
   * Navigated to **System > Cert. Manager > CAs**.
   * Imported Base-64 X.509 text exported from `WinSRV3`.
   * Set **Next Certificate Serial:** `1`.
2. **Server Certificate Provisioning:**
   * Navigated to **System > Cert. Manager > Certificates**.
   * Created server certificate `OpenVPN_Server_Cert` signed by the internal CA authority.
   * Subject Alternative Names: Included WAN IP `192.168.99.123`.
3. **OpenVPN Server Daemon Configuration:**
   * Navigated to **VPN > OpenVPN > Servers**.
   * **Server Mode:** `Remote Access (User Auth)`
   * **Backend for Authentication:** `WinSRV1_AD`
   * **Protocol:** `UDP on IPv4 only`
   * **Interface:** `WAN`
   * **Local Port:** `1194`
   * **Server Certificate:** `OpenVPN_Server_Cert`
   * **IPv4 Tunnel Network:** `10.0.8.0/24`
   * **IPv4 Local Network(s):** `172.16.100.0/24, 192.168.2.0/24, 192.168.1.0/24`
   * **DNS Default Domain:** `manila.com`
   * **DNS Server 1:** `192.168.2.10`
4. **Client Export & Enrollment:**
   * Navigated to **VPN > OpenVPN > Client Export**.
   * Exported standard bundle/installer configured for WAN IP `192.168.99.123`.
   * Installed profile on **WinClient3** and authenticated user `C1`.

---

## 4. Operational Verification Commands & Evidence

### 1. DNS Resolution & Domain Filtering
```cmd
:: Executed on WINSRV1 (Forwarder verification)
nslookup www.nationalmuseum.gov.ph 192.168.99.254
nslookup www.nationalmuseum.gov.ph 127.0.0.1

:: Executed on Client1 (Client-side access and filtering verification)
ipconfig /flushdns
nslookup www.nationalmuseum.gov.ph
nslookup www.starcity.com.ph
```
* **Result:** `www.nationalmuseum.gov.ph` resolved successfully and web pages loaded. Access to `www.starcity.com.ph` timed out/was blocked, generating alerts on pfSense with SID `100002`.

### 2. LDAP Authentication Diagnostics
* Executed inside pfSense WebGUI: **Diagnostics > Authentication**.
  * Server: `WinSRV1_AD`
  * Username: `C1`
  * Password: `P@ssw0rd`
* **Result:** `User C1 authenticated successfully. This user is a member of groups: VPN Users, WebUsers, Domain Users.`

### 3. OpenVPN Remote Tunnel & DMZ Access
* Executed from connected remote client (`WinClient3` - `10.0.8.2`):
```cmd
:: Verify routing to Domain Controller & Services
ping 192.168.2.10
nslookup winsrv1.manila.com 192.168.2.10

:: Verify routing to DMZ Web Server
ping 192.168.1.10
```
* Web browser verification: Navigation to `https://192.168.1.10` loaded internal DMZ resources across the encrypted tunnel.

### 4. Snort XMAS Scan Detection Test
```cmd
:: Executed on WinClient3 targeting pfSense WAN
nmap -sX -p 80,443 192.168.99.123
```
* **Result:** Packets matching flags `FPU` dropped per SID `100001` and registered under **Services > Snort > Alerts**.