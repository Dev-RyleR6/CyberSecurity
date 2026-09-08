# MA1 Setup

  **FOR EVENT ORGANIZERS / EXPERTS ONLY**  

**VM Setup & Configuration Guide**

Module A1 — Apache Website Security Assessment

*WorldSkills ASEAN Manila 2025 — Cyber Security Skill 54*

**This guide is for setting up competition-ready VMs on the ESXi host. Do not distribute to competitors.**

## **Quick Reference — Final VM State**

| Item | Value |
| :---- | :---- |
| **ESXi Host IP** | 192.168.1.1 |
| **ESXi Expert User** | amuser  /  Mt Blanc |
| **Competitor Laptop User** | Competitor0  /  CharterDressing |
| **All VM Password** | P@ssw0rd (root, Administrator, all domain users) |
| **DC IP** | 192.168.1.10 (static) |
| **WWW IP** | 192.168.1.20 (static) |
| **AMclient IP** | DHCP from DC |
| **Domain** | grimshay.ca |
| **Website URL** | https://www.grimshay.ca |
| **AD Group to create** | webusers (members: user1, user2 — non-members: user3, user4) |
| **Apache vhost root** | /var/www/grimshay/ |

**⚠ Warning:** *The WWW machine must be deliberately misconfigured. The whole point of MA1 is for competitors to find these vulnerabilities. Do NOT harden the Apache installation — follow Part 4 exactly to plant the required weaknesses.*

# **Part 1 — ESXi Host Preparation**

Before creating any VMs, verify the ESXi host is ready to accept connections from competitor laptops.

## **Step 1: Configure ESXi Network**

**Step 1:** Log in to the ESXi host web UI at: https://192.168.1.1

**Step 2:** Go to Networking → Virtual Switches

**Step 3:** Confirm a vSwitch exists connecting all VMs to the same LAN (192.168.1.0/24)

**Step 4:** Create a Port Group named LAN if it does not exist:

* Click Add Port Group

  * Name: LAN

  * VLAN ID: 0

  * Virtual Switch: vSwitch0

**Step 5:** All four VMs (DC, WWW, AMclient1, AMclient2) must be connected to this LAN port group

## **Step 2: Create the Expert ESXi Account**

**Step 1:** Go to Host → Manage → Security & Users → Users

**Step 2:** Click Add User

**Step 3:** Username: amuser

**Step 4:** Password: Mt Blanc

**Step 5:** Assign role: Read Only (competitors must only see their own VMs, not modify ESXi)

**Step 6:** Click Add

**📝 Note:** *The amuser account is what competitors use to log in to VMware Workstation to access their VMs. It should have Read Only rights at the host level but Full Access to the specific VM folder for each team.*

## **Step 3: Create the Competitor Laptop Local Account**

**Step 1:** On each competitor laptop, open Settings → Accounts → Other Users → Add someone else

**Step 2:** Username: Competitor0

**Step 3:** Password: CharterDressing

**Step 4:** Account type: Standard User

**Step 5:** Install VMware Workstation on the laptop if not already present

**Step 6:** Pre-configure VMware Workstation to connect to: 192.168.1.1 with amuser / Mt Blanc

**✓ Tip:** *Install Chrome, Putty, and Wireshark on the AMclient VMs — not on the competitor laptops. The laptops only need VMware Workstation.*

# **Part 2 — DC VM Setup (Domain Controller)**

The DC machine hosts Active Directory for the grimshay.ca domain and serves as the DNS server for all VMs.

## **VM Specifications**

| Spec | Value |
| :---- | :---- |
| **VM Name** | DC |
| **OS** | Windows Server 2019 or 2022 (Desktop Experience) |
| **RAM** | 4 GB minimum |
| **Disk** | 60 GB |
| **Network** | LAN port group — 192.168.1.10 / 255.255.255.0 |
| **Gateway** | 192.168.1.1 (ESXi host acts as gateway — or leave blank if no internet routing needed) |
| **DNS** | 192.168.1.10 (points to itself after AD DS install) |
| **Hostname** | DC |
| **Domain** | grimshay.ca |
| **Administrator Password** | P@ssw0rd |

## **Step 1: Install Windows Server and Set IP**

**Step 1:** Install Windows Server — select Desktop Experience during installation

**Step 2:** Set Administrator password to: P@ssw0rd

**Step 3:** Open Control Panel → Network and Sharing Center → Change adapter settings

**Step 4:** Right-click the NIC → Properties → IPv4 → Use the following address:

IP Address:   192.168.1.10

Subnet Mask:  255.255.255.0

Gateway:      192.168.1.1

DNS Server:   192.168.1.10  (itself — update after AD DS is installed)

**Step 5:** Set hostname to DC: Right-click Start → System → Rename this PC → DC → Restart

## **Step 2: Install AD DS and Promote to Domain Controller**

**Step 1:** Open Server Manager → Manage → Add Roles and Features

**Step 2:** Select Active Directory Domain Services → Add Features → Next → Install

**Step 3:** After installation click Promote this server to a domain controller

**Step 4:** Select Add a new forest

**Step 5:** Root domain name: grimshay.ca → Next

**Step 6:** Set DSRM password: P@ssw0rd → Next through all defaults → Install

**Step 7:** Server will restart automatically

**Step 8:** After restart, log in as: GRIMSHAY\\Administrator / P@ssw0rd

## **Step 3: Create AD Users and Groups**

Open Active Directory Users and Computers from Server Manager → Tools.

### **Create the webusers Group**

**Step 1:** Expand grimshay.ca → right-click Users → New → Group

**Step 2:** Group name: webusers

**Step 3:** Group scope: Global — Group type: Security → OK

### **Create Domain Users**

**Step 1:** Right-click Users → New → User for each of the following:

| Username | Full Name / Notes |
| :---- | :---- |
| **user1** | Web User 1 — ADD to webusers group |
| **user2** | Web User 2 — ADD to webusers group |
| **user3** | Regular User 3 — do NOT add to webusers |
| **user4** | Regular User 4 — do NOT add to webusers |
| **competitor** | Generic competitor test account — do NOT add to webusers |

**Step 2:** Set password P@ssw0rd for all users — uncheck 'User must change password at next logon' — check 'Password never expires'

**Step 3:** Add user1 and user2 to webusers group:

* Double-click user1 → Member Of → Add → type webusers → Check Names → OK

  * Repeat for user2

## **Step 4: Configure DNS for www.grimshay.ca**

**Step 1:** Open Server Manager → Tools → DNS

**Step 2:** Expand DC → Forward Lookup Zones → grimshay.ca

**Step 3:** Right-click grimshay.ca → New Host (A or AAAA)

**Step 4:** Fill in:

Name:       www

IP Address: 192.168.1.20

**Step 5:** Click Add Host → OK

**Step 6:** Verify: open Command Prompt on DC and run:

nslookup www.grimshay.ca 192.168.1.10

Expected: Address: 192.168.1.20

## **Step 5: Configure DHCP for AMclients**

**Step 1:** Open Server Manager → Manage → Add Roles and Features → DHCP Server → Install

**Step 2:** After install, open Tools → DHCP

**Step 3:** Expand DC → IPv4 → right-click → New Scope

**Step 4:** Configure the scope:

Name:           LAN Clients

Start IP:       192.168.1.100

End IP:         192.168.1.200

Subnet Mask:    255.255.255.0

Default Gateway: 192.168.1.1

DNS Server:     192.168.1.10

**Step 5:** Activate the scope

# **Part 3 — WWW VM Setup (Base Linux \+ Apache)**

The WWW machine is the primary assessment target. First, set it up cleanly, then apply deliberate misconfigurations in Part 4\.

## **VM Specifications**

| Spec | Value |
| :---- | :---- |
| **VM Name** | WWW |
| **OS** | Rocky Linux 9 (or CentOS Stream 9\) |
| **RAM** | 2 GB minimum |
| **Disk** | 40 GB |
| **Network** | LAN port group — 192.168.1.20 / 255.255.255.0 |
| **Gateway** | 192.168.1.1 |
| **DNS** | 192.168.1.10 (DC) |
| **Hostname** | www.grimshay.ca |
| **root Password** | P@ssw0rd |
| **competitor account** | competitor / P@ssw0rd |

## **Step 1: Install Rocky Linux 9**

**Step 1:** Boot the VM from the Rocky Linux 9 ISO

**Step 2:** Choose: Server (no GUI) installation type — minimal install is fine

**Step 3:** During installation set:

* Hostname: www.grimshay.ca

  * Root password: P@ssw0rd

  * Create additional user: competitor / P@ssw0rd

**Step 4:** Complete installation and reboot

## **Step 2: Configure Static IP**

**Step 1:** Log in as root

**Step 2:** Edit the network configuration:

nmcli con mod ens160 ipv4.method manual \\

  ipv4.addresses 192.168.1.20/24 \\

  ipv4.gateway 192.168.1.1 \\

  ipv4.dns 192.168.1.10

nmcli con up ens160

**Step 3:** Verify connectivity to DC:

ping 192.168.1.10

nslookup grimshay.ca 192.168.1.10

**📝 Note:** *The interface name may be different on your system (ens192, eth0, etc). Run: ip addr show — to find the correct interface name.*

## **Step 3: Install Apache (httpd)**

**Step 1:** Install Apache:

dnf install \-y httpd mod\_ssl

**Step 2:** Enable and start Apache:

systemctl enable httpd

systemctl start httpd

**Step 3:** Open firewall ports:

firewall-cmd \--permanent \--add-service=http

firewall-cmd \--permanent \--add-service=https

firewall-cmd \--reload

**Step 4:** Verify Apache is running:

systemctl status httpd

curl http://localhost

## **Step 4: Create the Website Content**

**Step 1:** Create the web root directory:

mkdir \-p /var/www/grimshay

**Step 2:** Create a simple index page:

cat \> /var/www/grimshay/index.html \<\< 'EOF'

\<\!DOCTYPE html\>

\<html\>\<head\>\<title\>Grimshay Associates\</title\>\</head\>

\<body\>

\<h1\>Welcome to Grimshay Associates\</h1\>

\<p\>This is an internal company website.\</p\>

\<p\>Access is restricted to authorized personnel only.\</p\>

\</body\>\</html\>

EOF

**Step 3:** Create subdirectories with files (needed for directory listing vulnerability):

mkdir \-p /var/www/grimshay/files

mkdir \-p /var/www/grimshay/images

mkdir \-p /var/www/grimshay/backup

echo 'employee\_data.csv' \> /var/www/grimshay/files/employee\_data.csv

echo 'salary\_2024.xlsx' \> /var/www/grimshay/files/salary\_2024.xlsx

echo 'config\_backup' \> /var/www/grimshay/backup/httpd.conf.bak

**📝 Note:** *These files simulate sensitive content that should not be publicly browsable. When directory listing is enabled, competitors will be able to see them — that is the vulnerability.*

## **Step 5: Create the Virtual Host Configuration**

**Step 1:** Create the vhost config file:

cat \> /etc/httpd/conf.d/grimshay.conf \<\< 'EOF'

\<VirtualHost \*:80\>

    ServerName www.grimshay.ca

    ServerAlias grimshay.ca

    DocumentRoot /var/www/grimshay

    ErrorLog /var/log/httpd/grimshay\_error.log

    CustomLog /var/log/httpd/grimshay\_access.log combined

\</VirtualHost\>

EOF

**Step 2:** Create the SSL vhost config:

cat \> /etc/httpd/conf.d/grimshay-ssl.conf \<\< 'EOF'

\<VirtualHost \*:443\>

    ServerName www.grimshay.ca

    DocumentRoot /var/www/grimshay

    SSLEngine on

    SSLCertificateFile /etc/pki/tls/certs/grimshay.crt

    SSLCertificateKeyFile /etc/pki/tls/private/grimshay.key

\</VirtualHost\>

EOF

## **Step 6: Create a Self-Signed SSL Certificate**

**Step 1:** Generate the self-signed certificate:

openssl req \-x509 \-nodes \-days 365 \-newkey rsa:2048 \\

  \-keyout /etc/pki/tls/private/grimshay.key \\

  \-out /etc/pki/tls/certs/grimshay.crt \\

  \-subj "/CN=www.grimshay.ca/O=Grimshay/C=CA"

**Step 2:** Restart Apache:

systemctl restart httpd

**Step 3:** Test HTTPS from the WWW machine itself:

curl \-k https://localhost

**📝 Note:** *The self-signed certificate is intentional. When competitors browse to https://www.grimshay.ca they will get a certificate warning. This is a vulnerability to document (untrusted/self-signed certificate).*

# **Part 4 — Planting Deliberate Vulnerabilities on WWW**

This is the most critical section for the competition. Apply ALL of the following misconfigurations. These are what competitors must discover and document.

**⚠ Warning:** *Apply every item in this section. Do not skip any. The competition scoring is based on competitors identifying these specific weaknesses.*

## **Vulnerability Summary Table**

| Vulnerability to Plant | Config Change Required | How Competitors Detect It |
| :---- | :---- | :---- |
| **Apache Version Disclosure** | Default httpd.conf — do not add ServerTokens Prod or ServerSignature Off | curl \-I shows Server: Apache/2.x.x (Rocky Linux) header |
| **Directory Listing Enabled** | Add Options Indexes to the Directory block in grimshay.conf | Browser to https://www.grimshay.ca/files/ shows file listing |
| **No Authentication / Access Control** | Do not add any AuthType, AuthName, or Require directives to the vhost | Any user (including non-webusers) can access the site without credentials |
| **Weak SSL Protocol** | Add SSLProtocol all (allows SSLv3, TLSv1.0) | SSL scan or grep ssl.conf shows old protocols allowed |
| **Missing Security Headers** | Do not add X-Frame-Options, X-Content-Type-Options, or CSP headers | curl \-I shows none of the security headers in the response |
| **World-Writable Web Directory** | chmod 777 /var/www/grimshay | ls \-la /var/www/grimshay shows drwxrwxrwx permissions |

## **Vulnerability 1: Enable Apache Version Disclosure**

**🔧 Expert Action: Verify that ServerTokens and ServerSignature are NOT set to hide the version.**

**Step 1:** Open the main httpd.conf:

nano /etc/httpd/conf/httpd.conf

**Step 2:** Search for ServerTokens — if it exists and is set to Prod, comment it out or delete it:

\# ServerTokens Prod    \<-- comment this out if present

**Step 3:** Search for ServerSignature — if set to Off, change it to On:

ServerSignature On

**Step 4:** Save and restart Apache:

systemctl restart httpd

**Step 5:** Verify the version is visible:

curl \-I http://localhost

You should see: Server: Apache/2.4.xx (Rocky Linux) in the response.

## **Vulnerability 2: Enable Directory Listing**

**🔧 Expert Action: Add the Indexes option to the grimshay vhost so competitors can browse directories.**

**Step 1:** Edit the grimshay vhost config:

nano /etc/httpd/conf.d/grimshay.conf

**Step 2:** Update the file to include a Directory block with Options Indexes:

\<VirtualHost \*:80\>

    ServerName www.grimshay.ca

    DocumentRoot /var/www/grimshay

    \<Directory /var/www/grimshay\>

        Options Indexes FollowSymLinks

        AllowOverride None

        Require all granted

    \</Directory\>

\</VirtualHost\>

**Step 3:** Apply the same Directory block to the SSL vhost in grimshay-ssl.conf

**Step 4:** Restart Apache:

systemctl restart httpd

**Step 5:** Verify from the server:

curl http://localhost/files/

You should see an HTML page listing: employee\_data.csv, salary\_2024.xlsx

## **Vulnerability 3: No Authentication (webusers Group Not Enforced)**

**🔧 Expert Action: Do NOT configure any authentication. The vhost must allow all traffic with no login prompt.**

**Step 1:** Verify there are NO Auth directives in any config file:

grep \-r 'AuthType\\|AuthName\\|Require valid-user' /etc/httpd/conf.d/

This command must return NO output. If any Auth lines exist, remove them.

**Step 2:** The Require directive in the Directory block should be:

Require all granted

NOT: Require group webusers

**Step 3:** Verify by accessing the site from AMclient as user3 (not in webusers group) — the site should load without any username/password prompt.

## **Vulnerability 4: Weak SSL/TLS Protocol Configuration**

**🔧 Expert Action: Allow old SSL/TLS protocols in the SSL configuration.**

**Step 1:** Edit the main SSL configuration:

nano /etc/httpd/conf.d/ssl.conf

**Step 2:** Find the SSLProtocol line and change it to allow all protocols:

SSLProtocol all

**Step 3:** Comment out any SSLCipherSuite hardening lines if present:

\# SSLCipherSuite HIGH:\!aNULL:\!MD5

**Step 4:** Save and restart:

systemctl restart httpd

**📝 Note:** *SSLProtocol all allows TLSv1.0, TLSv1.1, and even SSLv3 (on older Apache builds). This is a recognizable misconfiguration that competitors should spot when reviewing the config file.*

## **Vulnerability 5: Missing HTTP Security Headers**

**🔧 Expert Action: Ensure mod\_headers is loaded but NO security headers are configured.**

**Step 1:** Verify mod\_headers is loaded (it should be by default):

httpd \-M | grep headers

**Step 2:** Confirm there are NO Header directives in any config file:

grep \-r 'X-Frame-Options\\|X-Content-Type\\|Content-Security' /etc/httpd/conf.d/

This must return NO output. If any header lines exist, remove them.

**Step 3:** Verify from the server — these headers should be ABSENT from the response:

curl \-I http://localhost

The output must NOT contain: X-Frame-Options, X-Content-Type-Options, Content-Security-Policy, Strict-Transport-Security.

## **Vulnerability 6: World-Writable Web Directory**

**🔧 Expert Action: Set the web root permissions to 777 so any local user can modify web files.**

**Step 1:** Set world-writable permissions on the web root:

chmod 777 /var/www/grimshay

chmod 777 /var/www/grimshay/files

chmod 777 /var/www/grimshay/images

chmod 777 /var/www/grimshay/backup

**Step 2:** Also set the files themselves to 666:

chmod 666 /var/www/grimshay/index.html

chmod 666 /var/www/grimshay/files/\*

**Step 3:** Verify:

ls \-la /var/www/

You should see: drwxrwxrwx for the grimshay directory.

## **Final Apache Restart and Smoke Test**

**Step 1:** Restart Apache after all changes:

systemctl restart httpd

**Step 2:** Confirm Apache is running:

systemctl status httpd

**Step 3:** Confirm all vulnerabilities are in place:

curl \-I http://localhost           \# check Server: header shows version

curl http://localhost/files/       \# check directory listing shows files

curl \-k https://localhost          \# check HTTPS works with self-signed cert

ls \-la /var/www/grimshay           \# check 777 permissions

grep SSLProtocol /etc/httpd/conf.d/ssl.conf   \# check shows 'all'

**✓ Tip:** *Run all five checks above before closing the WWW VM. If any vulnerability is missing, fix it now before the competition starts.*

# **Part 5 — AMclient VM Setup (Domain-Joined Windows Clients)**

Two AMclient machines are provided for the two competitors to test concurrently. Both have identical configurations.

## **VM Specifications**

| Spec | Value |
| :---- | :---- |
| **VM Names** | AMclient1  and  AMclient2 |
| **OS** | Windows 10 or Windows 11 |
| **RAM** | 4 GB minimum each |
| **Disk** | 60 GB each |
| **Network** | LAN port group — DHCP from DC |
| **Domain** | grimshay.ca |
| **Local Admin Password** | P@ssw0rd |
| **Required Software** | Google Chrome, Putty, Wireshark |

## **Step 1: Install Windows and Set Hostname**

**Step 1:** Install Windows 10/11 — set local Administrator password to P@ssw0rd during setup

**Step 2:** For AMclient1: Right-click Start → System → Rename this PC → AMclient1 → Restart

**Step 3:** For AMclient2: same process, set name to AMclient2

## **Step 2: Configure Network (DHCP)**

**Step 1:** Open Control Panel → Network and Sharing Center → Change adapter settings

**Step 2:** Right-click NIC → Properties → IPv4

**Step 3:** Select Obtain an IP address automatically

**Step 4:** Select Obtain DNS server address automatically

**Step 5:** Click OK → verify DHCP assigns an address from 192.168.1.100–200 from DC

**Step 6:** Verify DNS resolves the domain:

nslookup grimshay.ca

Expected: Server: 192.168.1.10  /  Address: 192.168.1.10

## **Step 3: Join the Domain**

**Step 1:** Right-click Start → System → Advanced system settings → Computer Name → Change

**Step 2:** Select Member of: Domain

**Step 3:** Type: grimshay.ca → click OK

**Step 4:** Enter domain credentials when prompted: GRIMSHAY\\Administrator / P@ssw0rd

**Step 5:** Click OK on the welcome message → Restart

**Step 6:** After restart, log in as: GRIMSHAY\\user1 / P@ssw0rd to confirm domain join worked

**⚠ Warning:** *If domain join fails, verify: (1) DC is running, (2) AMclient's DNS is set to 192.168.1.10, (3) the DC DHCP scope has issued an IP to the AMclient. Run ipconfig /all on the AMclient to check.*

## **Step 4: Install Required Software**

**Step 1:** Install Google Chrome — download from google.com/chrome or copy installer from a USB

**Step 2:** Install Putty — download from putty.org or copy installer

**Step 3:** Install Wireshark — download from wireshark.org or copy installer

**Step 4:** During Wireshark install, select: Install Npcap — this is required for packet capture

**Step 5:** Create a desktop shortcut for each tool so competitors can find them quickly

**Step 6:** Repeat all software installs on AMclient2

**✓ Tip:** *Pre-copy the competition document (WSA2025\_TP54\_MA1\_actual\_en.docx) to the Desktop of Competitor0's account on each competitor laptop. This saves time at the start of the competition.*

## **Step 5: Configure Chrome to Ignore Self-Signed Certificate Errors (Optional)**

**📝 Note:** *By default Chrome will block access to HTTPS sites with self-signed certs. You can allow competitors to bypass this, or leave it as-is since the certificate error is itself a finding.*

**Step 1:** If you want competitors to be able to bypass the error (recommended for testing):

* Open Chrome → go to chrome://flags

  * Search for: Allow invalid certificates for resources loaded from localhost

  * This only helps for localhost — for remote sites, competitors click Advanced → Proceed anyway

**Step 2:** Alternatively, instruct competitors to click Advanced → Proceed to www.grimshay.ca (unsafe) to access the site past the warning

# **Part 6 — Placing the Competition Document on Competitor Laptops**

The competition document must be available digitally on the competitor laptop so competitors can fill in the Appendix and save their submission.

## **Step 1: Place the Document**

**Step 1:** Copy WSA2025\_TP54\_MA1\_actual\_en.docx to the Desktop of the Competitor0 account on each competitor laptop

**Step 2:** Verify the document opens correctly in Microsoft Word

**Step 3:** Verify the Appendix section with Executive Summary and Table 1 is present and empty — ready for competitors to fill in

## **Step 2: Verify the Submission Instructions**

The document instructs competitors to:

* Fill in the Executive Summary (max 500 words) in Appendix 1

* Fill in Table 1 with their two selected vulnerabilities

* Save the file with their country code in the filename (e.g., PHL\_WSA2025\_MA1.docx)

* Leave the saved file on the Desktop for expert collection

**⚠ Warning:** *As an expert, you collect the completed document from the competitor laptop Desktop after they signal they are done. Do not shut down the VMs before you have collected and reviewed their submission.*

# **Part 7 — Pre-Competition Checklist**

Run through every item below on competition day before competitors enter the room.

## **ESXi Host**

| Check | Expected State |
| :---- | :---- |
| **ESXi accessible at 192.168.1.1** | Web UI loads, amuser can log in with password Mt Blanc |
| **LAN virtual switch exists** | All VMs connected to LAN port group 192.168.1.0/24 |
| **Competitor laptops can connect via VMware Workstation** | VMs listed in VMware Workstation after connecting to 192.168.1.1 |

## **DC Virtual Machine**

| Check | Expected State |
| :---- | :---- |
| **DC powered on, AD DS green in Server Manager** | Active Directory operational |
| **Domain: grimshay.ca** | Active Directory domain exists |
| **DNS record: www.grimshay.ca → 192.168.1.20** | nslookup www.grimshay.ca returns 192.168.1.20 |
| **webusers group exists with user1 and user2 as members** | Visible in AD Users and Computers |
| **user3, user4, competitor accounts exist, NOT in webusers** | Verified in AD |
| **DHCP scope 192.168.1.100–200 active** | Scope shows Active in DHCP console |

## **WWW Virtual Machine**

| Check | Expected State |
| :---- | :---- |
| **IP: 192.168.1.20 static** | ip addr show confirms 192.168.1.20/24 |
| **Apache running** | systemctl status httpd shows Active (running) |
| **https://www.grimshay.ca loads from AMclient** | Site loads (self-signed cert warning expected) |
| **Server: header shows Apache version** | curl \-I shows Server: Apache/2.4.xx (Rocky Linux) |
| **Directory listing at /files/ works** | Browser shows employee\_data.csv and salary\_2024.xlsx |
| **No authentication required on the site** | user3 (non-webusers) can access without password |
| **SSLProtocol all in ssl.conf** | grep SSLProtocol /etc/httpd/conf.d/ssl.conf shows 'all' |
| **No security headers in response** | curl \-I shows no X-Frame-Options or CSP |
| **/var/www/grimshay is 777** | ls \-la /var/www/ shows drwxrwxrwx for grimshay |

## **AMclient VMs**

| Check | Expected State |
| :---- | :---- |
| **AMclient1 and AMclient2 joined to grimshay.ca** | whoami shows GRIMSHAY\\username after domain login |
| **Chrome installed** | Google Chrome shortcut on Desktop |
| **Putty installed** | Putty shortcut on Desktop |
| **Wireshark installed** | Wireshark shortcut on Desktop |
| **user1 (webusers member) can log in** | Login as GRIMSHAY\\user1 / P@ssw0rd succeeds |
| **user3 (non-member) can log in** | Login as GRIMSHAY\\user3 / P@ssw0rd succeeds |

## **Competitor Laptops**

| Check | Expected State |
| :---- | :---- |
| **Competitor0 account exists with password CharterDressing** | Login succeeds |
| **VMware Workstation installed** | Application opens and connects to 192.168.1.1 |
| **Competition document on Desktop** | WSA2025\_TP54\_MA1\_actual\_en.docx present and opens |
| **Microsoft Word installed** | Document opens and is editable |

**✓ Tip:** *If any check fails, go back to the relevant Part and re-do that section. Fix in this order: ESXi network → DC (AD, DNS, DHCP) → WWW (Apache \+ vulnerabilities) → AMclient (domain join \+ software) → laptops.*

*WorldSkills ASEAN Manila 2025 — Cyber Security Skill 54 — Module A1 — EXPERT SETUP GUIDE — NOT FOR DISTRIBUTION TO COMPETITORS*

# MA1 Tasks

**Network Setup Guide**

Module A1 — Apache Website Security Assessment

*GUI Version — WorldSkills ASEAN Manila 2025*

Cyber Security Skill 54 — Module A1 — Based on WSA2025\_TP54\_MA1 Competition Document

## **Quick Reference**

| Item | Value |
| :---- | :---- |
| **ESXi IP Address** | 192.168.1.1 |
| **ESXi User** | amuser |
| **ESXi Password** | Mt Blanc |
| **Competitor Laptop User** | Competitor0 |
| **Competitor Laptop Password** | CharterDressing |
| **All VM Account Password** | P@ssw0rd |
| **DC Role** | Active Directory, DNS |
| **WWW Role** | Apache Web Server (www.grimshay.ca) |
| **AMclient Role** | Domain-joined test clients |
| **Assessment Scope** | Apache installation and www.grimshay.ca website only |

**📝 Note:** *Module A1 is a security assessment exercise, NOT a penetration test. You are investigating an existing configuration on the WWW machine and documenting vulnerabilities found.*

# **Part 1 — Network Overview and VM Summary**

The Module A1 environment consists of three types of machines connected on a shared LAN. All machines are hosted on an ESXi server and accessed via VMware Workstation from the competitor laptops.

## **VM Summary Table**

| VM Name | Role | IP Address | OS / Notes | Credentials |
| :---- | :---- | :---- | :---- | :---- |
| DC | Domain Controller / DNS | 192.168.1.x | Windows Server — AD DS, DNS. Hosts domain grimshay.ca with webusers group. | Administrator / P@ssw0rd |
| WWW | Apache Web Server | 192.168.1.x | Linux — Apache httpd hosting www.grimshay.ca. Primary assessment target. | root / P@ssw0rd |
| AMclient 1 | Domain Client | DHCP | Windows — domain joined. Chrome, Putty, Wireshark installed. | Domain user / P@ssw0rd |
| AMclient 2 | Domain Client | DHCP | Windows — domain joined. Chrome, Putty, Wireshark installed. | Domain user / P@ssw0rd |

**📝 Note:** *The exact IP addresses for DC and WWW are not printed in the competition document — they are visible in the VMware topology diagram on competition day. Verify all IPs when you first log in.*

## **Network Topology Summary**

* All VMs are on the same LAN segment (192.168.1.0/24) connected through the ESXi virtual switch.

* The DC is the DNS server for all machines. AMclients receive their IP via DHCP from the DC.

* The WWW machine has a static IP and hosts the website at www.grimshay.ca.

* The DC holds an AD group called webusers — only members of this group should have access to the website.

* There is no internet connectivity during the competition — the ISP machine is not present in MA1.

# **Part 2 — Accessing the Environment**

## **Step 1: Log On to the Competitor Laptop**

**Step 1:** Power on the competitor laptop

**Step 2:** Log in with:

User:     Competitor0

Password: CharterDressing

## **Step 2: Connect to ESXi via VMware Workstation**

**Step 1:** Open VMware Workstation on the competitor laptop

**Step 2:** Go to File → Connect to Server

**Step 3:** Enter the ESXi server details:

IP:       192.168.1.1

User:     amuser

Password: Mt Blanc

**Step 4:** Click Connect

**Step 5:** You will see the list of VMs for your team — do NOT access VMs from other teams

## **Step 3: Power On and Verify VMs**

**Step 1:** Right-click the DC VM → Power On

**Step 2:** Right-click the WWW VM → Power On

**Step 3:** Right-click AMclient 1 → Power On

**Step 4:** Right-click AMclient 2 → Power On (optional — for concurrent testing)

**Step 5:** Wait for all machines to fully boot before proceeding

**✓ Tip:** *Boot order: start DC first, wait \~60 seconds for AD to initialize, then boot WWW and AMclients.*

## **Step 4: Verify DC is Operational**

**Step 1:** Open the DC console in VMware Workstation

**Step 2:** Log in as: Administrator / P@ssw0rd

**Step 3:** Open Server Manager and confirm AD DS and DNS show green status

**Step 4:** Open Active Directory Users and Computers

**Step 5:** Expand the domain and verify the webusers group exists

**Step 6:** Note which users are members of webusers — this is relevant for website access testing

## **Step 5: Verify WWW Machine Network**

**Step 1:** Open the WWW console in VMware Workstation

**Step 2:** Log in as: root / P@ssw0rd

**Step 3:** Run the following to confirm the IP address:

ip addr show

\# OR

ifconfig

**Step 4:** Confirm the machine can resolve the domain:

nslookup grimshay.ca 192.168.1.x   \# replace with DC IP

**Step 5:** Confirm Apache is running:

systemctl status httpd

\# OR

service apache2 status

**✓ Tip:** *If Apache is not running, start it: systemctl start httpd — but note this for your vulnerability report as an unexpected service state.*

## **Step 6: Verify Website Access from AMclient**

**Step 1:** On AMclient 1, log in as a domain user (password: P@ssw0rd)

**Step 2:** Open Chrome

**Step 3:** Navigate to: https://www.grimshay.ca

**Step 4:** Note whether you are prompted for credentials, get a certificate error, or gain direct access

**Step 5:** Try with both a webusers group member and a non-member — document the difference

**⚠ Warning:** *Certificate errors on the HTTPS site are expected and important to note — self-signed or misconfigured certificates are a potential vulnerability to document.*

# **Part 3 — Apache Configuration Investigation**

This is the core of the MA1 assessment. Examine the Apache installation on the WWW machine and identify security issues. Work through each check below methodically.

## **Check 1: Apache Version Disclosure**

**Step 1:** On the WWW machine run:

httpd \-v

\# OR

apache2 \-v

**Step 2:** Also check the HTTP response header from AMclient using Putty or Chrome Developer Tools:

curl \-I http://www.grimshay.ca

**Step 3:** Look at the Server: header in the response

A response like Server: Apache/2.4.x (CentOS) exposes exact version information to attackers.

**📝 Note:** *Version disclosure allows attackers to look up known CVEs for that exact Apache version. The fix is to add ServerTokens Prod and ServerSignature Off to the Apache config.*

## **Check 2: Directory Listing Enabled**

**Step 1:** In Chrome on AMclient, navigate to a subdirectory of the site — try:

http://www.grimshay.ca/images/

http://www.grimshay.ca/files/

**Step 2:** If you see an Index of / page listing directory contents, directory listing is enabled

**Step 3:** On the WWW machine check the Apache config:

grep \-r 'Options' /etc/httpd/conf/ /etc/httpd/conf.d/

\# OR

grep \-r 'Options' /etc/apache2/

Look for Options Indexes — the word Indexes enables directory browsing.

**📝 Note:** *Directory listing allows anyone to browse the web root file structure, potentially exposing backup files, configuration files, and sensitive data.*

## **Check 3: SSL/TLS Certificate and Configuration**

**Step 1:** From AMclient, attempt to connect via HTTPS:

https://www.grimshay.ca

**Step 2:** Check for certificate warnings — is it self-signed? Expired? Wrong hostname?

**Step 3:** On WWW check the SSL configuration:

grep \-r 'SSLProtocol\\|SSLCipherSuite\\|SSLCertificate' /etc/httpd/conf.d/

**Step 4:** Check if weak protocols are allowed:

grep \-i 'SSLv2\\|SSLv3\\|TLSv1 ' /etc/httpd/conf.d/ssl.conf

Old protocols like SSLv2, SSLv3, and TLSv1.0 have known critical vulnerabilities (POODLE, DROWN).

## **Check 4: Authentication and Access Control (webusers group)**

**Step 1:** The competition brief states: only members of the webusers group should have access

**Step 2:** Test by logging into AMclient as a non-webusers domain user and browsing to the site

**Step 3:** On the WWW machine check the Apache .htaccess or config for authentication directives:

find /var/www/ \-name '.htaccess' \-exec cat {} \\;

grep \-r 'AuthType\\|AuthName\\|Require\\|Allow\\|Deny' /etc/httpd/conf/ /etc/httpd/conf.d/

If there is no authentication configured, anyone who can reach the web server can access the site regardless of domain group membership.

**📝 Note:** *If the site is supposed to restrict access to webusers group members only, the absence of authentication or authorization controls is a critical misconfiguration.*

## **Check 5: File and Directory Permissions on Web Root**

**Step 1:** On the WWW machine inspect the web root permissions:

ls \-la /var/www/html/

ls \-la /var/www/grimshay/   \# or wherever the vhost root is

**Step 2:** Check the ownership of web files:

stat /var/www/html/index.html

**Step 3:** Look for world-writable files or directories:

find /var/www/ \-perm \-o+w \-type f

find /var/www/ \-perm \-o+w \-type d

World-writable web directories allow any user on the system to modify web content, enabling defacement or injection attacks.

## **Check 6: Apache Running as Root**

**Step 1:** On the WWW machine check which user Apache runs as:

ps aux | grep httpd

ps aux | grep apache2

**Step 2:** The worker processes should be running as apache or www-data, NOT as root

**Step 3:** Also verify the Apache configuration:

grep \-i 'User\\|Group' /etc/httpd/conf/httpd.conf

Apache running as root means a successful exploit of any web vulnerability gives the attacker root-level access to the entire system.

## **Check 7: Unnecessary Apache Modules**

**Step 1:** List loaded Apache modules:

httpd \-M 2\>&1 | sort

\# OR

apache2ctl \-M 2\>&1 | sort

**Step 2:** Look for modules that should not be loaded in a production web server:

* mod\_status — exposes server statistics at /server-status

  * mod\_info — exposes full server config at /server-info

  * mod\_userdir — allows \~/username URLs

  * mod\_autoindex — enables directory listing

**Step 3:** Test if server-status is publicly accessible from AMclient:

http://www.grimshay.ca/server-status

**📝 Note:** *mod\_status and mod\_info can expose the server's internal configuration, loaded modules, and active requests to any visitor. They should be disabled or restricted to localhost.*

## **Check 8: HTTP Security Headers**

**Step 1:** From AMclient use curl or Chrome DevTools to examine HTTP response headers:

curl \-I https://www.grimshay.ca

**Step 2:** Check for the presence of security headers:

* X-Frame-Options — prevents clickjacking

  * X-Content-Type-Options — prevents MIME sniffing

  * Content-Security-Policy — restricts resource loading

  * Strict-Transport-Security (HSTS) — enforces HTTPS

  * Referrer-Policy — controls referrer information

The absence of security headers makes the site vulnerable to a range of client-side attacks including clickjacking and MIME-type confusion attacks.

# **Part 4 — Linux System Checks on WWW**

While the scope is Apache and the website, these system-level checks provide supporting context for your assessment findings.

## **Check OS and Installed Software**

**Step 1:** Identify the OS version:

cat /etc/os-release

uname \-a

**Step 2:** Check the Apache package version:

rpm \-q httpd      \# RHEL/CentOS

dpkg \-l apache2   \# Debian/Ubuntu

**Step 3:** Check for available updates (informational — no internet, so this may fail):

yum check-update httpd   \# RHEL/CentOS

apt list \--upgradable    \# Debian/Ubuntu

## **Check the Virtual Host Configuration**

**Step 1:** Find the virtual host config file for www.grimshay.ca:

ls /etc/httpd/conf.d/

ls /etc/apache2/sites-enabled/

**Step 2:** Read the vhost config:

cat /etc/httpd/conf.d/grimshay.conf

**Step 3:** Note and document:

* DocumentRoot path and permissions

* ServerName and ServerAlias values

* Any Directory directives and their Options

* Any authentication (Auth\*) directives

* SSL certificate paths and protocols

## **Check Local Firewall**

**Step 1:** Check firewall status on WWW:

systemctl status firewalld

iptables \-L \-n

**Step 2:** Verify which ports are open:

ss \-tlnp

netstat \-tlnp

**Step 3:** Document any unexpected open ports (beyond 80 and 443 for a web server)

**✓ Tip:** *An open port 22 (SSH) is expected for administration. Open ports beyond 80, 443, 22 should be noted and explained.*

# **Part 5 — Completing the Competition Appendix**

The deliverable for MA1 is an updated version of the competition document with the Appendix filled in. The Appendix has two sections: Executive Summary and Table 1\.

## **Executive Summary (max 500 words)**

Write a concise professional security assessment summary. Structure it as follows:

### **Paragraph 1 — Scope and Purpose**

State what was assessed, on which machine, and the overall purpose of the assessment. Example:

*A security assessment was conducted on the Apache web server hosting www.grimshay.ca. The assessment focused on the Apache installation configuration, website access controls, SSL/TLS configuration, and file system permissions. The objective was to determine whether the server is appropriately secured for hosting content accessible only to members of the webusers domain group.*

### **Paragraph 2 — Summary of Findings**

Briefly describe the overall security posture — how many issues were found and their severity. Example:

*The assessment identified multiple security weaknesses in the Apache configuration. The most critical finding is the absence of authentication controls, which means the site is accessible to any user regardless of domain group membership, contrary to the stated security requirement. Additional findings include Apache version disclosure, enabled directory listing, weak SSL configuration, and missing HTTP security headers.*

### **Paragraph 3 — Recommendations**

Provide a high-level recommendation. Example:

*Immediate remediation is recommended for the authentication control gap and SSL configuration issues. The server should not be used to host sensitive content until these issues are resolved. A full review of the Apache configuration against a security baseline (such as the CIS Apache HTTP Server Benchmark) is recommended before production deployment.*

**📝 Note:** *Keep the summary under 500 words. Focus on the two vulnerabilities you selected for Table 1, but also mention the overall security posture briefly.*

## 

## 

## **Table 1 — Two Selected Vulnerabilities**

Select the two vulnerabilities you consider most critical. Below is a reference table of common findings from this type of environment to help you complete Table 1 on competition day:

| Vulnerability | Location | Risk Level | Why It Matters | Recommended Fix |
| :---- | :---- | :---- | :---- | :---- |
| **No Authentication on Website** | /etc/httpd/ conf.d/ | **Critical** | Any network user can access the site. The webusers group access restriction stated in the brief is not enforced. An attacker or unauthorized user can read all website content without any credentials. | Implement Apache authentication using mod\_authnz\_ldap to authenticate against Active Directory, or use .htaccess with AuthType Basic / Digest and Require group webusers. |
| **Apache Version Disclosure** | HTTP Response Header | **High** | The Server: header reveals the exact Apache version and OS. Attackers can look up known CVEs for that version and craft targeted exploits without needing to probe further. | Add ServerTokens Prod and ServerSignature Off to httpd.conf. This reduces the Server header to just 'Apache' with no version details. |
| **Directory Listing Enabled** | /etc/httpd/ conf.d/ or .htaccess | **High** | Visitors can browse the full directory structure of the web root. This can expose backup files, configuration files, and other sensitive data not intended to be public. | Remove the Indexes option from all Directory directives: Options \-Indexes. Apply this in the VirtualHost config and the default httpd.conf. |
| **Weak SSL/TLS Config** | /etc/httpd/ conf.d/ssl.conf | **High** | Allowing SSLv3 or TLSv1.0 exposes the server to POODLE and BEAST attacks. A self-signed or expired certificate causes browser warnings and does not provide trusted encryption. | Set SSLProtocol all \-SSLv3 \-TLSv1 \-TLSv1.1 and use a certificate signed by a trusted CA. Disable weak cipher suites. |
| **Missing Security Headers** | HTTP Response Headers | **Medium** | Without X-Frame-Options, X-Content-Type-Options, and CSP, the site is vulnerable to clickjacking, MIME confusion, and cross-site scripting injection via external resources. | Add Header always set X-Frame-Options DENY, Header always set X-Content-Type-Options nosniff, and Header always set X-XSS-Protection 1; mode=block to the Apache config. Enable mod\_headers. |
| **World-Writable Web Files** | /var/www/html/ | **High** | If web files or directories are writable by all users, any local user account on the system can modify web content. This can lead to defacement or injection of malicious code. | Set correct ownership: chown \-R apache:apache /var/www/html and restrict permissions: chmod \-R 755 /var/www/html with 644 for files. |

**📝 Note:** *On competition day, pick the two vulnerabilities you actually confirmed exist on the WWW machine. Do not list a vulnerability you did not verify. The table above covers the most common findings for this type of assessment.*

# **Part 6 — Saving and Submitting Your Work**

The competition document must be completed with your Appendix and saved correctly for expert collection.

## **Step 1: Complete the Appendix in the Competition Document**

**Step 1:** Open the competition document on the competitor laptop (it will be in Documents or on the Desktop)

**Step 2:** Navigate to Appendix 1

**Step 3:** Write your Executive Summary (max 500 words) in the designated section

**Step 4:** Fill in Table 1 with your two selected vulnerabilities

**Step 5:** Save the document

## **Step 2: Save with Country Code Label**

**Step 1:** Save the completed document with your country code in the filename. Example:

PHL\_WSA2025\_MA1\_Assessment.docx

**Step 2:** Save the file to the Desktop of the competitor laptop

**Step 3:** Verify the file is visible on the Desktop

**⚠ Warning:** *The competition document states the file must be labelled with your country code and left on the desktop for expert collection. If the file is not findable, your assessment work will not be marked.*

## **Step 3: Signal to Experts**

**Step 1:** Raise your hand or signal to the expert when your document is ready

**Step 2:** Do not shut down the VMs — experts may need to verify your findings directly on the machines

**Step 3:** Ensure both competitors have agreed on the final two vulnerabilities before submitting

**✓ Tip:** *Before signalling, do one final check: open the saved file and verify the Executive Summary and Table 1 are fully filled in and readable.*

# **Part 7 — Troubleshooting**

## **Website Not Accessible from AMclient**

* Verify the WWW machine is powered on and Apache is running: systemctl status httpd

* Check the WWW IP address with ip addr show and verify it is on the same subnet as AMclient

* Verify the DC is running and the DNS record for www.grimshay.ca resolves to the WWW machine IP

* Try accessing by IP directly in Chrome: http://192.168.1.x — if this works, the issue is DNS

* Check the local firewall on WWW: iptables \-L \-n or firewall-cmd \--list-all

## **Cannot Log In to WWW via Putty (SSH)**

* Verify SSH is running: systemctl status sshd

* Check the WWW IP address is correct

* Use root / P@ssw0rd as credentials

* If SSH is on a non-standard port, check: ss \-tlnp | grep ssh

## **AD / Domain Users Not Working on AMclient**

* Verify the DC is running and AD DS shows green in Server Manager

* Try logging in with the local account first to rule out machine issues

* Check the DC IP is set as the DNS server on AMclient: ipconfig /all

* Run: nltest /dsgetdc:grimshay.ca from AMclient to verify domain connectivity

## **Chrome Shows Certificate Error on HTTPS Site**

This is expected and is itself a finding to document. Note what type of error:

* NET::ERR\_CERT\_AUTHORITY\_INVALID — self-signed certificate not trusted by the browser

* NET::ERR\_CERT\_COMMON\_NAME\_INVALID — certificate hostname does not match www.grimshay.ca

* NET::ERR\_CERT\_DATE\_INVALID — certificate has expired

In Chrome, click Advanced → Proceed to continue past the warning for testing purposes.

# **Part 8 — Final Checklist Before Signalling Done**

## **VM and Network**

| Check | How to Verify | Expected |
| :---- | :---- | :---- |
| All VMs powered on | VMware Workstation VM list | All show green / running |
| DC AD DS running | Server Manager on DC | Green status |
| WWW Apache running | systemctl status httpd on WWW | Active (running) |
| Website accessible from AMclient | Chrome → https://www.grimshay.ca | Site loads (cert warning OK) |
| webusers group exists on DC | AD Users and Computers on DC | webusers group visible |

## **Assessment Findings**

| Check | How to Verify | Expected |
| :---- | :---- | :---- |
| Apache version checked | httpd \-v \+ curl \-I output | Version noted |
| Directory listing tested | Browser → site subdirectory | Enabled or disabled noted |
| SSL/TLS config reviewed | ssl.conf reviewed on WWW | Protocol versions noted |
| Authentication config reviewed | httpd.conf \+ .htaccess reviewed | Auth present or absent noted |
| Security headers checked | curl \-I response headers | Present or missing noted |
| File permissions checked | ls \-la /var/www/ on WWW | Permissions noted |

## **Submission**

| Check | How to Verify | Expected |
| :---- | :---- | :---- |
| Executive Summary written | Open competition document | Appendix 1 section filled |
| Table 1 completed | Open competition document | Two vulnerabilities listed |
| File saved on Desktop | Check Desktop of competitor laptop | File visible with country code |
| File name includes country code | Check filename | e.g. PHL\_WSA2025\_MA1.docx |

**✓ Tip:** *Fix issues in order: network connectivity → DC and AD → WWW and Apache → website access → assessment checks → document submission.*

*WorldSkills ASEAN Manila 2025 — Cyber Security Skill 54 — Module A1 — Network Setup & Assessment Guide*

# Module A1 2025

## **Phase 1: Network & ESXi Base Configuration**

Before creating virtual machines, you need to set up an isolated network space within ESXi so the VMs can talk to each other without conflicting with your home network.

1. **Create a Virtual Switch (vSwitch):**  
   * Log into your ESXi web interface (`https://192.168.1.1` or whatever IP your host currently uses).  
   * Go to **Networking** \> **Virtual switches** \> **Add standard virtual switch**. Name it something like `WorldSkills-vSwitch`.  
2. **Create a Port Group:**  
   * Go to the **Port groups** tab \> **Add port group**.  
   * Name it `WorldSkills-Network` and tie it to the `WorldSkills-vSwitch` you just made. All your VMs will plug into this network.

---

## **Phase 2: Building the 3 Virtual Machines**

Since the competition uses the generic password `P@ssw0rd` for everything, use that during your OS installations to keep things authentic.

### **1\. The Active Directory Domain Controller (DC Machine)**

This machine handles identity management and DNS.

* **OS to install:** Windows Server (e.g., 2019 or 2022 Evaluation edition).  
* **Network Adapter:** Assign to `WorldSkills-Network`.  
* **Post-Installation Setup:**  
  * Assign a static IP (e.g., `192.168.1.10`, Subnet: `255.255.255.0`, Gateway: `192.168.1.1`). Set its own DNS to `127.0.0.1`.  
  * Open Server Manager and click **Add Roles and Features**.  
  * Install **Active Directory Domain Services (AD DS)** and **DNS Server**.  
  * Promote the server to a Domain Controller. Create a new forest with the domain name `grimshay.ca`.  
  * Open **Active Directory Users and Computers**, create a global security group named `webusers`, and add a few test user accounts to it.

### **2\. The Linux Web Server (WWW Machine)**

This is your target machine hosting Apache.

* **OS to install:** Any standard enterprise Linux distro (such as Ubuntu Server or Rocky Linux).  
* **Network Adapter:** Assign to `WorldSkills-Network`.  
* **Post-Installation Setup:**  
  * Assign a static IP (e.g., `192.168.1.20`). Set the primary DNS server IP to point to your **DC Machine** (`192.168.1.10`).  
  * Install the Apache web server package (`sudo apt install apache2` or `sudo dnf install httpd`).  
  * **Configure the Website:** Set up a basic HTML landing page inside the web root folder representing `www.grimshay.ca`.  
  * **The Twist (The Vulnerability):** The competition task implies that *only* members of the `webusers` Active Directory group should have access, but something is misconfigured. To simulate this vulnerability, you could misconfigure Apache’s `.htaccess` file, leave an open directory indexing vulnerability, or fail to properly bind Apache to the LDAP/Active Directory authentication module (`mod_authnz_pam` or `mod_auth_nz_ldap`).

### **3\. The Competitor Workstation (AMclient Machine)**

This is where you will sit to perform the assessment.

* **OS to install:** Windows 10/11 or a Linux Desktop.  
* **Network Adapter:** Assign to `WorldSkills-Network`.  
* **Post-Installation Setup:**  
  * Assign a static IP (e.g., `192.168.1.30`) and set its DNS to point to the DC (`192.168.1.10`).  
  * Join the computer to the `grimshay.ca` domain.  
  * Install the following testing tools required by the project:  
    * Google Chrome (to test web browser authentication).  
    * PuTTY (to test SSH access to the Linux server).  
    * Wireshark (to capture network packets and see if passwords/traffic are traveling in unencrypted cleartext).

---

## **Phase 3: Verifying the Lab**

Once all three are running:

1. Log into the **AMclient** machine using a domain account you created.  
2. Open Chrome and browse to `http://www.grimshay.ca`. Ensure the site resolves (if it doesn't, check your DNS settings on the DC).  
3. Start running Wireshark, attempt to log into things, and practice hunting for the configuration flaws just like you will on competition day\!

# TEST Project A1: Task

## **MA1: WorldSkill Competition — Cyber Security Training Plan**

### **Recommended Competitor Workflow**

### 

### **Phase 1 — Access the Environment**

Competitor starts from the workstation/laptop, then opens VMware Workstation to access their assigned VMs.

They should confirm they can access:

* AMClient1  
* AMClient2  
* DC  
* WWW server

They should verify basic connectivity:

Bash

ping 172.16.100.10

ping 172.16.100.13

nslookup www.grimshay.ca

Expected DNS should resolve:

Plaintext

www.grimshay.ca \-\> 172.16.100.13

> **Note:** If DNS does not resolve, they should check whether the client is using the DC as DNS.

---

### **Phase 2 — Test Website Access from Client Machines**

From AMClient1 or AMClient2, open Chrome: https://www.grimshay.ca

Then test access using different accounts:

* Domain users who should have access  
* Domain users who should NOT have access  
* Local account access

> 🚨 **The Key Security Question:**

> Can users outside the webusers group access the website? If yes, that is a major access-control finding.

They should also check:

* Does the website require authentication?  
* Does it use HTTPS?  
* Does the certificate show errors?  
* Does it expose directory listings?  
* Does it expose default Apache pages?  
* Does it allow anonymous access?  
* Does it leak server version information?

---

### **Phase 3 — Inspect Apache Configuration**

Using PuTTY, SSH into the Linux web server: 172.16.100.13, then inspect Apache.

> 💻 **Common Inspection Commands**

> Bash

hostname  
ip addr  
sudo apache2ctl \-S  
sudo apache2ctl \-M  
sudo systemctl status apache2  
> 

> 📂 **Check Apache Site Configuration**

> Bash

ls \-la /etc/apache2/sites-enabled/  
ls \-la /etc/apache2/sites-available/  
cat /etc/apache2/sites-enabled/\*.conf  
> 

> 🔒 **Check Authentication / Access Rules**

> Bash

grep \-R "Require" /etc/apache2/  
grep \-R "Auth" /etc/apache2/  
grep \-R "Options" /etc/apache2/  
> 

> 🔑 **Check Website Directory Permissions**

> Bash

ls \-la /var/www/  
ls \-la /var/www/html/  
find /var/www \-type f \-maxdepth 3 \-ls  
> 

*Note: The competitor is looking for misconfiguration, not trying to break the system.*

---

### **Phase 4 — Identify the Two Most Critical Website Vulnerabilities**

Possible categories they should investigate:

| Area | What to Check |
| :---- | :---- |
| **Access control** | Only webusers group should access the site |
| **HTTPS/TLS** | Certificate validity, weak SSL/TLS config |
| **Apache config** | Directory listing, default pages, exposed server banner |
| **File permissions** | Web files readable/writable by wrong users |
| **Authentication** | Anonymous access, incorrect AD group mapping |
| **Sensitive files** | Backup files, config files, credentials inside web directory |
| **Logging** | Missing or weak audit logs |
| **Security headers** | Missing HSTS, X-Frame-Options, X-Content-Type-Options |

For this project, prioritize findings around:

1. **Broken access control to the website**  
2. **Apache/web server misconfiguration exposing sensitive information or unsafe access**

*Reasoning: The client specifically stated that only the webusers domain group should have access.*

---

## **7\. Example Assessment Logic**

The competitor should execute and document findings using this mental framework:

> 📁 **Finding Example 1 — Website Accessible by Unauthorized Users**

> * **Test:** Login as a normal domain user not in webusers. Open https://www.grimshay.ca.  
> * **Vulnerability:** Improper access control / authorization misconfiguration  
> * **Risk:** **High or Critical**  
> * **Why it matters:** Users outside the approved group can view or interact with protected website content.  
> * **Recommendation:** Configure Apache authentication and authorization to allow only members of the domain webusers group. Validate group membership against Active Directory and deny all other users by default.

> 📁 **Finding Example 2 — Directory Listing Enabled**

> * **Test:** Browse website folders manually. Check Apache Options configuration. Look for Options Indexes.  
> * **Vulnerability:** Directory listing enabled on Apache  
> * **Risk:** **Medium or High**  
> * **Why it matters:** Attackers or unauthorized users may enumerate files, backups, scripts, or sensitive content.  
> * **Recommendation:** Disable directory indexing by removing Indexes from Apache Options and adding Options \-Indexes.

---

## **8\. Suggested Executive Summary Structure**

The final report submission should align with this format:

> An assessment was performed on the Apache-hosted www.grimshay.ca website within the provided lab environment. The review focused on validating whether the website configuration supports the client requirement that only members of the domain webusers group should access the site.

> Testing was conducted from the domain-joined AMClient systems and configuration review was performed on the Linux web server hosting Apache. The assessment identified security weaknesses affecting access control and web server configuration. These issues may allow unauthorized access or expose unnecessary information about the server and hosted content.

> The recommended priority is to correct the Apache authentication and authorization configuration, validate Active Directory group enforcement, restrict access by default, and harden the Apache site configuration. Once remediation is completed, access should be retested using both authorized and unauthorized domain accounts.

---

## **9\. Final Process for the Competitor**

Use this exact execution sequence during practical runs:

1. Login to competitor workstation.  
2. Open assigned VMs through VMware Workstation.  
3. Confirm IP settings of AMClient, DC, and WWW server.  
4. Verify DNS resolution for www.grimshay.ca.  
5. Test website access from Chrome.  
6. Test access using different domain/local users.  
7. SSH to Linux web server using PuTTY.  
8. Review Apache virtual host, authentication, authorization, TLS, and directory settings.  
9. Select the two highest-risk website vulnerabilities.  
10. Fill in Appendix 1\.  
11. Save the final file on the desktop using the required country code.  
12. Notify experts for collection.

# MA2 Competition Tasks

1\. Verify all VM IPs are correct  (atoa)  
2\. Add port groups (Ryle)  
3\. pfSense basic setup (password, DHCP) ([ryleanthony gabotero](mailto:ryleanthony.gabotero@foundationu.com))   
4\. Firewall rules ([ryleanthony gabotero](mailto:ryleanthony.gabotero@foundationu.com))  
5\. WINSRV1 password policies and login banner ([earljohn estandarte](mailto:earljohn.estandarte@foundationu.com))  
6\. WINSRV1 GPOs (control, registry, google) ([earljohn estandarte](mailto:earljohn.estandarte@foundationu.com))  
7\. WINSRV1 file share and auditing ([earljohn estandarte](mailto:earljohn.estandarte@foundationu.com))  
8\. PKI certenroll GPO ([ryleanthony gabotero](mailto:ryleanthony.gabotero@foundationu.com))  
9\. LINSRV1 domain join ([earljohn estandarte](mailto:earljohn.estandarte@foundationu.com))  
10\. LINSRV1 SSH config ([earljohn estandarte](mailto:earljohn.estandarte@foundationu.com))  
11\. LINSRV1 website HTTPS ([earljohn estandarte](mailto:earljohn.estandarte@foundationu.com))  
12\. LINSRV1 SELinux ([earljohn estandarte](mailto:earljohn.estandarte@foundationu.com))  
13\. OpenVPN setup ([ryleanthony gabotero](mailto:ryleanthony.gabotero@foundationu.com))  
14\. Snort setup ([ryleanthony gabotero](mailto:ryleanthony.gabotero@foundationu.com))  
15\. Test everything with clients ([ryleanthony gabotero](mailto:ryleanthony.gabotero@foundationu.com))  
16\. Fill in Table 2 GPO recommendations (atoa)

# MA2 Firewall: Ryle

## **Complete pfSense Firewall Guide for Manila Competition**

---

## **Understanding the Firewall's Role**

Internet (ISP)  
      ↓  
pfSense ← controls ALL traffic between networks  
      ↓  
├── SERVERS (192.168.2.0/24) → WINSRV1, WINSRV3, WINSRV4  
├── DMZ (192.168.1.0/24) → LINSRV1  
└── LAN (172.16.100.0/24) → Client1, Client2

---

## **Why pfSense?**

| Feature | Purpose |
| ----- | ----- |
| Firewall | Controls what traffic is allowed |
| Router | Routes traffic between VLANs |
| NAT | Allows internal clients to access internetexit |
| DHCP | Gives IPs to LAN clients |
| OpenVPN | Remote access for external users |
| Snort | Detects and logs suspicious traffic |

---

## **Part 1 — Initial Setup**

### **Task 1 — Change Admin Password**

**Why:** Default password `pfsense` is publicly known — must change immediately

**Step by Step:**

1. Login to `https://192.168.2.254`  
2. Go to **System → User Manager**  
3. Click **Admin** user  
4. Scroll to **Password**  
5. Enter:

New password:     P@ssw0rd  
Confirm password: P@ssw0rd

6. Click **Save**

---

### **Task 2 — Setup DHCP for LAN Clients**

**Why:** Client1 and Client2 need automatic IP addresses from pfSense

**Step by Step:**

1. Go to **Services → DHCP Server**  
2. Click **OPT2 (LAN clients)** tab  
3. Check ✅ **Enable DHCP server on OPT2 interface**  
4. Set range:

From: 172.16.100.100  
To:   172.16.100.200

5. Under **DNS Servers** enter:

172.16.100.254

6. Under **DNS Servers** enter:

DNS Server 1: 192.168.2.10

7. Click **Save**

---

## **Part 2 — Understanding Firewall Rules**

### **How pfSense Rules Work**

Rules are processed TOP to BOTTOM  
First matching rule WINS  
Default rule at bottom \= BLOCK ALL

### **Rule Components**

| Field | Meaning |
| ----- | ----- |
| Action | Pass, Block, Reject |
| Interface | Which network the traffic comes FROM |
| Protocol | TCP, UDP, ICMP, Any |
| Source | Where traffic originates |
| Destination | Where traffic is going |
| Port | Which service (80=HTTP, 443=HTTPS etc) |

---

## **Part 3 — Firewall Rules Setup**

### **Before Adding Rules**

Go to **Firewall → Rules** and you will see tabs for each interface:

WAN | LAN | OPT1 (DMZ) | OPT2 (LAN clients)

---

### **Rule Set 1 — LAN Clients to Internet**

**What:** Allow Client1 and Client2 to access internet

**Why:** Internal users need internet access

**Step by Step:**

1. Go to **Firewall → Rules → OPT2**  
2. Click **Add** (up arrow)  
3. Set:

Action:      Pass  
Interface:   OPT2  
Protocol:    Any  
Source:      OPT2 subnet  
Destination: Any  
Description: LAN to Internet

4. Click **Save → Apply Changes**

---

### **Rule Set 2 — LAN to DMZ**

**What:** Allow LAN clients to access LINSRV1 via SSH, HTTP, HTTPS, DNS

**Why:** Internal users need to access the web server and DNS on LINSRV1

**Step by Step:**

1. Go to **Firewall → Rules → OPT2**  
2. Click **Add**  
3. Set:

**For SSH:**

Action:      Pass  
Interface:   OPT2  
Protocol:    TCP  
Source:      OPT2 subnet  
Destination: 192.168.1.10 (LINSRV1)  
Port:        2022  
Description: LAN to DMZ SSH

**For HTTP:**

Action:      Pass  
Protocol:    TCP  
Source:      OPT2 subnet  
Destination: 192.168.1.10  
Port:        80  
Description: LAN to DMZ HTTP

**For HTTPS:**

Action:      Pass  
Protocol:    TCP  
Source:      OPT2 subnet  
Destination: 192.168.1.10  
Port:        443  
Description: LAN to DMZ HTTPS

**For DNS:**

Action:      Pass  
Protocol:    TCP/UDP  
Source:      OPT2 subnet  
Destination: 192.168.1.10  
Port:        53  
Description: LAN to DMZ DNS

4. Click **Save → Apply Changes** after each rule

---

### **Rule Set 3 — LAN to Servers**

**What:** Allow LAN clients to communicate with AD and PKI services

**Why:** Domain clients need to communicate with domain controller for authentication

**Step by Step:**

1. Go to **Firewall → Rules → OPT2**  
2. Add rules for each port:

**Create an Alias First (makes it easier):**

1. Go to **Firewall → Aliases**  
2. Click **Add**  
3. Set:

Name:        AD\_Ports  
Type:        Port  
Description: Active Directory Ports

4. Add these ports:

53, 88, 135, 137, 138, 139, 389, 443, 445, 464, 636, 3268, 3269, 9389

5. Click **Save → Apply Changes**

**Then Create Rule:**

Action:      Pass  
Interface:   OPT2  
Protocol:    TCP/UDP  
Source:      OPT2 subnet  
Destination: 192.168.2.0/24 (SERVERS subnet)  
Port:        AD\_Ports (alias)  
Description: LAN to Servers AD and PKI

---

### **Rule Set 4 — WAN Inbound to DMZ**

**What:** Allow internet users to access LINSRV1 website and DNS

**Why:** LINSRV1 hosts public website accessible from internet

**Step by Step:**

#### **First Create NAT Port Forward**

1. Go to **Firewall → NAT → Port Forward**  
2. Click **Add**

**For HTTP:**

Interface:      WAN  
Protocol:       TCP  
Destination:    WAN address  
Destination port: 80  
Redirect target IP: 192.168.1.10  
Redirect target port: 80  
Description:    WAN to DMZ HTTP

**For HTTPS:**

Interface:      WAN  
Protocol:       TCP  
Destination:    WAN address  
Destination port: 443  
Redirect target IP: 192.168.1.10  
Redirect target port: 443  
Description:    WAN to DMZ HTTPS

**For DNS:**

Interface:      WAN  
Protocol:       UDP  
Destination:    WAN address  
Destination port: 53  
Redirect target IP: 192.168.1.10  
Redirect target port: 53  
Description:    WAN to DMZ DNS

3. Click **Save → Apply Changes**

#### **Firewall Rules Are Auto-Created**

pfSense automatically creates firewall rules when you add port forwards. Check **Firewall → Rules → WAN** to verify.

---

### **Rule Set 5 — Servers Outbound**

**What:** Allow servers to communicate as necessary for services

**Why:** Servers need outbound access for updates, AD replication etc

**Step by Step:**

1. Go to **Firewall → Rules → LAN (SERVERS)**  
2. Click **Add**  
3. Set:

Action:      Pass  
Interface:   LAN  
Protocol:    Any  
Source:      192.168.2.0/24  
Destination: Any  
Description: Servers outbound

4. Click **Save → Apply Changes**

---

### **Rule Set 6 — DMZ to Servers**

**What:** Allow LINSRV1 to communicate with AD for domain join

**Why:** LINSRV1 needs to authenticate against AD

**Step by Step:**

**Create Alias for DMZ AD Ports:**

Name:  DMZ\_AD\_Ports  
Ports: 53, 88, 135, 137, 138, 389, 445, 464, 636, 3268, 3269

**Create Rule:**

1. Go to **Firewall → Rules → OPT1 (DMZ)**  
2. Click **Add**  
3. Set:

Action:      Pass  
Interface:   OPT1  
Protocol:    TCP/UDP  
Source:      192.168.1.10  
Destination: 192.168.2.0/24  
Port:        DMZ\_AD\_Ports  
Description: DMZ to Servers AD

4. Click **Save → Apply Changes**

---

### **Rule Set 7 — Block moulinrouge/starcity**

**What:** Block internal clients from accessing www.starcity.com.ph

**Why:** Competition requirement — selective internet filtering

**Step by Step:**

#### **Create Alias for Blocked Sites**

1. Go to **Firewall → Aliases**  
2. Click **Add**  
3. Set:

Name:        Blocked\_Sites  
Type:        Host  
Description: Blocked websites

4. Add hostname:

www.starcity.com.ph

5. Click **Save → Apply Changes**

#### **Add Block Rule ABOVE Allow Rule**

1. Go to **Firewall → Rules → OPT2**  
2. Click **Add** (up arrow — adds to TOP)  
3. Set:

Action:      Block  
Interface:   OPT2  
Protocol:    TCP  
Source:      OPT2 subnet  
Destination: Blocked\_Sites  
Port:        80, 443  
Description: Block starcity.com.ph

4. Make sure this rule is **ABOVE** the general internet allow rule  
5. Click **Save → Apply Changes**

---

## **Part 4 — OpenVPN Setup**

### **What OpenVPN Does**

WinClient3 (Internet) ──VPN tunnel──→ pfSense ──→ Internal network

Allows external users to securely access internal resources.

### **Step by Step**

#### **Step 1 — Create CA Certificate**

1. Go to **System → Cert Manager → CAs**  
2. Click **Add**  
3. Set:

Descriptive name: Manila-VPN-CA  
Method:          Import an existing Certificate Authority

4. Import certificate from WINSRV3:  
   * On WINSRV3 export the CA certificate  
   * Paste the certificate content here  
5. Click **Save**

#### **Step 2 — Create Server Certificate**

1. Go to **System → Cert Manager → Certificates**  
2. Click **Add/Sign**  
3. Set:

Method:          Create an internal Certificate  
Descriptive name: Manila-VPN-Server  
Certificate authority: Manila-VPN-CA  
Certificate Type: Server Certificate

4. Click **Save**

#### **Step 3 — Configure OpenVPN Server**

1. Go to **VPN → OpenVPN → Servers**  
2. Click **Add**  
3. Fill in:

**General Settings:**

Server mode:     Remote Access (SSL/TLS \+ User Auth)  
Backend for auth: Local Database  
Protocol:        UDP on IPv4  
Interface:       WAN  
Port:            1194  
Description:     Manila VPN

**Cryptographic Settings:**

TLS Configuration: checked  
Peer Certificate Authority: Manila-VPN-CA  
Server certificate: Manila-VPN-Server  
DH Parameter Length: 2048

**Tunnel Settings:**

IPv4 Tunnel Network: 10.0.8.0/24  
IPv4 Local Network:  172.16.100.0/24, 192.168.2.0/24, 192.168.1.0/24

**Client Settings:**

DNS Server 1: 192.168.2.10

4. Click **Save**

#### **Step 4 — Add VPN Firewall Rules**

1. Go to **Firewall → Rules → OpenVPN**  
2. Click **Add**  
3. Set:

Action:      Pass  
Protocol:    Any  
Source:      Any  
Destination: Any  
Description: Allow VPN traffic

4. Click **Save → Apply Changes**

#### **Step 5 — Add WAN Rule for VPN**

1. Go to **Firewall → Rules → WAN**  
2. Click **Add**  
3. Set:

Action:      Pass  
Protocol:    UDP  
Source:      Any  
Destination: WAN address  
Port:        1194  
Description: OpenVPN WAN

4. Click **Save → Apply Changes**

#### **Step 6 — Create VPN User**

1. Go to **System → User Manager**  
2. Click **Add**  
3. Set:

Username: vpnuser  
Password: P@ssw0rd

4. Check ✅ **Click to create a user certificate**  
5. Click **Save**

#### **Step 7 — Export VPN Client Config**

1. Go to **VPN → OpenVPN → Client Export**  
2. Find your VPN server  
3. Click **Export** for WinClient3  
4. Download the `.ovpn` file  
5. Install on WinClient3

---

## **Part 5 — Snort IDS Setup**

### **What Snort Does**

Monitors network traffic  
        ↓  
Detects suspicious patterns  
        ↓  
Logs or blocks matching traffic

### **Step by Step**

#### **Step 1 — Install Snort**

1. Go to **System → Package Manager → Available Packages**  
2. Search for `Snort`  
3. Click **Install**  
4. Wait for installation

#### **Step 2 — Configure Snort Interface**

1. Go to **Services → Snort**  
2. Click **Snort Interfaces**  
3. Click **Add**  
4. Set:

Interface:   WAN  
Description: WAN Snort

5. Click **Save**

#### **Step 3 — Enable Logging**

1. Click **WAN Settings** tab  
2. Check ✅ **Send Alerts to System Log**  
3. Check ✅ **Log to System Log**  
4. Click **Save**

#### **Step 4 — Add Custom Rule for XMAS Scan**

**Understanding the Rule:**

drop tcp any any \<\> a.b.c.d/24 and (flags: FPU; msg: "Possible XMAS scan"; sid: 100001\)

| Part | Meaning |
| ----- | ----- |
| drop | Block the traffic |
| tcp | TCP protocol |
| any any | Any source IP and port |
| \<\> | Bidirectional |
| a.b.c.d/24 | Replace with your WAN subnet |
| flags: FPU | FIN, PUSH, URG flags set (XMAS scan) |
| msg | Alert message |
| sid | Rule ID |

**Step by Step:**

1. Go to **Services → Snort**  
2. Click **WAN Rules** tab  
3. Click **Custom Rules**  
4. Find your WAN IP subnet (check **Status → Interfaces → WAN**)  
5. Replace `a.b.c.d/24` with your WAN subnet e.g. `192.168.99.0/24`  
6. Add the rule:

drop tcp any any \<\> 192.168.99.0/24 any (flags:FPU; msg:"Possible XMAS scan"; sid:100001;)

7. Click **Save**

#### **Step 5 — Add Rule for starcity.com.ph**

1. Still in **Custom Rules**  
2. Add:

alert tcp any any \-\> any 80 (msg:"Access to starcity.com.ph"; content:"starcity.com.ph"; sid:100002;)  
alert tcp any any \-\> any 443 (msg:"Access to starcity.com.ph"; content:"starcity.com.ph"; sid:100003;)

3. Click **Save**

#### **Step 6 — Start Snort**

1. Go to **Services → Snort → Snort Interfaces**  
2. Click **Start** button next to WAN interface  
3. Green indicator means running ✅

#### **Step 7 — Test Snort**

1. From WinClient3 run nmap:

nmap \-sX \[pfSense WAN IP\]

2. Check Snort alerts:  
   * Go to **Services → Snort → Alerts**  
   * Should see XMAS scan alert ✅

---

## **Part 6 — Verify Everything Works**

### **Test Checklist**

#### **Basic Connectivity**

From Client1:  
ping 192.168.2.254 → pfSense LAN ✅  
ping 192.168.2.10 → WINSRV1 ✅  
ping 192.168.1.10 → LINSRV1 ✅

#### **Internet Access**

From Client1:  
Browse www.nationalmuseum.gov.ph → WORKS ✅  
Browse www.starcity.com.ph → BLOCKED ✅

#### **DMZ Access**

From Client1:  
Browse http://192.168.1.10 → WORKS ✅  
SSH to 192.168.1.10 port 2022 → WORKS ✅

#### **VPN**

From WinClient3 (Internet):  
Connect OpenVPN → WORKS ✅  
Access internal resources → WORKS ✅

#### **Snort**

From WinClient3:  
nmap \-sX \[WAN IP\] → Alert logged ✅  
Browse starcity.com.ph → Alert logged ✅

---

## **Complete Firewall Checklist**

| Task | Status |
| ----- | ----- |
| Change admin password | ⬜ |
| Setup LAN DHCP | ⬜ |
| LAN → Internet rule | ⬜ |
| LAN → DMZ rules | ⬜ |
| LAN → Servers rules | ⬜ |
| WAN → DMZ NAT | ⬜ |
| DMZ → Servers rules | ⬜ |
| Block starcity.com.ph | ⬜ |
| OpenVPN setup | ⬜ |
| Snort installation | ⬜ |
| Snort XMAS scan rule | ⬜ |
| Snort starcity alert | ⬜ |
| Test all rules | ⬜ |

---

Would you like me to go deeper into any specific part — OpenVPN, Snort, or the firewall rules?

# MA2 PKI Layer: Ryle

**PKI Setup Guide**

WINSRV3 (Issuing CA) & WINSRV4 (Offline Root CA)

*GUI Version — WorldSkills ASEAN Manila 2025*

Cyber Security Skill 54 — Module A2 — Based on WSA2025\_TP54 Competition Document

## **Quick Reference**

| Item | Value |
| :---- | :---- |
| **WINSRV3 Hostname** | WINSRV3.manila.com |
| **WINSRV3 IP Address** | 192.168.2.30/24 |
| **WINSRV4 Hostname** | WINSRV4.manila.com |
| **WINSRV4 IP Address** | 192.168.2.50/24 |
| **VLAN** | SERVERS |
| **Domain** | manila.com |
| **Administrator Password** | P@ssw0rd |
| **WINSRV3 Role** | Subordinate / Issuing CA |
| **WINSRV4 Role** | Offline Root CA (powered off) |
| **Gateway** | 192.168.2.254 (pfSense SERVERS interface) |

**📝 Note:** *On competition day, WINSRV3 is pre-configured as subordinate CA and WINSRV4 as the Offline Root CA. This guide covers completing the PKI competition tasks from that starting state.*

# **Part 1 — Overview of the 2-Tier PKI Architecture**

The competition uses a 2-Tier PKI (Public Key Infrastructure) hierarchy. Understanding this architecture before you start will save time during configuration.

## **Tier 1 — WINSRV4 (Offline Root CA)**

* Acts as the trust anchor for the entire domain.

* Already configured and powered off on competition day — do not power it on unless troubleshooting.

* In a real-world deployment, the Root CA is always kept offline and physically secured.

* Its certificate is distributed to all domain computers via Group Policy.

## **Tier 2 — WINSRV3 (Issuing / Subordinate CA)**

* Issues certificates to domain computers, users, and servers.

* Has been configured as a subordinate CA, signed by WINSRV4.

* Handles all day-to-day certificate operations.

* This is the machine you will be working with most.

**📝 Note:** *The competition document states: 'WinSRV4 will be offline Root CA — already configured and powered off. WinSRV3 will be the issuing CA — already configured as the subordinate CA for the domain.'*

# **Part 2 — Verify WINSRV3 Initial Configuration**

Before starting PKI tasks, confirm WINSRV3 is correctly set up.

## **Step 1: Verify IP Address**

**Step 1:** Log on to WINSRV3 with Administrator / P@ssw0rd

**Step 2:** Open Control Panel → Network and Sharing Center

**Step 3:** Click Change adapter settings

**Step 4:** Right-click the network adapter → Properties

**Step 5:** Double-click Internet Protocol Version 4 (TCP/IPv4)

**Step 6:** Verify the following settings:

IP Address:   192.168.2.30

Subnet Mask:  255.255.255.0

Gateway:      192.168.2.254

DNS Server:   192.168.2.10  (WINSRV1)

**Step 7:** Click OK → Close

**⚠ Warning:** *DNS must point to WINSRV1 (192.168.2.10). Certificate enrollment and domain communication depend on correct DNS resolution.*

## **Step 2: Verify AD CS Role (Certificate Services)**

**Step 1:** Open Server Manager

**Step 2:** In the left panel confirm AD CS (Active Directory Certificate Services) is listed

**Step 3:** It should show green status

**Step 4:** Open a PowerShell window and run:

certutil \-ping

Expected output: Successfully contacted CertSvc

**✓ Tip:** *If AD CS is not listed, you may need to install it. See Part 3 (Installing AD CS) if this happens.*

## **Step 3: Verify CA Name and Configuration**

**Step 1:** Open Server Manager → Tools → Certification Authority

**Step 2:** In the left panel you should see a CA named similar to:

Manila-Issuing-CA  (or similar naming)

**Step 3:** Right-click the CA name → Properties

**Step 4:** Click the General tab and verify:

* CA type: Subordinate certification authority

* CA certificate is valid and shows WinSRV4 or Root CA as the issuer

**Step 5:** Click OK

# **Part 3 — Installing AD CS on WINSRV3 (If Not Pre-Installed)**

**📝 Note:** *On competition day this should already be done. Only follow this section if you find AD CS is missing from Server Manager.*

## **Step 1: Add the AD CS Role**

**Step 1:** Open Server Manager

**Step 2:** Click Manage → Add Roles and Features

**Step 3:** Click Next through the wizard until you reach Server Roles

**Step 4:** Check Active Directory Certificate Services

**Step 5:** Click Add Features when prompted

**Step 6:** Click Next

**Step 7:** On the AD CS Role Services page, ensure Certification Authority is checked

**Step 8:** Click Next → Next → Install

**Step 9:** Wait for installation to complete

**Step 10:** Click Configure Active Directory Certificate Services on the destination server

## **Step 2: Configure as Subordinate CA**

**Step 1:** In the AD CS Configuration wizard, click Next

**Step 2:** Select Use existing AD credentials: MANILA\\Administrator

**Step 3:** Click Next

**Step 4:** Check Certification Authority → click Next

**Step 5:** Select Enterprise CA → click Next

**Step 6:** Select Subordinate CA → click Next

**Step 7:** Select Create a new private key → click Next

**Step 8:** Accept default cryptography settings (RSA 2048\) → click Next

**Step 9:** Set CA Name: Manila-Issuing-CA → click Next

**Step 10:** Select Send a certificate request to a parent CA

**Step 11:** Enter the Root CA: WINSRV4.manila.com\\Manila-Root-CA

**Step 12:** Click Next → Configure → Close

**⚠ Warning:** *If WINSRV4 is powered off, you will need to save the certificate request to a file, transfer it to WINSRV4, issue the certificate, and then install it back on WINSRV3. This is the standard offline Root CA procedure.*

# **Part 4 — Creating the certenroll GPO for Auto-Enrollment**

The competition requires creating a GPO named 'certenroll' so domain computers automatically receive a certificate from WINSRV3 via auto-enrollment.

## **Step 1: Open Group Policy Management on WINSRV1**

**Step 1:** Log on to WINSRV1 (192.168.2.10) as Administrator

**Step 2:** Open Server Manager → Tools → Group Policy Management

**Step 3:** In the left panel expand:

Forest: manila.com

  └─ Domains

      └─ manila.com

## **Step 2: Create the certenroll GPO**

**Step 1:** Right-click manila.com → Create a GPO in this domain and Link it here

**Step 2:** In the New GPO dialog type exactly:

certenroll

**Step 3:** Click OK

**Step 4:** Right-click the certenroll GPO → Edit

## **Step 3: Configure Auto-Enrollment for Computers**

**Step 1:** In the GPO Editor navigate to:

Computer Configuration

  └─ Policies

      └─ Windows Settings

          └─ Security Settings

              └─ Public Key Policies

**Step 2:** Double-click Certificate Services Client – Auto-Enrollment

**Step 3:** In the Configuration Model dropdown select: Enabled

**Step 4:** Check ✅ Renew expired certificates, update pending certificates and remove revoked certificates

**Step 5:** Check ✅ Update certificates that use certificate templates

**Step 6:** Click OK

## **Step 4: Configure Auto-Enrollment for Users**

**Step 1:** In the same GPO Editor navigate to:

User Configuration

  └─ Policies

      └─ Windows Settings

          └─ Security Settings

              └─ Public Key Policies

**Step 2:** Double-click Certificate Services Client – Auto-Enrollment

**Step 3:** In the Configuration Model dropdown select: Enabled

**Step 4:** Check ✅ Renew expired certificates, update pending certificates and remove revoked certificates

**Step 5:** Check ✅ Update certificates that use certificate templates

**Step 6:** Click OK

**Step 7:** Close the GPO Editor

## **Step 5: Also Configure in Default Domain Policy (Best Practice)**

**📝 Note:** *Auto-enrollment in the Default Domain Policy ensures all computers including WINSRV3 itself receive certificates. The certenroll GPO handles this for most clients.*

**Step 1:** In Group Policy Management right-click Default Domain Policy → Edit

**Step 2:** Repeat Steps 3 and 4 above for both Computer and User Configuration

**Step 3:** Close GPO Editor

## **Step 6: Force Policy Update**

**Step 1:** Open Command Prompt on WINSRV1 and run:

gpupdate /force

**Step 2:** On each client machine (Client1, Client2, WINSRV3) open Command Prompt and run:

gpupdate /force

certutil \-pulse

**✓ Tip:** *certutil \-pulse forces the auto-enrollment client to check for new certificates immediately without waiting for the next policy refresh cycle.*

# **Part 5 — Distributing the Root CA Certificate via GPO**

For internal clients not to get certificate errors when browsing to internal HTTPS websites, the Root CA certificate (from WINSRV4) must be in the Trusted Root Certification Authorities store on all domain machines. This is done via the certenroll GPO.

## **Step 1: Export the Root CA Certificate from WINSRV3**

**Step 1:** On WINSRV3 open a Command Prompt and run:

certutil \-ca.cert C:\\RootCA.cer

**Step 2:** Alternatively, open Certification Authority console → right-click the CA name → Properties

**Step 3:** Click the General tab → View Certificate

**Step 4:** Click the Details tab → Copy to File

**Step 5:** In the wizard select DER encoded binary X.509 (.CER)

**Step 6:** Save as: C:\\RootCA.cer

**Step 7:** Copy RootCA.cer to WINSRV1 (e.g., via network share or USB)

## **Step 2: Add Root CA to the certenroll GPO**

**Step 1:** On WINSRV1 open Group Policy Management

**Step 2:** Right-click the certenroll GPO → Edit

**Step 3:** Navigate to:

Computer Configuration

  └─ Policies

      └─ Windows Settings

          └─ Security Settings

              └─ Public Key Policies

                  └─ Trusted Root Certification Authorities

**Step 4:** Right-click Trusted Root Certification Authorities → Import

**Step 5:** In the Certificate Import Wizard click Next

**Step 6:** Browse to and select RootCA.cer → click Next

**Step 7:** Select Place all certificates in the following store: Trusted Root Certification Authorities

**Step 8:** Click Next → Finish

**Step 9:** Click OK on the import success dialog

**Step 10:** Close the GPO Editor

## **Step 3: Apply to All Clients**

**Step 1:** On WINSRV1 run:

gpupdate /force

**Step 2:** On Client1 and Client2 run:

gpupdate /force

**✓ Tip:** *After gpupdate, you can verify the Root CA is trusted by opening certmgr.msc on a client → Trusted Root Certification Authorities → Certificates → look for Manila-Root-CA*

# **Part 6 — Issuing a Certificate to WINSRV3 for IIS (Web Server)**

The competition requires WINSRV3 to receive a Web Server certificate for IIS via auto-enrollment. This allows the IIS website on WINSRV3 to use HTTPS without certificate warnings on domain clients.

## **Step 1: Verify the Web Server Certificate Template**

**Step 1:** On WINSRV3 open Certification Authority console

**Step 2:** Expand Manila-Issuing-CA → Certificate Templates

**Step 3:** Verify Web Server template is listed

**Step 4:** If not listed: right-click Certificate Templates → New → Certificate Template to Issue

**Step 5:** Select Web Server → click OK

## **Step 2: Request a Web Server Certificate via IIS**

**Step 1:** On WINSRV3 open Server Manager → Tools → Internet Information Services (IIS) Manager

**Step 2:** Click the WINSRV3 server node in the left panel

**Step 3:** In the middle panel double-click Server Certificates

**Step 4:** In the right Actions panel click Create Domain Certificate

**Step 5:** Fill in the certificate details:

Common name:    winsrv3.manila.com

Organization:   Manila

Org. Unit:      IT

City/Locality:  Manila

State:          Metro Manila

Country:        PH

**Step 6:** Click Next

**Step 7:** Click Select — choose Manila-Issuing-CA (WINSRV3)

**Step 8:** Friendly name: WINSRV3 Web Certificate

**Step 9:** Click Finish

## **Step 3: Bind the Certificate to the HTTPS Site**

**Step 1:** In IIS Manager expand WINSRV3 → Sites

**Step 2:** Click the Default Web Site (or your target site)

**Step 3:** In the right Actions panel click Bindings

**Step 4:** Click Add

**Step 5:** Set Type: https

**Step 6:** Port: 443

**Step 7:** SSL Certificate: select WINSRV3 Web Certificate

**Step 8:** Click OK → Close

**✓ Tip:** *Test by opening a browser on Client1 and browsing to https://winsrv3.manila.com — there should be no certificate warning if the certenroll GPO distributed the Root CA correctly.*

# **Part 7 — Installing a Certificate on LINSRV1 (Linux Web Server)**

The competition requires LINSRV1 (192.168.1.10) to use HTTPS. The preferred method is a certificate signed by WINSRV3. A self-signed certificate fallback is also documented below.

## **Method A — CA-Signed Certificate from WINSRV3 (Preferred)**

### **Step 1: Generate a CSR on LINSRV1**

**Step 1:** Log on to LINSRV1 as root or competitor with sudo

**Step 2:** Run the following command to generate a private key and CSR:

openssl req \-new \-newkey rsa:2048 \-nodes \\

  \-keyout /etc/pki/tls/private/linsrv1.key \\

  \-out /tmp/linsrv1.csr \\

  \-subj "/CN=linsrv1.manila.com/O=Manila/C=PH"

**Step 3:** Copy the CSR file to WINSRV3:

scp /tmp/linsrv1.csr administrator@192.168.2.30:C:\\linsrv1.csr

### **Step 2: Sign the CSR on WINSRV3**

**Step 1:** On WINSRV3 open a Command Prompt and run:

certreq \-submit \-attrib "CertificateTemplate:WebServer" C:\\linsrv1.csr C:\\linsrv1.cer

**Step 2:** If the above fails, open Certification Authority console

**Step 3:** Right-click Manila-Issuing-CA → All Tasks → Submit new request

**Step 4:** Browse to C:\\linsrv1.csr → Open

**Step 5:** Go to Pending Requests → right-click the request → Issue

**Step 6:** Go to Issued Certificates → double-click the new cert → Details → Copy to File

**Step 7:** Save as C:\\linsrv1.cer (DER format) or C:\\linsrv1.pem (Base-64)

**Step 8:** Also export the CA chain: run in Command Prompt:

certutil \-ca.cert C:\\issuing-ca.cer

### **Step 3: Install the Certificate on LINSRV1**

**Step 1:** Copy the certificate files back to LINSRV1:

\# From LINSRV1 terminal:

scp administrator@192.168.2.30:C:\\linsrv1.cer /etc/pki/tls/certs/linsrv1.cer

scp administrator@192.168.2.30:C:\\issuing-ca.cer /etc/pki/tls/certs/issuing-ca.cer

**Step 2:** Convert to PEM if needed:

openssl x509 \-inform DER \-in /etc/pki/tls/certs/linsrv1.cer \\

  \-out /etc/pki/tls/certs/linsrv1.pem

**Step 3:** Edit the Apache SSL configuration:

nano /etc/httpd/conf.d/ssl.conf

**Step 4:** Set the following directives:

SSLCertificateFile     /etc/pki/tls/certs/linsrv1.pem

SSLCertificateKeyFile  /etc/pki/tls/private/linsrv1.key

SSLCertificateChainFile /etc/pki/tls/certs/issuing-ca.cer

**Step 5:** Restart Apache:

systemctl restart httpd

## **Method B — Self-Signed Certificate (Fallback)**

**📝 Note:** *Use this method only if you cannot get WINSRV3 to sign the certificate.*

**Step 1:** On LINSRV1 run:

openssl req \-x509 \-nodes \-days 365 \-newkey rsa:2048 \\

  \-keyout /etc/pki/tls/private/linsrv1.key \\

  \-out /etc/pki/tls/certs/linsrv1.pem \\

  \-subj "/CN=linsrv1.manila.com/O=Manila/C=PH"

**Step 2:** Edit /etc/httpd/conf.d/ssl.conf:

SSLCertificateFile    /etc/pki/tls/certs/linsrv1.pem

SSLCertificateKeyFile /etc/pki/tls/private/linsrv1.key

**Step 3:** Restart Apache:

systemctl restart httpd

**⚠ Warning:** *With a self-signed certificate, domain clients will still get a certificate warning unless you manually import the self-signed cert into their Trusted Root store. A CA-signed cert distributed via GPO is strongly preferred.*

# **Part 8 — Configuring the OpenVPN Certificate**

The competition requires OpenVPN on pfSense to use a certificate produced by WINSRV3 as the CA cert. Below is how to export the CA certificate and import it into pfSense.

## **Step 1: Export the Issuing CA Certificate from WINSRV3**

**Step 1:** On WINSRV3 open Certification Authority console

**Step 2:** Right-click Manila-Issuing-CA → Properties

**Step 3:** Click View Certificate → Details tab → Copy to File

**Step 4:** In the Export Wizard select Base-64 encoded X.509 (.CER)

**Step 5:** Save as C:\\issuing-ca.cer

**Step 6:** Open the file in Notepad to view the PEM-format certificate (begins with \-----BEGIN CERTIFICATE-----)

## **Step 2: Import CA Certificate into pfSense**

**Step 1:** Log on to pfSense at https://192.168.2.254 using admin / P@ssw0rd

**Step 2:** Go to System → Certificate Manager → CAs

**Step 3:** Click Add

**Step 4:** Fill in:

Descriptive Name: Manila-Issuing-CA

Method:           Import an existing Certificate Authority

**Step 5:** Paste the contents of issuing-ca.cer into the Certificate data field

**Step 6:** Click Save

## **Step 3: Create a Server Certificate for OpenVPN**

**Step 1:** In pfSense go to System → Certificate Manager → Certificates

**Step 2:** Click Add/Sign

**Step 3:** Select Method: Sign a Certificate Signing Request

**Step 4:** Fill in the certificate details:

Descriptive Name: OpenVPN-Server

Certificate Type: Server Certificate

**Step 5:** Click Save

**✓ Tip:** *If you prefer to create the OpenVPN server certificate directly from WINSRV3 and import it into pfSense, follow the same CSR/sign process from Part 7 but use the OpenVPN Server certificate template.*

# **Part 9 — Troubleshooting PKI Issues**

## **Certificate Auto-Enrollment Not Working**

* Run gpupdate /force on the client, then certutil \-pulse

* Check the Application event log for CAPI2 or AutoEnrollment errors

* Verify the certenroll GPO is linked to the domain and not blocked by inheritance

* Ensure the CA is online: certutil \-ping from the client

* Verify the DNS A record for WINSRV3.manila.com resolves to 192.168.2.30

## **Certificate Warning Still Appears in Browser**

* Verify the Root CA is in Trusted Root Certification Authorities on the client: run certmgr.msc

* Check the certificate Common Name matches the hostname you are browsing to

* Ensure the HTTPS binding in IIS uses the correct certificate

* Run gpupdate /force on the client and restart the browser

## **IIS Certificate Request Fails**

* Confirm the Web Server certificate template is published: Certification Authority → Certificate Templates

* Check that WINSRV3's CA service is running: services.msc → Active Directory Certificate Services

* Verify firewall allows TCP 135 and TCP 49152-65535 (RPC) between WINSRV3 and clients in the SERVERS VLAN

## **LINSRV1 Apache Cannot Start After SSL Config**

**Step 1:** Check for config syntax errors:

httpd \-t

**Step 2:** Check certificate and key file permissions:

ls \-la /etc/pki/tls/certs/linsrv1.pem

ls \-la /etc/pki/tls/private/linsrv1.key

**Step 3:** Verify SELinux context (if SELinux is enabled):

restorecon \-Rv /etc/pki/tls/

**Step 4:** Check Apache error logs:

tail \-50 /var/log/httpd/error\_log

# **Part 10 — Final Validation Checklist**

Run through every item before signaling completion to the experts.

## **WINSRV3 — Issuing CA**

| Check | How to Verify | Expected |
| :---- | :---- | :---- |
| AD CS is running | certutil \-ping | Successfully contacted CertSvc |
| CA name correct | Certification Authority console | Manila-Issuing-CA visible |
| CA is Subordinate type | CA Properties → General tab | Subordinate certification authority |
| Web Server cert on WINSRV3 | IIS Manager → Server Certificates | WINSRV3 Web Certificate present |
| HTTPS binding in IIS | IIS → Default Web Site → Bindings | Port 443 with certificate |

## **GPO — certenroll**

| Check | How to Verify | Expected |
| :---- | :---- | :---- |
| GPO linked to domain | Group Policy Management → manila.com | certenroll GPO listed |
| Auto-enrollment enabled (Computer) | GPO → Computer Config → Public Key Policies | Enabled with both checkboxes |
| Auto-enrollment enabled (User) | GPO → User Config → Public Key Policies | Enabled with both checkboxes |
| Root CA in GPO | GPO → Computer Config → Trusted Root CAs | Manila-Root-CA certificate |

## **Client Verification**

| Check | How to Verify | Expected |
| :---- | :---- | :---- |
| Root CA trusted on Client1 | certmgr.msc → Trusted Root → Certificates | Manila-Root-CA present |
| Personal cert issued to client | certmgr.msc → Personal → Certificates | Cert from Manila-Issuing-CA |
| No cert warning on WINSRV3 site | Browser → https://winsrv3.manila.com | Padlock icon, no warning |
| No cert warning on LINSRV1 site | Browser → https://linsrv1.manila.com | Padlock icon, no warning |
| No cert warning on w3.manila.com | Browser → https://w3.manila.com | Padlock icon, no warning |

## **LINSRV1 — Linux Web Server**

| Check | How to Verify | Expected |
| :---- | :---- | :---- |
| Apache running with SSL | systemctl status httpd | Active (running) |
| HTTPS accessible from LAN | From Client1: curl \-k https://192.168.1.10 | HTML response |
| HTTPS accessible from Internet | From Client3: curl https://linsrv1.manila.com | HTML response |
| Certificate valid | openssl s\_client \-connect linsrv1.manila.com:443 | Verify OK |

**✓ Tip:** *Fix issues in order: network connectivity → DNS → CA service → certificate templates → GPO → client enrollment. Most failures trace back to DNS or firewall rules between VLANs.*

*WorldSkills ASEAN Manila 2025 — Cyber Security Skill 54 — Module A2 — PKI Reference Guide*

# PKI Layer: 2026

# **PKI Setup Guide — Single-Tier CA (WSPH2026 Alignment)**

*Adapted from the WSA2025 2-tier guide to match Test Project A2 (Cybersecurity 54, 21st National Skills Olympics — Manila, WSPH2026\_TP54\_MA2).*

**What changed from the WSA2025 version, and why:**

| WSA2025 guide | This version | Reason |
| ----- | ----- | ----- |
| 2-tier: WINSRV4 (offline Root) \+ WINSRV3 (subordinate/issuing) | **1 tier: WinSRV1 is the CA** — Root CA that issues directly | Test Project: *"a national one-day module uses a single-tier issuing CA unless otherwise specified by the Organizer"* |
| Dedicated CA VMs | No dedicated CA VM in Table 1 — only WinSRV1 (AD DS/DNS/policy) and LinuxWeb exist | This year's environment doesn't provision a separate PKI server |
| Cert target: WINSRV3 (IIS) \+ LINSRV1 (Apache) | Cert target: **LinuxWeb only** | Test Project's HTTPS requirement is scoped to the DMZ web service — no internal IIS site is asked for |
| CSR submitted directly DMZ↔CA server | **CSR routed through Client1 (LAN)** | Table 2 only allows DMZ→Servers "AD DS required ports (minimum)" — no HTTP/certsrv path. LAN→Servers explicitly allows "PKI as required," so the LAN tier is the legal path |

> Placeholders `<DOMAIN>`, `<ADMIN_PASSWORD>`, and any exact hostnames not in Table 1 are unknown until Competition Day Instructions are issued — swap them in on the day. Everything else (IPs, VLANs, VM names) is taken directly from Test Project Table 1\.

## **Quick Reference**

| Item | Value |
| ----- | ----- |
| CA host | **WinSRV1** (192.168.2.10, Servers VLAN) |
| CA architecture | Single tier — Enterprise **Root CA**, issues directly (no subordinate) |
| CA role services | Certification Authority **\+ Certification Authority Web Enrollment** |
| Cert target | **LinuxWeb** (192.168.1.10, DMZ) |
| Domain | `<DOMAIN>` — from Competition Day Instructions |
| Client trust mechanism | `certenroll` GPO — auto-enrollment (Computer \+ User) |
| CSR submission path | LinuxWeb → Client1 (scp) → WinSRV1 `/certsrv` (browser) → back to Client1 → LinuxWeb (scp) |

---

# **Part 1 — Why Single-Tier, and What It Means in the Wizard**

A single-tier CA is one server that is both the trust anchor *and* the day-to-day issuer — there's no offline root signing a separate subordinate. In the AD CS Configuration Wizard this is set up by picking **CA Type: Root CA** (not *Subordinate CA*) — the wizard doesn't have an option literally called "issuing CA"; a Root CA that has no parent and issues certificates directly *is* the single-tier issuing CA the Test Project asks for.

Because WinSRV1 is already the domain controller, making it an **Enterprise** Root CA (rather than Standalone) is the right call: Enterprise CAs auto-publish their certificate to Active Directory, integrate with certificate templates, and support GPO-based auto-enrollment — which is what makes the "internal clients trust the CA via policy" requirement both practical and verifiable.

**⚠ Trade-off to flag to your teammate:** putting the CA on the same box as the domain controller is not best practice in the real world, but it's what this year's VM list forces — there's no separate server to put it on unless the Organizer says otherwise on the day.

---

# **Part 2 — Install AD CS on WinSRV1**

## **Step 1: Add the Role**

**Step 1:** On WinSRV1, open Server Manager → Manage → Add Roles and Features

**Step 2:** Next through to Server Roles → check **Active Directory Certificate Services**

**Step 3:** Click Add Features when prompted → Next

**Step 4:** On Role Services, check:

* **Certification Authority**  
* **Certification Authority Web Enrollment** (needed — see Part 4 on why LinuxWeb can't reach this directly)

**Step 5:** Next → Next → Install → wait for completion

**Step 6:** Click **Configure Active Directory Certificate Services on the destination server**

## **Step 2: Configure as a Single-Tier Enterprise Root CA**

**Step 1:** Credentials: use existing AD credentials (domain admin) → Next

**Step 2:** Role Services: check both **Certification Authority** and **Certification Authority Web Enrollment** → Next

**Step 3:** Setup Type: select **Enterprise CA** → Next

**Step 4:** CA Type: select **Root CA** → Next

**Step 5:** Private Key: **Create a new private key** → Next

**Step 6:** Cryptography: accept defaults (RSA 2048, SHA256) unless Competition Day Instructions specify otherwise → Next

**Step 7:** CA Name: give it something identifiable, e.g. `<DOMAIN-SHORT>-CA` — Next

**Step 8:** Validity Period: default (5 years) is fine for a one-day module → Next

**Step 9:** Certificate database locations: accept defaults → Next → **Configure** → Close

**✓ Tip:** *Because this is an Enterprise Root CA, its certificate is automatically published to Active Directory and to the domain's NTAuth store — this replaces the manual "export RootCA.cer, import into a GPO's Trusted Root store" dance from the 2-tier guide. You still need the `certenroll` GPO for auto-enrollment (Part 3), but you generally don't need to hand-import the root cert.*

## **Step 3: Verify**

**Step 1:** Open PowerShell on WinSRV1 and run:

certutil \-ping

Expected: `Successfully contacted CertSvc`

**Step 2:** Open Server Manager → Tools → Certification Authority → confirm your CA name is listed and shows green status

---

# **Part 3 — `certenroll` GPO: Client Trust and Auto-Enrollment**

## **Step 1: Create and Link the GPO**

**Step 1:** On WinSRV1, open Group Policy Management

**Step 2:** Right-click `<DOMAIN>` → Create a GPO in this domain and Link it here

**Step 3:** Name it exactly `certenroll` → OK → Edit

## **Step 2: Enable Auto-Enrollment (Computer)**

**Step 1:** Navigate to Computer Configuration → Policies → Windows Settings → Security Settings → Public Key Policies

**Step 2:** Double-click **Certificate Services Client – Auto-Enrollment**

**Step 3:** Configuration Model: **Enabled**

**Step 4:** Check both boxes (renew expired/update pending, and update certs using templates) → OK

## **Step 3: Enable Auto-Enrollment (User)**

**Step 1:** Repeat the same three checks under User Configuration → Policies → Windows Settings → Security Settings → Public Key Policies → same dialog → OK

**Step 2:** Close the GPO Editor

## **Step 4: Apply and Verify**

**Step 1:** On WinSRV1: `gpupdate /force`

**Step 2:** On Client1 and Client2: `gpupdate /force` then `certutil -pulse`

**Step 3:** On a client, run `certmgr.msc` → **Trusted Root Certification Authorities** → confirm the CA's certificate is present

**✓ Tip:** *If it's not there yet, it will appear after the next `gpupdate` cycle since Enterprise CA certs propagate through AD automatically — give it a minute and re-check before troubleshooting.*

---

# **Part 4 — Issuing a Server Certificate to LinuxWeb**

This is the part the WSA2025 WinSRV1 guide never covered, and the one Table 2's firewall rules actually constrain the shape of. **DMZ → Servers is limited to "AD DS required ports (minimum)"** — there's no HTTP/certsrv path allowed directly from LinuxWeb to WinSRV1. But **LAN → Servers explicitly allows "PKI as required,"** and **LAN → DMZ allows SSH**. So the CSR has to be relayed through a LAN client.

## **Step 0: Publish the Web Server Template**

**Step 1:** On WinSRV1, open Certification Authority console → expand your CA → Certificate Templates

**Step 2:** Right-click → New → Certificate Template to Issue → select **Web Server** → OK

*(Skip this if it's already listed.)*

## **Step 1: Generate the CSR on LinuxWeb**

**Step 1:** SSH into LinuxWeb

**Step 2:** Run:

openssl req \-new \-newkey rsa:2048 \-nodes \\  
 \-keyout /etc/pki/tls/private/linuxweb.key \\  
 \-out /tmp/linuxweb.csr \\  
 \-subj "/CN=linuxweb.\<DOMAIN\>/O=WSPH2026/C=PH"

## **Step 2: Relay the CSR Through Client1 (LAN)**

**Step 1:** From LinuxWeb, copy the CSR to Client1 (LAN → DMZ, allowed):

scp /tmp/linuxweb.csr \<user\>@\<client1-ip\>:C:\\Temp\\linuxweb.csr

**Step 2:** On Client1, open a browser to WinSRV1's web enrollment page (LAN → Servers PKI path, allowed):

http://192.168.2.10/certsrv

**Step 3:** Log in with a domain account, click **Request a certificate → advanced certificate request → Submit a certificate request by using a base-64-encoded... file**

**Step 4:** Open `linuxweb.csr` in Notepad, paste its full contents into the request field

**Step 5:** Certificate Template: select **Web Server** → Submit

**Step 6:** If issued immediately, download the certificate (Base-64 encoded) as `linuxweb.cer`. If it shows **Pending**, an admin approves it via Certification Authority console → Pending Requests → right-click → **Issue**, then Client1 goes back to `/certsrv` → **View the status of a pending certificate request** to download it.

**Step 7:** While on `/certsrv`, also download the **CA certificate** itself (Download a CA certificate → Base-64 → save as `ca-root.cer`) — since this is single-tier, this one file *is* the full chain.

## **Step 3: Relay the Cert Back and Install**

**Step 1:** From Client1, copy both files back to LinuxWeb:

scp C:\\Temp\\linuxweb.cer \<user\>@192.168.1.10:/tmp/  
 scp C:\\Temp\\ca-root.cer \<user\>@192.168.1.10:/tmp/

**Step 2:** On LinuxWeb, move them into place and convert if needed:

sudo mv /tmp/linuxweb.cer /etc/pki/tls/certs/linuxweb.cer  
 sudo mv /tmp/ca-root.cer /etc/pki/tls/certs/ca-root.cer  
 openssl x509 \-inform DER \-in /etc/pki/tls/certs/linuxweb.cer \-out /etc/pki/tls/certs/linuxweb.pem 2\>/dev/null || cp /etc/pki/tls/certs/linuxweb.cer /etc/pki/tls/certs/linuxweb.pem

**Step 3:** Edit the Apache SSL config:

sudo nano /etc/httpd/conf.d/ssl.conf

Set:

SSLCertificateFile /etc/pki/tls/certs/linuxweb.pem  
 SSLCertificateKeyFile /etc/pki/tls/private/linuxweb.key  
 SSLCertificateChainFile /etc/pki/tls/certs/ca-root.cer

**Step 4:** Fix SELinux context before you enable enforcing mode (see Part 6\) so this doesn't silently break later:

sudo restorecon \-Rv /etc/pki/tls/

**Step 5:** Restart Apache:

sudo systemctl restart httpd

---

# **Part 5 — Functional Validation**

| Check | How | Expected |
| ----- | ----- | ----- |
| CA online | `certutil -ping` on WinSRV1 | Successfully contacted CertSvc |
| CA type correct | Certification Authority console → CA Properties → General | Root CA, no parent CA listed |
| `certenroll` GPO linked | Group Policy Management | Listed under `<DOMAIN>`, both Computer/User auto-enrollment enabled |
| CA trusted on clients | `certmgr.msc` on Client1/Client2 → Trusted Root | CA certificate present |
| Apache running with SSL | `systemctl status httpd` on LinuxWeb | active (running) |
| HTTPS from LAN | `curl -v https://192.168.1.10` from Client1 | Valid handshake, no `-k` needed once CA is trusted |
| HTTPS from an external client | From WinClient3, per firewall rules | Only if WAN→DMZ HTTPS is explicitly required by the task |
| Cert chain valid | `openssl s_client -connect linuxweb.<DOMAIN>:443 -CAfile /etc/pki/tls/certs/ca-root.cer` | `Verify return code: 0 (ok)` |
| No browser warning | Browse to `https://linuxweb.<DOMAIN>` from Client1 | Padlock, no warning |

---

# **Part 6 — Interaction With Linux Hardening**

The Test Project's Linux hardening tasks (non-default SSH port, no direct root login, SELinux enforcing) run *after* this PKI work, and they're the most common way HTTPS silently breaks. Before you sign off:

* After `setenforce 1` (or reboot into enforcing), re-run `sudo restorecon -Rv /etc/pki/tls/` — if the cert/key files were moved or created before contexts were fixed, Apache will fail to read them under enforcing mode with no obvious error beyond the Apache log.  
* Check `tail -50 /var/log/httpd/error_log` if `systemctl status httpd` shows a failure after enabling SELinux.  
* If you move SSH to a non-default port, that's a separate SELinux port-context fix (`semanage port -a -t ssh_port_t -p tcp <port>`) — unrelated to the cert, but worth doing back-to-back since both are "harden then re-verify service" steps.  
* Re-run the Part 5 HTTPS checks *after* hardening, not just once before it — the Test Project's PKI section and Linux hardening section are graded as functioning together, not independently.

---

# **Part 7 — Troubleshooting**

**`/certsrv` not reachable from Client1**

* Confirm the Web Enrollment role installed (Part 2, Step 1\) — it's a separate checkbox from Certification Authority  
* Check the LAN→Servers firewall rule actually permits the port (80/443) you're using for `/certsrv`

**Web Server template missing from the template list on submission**

* Go back to Part 4, Step 0 — the template has to be explicitly issued by the CA before it appears

**Certificate request stuck as Pending**

* Check the CA's issuance settings — Enterprise CAs matching an issued template *can* auto-issue, but templates can still be configured to require manager approval. Issue it manually via Pending Requests if so.

**Auto-enrollment not delivering the CA cert to clients**

* `gpupdate /force` then `certutil -pulse` on the client  
* Verify the `certenroll` GPO isn't blocked by inheritance or scoped out by security filtering

**Apache won't start after the SSL config change**

* `httpd -t` for syntax errors  
* Check file permissions on the key/cert  
* `restorecon -Rv /etc/pki/tls/` if SELinux is already enforcing

---

*Adapted for WSPH2026\_TP54\_MA2 from the WSA2025 2-tier PKI reference. Replace `<DOMAIN>` and `<ADMIN_PASSWORD>` with the values from Competition Day Instructions before use.*

# MA2 ISP Layer: Earl

**ISP VM Complete Setup Guide**

*Rocky Linux — Simulated Internet for Competition*

WorldSkills ASEAN Manila 2025 — Cyber Security Skill 54 — Module A2

**Quick Reference**

| Item | Value |
| :---- | :---- |
| VM Name | ISP |
| OS | Rocky Linux 9.x |
| VLAN | Internet |
| IP Address | Dynamic — assigned based on your Internet subnet |
| Services | DHCP, DNS, Apache HTTPS (two websites) |
| Website 1 | https://www.nationalmuseum.gov.ph |
| Website 2 | https://www.starcity.com.ph |
| DHCP Serves | pfSense WAN interface and Client3 |
| Credentials | N/A — competitors do not log in on competition day |
| Root Password (practice) | P@ssw0rd |

**What This VM Does**

| Service | Purpose | Used By |
| :---- | :---- | :---- |
| DHCP Server | Assigns IP addresses to WAN-facing devices | pfSense WAN interface, Client3 |
| DNS Server (BIND) | Resolves nationalmuseum.gov.ph and starcity.com.ph to this VM's IP | pfSense, Client3, all internal clients via pfSense forwarding |
| Apache HTTPS | Hosts two fake websites with self-signed certificates for firewall testing | Internal clients testing firewall rules, WinClient3 |
| Rocky Linux Repository (optional) | Hosts local package mirror for LINSRV1 to install packages without real internet | LINSRV1 when DMZ has no internet |

**📝 Note:** *On competition day this VM is pre-built and running. Competitors do not need to log in. This guide is for practice lab setup only.*

# **Part 1 — Create ISP VM in ESXi**

Create the ISP virtual machine in ESXi before installing Rocky Linux.

## **Step 1: Open ESXi Web Interface**

**Step 1:** Open browser on competitor laptop

**Step 2:** Navigate to:

https://192.168.1.1

**Step 3:** Login with ESXi credentials:

User:     pmuser

Password: Paul Bocuse

## **Step 2: Create the VM**

**Step 1:** Click Virtual Machines in the left panel

**Step 2:** Click Create / Register VM

**Step 3:** Select Create a new virtual machine → click Next

**Step 4:** Fill in VM details:

Name:             ISP

Compatibility:    ESXi 6.7 or later

Guest OS Family:  Linux

Guest OS Version: Rocky Linux 9 (64-bit)

**Step 5:** Click Next

**Step 6:** Select datastore1 → click Next

**Step 7:** Customize hardware settings:

CPU:           2 vCPUs

RAM:           2 GB

Hard Disk:     40 GB

Network:       Internet  ← CRITICAL: must be Internet port group

CD/DVD Drive:  Datastore ISO file → select Rocky Linux 9 ISO

**Step 8:** Click Next → Finish

**⚠ Warning:** *Network adapter MUST be connected to the Internet port group. This is the same VLAN as pfSense WAN and Client3.*

# **Part 2 — Install Rocky Linux 9**

Install Rocky Linux using Minimal or Server installation. ISP does not need a GUI.

## **Step 1: Boot from ISO**

**Step 1:** Power on the ISP VM in ESXi

**Step 2:** Open the VM console

**Step 3:** When boot menu appears select:

Install Rocky Linux 9

**Step 4:** Press Enter

## **Step 2: Language and Keyboard**

**Step 1:** Select: English (United States)

**Step 2:** Click Continue

## **Step 3: Software Selection**

**Step 1:** Click Software Selection

**Step 2:** For Base Environment select:

Server

**Step 3:** On the right side check these add-ons:

* ✅ Basic Web Server — includes Apache httpd

* ✅ System Tools

* ✅ Network Servers — includes DHCP and BIND DNS

**Step 4:** Click Done

**📝 Note:** *Selecting 'Server' with Network Servers add-on installs DHCP (dhcpd) and DNS (BIND/named) packages automatically.*

## **Step 4: Network Configuration**

**Step 1:** Click Network & Host Name

**Step 2:** Click the toggle to turn ON the network adapter

**Step 3:** Click Configure

**Step 4:** Click IPv4 Settings tab

**Step 5:** Change Method to: Automatic (DHCP)

**📝 Note:** *ISP gets its own IP via DHCP from the physical router or ESXi management network on the Internet VLAN. Leave as DHCP here — you will set a static IP after installation if needed.*

**Step 6:** Set Hostname:

ISP

**Step 7:** Click Done

## **Step 5: Installation Destination**

**Step 1:** Click Installation Destination

**Step 2:** Select the 40GB disk

**Step 3:** Select: Automatic partitioning

**Step 4:** Click Done

## **Step 6: Root Password**

**Step 1:** Click Root Password

**Step 2:** Enter: P@ssw0rd

**Step 3:** Confirm: P@ssw0rd

**Step 4:** Click Done

## **Step 7: Begin Installation**

**Step 1:** Click Begin Installation

**Step 2:** Wait approximately 10-15 minutes for installation to complete

**Step 3:** When complete click Reboot System

**Step 4:** After reboot unmount ISO:

* Go to ESXi → ISP VM → Edit Settings

* CD/DVD Drive → uncheck Connected

* Click Save

# **Part 3 — Initial Setup After Installation**

## **Step 1: Find the VM IP Address**

**Step 1:** Login to ISP VM console as root

**Step 2:** Run:

ip a

**Step 3:** Note the IP address on ens33 (or your interface)

**Step 4:** This is the IP assigned by the physical router DHCP

**Step 5:** Run ip route to find the gateway:

ip route

**Step 6:** Note the default gateway — this is your Internet subnet

**📝 Note:** *The ISP VM IP and subnet depends on your physical network. For example if your physical router uses 192.168.99.0/24, ISP will get an IP like 192.168.99.x.*

## **Step 2: Set Static IP (Recommended)**

Give ISP a static IP so it stays consistent for DHCP and DNS configuration.

**Step 1:** Check your interface name:

nmcli device status

**Step 2:** Set static IP — replace ens33 with your interface name and adjust subnet to match your network:

sudo nmcli con mod "ens33" ipv4.addresses 192.168.99.1/24

sudo nmcli con mod "ens33" ipv4.gateway 192.168.99.254

sudo nmcli con mod "ens33" ipv4.dns 8.8.8.8

sudo nmcli con mod "ens33" ipv4.method manual

sudo nmcli con down "ens33" && sudo nmcli con up "ens33"

**⚠ Warning:** *Adjust 192.168.99.1 and 192.168.99.254 to match your actual Internet VLAN subnet. Check what IP pfSense WAN gets from DHCP to determine the correct subnet.*

## **Step 3: Set Hostname**

sudo hostnamectl set-hostname ISP

hostnamectl

## **Step 4: Install Required Packages**

**Step 1:** Update and install all needed services:

sudo dnf update \-y

sudo dnf install \-y httpd mod\_ssl openssl dhcp-server bind bind-utils firewalld

## **Step 5: Enable Services**

sudo systemctl enable \--now httpd

sudo systemctl enable \--now named

sudo systemctl enable \--now dhcpd

sudo systemctl enable \--now firewalld

# **Part 4 — Configure DHCP Server**

The ISP DHCP server assigns IP addresses to pfSense WAN interface and Client3. This simulates how a real ISP provides IPs to customers.

## **Understanding the DHCP Role**

| Device | Gets IP From ISP DHCP | Purpose |
| :---- | :---- | :---- |
| pfSense WAN | Yes — DHCP client on WAN interface | pfSense gets internet-facing IP from ISP |
| Client3 | Yes — DHCP client on Internet VLAN | External client for testing VPN and DMZ access |
| pfSense LAN clients | No — pfSense provides DHCP to LAN | Internal clients get IPs from pfSense |

## **Step 1: Edit DHCP Configuration**

**Step 1:** Open the DHCP server configuration file:

sudo vi /etc/dhcp/dhcpd.conf

**Step 2:** Replace the entire file content with this configuration — adjust subnet to match your Internet VLAN:

\# ISP DHCP Server Configuration  
\# WorldSkills ASEAN Manila 2025  
   
default-lease-time 600;  
max-lease-time 7200;  
authoritative;  
   
\# Internet VLAN subnet  
\# Adjust this subnet to match your actual Internet VLAN  
subnet 192.168.99.0 netmask 255.255.255.0 {  
    range 192.168.99.100 192.168.99.200;  
    option routers 192.168.99.1;  
    option domain-name-servers 192.168.99.1;  
    option domain-name "isp.local";  
    default-lease-time 600;  
    max-lease-time 7200;  
}

**Step 3:** Save the file: Esc → :wq → Enter

## **Step 2: Specify Network Interface for DHCP**

**Step 1:** Edit the DHCP service configuration:

sudo vi /etc/sysconfig/dhcpd

**Step 2:** Add or modify this line — replace ens33 with your interface name:

DHCPDARGS=ens33

**Step 3:** Save: Esc → :wq → Enter

## **Step 3: Start and Enable DHCP**

sudo systemctl restart dhcpd

sudo systemctl enable dhcpd

sudo systemctl status dhcpd

Should show: active (running)

## **Step 4: Open Firewall for DHCP**

sudo firewall-cmd \--permanent \--add-service=dhcp

sudo firewall-cmd \--reload

## **Step 5: Verify DHCP is Working**

**Step 1:** On pfSense console select Option 2 → WAN

**Step 2:** Select Configure IPv4 via DHCP: y

**Step 3:** pfSense WAN should receive an IP from ISP DHCP pool

**Step 4:** Check pfSense Status → Dashboard → WAN shows IP in 192.168.99.100-200 range

**Step 5:** On Client3 — verify it receives DHCP:

ipconfig

Should show IP in range 192.168.99.100-200

**✓ Tip:** *If DHCP is not working, check: sudo journalctl \-u dhcpd \-n 20 for error messages.*

# **Part 5 — Configure DNS Server (BIND)**

The ISP DNS server resolves the two competition test websites (nationalmuseum.gov.ph and starcity.com.ph) to the ISP VM's own IP address. This allows internal clients to find and access the fake websites.

## **Step 1: Edit Main BIND Configuration**

**Step 1:** Open the named configuration file:

sudo vi /etc/named.conf

**Step 2:** Find the options section and modify:

options {  
    listen-on port 53 { any; };  
    listen-on-v6 port 53 { ::1; };  
    directory       "/var/named";  
    dump-file       "/var/named/data/cache\_dump.db";  
    statistics-file "/var/named/data/named\_stats.txt";  
    memstatistics-file "/var/named/data/named\_mem\_stats.txt";  
    allow-query     { any; };  
    recursion yes;  
    forwarders { 8.8.8.8; 8.8.4.4; };  
    forward only;  
    dnssec-validation no;  
};

**Step 3:** At the bottom of named.conf add zone declarations:

zone "nationalmuseum.gov.ph" IN {  
    type master;  
    file "/var/named/nationalmuseum.gov.ph.zone";  
};  
   
zone "starcity.com.ph" IN {  
    type master;  
    file "/var/named/starcity.com.ph.zone";  
};

**Step 4:** Save: Esc → :wq → Enter

## **Step 2: Create Zone File for nationalmuseum.gov.ph**

**Step 1:** Create the zone file — replace 192.168.99.1 with your actual ISP VM IP:

sudo vi /var/named/nationalmuseum.gov.ph.zone

**Step 2:** Add this content:

$TTL 86400  
@   IN  SOA ns1.nationalmuseum.gov.ph. admin.nationalmuseum.gov.ph. (  
            2025010101 ; Serial  
            3600       ; Refresh  
            1800       ; Retry  
            604800     ; Expire  
            86400 )    ; Minimum TTL  
   
@       IN  NS  ns1.nationalmuseum.gov.ph.  
@       IN  A   192.168.99.1  
www     IN  A   192.168.99.1  
ns1     IN  A   192.168.99.1

**Step 3:** Save: Esc → :wq → Enter

## **Step 3: Create Zone File for starcity.com.ph**

**Step 1:** Create the zone file:

sudo vi /var/named/starcity.com.ph.zone

**Step 2:** Add this content:

$TTL 86400  
@   IN  SOA ns1.starcity.com.ph. admin.starcity.com.ph. (  
            2025010101 ; Serial  
            3600       ; Refresh  
            1800       ; Retry  
            604800     ; Expire  
            86400 )    ; Minimum TTL  
   
@       IN  NS  ns1.starcity.com.ph.  
@       IN  A   192.168.99.1  
www     IN  A   192.168.99.1  
ns1     IN  A   192.168.99.1

**Step 3:** Save: Esc → :wq → Enter

## **Step 4: Set Correct Ownership on Zone Files**

sudo chown named:named /var/named/nationalmuseum.gov.ph.zone

sudo chown named:named /var/named/starcity.com.ph.zone

sudo chmod 640 /var/named/nationalmuseum.gov.ph.zone

sudo chmod 640 /var/named/starcity.com.ph.zone

## **Step 5: Start and Enable DNS**

sudo named-checkconf

sudo named-checkzone nationalmuseum.gov.ph /var/named/nationalmuseum.gov.ph.zone

sudo named-checkzone starcity.com.ph /var/named/starcity.com.ph.zone

All three should return OK. Then:

sudo systemctl restart named

sudo systemctl enable named

sudo systemctl status named

## **Step 6: Open Firewall for DNS**

sudo firewall-cmd \--permanent \--add-service=dns

sudo firewall-cmd \--reload

## **Step 7: Verify DNS Resolution**

**Step 1:** Test locally on ISP VM:

nslookup www.nationalmuseum.gov.ph 192.168.99.1

nslookup www.starcity.com.ph 192.168.99.1

**Step 2:** Both should return: 192.168.99.1 (or your ISP VM IP)

**Step 3:** Test from pfSense Diagnostics → Ping → use ISP IP as DNS:

nslookup www.nationalmuseum.gov.ph

**✓ Tip:** *If named fails to start, check: sudo journalctl \-u named \-n 30 for detailed errors.*

# **Part 6 — Generate Self-Signed SSL Certificates**

Both websites use HTTPS with self-signed certificates as stated in the competition document: 'created intentionally with a self-signed certificate.' This tests whether clients can browse despite certificate warnings, and whether pfSense firewall rules work correctly.

## **Step 1: Create Certificate Directory**

sudo mkdir \-p /etc/pki/tls/isp

sudo chmod 700 /etc/pki/tls/isp

## **Step 2: Generate Certificate for nationalmuseum.gov.ph**

**Step 1:** Generate self-signed certificate — type all on ONE LINE:

sudo openssl req \-x509 \-nodes \-newkey rsa:2048 \-keyout /etc/pki/tls/isp/nationalmuseum.key \-out /etc/pki/tls/isp/nationalmuseum.crt \-days 365 \-subj "/C=PH/ST=Metro Manila/L=Manila/O=ISP/CN=www.nationalmuseum.gov.ph"

## **Step 3: Generate Certificate for starcity.com.ph**

sudo openssl req \-x509 \-nodes \-newkey rsa:2048 \-keyout /etc/pki/tls/isp/starcity.key \-out /etc/pki/tls/isp/starcity.crt \-days 365 \-subj "/C=PH/ST=Metro Manila/L=Manila/O=ISP/CN=www.starcity.com.ph"

## **Step 4: Set Certificate Permissions**

sudo chmod 600 /etc/pki/tls/isp/\*.key

sudo chmod 644 /etc/pki/tls/isp/\*.crt

sudo restorecon \-Rv /etc/pki/tls/isp

# 

# 

# 

# **Part 7 — Configure Apache Virtual Hosts**

Create two separate HTTPS virtual host configurations for the two competition test websites.

## **Step 1: Create Website Directories**

sudo mkdir \-p /var/www/nationalmuseum

sudo mkdir \-p /var/www/starcity

## **Step 2: Create nationalmuseum.gov.ph Homepage**

**Step 1:** Create the index.html:

sudo vi /var/www/nationalmuseum/index.html

**Step 2:** Add content:

\<\!DOCTYPE html\>  
\<html\>  
\<head\>  
    \<title\>National Museum of the Philippines\</title\>  
    \<style\>  
        body { font-family: Arial; text-align: center; padding: 50px; background: \#f0f0f0; }  
        h1 { color: \#1a5276; }  
        p { color: \#555; }  
    \</style\>  
\</head\>  
\<body\>  
    \<h1\>National Museum of the Philippines\</h1\>  
    \<p\>WorldSkills ASEAN Manila 2025 \- Firewall Test Website\</p\>  
    \<p\>If you can see this page, firewall rules allow access to nationalmuseum.gov.ph\</p\>  
\</body\>  
\</html\>

**Step 3:** Save: Esc → :wq → Enter

## **Step 3: Create starcity.com.ph Homepage**

**Step 1:** Create the index.html:

sudo vi /var/www/starcity/index.html

**Step 2:** Add content:

\<\!DOCTYPE html\>  
\<html\>  
\<head\>  
    \<title\>Star City Amusement Park\</title\>  
    \<style\>  
        body { font-family: Arial; text-align: center; padding: 50px; background: \#fff3cd; }  
        h1 { color: \#856404; }  
        p { color: \#555; }  
    \</style\>  
\</head\>  
\<body\>  
    \<h1\>Star City Amusement Park\</h1\>  
    \<p\>WorldSkills ASEAN Manila 2025 \- Firewall Test Website (SHOULD BE BLOCKED)\</p\>  
    \<p\>If you can see this page, firewall rules are NOT blocking starcity.com.ph correctly\!\</p\>  
\</body\>  
\</html\>

**Step 3:** Save: Esc → :wq → Enter

## **Step 4: Set SELinux Context on Website Directories**

sudo semanage fcontext \-a \-t httpd\_sys\_content\_t "/var/www/nationalmuseum(/.\*)?"

sudo semanage fcontext \-a \-t httpd\_sys\_content\_t "/var/www/starcity(/.\*)?"

sudo restorecon \-Rv /var/www/nationalmuseum

sudo restorecon \-Rv /var/www/starcity

## **Step 5: Create Apache Virtual Host for nationalmuseum.gov.ph**

**Step 1:** Create the Apache config file:

sudo vi /etc/httpd/conf.d/nationalmuseum.conf

**Step 2:** Add this content:

\# HTTP \- redirect to HTTPS  
\<VirtualHost \*:80\>  
    ServerName www.nationalmuseum.gov.ph  
    ServerAlias nationalmuseum.gov.ph  
    Redirect permanent / https://www.nationalmuseum.gov.ph/  
\</VirtualHost\>  
   
\# HTTPS  
\<VirtualHost \*:443\>  
    ServerName www.nationalmuseum.gov.ph  
    ServerAlias nationalmuseum.gov.ph  
    DocumentRoot /var/www/nationalmuseum  
   
    SSLEngine on  
    SSLCertificateFile    /etc/pki/tls/isp/nationalmuseum.crt  
    SSLCertificateKeyFile /etc/pki/tls/isp/nationalmuseum.key  
   
    \<Directory /var/www/nationalmuseum\>  
        AllowOverride None  
        Require all granted  
        Options \-Indexes  
    \</Directory\>  
   
    ErrorLog  /var/log/httpd/nationalmuseum-error.log  
    CustomLog /var/log/httpd/nationalmuseum-access.log combined  
\</VirtualHost\>

**Step 3:** Save: Esc → :wq → Enter

## **Step 6: Create Apache Virtual Host for starcity.com.ph**

**Step 1:** Create the Apache config file:

sudo vi /etc/httpd/conf.d/starcity.conf

**Step 2:** Add this content:

\# HTTP \- redirect to HTTPS  
\<VirtualHost \*:80\>  
    ServerName www.starcity.com.ph  
    ServerAlias starcity.com.ph  
    Redirect permanent / https://www.starcity.com.ph/  
\</VirtualHost\>  
   
\# HTTPS  
\<VirtualHost \*:443\>  
    ServerName www.starcity.com.ph  
    ServerAlias starcity.com.ph  
    DocumentRoot /var/www/starcity  
   
    SSLEngine on  
    SSLCertificateFile    /etc/pki/tls/isp/starcity.crt  
    SSLCertificateKeyFile /etc/pki/tls/isp/starcity.key  
   
    \<Directory /var/www/starcity\>  
        AllowOverride None  
        Require all granted  
        Options \-Indexes  
    \</Directory\>  
   
    ErrorLog  /var/log/httpd/starcity-error.log  
    CustomLog /var/log/httpd/starcity-access.log combined  
\</VirtualHost\>

**Step 3:** Save: Esc → :wq → Enter

## **Step 7: Test and Restart Apache**

**Step 1:** Test configuration syntax:

sudo apachectl configtest

Must show: Syntax OK before proceeding

**Step 2:** Restart Apache:

sudo systemctl restart httpd

sudo systemctl enable httpd

**Step 3:** Verify Apache is running:

sudo systemctl status httpd

# **Part 8 — Configure Local Firewall**

Open required ports on the ISP VM's local firewall (firewalld).

## **Step 1: Open Required Ports**

sudo firewall-cmd \--permanent \--add-service=http

sudo firewall-cmd \--permanent \--add-service=https

sudo firewall-cmd \--permanent \--add-service=dns

sudo firewall-cmd \--permanent \--add-service=dhcp

sudo firewall-cmd \--reload

## **Step 2: Verify Firewall Rules**

sudo firewall-cmd \--list-all

Should show services include: http https dns dhcp

# **Part 9 — Configure SELinux**

SELinux is enabled by default on Rocky Linux. Configure it to allow Apache and DNS to run correctly.

## **Step 1: Verify SELinux Status**

sestatus

Should show: SELinux status: enabled

## **Step 2: Allow Apache Network Connections**

sudo setsebool \-P httpd\_can\_network\_connect on

sudo setsebool \-P httpd\_can\_network\_relay on

## **Step 3: Allow NAMED (DNS) to Work**

sudo setsebool \-P named\_write\_master\_zones on

## **Step 4: Verify SELinux Contexts**

ls \-Z /var/www/nationalmuseum/

ls \-Z /var/www/starcity/

Should show httpd\_sys\_content\_t context on files

# **Part 10 — Optional: Rocky Linux Package Repository**

If you have the Rocky Linux 9 DVD ISO, you can host a local package repository on ISP so LINSRV1 can install packages without real internet access.

**📝 Note:** *This is optional. On competition day, LINSRV1 has all packages pre-installed. This is only needed when building from scratch for practice.*

## **Step 1: Mount Rocky Linux DVD ISO**

**Step 1:** In ESXi attach Rocky Linux DVD ISO to ISP VM

**Step 2:** On ISP VM mount the ISO:

sudo mkdir \-p /mnt/rockyiso

sudo mount /dev/cdrom /mnt/rockyiso

## **Step 2: Copy Packages to Web Directory**

sudo mkdir \-p /var/www/html/rockylinux/9/BaseOS/x86\_64/os

sudo mkdir \-p /var/www/html/rockylinux/9/AppStream/x86\_64/os

sudo cp \-r /mnt/rockyiso/BaseOS/. /var/www/html/rockylinux/9/BaseOS/x86\_64/os/

sudo cp \-r /mnt/rockyiso/AppStream/. /var/www/html/rockylinux/9/AppStream/x86\_64/os/

## **Step 3: Install createrepo and Generate Metadata**

sudo dnf install createrepo\_c \-y

sudo createrepo /var/www/html/rockylinux/9/BaseOS/x86\_64/os/

sudo createrepo /var/www/html/rockylinux/9/AppStream/x86\_64/os/

## **Step 4: Set SELinux Context**

sudo semanage fcontext \-a \-t httpd\_sys\_content\_t "/var/www/html/rockylinux(/.\*)?"

sudo restorecon \-Rv /var/www/html/rockylinux

## **Step 5: Restart Apache**

sudo systemctl restart httpd

## 

## 

## **Step 6: Configure LINSRV1 to Use ISP Repository**

On LINSRV1 create a repo file pointing to ISP:

sudo vi /etc/yum.repos.d/isp.repo

\[isp-baseos\]  
name=ISP BaseOS  
baseurl=http://192.168.99.1/rockylinux/9/BaseOS/x86\_64/os/  
enabled=1  
gpgcheck=0  
sslverify=0  
   
\[isp-appstream\]  
name=ISP AppStream  
baseurl=http://192.168.99.1/rockylinux/9/AppStream/x86\_64/os/  
enabled=1  
gpgcheck=0  
sslverify=0

**Step 7:** Test on LINSRV1:

sudo dnf repolist

sudo dnf install realmd adcli \-y

# **Part 11 — Full Verification**

Test all ISP services before using the competition network.

## **Verify DHCP**

| Test | How | Expected |
| :---- | :---- | :---- |
| pfSense WAN gets IP | pfSense Status → Dashboard → WAN | IP in 192.168.99.100-200 range |
| Client3 gets IP | ipconfig on Client3 | IP in 192.168.99.100-200 range |
| DHCP gives correct DNS | ipconfig /all on Client3 | DNS \= 192.168.99.1 |
| DHCP gives correct gateway | ipconfig /all on Client3 | Gateway \= 192.168.99.1 |

## **Verify DNS**

| Test | Command | Expected |
| :---- | :---- | :---- |
| nationalmuseum.gov.ph resolves | nslookup www.nationalmuseum.gov.ph 192.168.99.1 | 192.168.99.1 |
| starcity.com.ph resolves | nslookup www.starcity.com.ph 192.168.99.1 | 192.168.99.1 |
| DNS from pfSense works | Diagnostics → Ping → ping www.nationalmuseum.gov.ph | Resolves and pings |

## **Verify Websites**

| Test | How | Expected |
| :---- | :---- | :---- |
| nationalmuseum HTTPS | curl \-k https://www.nationalmuseum.gov.ph on ISP VM | Returns HTML page |
| starcity HTTPS | curl \-k https://www.starcity.com.ph on ISP VM | Returns HTML page |
| Browse from Client3 | Open browser on Client3 → https://www.nationalmuseum.gov.ph | Page loads (with cert warning — self-signed) |
| Browse from LAN client | Browser on Client1 → https://www.nationalmuseum.gov.ph | Page loads if firewall allows |
| starcity blocked from LAN | Browser on Client1 → https://www.starcity.com.ph | Should be blocked by pfSense |

## **Verify Firewall Testing Purpose**

The two websites serve different firewall testing purposes:

| Website | Firewall Rule | Expected from LAN | Expected from Client3 |
| :---- | :---- | :---- | :---- |
| nationalmuseum.gov.ph | ALLOWED — pfSense permits LAN to access this | ✅ Page loads | ✅ Page loads |
| starcity.com.ph | BLOCKED — pfSense blocks LAN from this | ❌ Connection refused/timeout | ✅ Page loads (external) |

**✓ Tip:** *If LAN clients cannot reach nationalmuseum.gov.ph but can reach 8.8.8.8, the issue is pfSense DNS not forwarding to ISP DNS. Check pfSense Services → DNS Resolver → Enable Forwarding Mode.*

# **Part 12 — Troubleshooting Reference**

| Problem | Likely Cause | Fix |
| :---- | :---- | :---- |
| pfSense WAN gets no IP | ISP DHCP not running or wrong subnet | sudo systemctl restart dhcpd; check subnet matches Internet VLAN |
| DNS not resolving from pfSense | pfSense not using ISP as upstream DNS | pfSense: Services → DNS Resolver → Enable Forwarding → DNS \= ISP IP |
| Website not loading — connection refused | Apache not running or firewall blocking 443 | sudo systemctl restart httpd; firewall-cmd \--add-service=https |
| Certificate error in browser | Self-signed certificate — expected\! | Click Advanced → Accept risk. This is intentional per competition document |
| named fails to start | Zone file syntax error | sudo named-checkzone \+ named-checkconf to find errors |
| dhcpd fails to start | Wrong subnet or interface | Check /etc/dhcp/dhcpd.conf subnet matches actual Internet VLAN |
| SELinux blocking Apache | Wrong file context | sudo restorecon \-Rv /var/www/nationalmuseum /var/www/starcity |
| Client3 cannot browse websites | DNS not configured on Client3 DHCP | Verify DHCP option 6 (DNS) points to ISP IP |

# **Final Configuration Summary**

| Service | Config File | Status Check |
| :---- | :---- | :---- |
| DHCP Server | /etc/dhcp/dhcpd.conf | systemctl status dhcpd |
| DNS Server (BIND) | /etc/named.conf \+ zone files | systemctl status named |
| Apache HTTP/HTTPS | /etc/httpd/conf.d/\*.conf | systemctl status httpd |
| Firewall | firewall-cmd rules | firewall-cmd \--list-all |
| SELinux | /etc/selinux/config | sestatus |

*WorldSkills ASEAN Manila 2025 — Cyber Security Skill 54 — ISP VM Setup Guide (Rocky Linux)*

# MA2 DC Layer: Earl

## **WINSRV1 Complete Guide for Manila Competition**

---

## **What WINSRV1 Does**

WINSRV1 is the Domain Controller providing:  
├── Active Directory (AD)  
├── DNS for internal network  
├── DHCP for LAN clients  
└── File Server (shares)

---

## **Part 1 — Password Policies**

### **Task 1 — Default Domain Password Policy**

**What:** All domain users must have:

* Minimum 8 characters  
* Changed every month

**Why:** Prevents weak passwords across entire domain

**Step by Step:**

1. Open **Server Manager → Tools → Group Policy Management**  
2. Expand **Forest → Domains → manila.com**  
3. Right-click **Default Domain Policy**  
4. Click **Edit**  
5. Navigate to:

Computer Configuration → Policies → Windows Settings →  
Security Settings → Account Policies → Password Policy

6. Configure each setting:

**Minimum Password Length:**

1. Double-click **Minimum password length**  
2. Check ✅ **Define this policy setting**  
3. Set to `8`  
4. Click **OK**

**Maximum Password Age (monthly):**

1. Double-click **Maximum password age**  
2. Check ✅ **Define this policy setting**  
3. Set to `30`  
4. Click **OK**

**Password Complexity:**

1. Double-click **Password must meet complexity requirements**

2. Select **Enabled**

3. Click **OK**

4. Close GPO Editor

5. Run on WINSRV1:

gpupdate /force

---

### **Task 2 — Fine-Grained Password Policy for Executive Group**

**What:** Executive group members need 10 character passwords

**Why:** Higher privilege users need stronger passwords

**Step by Step:**

1. Open **Server Manager → Tools → Active Directory Administrative Center**  
2. Click **manila (local)** in left panel  
3. Double-click **System** folder  
4. Double-click **Password Settings Container**  
5. On the right panel click **New → Password Settings**  
6. Fill in:

Name:                    Executive-PSO  
Precedence:              1  
Minimum password length: 10  
Maximum password age:    30 days  
Minimum password age:    1 day  
Password history:        24  
Complexity:              Enabled

7. Under **Directly Applies To** click **Add**  
8. Type `Executive` and click **OK**  
9. Click **OK** to save

**Verify it works:**

1. Open **Active Directory Users and Computers**  
2. Find an Executive group member  
3. Right-click → **Properties**  
4. Click **Attribute Editor tab**  
5. Look for `msDS-ResultantPSO`  
6. Should show `Executive-PSO`

---

## **Part 2 — Login Banner**

### **Task 3 — Create Login Banner**

**What:**

* Title: `WorldSkills ASEAN Manila`  
* Text: `authorized access only`

**Why:** Informs users they are accessing a secured system — legal requirement in many organizations

**Step by Step:**

1. Open **Group Policy Management**  
2. Right-click **Default Domain Policy → Edit**  
3. Navigate to:

Computer Configuration → Policies → Windows Settings →  
Security Settings → Local Policies → Security Options

4. Find **Interactive logon: Message title for users attempting to log on**  
5. Double-click it  
6. Check ✅ **Define this policy setting**  
7. Type:

WorldSkills ASEAN Manila

8. Click **OK**

9. Find **Interactive logon: Message text for users attempting to log on**

10. Double-click it

11. Check ✅ **Define this policy setting**

12. Type:

authorized access only

13. Click **OK**  
14. Close GPO Editor  
15. Run:

gpupdate /force

**Verify:** Lock the screen and log back in — you should see the banner before login.

---

## **Part 3 — GPO Tasks**

### **Task 4 — control GPO (Restrict Control Panel)**

**What:** Accounting users cannot access Control Panel

**Why:** Prevents unauthorized system changes by accounting staff

**Step by Step:**

#### **Create Accounting OU (if not exists)**

1. Open **Active Directory Users and Computers**  
2. Right-click **manila.com**  
3. Click **New → Organizational Unit**  
4. Name it `Accounting`  
5. Move accounting users into this OU

#### **Create GPO**

1. Open **Group Policy Management**  
2. Right-click **manila.com**  
3. Click **Create a GPO in this domain and Link it here**  
4. Name it:

control

5. Click **OK**

#### **Edit GPO**

1. Right-click **control → Edit**  
2. Navigate to:

User Configuration → Policies → Administrative Templates →  
Control Panel

3. Double-click **Prohibit access to Control Panel and PC settings**  
4. Select **Enabled**  
5. Click **OK**  
6. Close GPO Editor

#### **Apply Only to Accounting Users**

1. Click **control** GPO in left panel  
2. Click **Scope** tab  
3. Under **Security Filtering** remove **Authenticated Users**  
4. Click **Add**  
5. Type `Accounting` (the group or OU)  
6. Click **OK**

**Verify:** Login as an accounting user → try to open Control Panel → should be blocked.

---

### **Task 5 — registry GPO (Prevent Registry Editing)**

**What:** Users in Manila OU cannot use registry editing tools

**Why:** Prevents unauthorized registry modifications

**Step by Step:**

#### **Create Manila OU (if not exists)**

1. Open **Active Directory Users and Computers**  
2. Right-click **manila.com**  
3. Click **New → Organizational Unit**  
4. Name it `Manila`  
5. Move relevant users into this OU

#### **Create GPO**

1. Open **Group Policy Management**  
2. Right-click **Manila** OU  
3. Click **Create a GPO in this domain and Link it here**  
4. Name it:

registry

5. Click **OK**

#### **Edit GPO**

1. Right-click **registry → Edit**  
2. Navigate to:

User Configuration → Policies → Administrative Templates →  
System

3. Double-click **Prevent access to registry editing tools**  
4. Select **Enabled**  
5. Under **Disable regedit from running silently** select **Yes**  
6. Click **OK**  
7. Close GPO Editor

**Verify:** Login as Manila OU user → try to run `regedit` → should be blocked.

---

### **Task 6 — google GPO (Chrome Homepage)**

**What:** Force all users to use `http://w3.manila.com` as Chrome homepage

**Why:** Ensures all users start from company website

**Step by Step:**

#### **Step 1 — Find Chrome Enterprise Bundle**

1. On WINSRV1 open **File Explorer**  
2. Navigate to:

C:\\Users\\Administrator\\Documents

3. Find `googleChromeEnterpriseBundle64.zip`

#### **Step 2 — Extract Bundle**

1. Right-click the zip file  
2. Click **Extract All**  
3. Extract to:

C:\\ChromeBundle

#### **Step 3 — Copy ADMX Files**

1. Open **File Explorer**  
2. Navigate to:

C:\\ChromeBundle\\Configuration\\admx

3. Copy `chrome.admx` and `google.admx`  
4. Paste to:

C:\\Windows\\SYSVOL\\sysvol\\manila.com\\Policies\\PolicyDefinitions\\

5. Also copy the `en-US` folder from admx to:

C:\\Windows\\SYSVOL\\sysvol\\manila.com\\Policies\\PolicyDefinitions\\en-US\\

#### **Step 4 — Create GPO**

1. Open **Group Policy Management**  
2. Right-click **manila.com**  
3. Click **Create a GPO in this domain and Link it here**  
4. Name it:

google

5. Click **OK**

#### **Step 5 — Edit GPO**

1. Right-click **google → Edit**  
2. Navigate to:

User Configuration → Policies → Administrative Templates →  
Google → Google Chrome

3. Double-click **Action on startup**

4. Select **Enabled**

5. Set to **Open a list of URLs**

6. Click **OK**

7. Find **URLs to open on startup**

8. Double-click it

9. Select **Enabled**

10. Click **Show**

11. Add:

http://w3.manila.com

9. Click **OK → OK**

#### **Alternative — Set Homepage Instead**

1. Find **Homepage URL**  
2. Double-click  
3. Select **Enabled**  
4. Enter:

http://w3.manila.com

5. Click **OK**

6. Find **Show Home button on toolbar**

7. Select **Enabled**

8. Click **OK**

#### **Step 6 — Verify**

1. Login as domain user on Client1  
2. Run `gpupdate /force`  
3. Open Chrome  
4. Should open `http://w3.manila.com` automatically

---

## **Part 4 — GPO Recommendations (Table 2\)**

**What:** Recommend 3 additional GPOs to secure the domain

**Why:** Shows understanding of security best practices

### **Recommended GPOs to Fill in Table 2**

#### **GPO 1 — Account Lockout Policy**

| Field | Value |
| ----- | ----- |
| Name | Account Lockout Policy |
| Path | Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Account Lockout Policy |
| Effect | Locks account after 5 failed login attempts for 30 minutes |
| Why chosen | Prevents brute force password attacks. More effective than just password complexity because it stops automated attack tools |

#### **GPO 2 — Disable USB Storage**

| Field | Value |
| ----- | ----- |
| Name | Disable Removable Storage |
| Path | Computer Configuration → Policies → Administrative Templates → System → Removable Storage Access |
| Effect | Prevents users from using USB drives and external storage |
| Why chosen | Prevents data theft and malware introduction via USB. More targeted than disabling all hardware |

#### **GPO 3 — Screen Lock Policy**

| Field | Value |
| ----- | ----- |
| Name | Screen Lock Timeout |
| Path | Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options → Interactive logon: Machine inactivity limit |
| Effect | Locks screen after 10 minutes of inactivity |
| Why chosen | Prevents unauthorized access to unattended workstations. Simpler to enforce via GPO than relying on users to manually lock |

---

## **Part 5 — File Share**

### **Task 7 — Create Pictures Share**

**What:**

* Path: `C:\shares\pictures`  
* Share name: `pictures`  
* Permissions:  
  * Customer Service → Read  
  * Graphics → Modify  
  * IT → Full Control  
  * Everyone else → No access

**Why:** Principle of least privilege — users only get access they need

**Step by Step:**

#### **Create Folder**

1. Open **File Explorer**  
2. Navigate to `C:\`  
3. Create folder `shares`  
4. Inside `shares` create folder `pictures`

#### **Share the Folder**

1. Right-click `pictures` folder  
2. Click **Properties**  
3. Click **Sharing tab**  
4. Click **Advanced Sharing**  
5. Check ✅ **Share this folder**  
6. Set share name:

pictures

7. Click **Permissions**  
8. Remove **Everyone**  
9. Click **Add** and add each group:

**Customer Service:**

Permission: Read ✅

**Graphics:**

Permission: Change ✅ (this is Modify)

**IT:**

Permission: Full Control ✅

10. Click **OK**

#### **Set NTFS Permissions**

1. Click **Security tab**  
2. Click **Advanced**  
3. Click **Disable inheritance**  
4. Select **Convert inherited permissions**  
5. Remove all groups except:  
   * SYSTEM  
   * Administrators  
6. Click **Add** for each group:

**Customer Service:**

Type:  Allow  
Permissions: Read & Execute, List, Read

**Graphics:**

Type:  Allow  
Permissions: Modify, Read & Execute, List, Read, Write

**IT:**

Type:  Allow  
Permissions: Full Control

7. Click **OK → Apply → OK**

---

### **Task 8 — Set Auditing on manila.jpg**

**What:** Log when `manila.jpg` is read by any group member

**Why:** Track who accesses sensitive files — security audit requirement

**Step by Step:**

#### **Step 1 — Enable Audit Policy via GPO**

1. Open **Group Policy Management**  
2. Edit **Default Domain Policy**  
3. Navigate to:

Computer Configuration → Policies → Windows Settings →  
Security Settings → Advanced Audit Policy Configuration →  
Object Access

4. Double-click **Audit File System**  
5. Check ✅ **Configure the following audit events**  
6. Check ✅ **Success**  
7. Check ✅ **Failure**  
8. Click **OK**

#### **Step 2 — Enable Auditing on the File**

1. Navigate to `C:\shares\pictures`  
2. Right-click `manila.jpg`  
3. Click **Properties**  
4. Click **Security tab → Advanced**  
5. Click **Auditing tab**  
6. Click **Add**  
7. Click **Select a principal**  
8. Type `Everyone` → click **OK**  
9. Set:

Type:  All  
Permissions: Read

10. Click **OK → Apply → OK**

#### **Step 3 — Verify Auditing Works**

1. Login as any domain user  
2. Open `manila.jpg`  
3. Open **Event Viewer**  
4. Go to:

Windows Logs → Security

5. Look for **Event ID 4663** — file access logged ✅

---

## **Complete WINSRV1 Checklist**

| Task | What | Status |
| ----- | ----- | ----- |
| Default password policy | 8 chars, 30 days | ⬜ |
| Fine-grained PSO | Executive group 10 chars | ⬜ |
| Login banner title | WorldSkills ASEAN Manila | ⬜ |
| Login banner text | authorized access only | ⬜ |
| control GPO | Block Control Panel for Accounting | ⬜ |
| registry GPO | Block regedit for Manila OU | ⬜ |
| google GPO | Force Chrome homepage | ⬜ |
| GPO recommendations | Fill Table 2 | ⬜ |
| pictures share | Create with correct permissions | ⬜ |
| manila.jpg auditing | Log all reads | ⬜ |

---

Which task would you like to start with first?

# MA2 LINSRV1: Earl

**LINSRV1 Complete Setup Guide**

*WorldSkills ASEAN Manila 2025*

Cyber Security — Skill 54 — Module A2

*Based on Actual Competition Experience*

**Quick Reference**

| Item | Value |
| :---- | :---- |
| Hostname | LINSRV1.manila.com |
| IP Address | 192.168.1.10/24 |
| Gateway | 192.168.1.254 (pfSense DMZ) |
| DNS | 192.168.2.10 (WINSRV1) |
| Domain | manila.com |
| VLAN | DMZ |
| SSH Port | 2022 |
| Website Path | /var/www/manila |
| Root Password | P@ssw0rd |
| Allowed SSH Users | C1 and C2 only |

**⚠** *On competition day, LINSRV1 is pre-built with all packages installed. This guide covers building from scratch for practice purposes.*

# **Part 1 — VM Creation in ESXi**

Create the LINSRV1 virtual machine in ESXi before installing the OS.

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | Go to ESXi → Virtual Machines → Create/Register VM | Start VM creation wizard |
| **2** | Name: LINSRV1, OS: Linux, Version: Rocky Linux 9 (64-bit) | Set VM name and OS type |
| **3** | CPU: 2, RAM: 4GB, Storage: 60GB | Allocate resources per training plan |
| **4** | Network: DMZ port group | Connect to DMZ network — CRITICAL |
| **5** | CD/DVD: Mount Rocky Linux 9.8 DVD ISO | Attach installation media |

**⚠** *Network adapter MUST be set to DMZ port group. Wrong port group \= cannot reach WINSRV1 or pfSense.*

# **Part 2 — Rocky Linux Installation**

Use Rocky Linux 9.8 DVD ISO. Select Server base environment for more packages.

## **Installation Steps**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | Boot from ISO → Install Rocky Linux 9 | Select install option from boot menu |
| **2** | Software Selection → Server | Select Server (not Minimal) for more packages |
| **3** | Network → Set static IP during install | Configure IP before installation completes |
| **4** | Hostname: LINSRV1 | Set hostname during installation |
| **5** | Root password: P@ssw0rd | Match all other VMs in competition |
| **6** | Create user: competitor / P@ssw0rd | Do NOT make administrator |

## **Network Configuration During Install**

In the Network & Hostname screen:

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | IP Address: 192.168.1.10 | Static IP in DMZ subnet |
| **2** | Subnet Mask: 255.255.255.0 | Standard /24 subnet |
| **3** | Gateway: 192.168.1.254 | pfSense DMZ interface |
| **4** | DNS: 192.168.2.10 | WINSRV1 for domain resolution |
| **5** | Search domain: manila.com | For short name resolution |

**⚠** *Directory Client add-on is NOT available in Rocky Linux 9.8 DVD ISO. Domain join packages must be installed manually after OS installation.*

# **Part 3 — Initial Setup After Installation**

## **3.1 Set Hostname**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo hostnamectl set-hostname LINSRV1.manila.com | Set fully qualified domain name as hostname |
| **2** | hostnamectl | Verify hostname was set correctly |

## **3.2 Verify Network**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | ip a | Check IP address is 192.168.1.10 |
| **2** | ping \-c 4 192.168.1.254 | Verify pfSense DMZ gateway is reachable |
| **3** | ping \-c 4 192.168.2.10 | Verify WINSRV1 is reachable |
| **4** | nslookup manila.com 192.168.2.10 | Verify DNS resolution works |

## **3.3 Fix Network if Needed**

If IP settings are wrong after installation, fix with nmcli:

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | nmcli con show | Find your connection name (e.g. ens33, System ens160) |
| **2** | sudo nmcli con mod "ens33" ipv4.addresses 192.168.1.10/24 | Set static IP |
| **3** | sudo nmcli con mod "ens33" ipv4.gateway 192.168.1.254 | Set gateway |
| **4** | sudo nmcli con mod "ens33" ipv4.dns 192.168.2.10 | Set DNS to WINSRV1 |
| **5** | sudo nmcli con mod "ens33" ipv4.dns-search manila.com | Set search domain |
| **6** | sudo nmcli con mod "ens33" ipv4.method manual | Set to static (not DHCP) |
| **7** | sudo nmcli con down "ens33" && sudo nmcli con up "ens33" | Apply changes |

**⚠** *Replace 'ens33' with your actual interface name from nmcli con show.*

# **Part 4 — Install Required Packages**

Rocky Linux 9.8 Minimal/Server does not include all required competition packages. Install them via internet or ISP repository.

## **4.1 Enable Internet Access Temporarily**

Since DMZ has no internet by default, temporarily allow internet via pfSense:

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | Login pfSense → Firewall → NAT → Outbound | Go to NAT outbound settings |
| **2** | Select Automatic Outbound NAT → Save → Apply | Enable NAT for all interfaces including DMZ |
| **3** | Firewall → Rules → OPT1 (DMZ) → Add Pass Any rule | Allow DMZ to reach internet temporarily |

## **4.2 Fix DNS for Package Downloads**

Add Google DNS temporarily for package downloads:

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo vi /etc/resolv.conf | Edit DNS configuration |
| **2** | Add: nameserver 8.8.8.8 | Google DNS for resolving mirror hostnames |
| **3** | ping \-c 4 google.com | Verify internet DNS works before installing |

## **4.3 Install All Required Packages**

Install everything needed for the competition in one command:

sudo dnf install \-y httpd mod\_ssl openssl firewalld policycoreutils-python-utils setroubleshoot-server setools-console realmd sssd sssd-tools oddjob oddjob-mkhomedir adcli samba-common-tools krb5-workstation bind-utils chrony audit rsyslog vim wget curl net-tools tcpdump pam\_pwquality

| Package | Purpose |
| :---- | :---- |
| httpd | Apache web server |
| mod\_ssl | HTTPS/SSL support for Apache |
| openssl | Certificate generation and testing |
| firewalld | Linux local firewall |
| policycoreutils-python-utils | SELinux tools including semanage |
| realmd | Domain discovery and join |
| sssd \+ sssd-tools | System Security Services Daemon for AD auth |
| oddjob \+ oddjob-mkhomedir | Auto-create home directories for domain users |
| adcli | Active Directory command line interface |
| samba-common-tools | Samba/AD support tools including net ads |
| krb5-workstation | Kerberos authentication tools |
| pam\_pwquality | Password complexity enforcement |
| chrony | Time synchronization with domain controller |

**⚠** *If internet is not available, use: adcli join manila.com \-U Administrator (does not require internet). Domain join packages may already be partially installed.*

# **Part 5 — Enable Core Services**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo systemctl enable \--now firewalld | Start and enable firewall at boot |
| **2** | sudo systemctl enable \--now httpd | Start and enable Apache at boot |
| **3** | sudo systemctl enable \--now sshd | Start and enable SSH at boot |
| **4** | sudo systemctl enable \--now chronyd | Start and enable time sync at boot |
| **5** | sudo systemctl enable \--now auditd | Start and enable audit logging |

**✓** *Always verify with: systemctl status httpd \--no-pager and sestatus*

# **Part 6 — Configure SELinux**

The competition requires SELinux enabled AND Apache fully operational. These are not mutually exclusive — just need correct configuration.

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo setenforce 1 | Set SELinux to enforcing mode immediately |
| **2** | sudo sed \-i 's/^SELINUX=.\*/SELINUX=enforcing/' /etc/selinux/config | Make enforcing mode permanent across reboots |
| **3** | sestatus | Verify: Current mode: enforcing, Mode from config: enforcing |
| **4** | sudo semanage port \-a \-t http\_port\_t \-p tcp 443 | Allow Apache on HTTPS port |
| **5** | sudo semanage port \-a \-t ssh\_port\_t \-p tcp 2022 | Allow SSH on port 2022 — REQUIRED or sshd fails |
| **6** | sudo setsebool \-P httpd\_can\_network\_connect on | Allow Apache to make network connections |

**⚠** *semanage port for SSH port 2022 MUST be done BEFORE restarting sshd on port 2022\. Without it: 'Bind to port 2022 failed: Permission denied'*

# **Part 7 — Create Website**

The competition document specifies website content is in /var/www/manila.

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo mkdir \-p /var/www/manila | Create website directory |
| **2** | sudo vi /var/www/manila/index.html | Create website content — add competition HTML here |
| **3** | sudo chown \-R apache:apache /var/www/manila | Set Apache as owner so it can read files |
| **4** | sudo semanage fcontext \-a \-t httpd\_sys\_content\_t "/var/www/manila(/.\*)?" | Set SELinux context for website directory |
| **5** | sudo restorecon \-Rv /var/www/manila | Apply SELinux context to all files in directory |

## **Sample index.html**

\<h1\>WorldSkills ASEAN Manila \- LINSRV1 Website\</h1\>

# **Part 8 — Configure Apache HTTP (Port 80\)**

Create Apache virtual host for HTTP:

sudo vi /etc/httpd/conf.d/manila.conf

Add this content:

\<VirtualHost \*:80\>  
    ServerName w3.manila.com  
    ServerAlias linsrv1.manila.com  
    DocumentRoot /var/www/manila  
    \<Directory /var/www/manila\>  
        AllowOverride None  
        Require all granted  
        Options \-Indexes  
    \</Directory\>  
    ErrorLog /var/log/httpd/manila-error.log  
    CustomLog /var/log/httpd/manila-access.log combined  
\</VirtualHost\>

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo apachectl configtest | ALWAYS test config before restarting — prevents Apache going down |
| **2** | sudo systemctl restart httpd | Apply new configuration |
| **3** | curl http://localhost | Verify HTTP works locally |

# **Part 9 — Configure HTTPS (Port 443\)**

Two approaches: self-signed (quick) or CA-signed from WINSRV3 (preferred).

## **9.1 Create Certificate Directory**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo mkdir \-p /etc/pki/tls/wsc | Create directory for competition SSL certificates |
| **2** | sudo chmod 700 /etc/pki/tls/wsc | Only root can access certificate directory |

## **9.2 Generate CSR with Subject Alternative Names**

Create OpenSSL config file to include both hostnames in one certificate:

sudo vi /tmp/manila-openssl.cnf

Add this content:

\[req\]  
default\_bits \= 2048  
prompt \= no  
distinguished\_name \= dn  
req\_extensions \= req\_ext  
   
\[dn\]  
CN \= linsrv1.manila.com  
C \= PH  
ST \= Metro Manila  
L \= Manila  
O \= WorldSkills  
OU \= Cyber  
   
\[req\_ext\]  
subjectAltName \= @alt\_names  
   
\[alt\_names\]  
DNS.1 \= linsrv1.manila.com  
DNS.2 \= w3.manila.com  
IP.1 \= 192.168.1.10

Generate CSR and private key using the config file — type ALL on ONE LINE:

sudo openssl req \-new \-newkey rsa:2048 \-nodes \-keyout /etc/pki/tls/wsc/manila.key \-out /etc/pki/tls/wsc/manila.csr \-config /tmp/manila-openssl.cnf

**⚠** *Do NOT use backslash line continuation in ESXi console — type entire command on one line or it fails with 'Extra option: keyout'*

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo cat /etc/pki/tls/wsc/manila.csr | View CSR — copy all content including BEGIN/END lines for submission to WINSRV3 |

## **9.3 Submit CSR to WINSRV3 (from WINSRV1 browser)**

Since LINSRV1 has no GUI, use WINSRV1 browser:

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | Open browser on WINSRV1 | WINSRV1 has GUI and can reach WINSRV3 |
| **2** | Navigate to: http://192.168.2.30/certsrv | WINSRV3 Certificate Authority web enrollment |
| **3** | Login: manila\\Administrator / P@ssw0rd | Domain admin credentials |
| **4** | Request certificate → Advanced → Submit base-64 CSR | Paste CSR content from LINSRV1 |
| **5** | Certificate Template: Web Server → Submit | Select correct template for web server cert |
| **6** | Download Base 64 certificate → save as C:\\manila.crt | Download signed certificate |
| **7** | Also download CA certificate → save as C:\\ManilaRootCA.crt | Needed for client trust and LINSRV1 chain verification |

## **9.4 Transfer Certificates to LINSRV1**

Use SCP from WINSRV1 PowerShell. Note: use uppercase \-P for SCP port:

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | scp \-P 2022 C:\\manila.crt C1@192.168.1.10:/tmp/manila.crt | Copy signed cert to LINSRV1 /tmp. Use C1 user since AllowUsers restricts SSH to C1 and C2 |
| **2** | scp \-P 2022 C:\\ManilaRootCA.crt C1@192.168.1.10:/tmp/ManilaRootCA.crt | Copy Root CA cert — needed for chain verification |

**⚠** *SCP uses uppercase \-P for port (unlike SSH which uses lowercase \-p). Also ensure /home/C1 has correct permissions: sudo chown C1:C1 /home/C1*

## **9.5 Install Certificates on LINSRV1**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo cp /tmp/manila.crt /etc/pki/tls/wsc/manila.crt | Move signed cert to SSL directory |
| **2** | sudo cp /tmp/ManilaRootCA.crt /etc/pki/ca-trust/source/anchors/ | Install Root CA into system trust store |
| **3** | sudo update-ca-trust extract | Rebuild CA trust database — run after adding any new CA |
| **4** | sudo trust list | grep \-i manila | Verify Manila Root CA is now trusted |
| **5** | sudo chmod 600 /etc/pki/tls/wsc/manila.key | Private key readable only by root — REQUIRED |
| **6** | sudo chmod 644 /etc/pki/tls/wsc/manila.crt | Certificate readable by Apache |
| **7** | sudo restorecon \-Rv /etc/pki/tls/wsc | Fix SELinux contexts on certificate files |

## **9.6 Create Apache SSL Virtual Host**

sudo vi /etc/httpd/conf.d/manila-ssl.conf

Add this content:

\<VirtualHost \*:443\>  
    ServerName w3.manila.com  
    ServerAlias linsrv1.manila.com  
    DocumentRoot /var/www/manila  
    SSLEngine on  
    SSLCertificateFile /etc/pki/tls/wsc/manila.crt  
    SSLCertificateKeyFile /etc/pki/tls/wsc/manila.key  
    SSLCACertificateFile /etc/pki/ca-trust/source/anchors/ManilaRootCA.crt  
    \<Directory /var/www/manila\>  
        AllowOverride None  
        Require all granted  
        Options \-Indexes  
    \</Directory\>  
    ErrorLog /var/log/httpd/manila-ssl-error.log  
    CustomLog /var/log/httpd/manila-ssl-access.log combined  
\</VirtualHost\>

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo apachectl configtest | Test config — must show 'Syntax OK' before restarting |
| **2** | sudo systemctl restart httpd | Apply SSL configuration |
| **3** | curl \-k https://localhost | Test HTTPS locally (-k skips cert verification) |
| **4** | curl https://linsrv1.manila.com | Test with cert verification — should work without \-k |
| **5** | curl https://w3.manila.com | Test w3 hostname — should work if SAN is included in cert |

**✓** *curl https://linsrv1.manila.com succeeding WITHOUT \-k flag means the CA certificate is properly trusted\!*

# **Part 10 — Configure Firewall (firewalld)**

Open only required ports. The competition document specifies DMZ should allow HTTP, HTTPS, DNS, and SSH.

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo firewall-cmd \--set-default-zone=dmz | Set DMZ as default zone — appropriate for server in DMZ network |
| **2** | sudo firewall-cmd \--permanent \--zone=dmz \--add-service=http | Allow HTTP (port 80\) permanently |
| **3** | sudo firewall-cmd \--permanent \--zone=dmz \--add-service=https | Allow HTTPS (port 443\) permanently |
| **4** | sudo firewall-cmd \--permanent \--zone=dmz \--add-service=dns | Allow DNS (port 53\) — for public DNS entries |
| **5** | sudo firewall-cmd \--permanent \--zone=dmz \--add-port=2022/tcp | Allow SSH on custom port 2022 permanently |
| **6** | sudo firewall-cmd \--reload | Apply all permanent rules without restart |
| **7** | sudo firewall-cmd \--list-all | Verify all rules are applied correctly |

**✓** *\--permanent flag makes rules survive reboot. Without it, rules are lost on restart.*

# **Part 11 — Configure SSH (Port 2022\)**

The competition requires SSH on port 2022, root login disabled, and only C1/C2 allowed.

## **11.1 SELinux Must Be Done First**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo semanage port \-a \-t ssh\_port\_t \-p tcp 2022 | Tell SELinux port 2022 is allowed for SSH — MUST do before restarting sshd |
| **2** | sudo semanage port \-m \-t ssh\_port\_t \-p tcp 2022 | Use \-m instead of \-a if port already exists (modify) |

**⚠** *Without semanage port, sshd will fail with: 'Bind to port 2022 on 0.0.0.0 failed: Permission denied'*

## **11.2 Edit SSH Configuration**

sudo vi /etc/ssh/sshd\_config

Set or add these lines:

Port 2022  
PermitRootLogin no  
PasswordAuthentication yes  
AllowUsers C1 C2 c1 c2 competitor

**⚠** *Keep 'competitor' in AllowUsers until C1/C2 SSH is tested and confirmed working. Remove it after.*

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo systemctl restart sshd | Apply SSH config changes |
| **2** | ss \-tulpn | grep ssh | Verify SSH is listening on port 2022 |

## **11.3 After C1/C2 Testing — Remove competitor**

sudo vi /etc/ssh/sshd\_config

Change AllowUsers to:

AllowUsers C1 C2 c1 c2

sudo systemctl restart sshd

# **Part 12 — Create Local C1 and C2 Users**

Create local C1 and C2 accounts as fallback. The competition document explicitly allows this if domain join fails.

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo useradd C1 | Create local user C1 |
| **2** | sudo useradd C2 | Create local user C2 |
| **3** | sudo passwd C1 | Set C1 password — enter P@ssw0rd |
| **4** | sudo passwd C2 | Set C2 password — enter P@ssw0rd |
| **5** | echo "C1 ALL=(ALL) ALL" | sudo tee /etc/sudoers.d/wsc-c1 | Give C1 full sudo rights |
| **6** | echo "C2 ALL=(ALL) ALL" | sudo tee /etc/sudoers.d/wsc-c2 | Give C2 full sudo rights |
| **7** | sudo chmod 440 /etc/sudoers.d/wsc-c1 /etc/sudoers.d/wsc-c2 | Set correct permissions on sudoers files |
| **8** | sudo visudo \-c | Validate sudoers syntax — should return 'parsed OK' |

## **Verify C1 and C2 sudo**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | su \- C1 | Switch to C1 user |
| **2** | sudo whoami | Should return: root |
| **3** | exit | Return to previous user |

# **Part 13 — Configure Password Policy**

The competition requires: 10 characters, monthly change, 25 day minimum, 25 day warning, complexity (upper+lower+number+symbol).

## **13.1 Password Complexity**

sudo vi /etc/security/pwquality.conf

Set these values:

minlen \= 10  
ucredit \= \-1  
lcredit \= \-1  
dcredit \= \-1  
ocredit \= \-1  
retry \= 3

| Setting | Value | Meaning |
| :---- | :---- | :---- |
| minlen | 10 | Minimum 10 characters |
| ucredit \= \-1 | At least 1 | Require uppercase letter |
| lcredit \= \-1 | At least 1 | Require lowercase letter |
| dcredit \= \-1 | At least 1 | Require digit/number |
| ocredit \= \-1 | At least 1 | Require symbol/special character |

## **13.2 Password Aging**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo chage \-M 30 \-m 25 \-W 25 C1 | Max age 30 days, min age 25 days, warn 25 days before expiry |
| **2** | sudo chage \-M 30 \-m 25 \-W 25 C2 | Same for C2 |
| **3** | sudo chage \-M 30 \-m 25 \-W 25 competitor | Same for competitor |
| **4** | chage \-l C1 | Verify password aging settings for C1 |

# **Part 14 — Domain Join to manila.com**

The competition requires LINSRV1 joined to manila.com. Follow this exact order based on what worked.

## **14.1 Prerequisites**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo timedatectl set-timezone Asia/Manila | Set correct timezone — critical for Kerberos |
| **2** | sudo chronyc makestep | Force time sync with domain controller immediately |
| **3** | timedatectl | Verify time matches WINSRV1 — must be within 5 minutes |
| **4** | nslookup manila.com 192.168.2.10 | Verify DNS resolves domain before joining |
| **5** | nslookup \_ldap.\_tcp.manila.com 192.168.2.10 | Verify LDAP SRV records exist |

**⚠** *Time sync is CRITICAL. Kerberos authentication fails if time difference between LINSRV1 and WINSRV1 is more than 5 minutes.*

## **14.2 Configure Kerberos**

sudo vi /etc/krb5.conf

Replace entire content with:

\[libdefaults\]  
    default\_realm \= MANILA.COM  
    dns\_lookup\_realm \= false  
    dns\_lookup\_kdc \= true  
    ticket\_lifetime \= 24h  
    renew\_lifetime \= 7d  
    forwardable \= true  
    rdns \= false  
   
\[realms\]  
    MANILA.COM \= {  
        kdc \= 192.168.2.10  
        admin\_server \= 192.168.2.10  
        default\_domain \= manila.com  
    }  
   
\[domain\_realm\]  
    .manila.com \= MANILA.COM  
    manila.com \= MANILA.COM

## **14.3 Join Domain**

Use adcli — does NOT require internet or repo access:

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | touch /etc/krb5.keytab && chmod 600 /etc/krb5.keytab | Create empty keytab file with correct permissions before joining |
| **2** | adcli join manila.com \-U Administrator \-v | Join domain using adcli with verbose output. No output \= SUCCESS |

**✓** *Success looks like blank output OR 'Added keytab entries' messages. No error \= joined\!*

## **14.4 Verify Domain Join**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | hostname \-d | Should return: manila.com |
| **2** | hostname \-f | Should return: LINSRV1.manila.com |
| **3** | klist \-k /etc/krb5.keytab | Should show keytab entries for LINSRV1 |

Verify on WINSRV1:

nslookup LINSRV1

Should return: 192.168.1.10

## **14.5 Configure SSSD**

sudo vi /etc/sssd/sssd.conf

Add this content:

\[sssd\]  
config\_file\_version \= 2  
domains \= manila.com  
services \= nss, pam  
   
\[domain/manila.com\]  
id\_provider \= ad  
auth\_provider \= ad  
access\_provider \= ad  
chpass\_provider \= ad  
ad\_domain \= manila.com  
ad\_server \= winsrv1.manila.com  
ad\_hostname \= linsrv1.manila.com  
krb5\_realm \= MANILA.COM  
cache\_credentials \= true  
use\_fully\_qualified\_names \= False  
fallback\_homedir \= /home/%u  
default\_shell \= /bin/bash  
ldap\_id\_mapping \= true

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | sudo chmod 600 /etc/sssd/sssd.conf | SSSD requires 600 permissions — fails to start otherwise |
| **2** | sudo chown root:root /etc/sssd/sssd.conf | Must be owned by root |
| **3** | sudo authselect select sssd with-mkhomedir \--force | Configure PAM to use SSSD for authentication |
| **4** | sudo systemctl enable \--now oddjobd | Enable auto home directory creation for domain users |
| **5** | sudo systemctl restart sssd | Apply SSSD configuration |
| **6** | sudo systemctl enable sssd | Start SSSD at boot |
| **7** | id C1 | Verify C1 domain user is found — should show uid and groups |
| **8** | id C2 | Verify C2 domain user is found |

# **Part 15 — Give Domain C1/C2 Sudo Rights**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | echo "C1 ALL=(ALL) ALL" | sudo tee /etc/sudoers.d/domain-c1 | Give domain C1 full sudo rights |
| **2** | echo "C2 ALL=(ALL) ALL" | sudo tee /etc/sudoers.d/domain-c2 | Give domain C2 full sudo rights |
| **3** | sudo chmod 440 /etc/sudoers.d/domain-c1 /etc/sudoers.d/domain-c2 | Set correct permissions |
| **4** | sudo visudo \-c | Validate sudoers — must show 'parsed OK' |

# **Part 16 — Fix Certificate Warning on Client Computers**

After installing CA-signed cert on LINSRV1, clients still get warnings because they don't trust Manila Root CA. Fix via GPO.

## **On WINSRV1 — Import Root CA via certenroll GPO**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | Open Group Policy Management on WINSRV1 | Server Manager → Tools → Group Policy Management |
| **2** | Find certenroll GPO → Right-click → Edit | Edit the existing certenroll GPO |
| **3** | Computer Config → Policies → Windows Settings → Security Settings → Public Key Policies → Trusted Root CAs | Navigate to trusted root certificates |
| **4** | Right-click → Import → Browse to C:\\ManilaRootCA.crt | Import the Root CA certificate |
| **5** | Place in Trusted Root Certification Authorities → Finish | Complete import wizard |

## **On Each Client**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | gpupdate /force | Force Group Policy update to receive Root CA certificate |
| **2** | Open browser → https://linsrv1.manila.com | Should show website WITHOUT certificate warning |
| **3** | Open browser → https://w3.manila.com | Should also work without warning if SAN is included |

**✓** *If browser still shows warning after gpupdate, restart the browser completely — cached security decisions may persist.*

# **Part 17 — Troubleshooting Reference**

| Error | Cause | Fix |
| :---- | :---- | :---- |
| Bind to port 2022 failed: Permission denied | SELinux blocking SSH on port 2022 | sudo semanage port \-a \-t ssh\_port\_t \-p tcp 2022 |
| Extra option: keyout | Missing dash before \-keyout in openssl command | Type full command on ONE line with all dashes |
| SSL: certificate subject name mismatch | Cert only has one hostname, not both | Regenerate CSR with SAN config file |
| SSL: unable to get local issuer certificate | Root CA not in trust store | sudo update-ca-trust extract |
| SSSD fails to start | sssd.conf wrong permissions | sudo chmod 600 /etc/sssd/sssd.conf |
| adcli: Server not found in Kerberos database | DNS not resolving or time mismatch | Check resolv.conf and run chronyc makestep |
| adcli: Couldn't add keytab entries | keytab file missing | touch /etc/krb5.keytab && chmod 600 /etc/krb5.keytab |
| realm join fails with repo error | Tries to install packages from internet | Use adcli join instead — no internet needed |
| Apache fails to start after config change | Syntax error in config file | Always run apachectl configtest first |
| SELinux blocking Apache website | Wrong SELinux context on files | sudo restorecon \-Rv /var/www/manila |
| SCP Permission denied | competitor not in AllowUsers or wrong home dir perms | Add competitor to AllowUsers temporarily or use C1 user |

# **Final Validation Checklist**

Run these commands on LINSRV1 before signaling completion:

## **System Checks**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | hostnamectl | Should show: LINSRV1.manila.com |
| **2** | ip a | Should show: 192.168.1.10/24 |
| **3** | sestatus | Should show: Current mode: enforcing |
| **4** | firewall-cmd \--list-all | Should show: http, https, dns, 2022/tcp |
| **5** | ss \-tulpn | egrep '2022|80|443' | Should show SSH on 2022, Apache on 80 and 443 |
| **6** | systemctl status httpd \--no-pager | Should show: active (running) |
| **7** | systemctl status sshd \--no-pager | Should show: active (running) |
| **8** | realm list | Should show manila.com as configured domain |
| **9** | id C1 && id C2 | Should return user information for both C1 and C2 |
| **10** | chage \-l C1 | Should show correct password aging settings |

## **From Client1/Client2**

| \# | Command | Explanation |
| :---- | :---- | :---- |
| **1** | ping 192.168.1.10 | LINSRV1 is reachable from LAN |
| **2** | nslookup linsrv1.manila.com | DNS resolves to 192.168.1.10 |
| **3** | nslookup w3.manila.com | DNS resolves to 192.168.1.10 |
| **4** | curl http://192.168.1.10 | HTTP website loads |
| **5** | curl https://linsrv1.manila.com | HTTPS works without certificate error |
| **6** | ssh C1@192.168.1.10 \-p 2022 | C1 can SSH in |
| **7** | ssh C2@192.168.1.10 \-p 2022 | C2 can SSH in |
| **8** | ssh root@192.168.1.10 \-p 2022 | Root SSH should FAIL |

## **Expected Final State**

| Test | Expected Result |
| :---- | :---- |
| Ping LINSRV1 | Pass |
| DNS linsrv1.manila.com | Pass — 192.168.1.10 |
| DNS w3.manila.com | Pass — 192.168.1.10 |
| HTTP website | Pass — shows content |
| HTTPS website | Pass — no certificate warning |
| SSH port 2022 | Pass |
| Root SSH login | Fail — permission denied |
| C1 SSH login | Pass |
| C2 SSH login | Pass |
| C1/C2 sudo whoami | Pass — returns root |
| Other users SSH | Fail — permission denied |
| firewalld running | Pass |
| SELinux enforcing | Pass |
| Apache \+ SELinux | Pass — both work together |
| Domain join | Pass — realm list shows manila.com |

*WorldSkills ASEAN Manila 2025 — Cyber Security Skill 54 — Based on actual competition practice experience*

# LINSRV1: 2026

# **LinuxWeb Setup Guide — WSPH2026 Alignment**

*Adapted from the WSA2025 LINSRV1 guide to match Test Project A2's DMZ web VM (Table 1: LinuxWeb, 192.168.1.10, DMZ). Covers the **Linux server hardening** section. HTTPS certificate issuance is not repeated here — see `PKI_Setup_Guide_Single_Tier_WSPH2026.md`.*

## **Quick Reference**

| Item | Value |
| ----- | ----- |
| Hostname | LinuxWeb.\<DOMAIN\> |
| IP Address | 192.168.1.10/24 |
| Gateway | 192.168.1.254 (pfSense DMZ) |
| DNS | 192.168.2.10 (WinSRV1) |
| VLAN | DMZ |
| SSH Port | \<SSH\_PORT\> — non-default, per Competition Day Instructions |
| Website Path | \<WEB\_ROOT\> — per Competition Day Instructions |
| Allowed SSH Users | \<SSH\_USER1\>, \<SSH\_USER2\> — per credentials provided on day |

---

## **What changed from the WSA2025 version, and why**

| WSA2025 part | Status | Reason |
| ----- | ----- | ----- |
| Parts 1–5: create the VM in ESXi, install Rocky Linux, temporarily open DMZ→Internet NAT to `dnf install` packages | **Dropped entirely** | Test Project: *"The virtual machines are pre-provisioned. Competitors must verify addressing and connectivity"* — and separately: *"External internet access is not available unless explicitly permitted by the Organizer."* Don't build from scratch and don't try to reach the internet — start from verification |
| Part 9 (CSR generation, submit to WINSRV3 via WINSRV1 browser, transfer back) | **See PKI\_Setup\_Guide\_Single\_Tier\_WSPH2026.md** | This year's CA is single-tier on WinSRV1, and Table 2's firewall rules mean the CSR has to relay through Client1 (LAN), not WinSRV1 directly — different path, not just a renamed one |
| Cert SAN covering two hostnames (`linsrv1`/`w3`) | **One hostname** (`linuxweb.<DOMAIN>`) | One DMZ web VM this year, not two site names on one box |
| Part 14 — domain join (Kerberos, SSSD, `adcli join`) | **Moved to optional, Part 10 below** | Test Project's Linux hardening section only lists SSH hardening, SELinux, and HTTPS — no domain join requirement anywhere in the text. Don't assume it's needed; confirm on the day |
| Part 15 — domain C1/C2 sudo rights | **Folded into optional Part 10** | Depends entirely on whether domain join turns out to be required |
| Part 13 — Linux password aging/complexity policy | **Moved to optional, Part 11 below** | The password-policy requirement in Test Project sits under the **AD/GPO** section for the Windows domain — nothing states a separate Linux-local policy is graded |
| `firewalld --add-service=dns` | **Dropped** | LinuxWeb is a web server, not a DNS server — no inbound DNS service needed here |
| SSH port 2022, root password `P@ssw0rd`, path `/var/www/manila` | **Placeholders** | WSA2025-specific values — don't assume they repeat; use whatever Competition Day Instructions give you |

---

# **Part 1 — Verify Initial Configuration**

Since the VM is pre-provisioned, this is a check, not a build step.

| \# | Command | Explanation |
| ----- | ----- | ----- |
| **1** | `hostnamectl` | Confirm hostname matches what's expected |
| **2** | `ip a` | Confirm IP is 192.168.1.10/24 |
| **3** | `ping -c 4 192.168.1.254` | Confirm pfSense DMZ gateway is reachable |
| **4** | `ping -c 4 192.168.2.10` | Confirm WinSRV1 is reachable |
| **5** | `nslookup <DOMAIN> 192.168.2.10` | Confirm DNS resolution works |

**If something's wrong**, fix with `nmcli` (same commands as before — `nmcli con mod "<iface>" ipv4.addresses/gateway/dns ...`) rather than reinstalling. Don't spend module time rebuilding a pre-provisioned box.

---

# **Part 2 — Confirm Pre-Installed Services and Packages**

| \# | Command | Explanation |
| ----- | ----- | ----- |
| **1** | `systemctl status httpd --no-pager` | Apache should already be installed and running |
| **2** | `systemctl status sshd --no-pager` | SSH should be running |
| **3** | `systemctl status firewalld --no-pager` | Local firewall should be running |
| **4** | `rpm -q httpd mod_ssl openssl policycoreutils-python-utils` | Confirm the packages you'll need for hardening are present |
| **5** | `sestatus` | Note current SELinux mode before you change anything |

If something required is genuinely missing and you can't reach it locally, that's a question for the Organizer — don't assume you can `dnf install` your way around it, since outbound internet isn't available by default this year.

---

# **Part 3 — Confirm Website Content**

| \# | Command | Explanation |
| ----- | ----- | ----- |
| **1** | `curl http://localhost` | Confirm the existing site serves something before you touch anything |
| **2** | `cat /etc/httpd/conf.d/*.conf` | Find the actual DocumentRoot in use — don't assume `/var/www/manila` |

Treat whatever's already configured as the baseline; only change content/paths if Competition Day Instructions explicitly ask you to.

---

# **Part 4 — SELinux: Enforcing Mode**

| \# | Command | Explanation |
| ----- | ----- | ----- |
| **1** | `sudo setenforce 1` | Enforcing mode immediately |
| **2** | `sudo sed -i 's/^SELINUX=.*/SELINUX=enforcing/' /etc/selinux/config` | Persist across reboot |
| **3** | `sestatus` | Confirm: Current mode: enforcing, Mode from config: enforcing |
| **4** | `sudo semanage port -a -t http_port_t -p tcp 443` | Allow Apache on 443 (skip if already present — use `-m` instead of `-a` to modify) |
| **5** | `sudo setsebool -P httpd_can_network_connect on` | Only needed if Apache proxies or calls out elsewhere |
| **6** | `sudo semanage fcontext -a -t httpd_sys_content_t "<WEB_ROOT>(/.*)?"` | Match SELinux context to wherever the actual DocumentRoot is |
| **7** | `sudo restorecon -Rv <WEB_ROOT>` | Apply it |

**✓ Test immediately:** `curl http://localhost` — if this breaks the moment you flip enforcing mode, it's almost always a context problem — `restorecon -Rv` the actual content path and cert path (see Part 9\) again.

---

# **Part 5 — Apache HTTP (Port 80\)**

Confirm the existing vhost is sane, or create one if it's not there:

sudo vi /etc/httpd/conf.d/site.conf

\<VirtualHost \*:80\>  
 ServerName linuxweb.\<DOMAIN\>  
 DocumentRoot \<WEB\_ROOT\>  
 \<Directory \<WEB\_ROOT\>\>  
 AllowOverride None  
 Require all granted  
 Options \-Indexes  
 \</Directory\>  
 ErrorLog /var/log/httpd/site-error.log  
 CustomLog /var/log/httpd/site-access.log combined  
 \</VirtualHost\>

| \# | Command | Explanation |
| ----- | ----- | ----- |
| **1** | `sudo apachectl configtest` | Always test before restarting |
| **2** | `sudo systemctl restart httpd` | Apply |
| **3** | `curl http://localhost` | Verify |

---

# **Part 6 — SSH Hardening**

| \# | Command | Explanation |
| ----- | ----- | ----- |
| **1** | `sudo semanage port -a -t ssh_port_t -p tcp <SSH_PORT>` | Do this **before** changing sshd's port — SELinux will block the bind otherwise |

sudo vi /etc/ssh/sshd\_config

Port \<SSH\_PORT\>  
 PermitRootLogin no  
 PasswordAuthentication yes  
 AllowUsers \<SSH\_USER1\> \<SSH\_USER2\>

**⚠** *Keep your own admin/fallback account in `AllowUsers` until you've confirmed the authorised accounts can actually log in — remove it after, same caution as before.*

| \# | Command | Explanation |
| ----- | ----- | ----- |
| **2** | `sudo systemctl restart sshd` | Apply |
| **3** | `ss -tulpn | grep ssh` | Confirm listening on `<SSH_PORT>` |
| **4** | `ssh root@192.168.1.10 -p <SSH_PORT>` from another box | Should **fail** |
| **5** | `ssh <SSH_USER1>@192.168.1.10 -p <SSH_PORT>` | Should succeed |

---

# **Part 7 — Local Authorised Accounts**

Unless domain join is confirmed as required (Part 10), authorised local accounts are the straightforward way to satisfy "restrict access to authorised accounts only":

| \# | Command | Explanation |
| ----- | ----- | ----- |
| **1** | `sudo useradd <SSH_USER1>` | Create local account |
| **2** | `sudo useradd <SSH_USER2>` | Create local account |
| **3** | `sudo passwd <SSH_USER1>` | Set password per Competition Day Instructions |
| **4** | `sudo passwd <SSH_USER2>` | Same |
| **5** | `echo "<SSH_USER1> ALL=(ALL) ALL" | sudo tee /etc/sudoers.d/user1` | Only if sudo rights are actually required by the task |
| **6** | `sudo chmod 440 /etc/sudoers.d/user1 /etc/sudoers.d/user2` | Correct permissions |
| **7** | `sudo visudo -c` | Must return "parsed OK" |

---

# **Part 8 — Local Firewall (firewalld)**

| \# | Command | Explanation |
| ----- | ----- | ----- |
| **1** | `sudo firewall-cmd --permanent --zone=dmz --add-service=http` | Allow 80 |
| **2** | `sudo firewall-cmd --permanent --zone=dmz --add-service=https` | Allow 443 |
| **3** | `sudo firewall-cmd --permanent --zone=dmz --add-port=<SSH_PORT>/tcp` | Allow your SSH port |
| **4** | `sudo firewall-cmd --reload` | Apply |
| **5** | `sudo firewall-cmd --list-all` | Verify — should show http, https, `<SSH_PORT>/tcp` only |

---

# **Part 9 — HTTPS / Certificate**

Full procedure is in `PKI_Setup_Guide_Single_Tier_WSPH2026.md` (CSR generation, relay through Client1, signing on WinSRV1, install back on LinuxWeb). Once you have the cert files in place, the Apache side looks like this:

sudo vi /etc/httpd/conf.d/site-ssl.conf

\<VirtualHost \*:443\>  
 ServerName linuxweb.\<DOMAIN\>  
 DocumentRoot \<WEB\_ROOT\>  
 SSLEngine on  
 SSLCertificateFile /etc/pki/tls/certs/linuxweb.pem  
 SSLCertificateKeyFile /etc/pki/tls/private/linuxweb.key  
 SSLCACertificateFile /etc/pki/tls/certs/ca-root.cer  
 \<Directory \<WEB\_ROOT\>\>  
 AllowOverride None  
 Require all granted  
 Options \-Indexes  
 \</Directory\>  
 ErrorLog /var/log/httpd/site-ssl-error.log  
 CustomLog /var/log/httpd/site-ssl-access.log combined  
 \</VirtualHost\>

| \# | Command | Explanation |
| ----- | ----- | ----- |
| **1** | `sudo apachectl configtest` | Must show Syntax OK |
| **2** | `sudo systemctl restart httpd` | Apply |
| **3** | `curl -k https://localhost` | Local test, skipping verification |
| **4** | `curl https://linuxweb.<DOMAIN>` | Should succeed **without** `-k` once the CA is trusted client-side |

*(File paths above match the PKI guide's Part 4 exactly — keep them in sync if you change one.)*

---

# **Part 10 — Domain Join (Optional — Confirm Before Doing This)**

Not called for anywhere in this year's Test Project text. Only do this if Competition Day Instructions explicitly say LinuxWeb must be domain-joined. If so, the WSA2025 procedure (Kerberos config, `adcli join`, SSSD) still applies — the mechanics don't change, just the domain name and hostnames:

* Set timezone, `chronyc makestep`, confirm time is within 5 minutes of WinSRV1 (Kerberos is time-sensitive)  
* `/etc/krb5.conf` — realm \= `<DOMAIN>` uppercase, kdc/admin\_server \= 192.168.2.10  
* `touch /etc/krb5.keytab && chmod 600 /etc/krb5.keytab`, then `adcli join <DOMAIN> -U Administrator -v`  
* `/etc/sssd/sssd.conf` (mode 600, root:root) pointing `ad_server`/`ad_hostname` at this year's actual names, then `authselect select sssd with-mkhomedir --force` and restart sssd  
* Verify with `realm list`, `id <user>` for each domain account expected to log in

If this isn't confirmed as required, skip it and rely on Part 7's local accounts — don't spend module time on Kerberos troubleshooting for a requirement that may not exist this year.

---

# **Part 11 — Linux Password Policy (Optional — Confirm Before Doing This)**

Not stated in Test Project's Linux hardening section — the password baseline requirement is written under **Identity and Policy (AD/GPO)**, i.e. the Windows domain. If Competition Day Instructions extend it to local Linux accounts too:

sudo vi /etc/security/pwquality.conf minlen \= \<N\>  
 ucredit \= \-1  
 lcredit \= \-1  
 dcredit \= \-1  
 ocredit \= \-1

sudo chage \-M \<max\> \-m \<min\> \-W \<warn\> \<SSH\_USER1\>

---

# **Part 12 — Troubleshooting**

| Error | Cause | Fix |
| ----- | ----- | ----- |
| `Bind to port <SSH_PORT> failed: Permission denied` | SELinux blocking that port for SSH | `sudo semanage port -a -t ssh_port_t -p tcp <SSH_PORT>` — must be done **before** restarting sshd |
| `SSL: certificate subject name mismatch` | ServerName doesn't match the cert's CN/SAN | Regenerate CSR with the right CN, or fix ServerName |
| `SSL: unable to get local issuer certificate` | CA cert not in the system trust store | `sudo cp <ca-root.cer> /etc/pki/ca-trust/source/anchors/ && sudo update-ca-trust extract` |
| Apache won't start after a config change | Syntax error | `apachectl configtest` first, always |
| SELinux blocking Apache from serving content or reading the cert | Wrong context on files | `sudo restorecon -Rv <WEB_ROOT>` and `sudo restorecon -Rv /etc/pki/tls/` |
| SCP permission denied | User not in `AllowUsers`, or home dir perms wrong | Check `sshd_config` and `chown` the home directory |

---

# **Part 13 — Final Validation Checklist**

| \# | Check | Expected |
| ----- | ----- | ----- |
| 1 | `hostnamectl` | Correct hostname |
| 2 | `ip a` | 192.168.1.10/24 |
| 3 | `sestatus` | Current mode: enforcing |
| 4 | `firewall-cmd --list-all` | http, https, `<SSH_PORT>/tcp` only |
| 5 | `ss -tulpn` | SSH on `<SSH_PORT>`, Apache on 80 and 443 |
| 6 | `systemctl status httpd` | active (running) |
| 7 | `systemctl status sshd` | active (running) |
| 8 | `ssh root@192.168.1.10 -p <SSH_PORT>` | **Fails** |
| 9 | `ssh <SSH_USER1>@192.168.1.10 -p <SSH_PORT>` | Succeeds |
| 10 | From Client1: `curl http://192.168.1.10` | HTTP loads |
| 11 | From Client1: `curl https://linuxweb.<DOMAIN>` | HTTPS works, no cert error |
| 12 | Re-run 6–11 **after** SELinux enforcing is on | Everything above still passes — this is the check that actually matters, not doing each step once in isolation |

# MA2 WINSRV1: Earl

**WINSRV1 Complete Setup Guide**

*GUI Version — WorldSkills ASEAN Manila 2025*

Cyber Security Skill 54 — Module A2 — Based on WSA2025\_TP54 Competition Document

**Quick Reference**

| Item | Value |
| :---- | :---- |
| Hostname | WINSRV1.manila.com |
| IP Address | 192.168.2.10/24 |
| Subnet Mask | 255.255.255.0 |
| Gateway | 192.168.2.254 (pfSense SERVERS interface) |
| DNS | 192.168.2.10 (itself) |
| VLAN | SERVERS |
| Domain | manila.com |
| Administrator Password | P@ssw0rd |
| Roles | AD DS, DNS, DHCP, File Server |

**📝 Note:** *On competition day WINSRV1 is pre-configured with AD DS, DNS, and DHCP. This guide covers completing the competition tasks.*

# **Part 1 — Verify Initial Configuration**

Before starting any tasks, verify WINSRV1 is correctly configured. Fixing network or domain issues first saves time later.

## **Step 1: Verify IP Address**

**Step 1:** Open Control Panel → Network and Sharing Center

**Step 2:** Click Change adapter settings

**Step 3:** Right-click your network adapter → Properties

**Step 4:** Double-click Internet Protocol Version 4 (TCP/IPv4)

**Step 5:** Verify these settings:

IP Address:   192.168.2.10

Subnet Mask:  255.255.255.0

Gateway:      192.168.2.254

DNS Server:   192.168.2.10

**Step 6:** Click OK → Close

**⚠ Warning:** *DNS must point to itself (192.168.2.10). If it points elsewhere, domain resolution will fail.*

## **Step 2: Verify Hostname and Domain**

**Step 1:** Right-click Start → System

**Step 2:** Verify Computer name shows WINSRV1

**Step 3:** Verify Domain shows manila.com

**Step 4:** Open PowerShell and run:

ipconfig /all

Verify output shows:

Host Name . . . . : WINSRV1

Primary Dns Suffix: manila.com

IPv4 Address. . . : 192.168.2.10

## **Step 3: Verify AD DS, DNS, DHCP Roles**

**Step 1:** Open Server Manager

**Step 2:** Check left panel shows:

* AD DS

* DNS

* DHCP

* File and Storage Services

**Step 3:** All should show green status

**✓ Tip:** *If any role shows red warning, click it to investigate before proceeding with other tasks.*

# **Part 2 — Default Domain Password Policy**

The competition requires all domain users to have a minimum 8-character password changed monthly.

**Step 1:** Open Server Manager → Tools → Group Policy Management

**Step 2:** In the left panel expand:

Forest: manila.com

  └─ Domains

      └─ manila.com

          └─ Group Policy Objects

**Step 3:** Right-click Default Domain Policy → Edit

**📝 Note:** *The Default Domain Policy applies to all users and computers in the domain.*

**Step 4:** In the GPO Editor navigate to:

Computer Configuration

  └─ Policies

      └─ Windows Settings

          └─ Security Settings

              └─ Account Policies

                  └─ Password Policy

**Step 5:** Configure Minimum Password Length:

* Double-click Minimum password length

* Check ✅ Define this policy setting

* Set value to: 8

* Click OK

**Step 6:** Configure Maximum Password Age (monthly):

* Double-click Maximum password age

* Check ✅ Define this policy setting

* Set value to: 30

* Click OK

**Step 7:** Enable Password Complexity:

* Double-click Password must meet complexity requirements

* Select Enabled

* Click OK

**Step 8:** Close GPO Editor

**Step 9:** Force policy update — open Command Prompt on WINSRV1:

gpupdate /force

**✓ Tip:** *Password complexity requires: uppercase, lowercase, digit, and special character by default in Windows.*

# **Part 3 — Fine-Grained Password Policy for Executive Group**

The competition requires members of the Executive group to have a minimum 10-character password. This overrides the default 8-character policy.

**Step 1:** Open Server Manager → Tools → Active Directory Administrative Center

**Step 2:** In the left panel click manila (local)

**Step 3:** In the main area double-click System folder

**Step 4:** Double-click Password Settings Container

**Step 5:** In the right panel under Tasks click New → Password Settings

**Step 6:** Fill in the Create Password Settings form:

Name:                    ExecutivePSO

Precedence:              1

Minimum password length: 10

Maximum password age:    30 days

Minimum password age:    1 day

Password history count:  24

Enforce password complexity: Enabled (checked)

**Step 7:** Under Directly Applies To section click Add

**Step 8:** In the search box type Executive → click Search

**Step 9:** Select the Executive group → click OK

**Step 10:** Click OK to save the Password Settings Object

## **Verify Fine-Grained Policy Applied**

**Step 1:** Open Active Directory Users and Computers

**Step 2:** Find a user who is a member of the Executive group

**Step 3:** Right-click the user → Properties

**Step 4:** Click Attribute Editor tab

**Step 5:** Look for msDS-ResultantPSO attribute

**Step 6:** Value should show: ExecutivePSO

**⚠ Warning:** *If Attribute Editor tab is not visible: View menu → Advanced Features*

# **Part 4 — Login Banner**

The competition requires a login banner with title 'WorldSkills ASEAN Manila' and text 'authorized access only' displayed before users log in.

## **Step 1: Open Group Policy Management**

**Step 1:** Open Server Manager

**Step 2:** Click Tools in the top menu

**Step 3:** Select Group Policy Management

## **Step 2: Create a New GPO**

**Step 1:** In the left panel expand:

Forest: manila.com

  └─ Domains

      └─ manila.com

**Step 2:** Right-click manila.com → Create a GPO in this domain and Link it here

**Step 3:** In the New GPO dialog type:

Login Banner

**Step 4:** Click OK

## **Step 3: Edit the GPO**

**Step 1:** In the left panel find the Login Banner GPO under manila.com

**Step 2:** Right-click Login Banner → Edit

**Step 3:** In the GPO Editor navigate to:

Computer Configuration

  └─ Policies

      └─ Windows Settings

          └─ Security Settings

              └─ Local Policies

                  └─ Security Options

## **Step 4: Configure the Message Title**

**Step 1:** In the right pane scroll down and find:

Interactive logon: Message title for users attempting to log on

**Step 2:** Double-click it

**Step 3:** Check ✅ Define this policy setting

**Step 4:** In the text field enter exactly:

WorldSkills ASEAN Manila

**Step 5:** Click OK

## **Step 5: Configure the Message Text**

**Step 1:** In the same Security Options pane find:

Interactive logon: Message text for users attempting to log on

**Step 2:** Double-click it

**Step 3:** Check ✅ Define this policy setting

**Step 4:** In the text field enter exactly:

authorized access only

**Step 5:** Click OK

## **Step 6: Close and Apply**

**Step 1:** Close the GPO Editor

**Step 2:** The GPO is already linked to manila.com from Step 2

**Step 3:** Force policy update on WINSRV1:

gpupdate /force

## **Step 7: Verify**

**Step 1:** Lock the screen or log off from any domain computer

**Step 2:** Before the login screen appears you should see:

Title:   WorldSkills ASEAN Manila

Message: authorized access only

**Step 3:** Click OK to proceed to login

**✓ Tip:** *This is a Computer Configuration setting so it applies to computers. Run gpupdate /force on each client or restart the client to see the banner.*

**📝 Note:** *If the banner does not appear, verify the GPO is linked to manila.com and not just created in Group Policy Objects folder.*

# **Part 5 — Control Panel Restriction GPO (control)**

The competition requires a GPO named 'control' that prevents Accounting users from accessing the Control Panel.

## **Step 1: Create the Accounting OU and Group (if not exists)**

**Step 1:** Open Server Manager → Tools → Active Directory Users and Computers

**Step 2:** Right-click manila.com → New → Organizational Unit

**Step 3:** Name it: Accounting

**Step 4:** Click OK

**Step 5:** Move accounting users into the Accounting OU by dragging them

**📝 Note:** *If the Accounting group already exists in AD, skip OU creation and just note which group contains accounting users.*

## **Step 2: Create the GPO**

**Step 1:** Open Group Policy Management

**Step 2:** Right-click manila.com → Create a GPO in this domain and Link it here

**Step 3:** Name it exactly:

control

**Step 4:** Click OK

## **Step 3: Edit the GPO**

**Step 1:** Right-click control GPO → Edit

**Step 2:** Navigate to:

User Configuration

  └─ Policies

      └─ Administrative Templates

          └─ Control Panel

**Step 3:** In the right pane find:

Prohibit access to Control Panel and PC settings

**Step 4:** Double-click it

**Step 5:** Select Enabled

**Step 6:** Click OK

**Step 7:** Close GPO Editor

## **Step 4: Apply Only to Accounting Users (Security Filtering)**

**Step 1:** In Group Policy Management click the control GPO

**Step 2:** In the right pane click Scope tab

**Step 3:** Under Security Filtering section:

* Select Authenticated Users

* Click Remove

* Click Yes to confirm

**Step 4:** Click Add

**Step 5:** Type Accounting in the search box → click Check Names

**Step 6:** Select the Accounting group → click OK

## **Step 5: Verify**

**Step 1:** Log in to a client machine as an Accounting user

**Step 2:** Run gpupdate /force

**Step 3:** Try to open Control Panel

**Step 4:** Should show: This operation has been cancelled due to restrictions in effect on this computer

**✓ Tip:** *Security Filtering ensures only Accounting users get this restriction. Other users can still access Control Panel.*

# **Part 6 — Registry Restriction GPO (registry)**

The competition requires a GPO named 'registry' that prevents users in the Manila OU from using registry editing tools like regedit.

## **Step 1: Create Manila OU (if not exists)**

**Step 1:** Open Active Directory Users and Computers

**Step 2:** Right-click manila.com → New → Organizational Unit

**Step 3:** Name it: Manila

**Step 4:** Click OK

**Step 5:** Move relevant users into Manila OU

## **Step 2: Create the GPO Linked to Manila OU**

**Step 1:** Open Group Policy Management

**Step 2:** Expand manila.com → find Manila OU

**Step 3:** Right-click Manila OU → Create a GPO in this domain and Link it here

**Step 4:** Name it exactly:

registry

**Step 5:** Click OK

**📝 Note:** *By linking the GPO directly to the Manila OU, it only applies to users inside that OU automatically.*

## **Step 3: Edit the GPO**

**Step 1:** Right-click registry GPO → Edit

**Step 2:** Navigate to:

User Configuration

  └─ Policies

      └─ Administrative Templates

          └─ System

**Step 3:** In the right pane find:

Prevent access to registry editing tools

**Step 4:** Double-click it

**Step 5:** Select Enabled

**Step 6:** Under Disable regedit from running silently select: Yes

**Step 7:** Click OK

**Step 8:** Close GPO Editor

## **Step 4: Verify**

**Step 1:** Log in as a user in the Manila OU

**Step 2:** Press Windows \+ R → type regedit → press Enter

**Step 3:** Should show: Registry editing has been disabled by your administrator

**✓ Tip:** *Also test that users NOT in Manila OU can still access regedit to confirm the restriction is scoped correctly.*

# **Part 7 — Google Chrome Homepage GPO (google)**

The competition requires a GPO named 'google' that forces Chrome to open http://w3.manila.com as the homepage for all users.

## **Step 1: Find the Chrome Enterprise Bundle**

**Step 1:** Open File Explorer on WINSRV1

**Step 2:** Navigate to:

C:\\Users\\Administrator\\Documents

**Step 3:** Find the file: googleChromeEnterpriseBundle64.zip

## **Step 2: Extract the Bundle**

**Step 1:** Right-click googleChromeEnterpriseBundle64.zip

**Step 2:** Click Extract All

**Step 3:** Extract to: C:\\ChromeBundle

**Step 4:** Click Extract

## **Step 3: Copy ADMX Files to Central Store**

**Step 1:** Open File Explorer → navigate to:

C:\\ChromeBundle\\Configuration\\admx

**Step 2:** Create the Central Store folder if it does not exist:

C:\\Windows\\SYSVOL\\sysvol\\manila.com\\Policies\\PolicyDefinitions

**Step 3:** Copy these files from C:\\ChromeBundle\\Configuration\\admx:

* chrome.admx → paste into PolicyDefinitions folder

* google.admx → paste into PolicyDefinitions folder

**Step 4:** Also copy the language files:

* Copy the en-US folder from admx directory

* Paste into PolicyDefinitions\\en-US folder

**📝 Note:** *If PolicyDefinitions folder does not exist, create it manually at the path shown above.*

## **Step 4: Create the GPO**

**Step 1:** Open Group Policy Management

**Step 2:** Right-click manila.com → Create a GPO in this domain and Link it here

**Step 3:** Name it exactly:

google

**Step 4:** Click OK

## **Step 5: Edit the GPO**

**Step 1:** Right-click google GPO → Edit

**Step 2:** Navigate to:

User Configuration

  └─ Policies

      └─ Administrative Templates

          └─ Google

              └─ Google Chrome

**📝 Note:** *If Google Chrome folder does not appear, wait a moment and press F5 to refresh. The ADMX files may take a moment to load.*

**Step 3:** Configure Homepage URL:

* Find: Homepage URL

* Double-click it

* Select: Enabled

* In the field enter: http://w3.manila.com

* Click OK

**Step 4:** Configure Action on Startup:

* Find: Action on startup

* Double-click it

* Select: Enabled

* From dropdown select: Open a list of URLs

* Click OK

**Step 5:** Configure URLs to open on startup:

* Find: URLs to open on startup

* Double-click it

* Select: Enabled

* Click Show button

* In the Value column enter: http://w3.manila.com

* Click OK → OK

**Step 6:** Configure Show Home Button:

* Find: Show Home button on toolbar

* Double-click it

* Select: Enabled

* Click OK

**Step 7:** Close GPO Editor

## **Step 6: Verify**

**Step 1:** On a client machine run: gpupdate /force

**Step 2:** Open Google Chrome

**Step 3:** Homepage should open: http://w3.manila.com

**Step 4:** Try to change homepage in Chrome settings

**Step 5:** Should be greyed out — managed by administrator

**⚠ Warning:** *The DNS record for w3.manila.com must exist and point to 192.168.1.10 (LINSRV1) for the homepage to load. See Part 10 for DNS configuration.*

# **Part 8 — PKI Auto Enrollment GPO (certenroll)**

The competition requires a GPO named 'certenroll' so that domain computers automatically receive certificates from WINSRV3 (the Issuing CA).

## **Step 1: Ensure Root CA Certificate is Available**

**Step 1:** Verify ManilaRootCA.crt file is available on WINSRV1

**Step 2:** It should be at C:\\ManilaRootCA.crt (downloaded from WINSRV3)

**Step 3:** If not available, copy from WINSRV3:

Open browser → http://192.168.2.30/certsrv

Login: manila\\Administrator / P@ssw0rd

Click: Download a CA certificate

Select: Base 64 → Download CA certificate

Save as: C:\\ManilaRootCA.crt

## **Step 2: Create the certenroll GPO**

**Step 1:** Open Group Policy Management

**Step 2:** Right-click manila.com → Create a GPO in this domain and Link it here

**Step 3:** Name it exactly:

certenroll

**Step 4:** Click OK

## **Step 3: Configure Auto-Enrollment**

**Step 1:** Right-click certenroll GPO → Edit

**Step 2:** Navigate to:

Computer Configuration

  └─ Policies

      └─ Windows Settings

          └─ Security Settings

              └─ Public Key Policies

**Step 3:** Double-click Certificate Services Client – Auto-Enrollment

**Step 4:** Change Configuration Model to: Enabled

**Step 5:** Check ✅ Renew expired certificates, update pending certificates

**Step 6:** Check ✅ Update certificates that use certificate templates

**Step 7:** Click OK

**Step 8:** Also double-click Certificate Services Client – Certificate Enrollment Policy

**Step 9:** Change Configuration Model to: Enabled

**Step 10:** Click OK

## **Step 4: Import Root CA into Trusted Root Store via GPO**

**Step 1:** Still inside certenroll GPO editor

**Step 2:** Navigate to:

Computer Configuration

  └─ Policies

      └─ Windows Settings

          └─ Security Settings

              └─ Public Key Policies

                  └─ Trusted Root Certification Authorities

**Step 3:** Right-click Trusted Root Certification Authorities → Import

**Step 4:** Click Next in the Certificate Import Wizard

**Step 5:** Click Browse → navigate to C:\\ManilaRootCA.crt

**Step 6:** Click Next

**Step 7:** Verify store is: Trusted Root Certification Authorities

**Step 8:** Click Next → Finish → OK

**Step 9:** Close GPO Editor

## **Step 5: Verify Auto-Enrollment Works**

**Step 1:** On Client1 or Client2 open Command Prompt

**Step 2:** Run:

gpupdate /force

**Step 3:** Open: certmgr.msc

**Step 4:** Expand Personal → Certificates

**Step 5:** Should see a certificate issued by Manila-Issuing-CA

**Step 6:** Also check Trusted Root Certification Authorities

**Step 7:** Should see Manila-Root-CA in the list

**✓ Tip:** *If certificate does not appear after gpupdate, try: certutil \-pulse to trigger enrollment manually.*

# **Part 9 — DNS Configuration**

WINSRV1 is the DNS server for the domain. DNS records must be created for LINSRV1 and w3.manila.com so clients can resolve them.

## **Step 1: Open DNS Manager**

**Step 1:** Open Server Manager → Tools → DNS

## **Step 2: Create A Record for linsrv1.manila.com**

**Step 1:** Expand WINSRV1 in the left panel

**Step 2:** Expand Forward Lookup Zones

**Step 3:** Click manila.com zone

**Step 4:** In the right pane right-click blank area → New Host (A or AAAA)

**Step 5:** Fill in:

Name:        linsrv1

IP Address:  192.168.1.10

**Step 6:** Check ✅ Create associated pointer (PTR) record

**Step 7:** Click Add Host

**Step 8:** Click OK

## **Step 3: Create A Record for w3.manila.com**

**Step 1:** Right-click blank area in manila.com zone → New Host (A or AAAA)

**Step 2:** Fill in:

Name:        w3

IP Address:  192.168.1.10

**Step 3:** Click Add Host → OK

## **Step 4: Verify DNS Records**

**Step 1:** Open Command Prompt on WINSRV1

**Step 2:** Test both records:

nslookup linsrv1.manila.com

nslookup w3.manila.com

**Step 3:** Both should return: 192.168.1.10

**Step 4:** Also test from Client1:

nslookup linsrv1.manila.com 192.168.2.10

nslookup w3.manila.com 192.168.2.10

**✓ Tip:** *If nslookup fails, ensure the DNS service is running: Services.msc → DNS → Started*

# **Part 10 — DHCP Configuration Verification**

pfSense handles DHCP for LAN clients, but verify WINSRV1 is set as the DNS server in DHCP options.

## **Option A — DHCP on pfSense (per competition document)**

**Step 1:** Login to pfSense: https://192.168.2.254

**Step 2:** Go to Services → DHCP Server

**Step 3:** Click OPT2 (LAN clients) tab

**Step 4:** Verify or set:

DNS Server 1: 192.168.2.10

**Step 5:** Click Save

## **Option B — If DHCP is on WINSRV1**

**Step 1:** Open Server Manager → Tools → DHCP

**Step 2:** Expand WINSRV1 → IPv4

**Step 3:** Right-click Scope → Properties

**Step 4:** Click DNS tab

**Step 5:** Verify DNS server is: 192.168.2.10

**Step 6:** Click OK

**✓ Tip:** *Clients should automatically get DNS \= 192.168.2.10 (WINSRV1) via DHCP. This ensures domain name resolution works.*

# **Part 11 — File Share Configuration (pictures)**

The competition requires a share named 'pictures' at C:\\shares\\pictures with specific permissions for different groups.

## **Required Permissions**

| Group | Share Permission | NTFS Permission |
| :---- | :---- | :---- |
| Customer Service | Read | Read & Execute |
| Graphics | Change | Modify |
| IT | Full Control | Full Control |
| Everyone / Others | No Access (Remove) | No Access (Remove) |

## **Step 1: Create the Folder**

**Step 1:** Open File Explorer

**Step 2:** Navigate to C:\\

**Step 3:** Create folder: shares

**Step 4:** Inside shares create folder: pictures

## **Step 2: Share the Folder**

**Step 1:** Right-click the pictures folder → Properties

**Step 2:** Click Sharing tab → Advanced Sharing

**Step 3:** Check ✅ Share this folder

**Step 4:** Share name: pictures

**Step 5:** Click Permissions

**Step 6:** Select Everyone → click Remove

**Step 7:** Add Customer Service group:

* Click Add → type Customer Service → Check Names → OK

* Check ✅ Read → click OK

**Step 8:** Add Graphics group:

* Click Add → type Graphics → Check Names → OK

* Check ✅ Change → click OK

**Step 9:** Add IT group:

* Click Add → type IT → Check Names → OK

* Check ✅ Full Control → click OK

**Step 10:** Click OK → OK to close Advanced Sharing

## **Step 3: Set NTFS Permissions**

**Step 1:** Click Security tab on the pictures folder Properties

**Step 2:** Click Advanced

**Step 3:** Click Disable Inheritance

**Step 4:** Select Convert inherited permissions into explicit permissions

**Step 5:** Remove all inherited groups except SYSTEM and Administrators

**Step 6:** Add Customer Service:

* Click Add → Select a principal → type Customer Service → OK

* Type: Allow

* Check: Read & Execute, List folder contents, Read

* Click OK

**Step 7:** Add Graphics:

* Click Add → Select a principal → type Graphics → OK

* Type: Allow

* Check: Modify, Read & Execute, List, Read, Write

* Click OK

**Step 8:** Add IT:

* Click Add → Select a principal → type IT → OK

* Type: Allow

* Check: Full Control

* Click OK

**Step 9:** Click Apply → OK → OK

**✓ Tip:** *Best practice: Share permissions should be set to Full Control for all groups, then control access via NTFS permissions only. However competition instructions specify share permissions per group so follow them exactly.*

# **Part 12 — Auditing manila.jpg**

The competition requires that access to manila.jpg is logged in the Security Event Log whenever any group member reads the file.

## **Step 1: Enable Audit Object Access Policy**

**Step 1:** Open Group Policy Management

**Step 2:** Right-click Default Domain Policy → Edit

**Step 3:** Navigate to:

Computer Configuration

  └─ Policies

      └─ Windows Settings

          └─ Security Settings

              └─ Advanced Audit Policy Configuration

                  └─ Audit Policies

                      └─ Object Access

**Step 4:** Double-click Audit File System

**Step 5:** Check ✅ Configure the following audit events

**Step 6:** Check ✅ Success

**Step 7:** Optionally check ✅ Failure

**Step 8:** Click OK

**Step 9:** Close GPO Editor

## **Step 2: Set Auditing on manila.jpg**

**Step 1:** Navigate to C:\\shares\\pictures in File Explorer

**Step 2:** Right-click manila.jpg → Properties

**Step 3:** Click Security tab

**Step 4:** Click Advanced

**Step 5:** Click Auditing tab

**Step 6:** Click Add

**Step 7:** Click Select a principal

**Step 8:** Type Everyone → Check Names → OK

**Step 9:** Set Type to: All

**Step 10:** Under Basic permissions check ✅ Read

**Step 11:** Click OK → Apply → OK → OK

## **Step 3: Apply Policy**

**Step 1:** Open Command Prompt on WINSRV1

**Step 2:** Run:

gpupdate /force

## **Step 4: Verify Auditing Works**

**Step 1:** Log in as any domain user on Client1

**Step 2:** Navigate to \\\\WINSRV1\\pictures

**Step 3:** Open or copy manila.jpg

**Step 4:** On WINSRV1 open Event Viewer:

Server Manager → Tools → Event Viewer

Windows Logs → Security

**Step 5:** Look for Event ID 4663

**Step 6:** Details should show: Object Name \= manila.jpg

**✓ Tip:** *Filter the Security log by Event ID 4663 to find file access events quickly: right-click Security log → Filter Current Log → Event ID: 4663*

# **Part 13 — Additional Security GPOs (Table 2 Recommendations)**

The competition asks you to recommend 3 additional GPOs to improve domain security. Fill in Table 2 on the competition answer sheet.

| GPO Name | Policy Path | Effect | Why Selected |
| :---- | :---- | :---- | :---- |
| Account Lockout Policy | Computer Config → Policies → Windows Settings → Security Settings → Account Policies → Account Lockout Policy | Locks account for 30 minutes after 5 failed login attempts | Prevents brute force password attacks. More effective than complexity alone because it stops automated tools regardless of password strength |
| Disable USB Storage | Computer Config → Policies → Administrative Templates → System → Removable Storage Access → All Removable Storage Classes: Deny all access | Prevents users from reading or writing to USB drives and external storage | Prevents data theft and malware introduction via removable media. Addresses insider threat and physical security vectors |
| Screen Lock Timeout | Computer Config → Policies → Windows Settings → Security Settings → Local Policies → Security Options → Interactive logon: Machine inactivity limit | Automatically locks screen after 10 minutes of inactivity | Prevents unauthorized access to unattended workstations. Simpler and more reliable than relying on users to manually lock their screens |

**📝 Note:** *Save this table as a document on the competitor laptop desktop so experts can collect it during assessment.*

# **Part 14 — Final Validation Checklist**

Run through every item before signaling completion to the experts.

## **Active Directory and Domain**

| Check | How to Verify | Expected |
| :---- | :---- | :---- |
| Domain is manila.com | System Properties | manila.com |
| WINSRV1 is Domain Controller | Server Manager → AD DS | Green status |
| Domain users can log in | Login as any domain user on Client1 | Success |
| Password policy enforced | Try creating user with 7-char password | Should fail |
| Executive group uses 10-char policy | msDS-ResultantPSO attribute on exec user | ExecutivePSO |

## **GPO Configurations**

| GPO | Test | Expected |
| :---- | :---- | :---- |
| Login Banner | Log off and log back on | Banner appears before login |
| control | Login as Accounting user → open Control Panel | Blocked |
| registry | Login as Manila OU user → run regedit | Blocked |
| google | Open Chrome as any user | http://w3.manila.com opens |
| certenroll | certmgr.msc → Personal → Certificates | Certificate from Manila-Issuing-CA |

## **DNS**

| Record | Command | Expected |
| :---- | :---- | :---- |
| linsrv1.manila.com | nslookup linsrv1.manila.com | 192.168.1.10 |
| w3.manila.com | nslookup w3.manila.com | 192.168.1.10 |
| winsrv1.manila.com | nslookup winsrv1.manila.com | 192.168.2.10 |
| winsrv3.manila.com | nslookup winsrv3.manila.com | 192.168.2.30 |

## **File Share**

| Test | Expected |
| :---- | :---- |
| Login as Customer Service user → access \\\\WINSRV1\\pictures → try to read | Success |
| Login as Customer Service user → try to create/delete file | Access Denied |
| Login as Graphics user → modify a file | Success |
| Login as IT user → Full Control | Success |
| Login as any other user → access share | Access Denied |
| Read manila.jpg → check Event Viewer Event ID 4663 | Event logged |

## **PKI and Certificates**

| Test | Expected |
| :---- | :---- |
| Open browser → https://linsrv1.manila.com | No certificate warning |
| Open browser → https://w3.manila.com | No certificate warning |
| certmgr.msc → Trusted Root → find Manila-Root-CA | Present |
| certmgr.msc → Personal → certificate from Manila-Issuing-CA | Present |

**✓ Tip:** *If any check fails, go back to the relevant part and re-verify. Fix issues in order: network → DNS → AD → GPO → PKI.*

*WorldSkills ASEAN Manila 2025 — Cyber Security Skill 54 — Module A2 — GUI Reference Guide*

# WINSRV1: 2026

# **WinSRV1 Setup Guide — WSPH2026 Alignment (Identity & Policy, DNS/DHCP)**

*Adapted from the WSA2025 WINSRV1 guide. Covers Test Project A2's **Identity and Policy (AD/GPO)** section and the DNS/DHCP items from the **ISP/Firewall** section. PKI/certificate work is not repeated here — see `PKI_Setup_Guide_Single_Tier_WSPH2026.md`, since the `certenroll` GPO now belongs to that document's single-tier CA setup on WinSRV1.*

## **Quick Recap (from the PKI guide)**

| Item | Value |
| ----- | ----- |
| Host | WinSRV1, 192.168.2.10, Servers VLAN |
| Domain | `<DOMAIN>` — per Competition Day Instructions |
| Also on this box | AD DS, DNS, and now the single-tier CA |

---

## **What's dropped from the WSA2025 version, and why**

| WSA2025 part | Status | Reason |
| ----- | ----- | ----- |
| Part 8 — `certenroll` PKI GPO | **Removed — see PKI guide** | Superseded by the single-tier CA setup; don't create it twice |
| Fine-Grained Password Policy for "Executive" group | **Dropped** | Test Project doesn't mention any Executive group or a fine-grained policy requirement — this was WSA2025-specific. Only add it back if Competition Day Instructions name a similar VIP/privileged group |
| File Share ("pictures") \+ Customer Service/Graphics/IT permission matrix | **Dropped** | No file-share task anywhere in Test Project A2 |
| Auditing `manila.jpg` specifically | **Dropped as written, generalized instead** | Tied to the dropped file-share task. Appendix 1 still lists "Audit / logging baseline" as a recommendation category, so Part 5 below gives you a generic, file-independent version you can point at *any* object if the Organizer wants a live demo |
| Chrome Homepage GPO | **Kept, but flagged conditional** | Only usable if a Chrome Enterprise ADMX bundle is actually staged on the competitor desktop this year — don't assume it will be |

Test Project's Identity and Policy section only strictly requires: (1) a domain password policy baseline, and (2) **at least two** additional workstation/user controls from Appendix 1's table — it doesn't require you to build all of Login Banner \+ Control Panel \+ Registry \+ Chrome. Treat Parts 2–5 below as a menu; pick two (Login Banner \+ Control Panel Restriction is the safest, most demonstrable pair) and leave the others as *documented recommendations* in Appendix 1 Table 3 rather than implementing all of them.

---

# **Part 1 — Verify Initial Configuration**

**Step 1:** Control Panel → Network and Sharing Center → Change adapter settings → adapter Properties → IPv4 — confirm:

IP: 192.168.2.10 Mask: 255.255.255.0 Gateway: 192.168.2.254 DNS: 192.168.2.10 (itself)

**Step 2:** Right-click Start → System — confirm computer name is WinSRV1 and domain is `<DOMAIN>`

**Step 3:** PowerShell → `ipconfig /all` — confirm the same

**Step 4:** Server Manager — confirm AD DS, DNS, DHCP (if present), and (from the PKI guide) AD CS all show green

---

# **Part 2 — Domain Password Policy Baseline (Required)**

**Step 1:** Group Policy Management → Forest → Domains → `<DOMAIN>` → Group Policy Objects → right-click **Default Domain Policy** → Edit

**Step 2:** Navigate to Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy

**Step 3:** Set the values **Competition Day Instructions specify** for this year. (WSA2025 used: minimum length 8, maximum age 30 days, complexity enabled — treat these as an example to practice with, not this year's actual numbers.)

**Step 4:** Close the editor, then on WinSRV1: `gpupdate /force`

**✓ Verification for the answer sheet:** try creating a test user with a password that violates the policy (e.g. too short) and confirm it's rejected — that's your "policy results" evidence.

---

# **Part 3 — Login Banner (Candidate Control \#1)**

**Step 1:** Group Policy Management → right-click `<DOMAIN>` → Create a GPO in this domain and Link it here → name it **Login Banner**

**Step 2:** Edit → Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options

**Step 3:** Set **Interactive logon: Message title for users attempting to log on** to your event/org name

**Step 4:** Set **Interactive logon: Message text for users attempting to log on** to your required text

**Step 5:** `gpupdate /force` on a client, then lock/log off — banner should appear before the login screen

---

# **Part 4 — Control Panel Restriction (Candidate Control \#2)**

**Step 1:** Create a GPO named **control**, linked at `<DOMAIN>` (or scoped to a specific OU/group if the task specifies one — WSA2025 used an "Accounting" group as an example target)

**Step 2:** Edit → User Configuration → Policies → Administrative Templates → Control Panel → **Prohibit access to Control Panel and PC settings** → Enabled

**Step 3:** If scoping to a specific group: GPO → Scope tab → Security Filtering → remove Authenticated Users, add the target group

**Step 4:** Verify: log in as a targeted user, `gpupdate /force`, try opening Control Panel — should be blocked; confirm an untargeted user is unaffected

---

# **Part 5 — Registry Tools Restriction (Optional Alternate Control)**

**Step 1:** Create a GPO named **registry**, linked to the target OU/group

**Step 2:** Edit → User Configuration → Policies → Administrative Templates → System → **Prevent access to registry editing tools** → Enabled

**Step 3:** Verify as a targeted user: Win+R → `regedit` → should show "Registry editing has been disabled by your administrator"

---

# **Part 5b — Chrome Homepage GPO (Optional, Conditional)**

Only attempt this if a Chrome Enterprise ADMX bundle is actually present on the competitor desktop — don't assume it will be provisioned this year.

**Step 1:** If present, extract it and copy `chrome.admx`/`google.admx` (+ the `en-US` language folder) into `C:\Windows\SYSVOL\sysvol\<DOMAIN>\Policies\PolicyDefinitions`

**Step 2:** Create a GPO named **google**, configure Homepage URL / Action on Startup / URLs to open on startup under User Configuration → Administrative Templates → Google → Google Chrome

**Step 3:** Verify on a client after `gpupdate /force`

If the bundle isn't available, document this as a **recommendation only** in Appendix 1 Table 3 rather than losing time hunting for it.

---

# **Part 6 — Generic Audit/Logging Baseline (Optional Control, Not Tied to a Specific File)**

If you want a live-demonstrable audit control instead of just a written recommendation:

**Step 1:** Default Domain Policy → Computer Configuration → Policies → Windows Settings → Security Settings → Advanced Audit Policy Configuration → Audit Policies → Object Access → **Audit File System** → Enabled, Success (and Failure if you want it)

**Step 2:** Pick **any** object relevant to this year's task (not `manila.jpg` — that was WSA2025's file-share scenario) — right-click it → Properties → Security → Advanced → Auditing tab → Add a principal, set Type: All, check the permissions you want logged

**Step 3:** `gpupdate /force`, then access the object as a test user, and confirm the event (Event ID 4663\) appears in Event Viewer → Windows Logs → Security

---

# **Part 7 — DNS Record for LinuxWeb**

**Step 1:** Server Manager → Tools → DNS → expand WinSRV1 → Forward Lookup Zones → `<DOMAIN>` zone

**Step 2:** Right-click → New Host (A or AAAA):

Name: linuxweb IP: 192.168.1.10

**Step 3:** Check "Create associated pointer (PTR) record" → Add Host → OK

**Step 4:** Verify from WinSRV1 and from Client1:

nslookup linuxweb.\<DOMAIN\>

Should return 192.168.1.10 both times

*(This single record replaces WSA2025's two hostnames — `linsrv1` and `w3` — since this year's environment has one DMZ web VM, not two separate sites.)*

---

# **Part 8 — DHCP Verification**

Per the Firewall configuration bullet: *"Enable DHCP for LAN clients; configure WinSRV1 as the DNS server for LAN clients."* This year DHCP is expected to be handled by **pfSense**, not WinSRV1 — verify rather than build it here.

**Step 1:** Log into pfSense → Services → DHCP Server → LAN tab

**Step 2:** Confirm DHCP is enabled for the LAN range and **DNS Server 1 is set to 192.168.2.10** (WinSRV1)

**Step 3:** On Client1/Client2, `ipconfig /all` — confirm they received a lease and that the DNS server listed is 192.168.2.10

---

# **Part 9 — Appendix 1, Table 3 (Draft)**

Use this as your starting draft for the answer sheet — fill in the "Notes/Verification" column with what you actually did once decided:

| Policy Area | Recommendation | Notes / Verification |
| ----- | ----- | ----- |
| Password / account policy baseline | Domain password policy per Part 2 | Implemented — tested with a policy-violating password |
| Workstation security hardening (user controls) | Login Banner (Part 3\) **and** Control Panel restriction (Part 4\) | Implemented — pick these two as your required "at least two" controls |
| Audit / logging baseline | Object Access auditing (Part 6\) | Implement if time allows; otherwise document as recommended |
| Removable media / device controls | Disable removable storage via Administrative Templates → System → Removable Storage Access | Recommend only unless task specifically calls for it |
| Browser / web access controls | Chrome homepage/managed settings (Part 5b) | Conditional on ADMX bundle availability — recommend only if not implemented |

---

# **Part 10 — Final Validation Checklist**

## **Domain / AD**

| Check | How | Expected |
| ----- | ----- | ----- |
| Domain is `<DOMAIN>` | System Properties | Matches |
| Password policy enforced | Try a policy-violating password | Rejected |

## **GPOs**

| GPO | Test | Expected |
| ----- | ----- | ----- |
| Login Banner | Log off/on | Banner appears pre-login |
| control | Login as targeted user → Control Panel | Blocked |
| registry (if implemented) | Login as targeted user → regedit | Blocked |
| google (if implemented) | Open Chrome | Configured homepage loads |

## **DNS / DHCP**

| Check | How | Expected |
| ----- | ----- | ----- |
| linuxweb.\<DOMAIN\> resolves | `nslookup` from Client1 | 192.168.1.10 |
| LAN clients get correct DNS via DHCP | `ipconfig /all` on Client1/Client2 | DNS \= 192.168.2.10 |

## **PKI**

See `PKI_Setup_Guide_Single_Tier_WSPH2026.md` — don't duplicate those checks here.

**✓ Tip:** *If something fails, work the dependency chain in order: network → DNS → AD/domain membership → GPO → the specific control being tested. Most GPO "it's not applying" issues trace back to `gpupdate` not having run, security filtering, or a link/inheritance block — not the policy setting itself.*

# Yes

**Part 13 — Additional Security GPO Recommendations (Table 2\)**

The competition asks for 3 GPOs that would further secure the domain. Here are the best industry-standard choices with full rationale:

---

| GPO Name | Policy Path | Effect | Why Selected Over Other Choices |
| ----- | ----- | ----- | ----- |
| **Account Lockout Policy** | Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Account Lockout Policy | Locks the account for 30 minutes after 5 consecutive failed login attempts. Resets the counter after 30 minutes. | Directly counters brute-force and password-spray attacks. Unlike password complexity alone — which only raises the bar for guessing — lockout actively stops automated tools regardless of password strength. Chosen over "minimum password length increase" because an 8-char complex password with lockout is more resistant than a 20-char password with unlimited attempts. |
| **Disable USB/Removable Storage** | Computer Configuration → Policies → Administrative Templates → System → Removable Storage Access → All Removable Storage Classes: Deny all access | Prevents all users from reading from or writing to USB drives, external hard drives, and removable media on domain computers. | Addresses two separate threat vectors simultaneously: data exfiltration by insiders and malware introduction via infected USB drives. Chosen over "AppLocker" or "Software Restriction Policies" because removable media is a physical attack vector that software controls alone cannot address — this eliminates the vector entirely rather than trying to control what runs from it. |
| **Screen Lock / Inactivity Timeout** | Computer Configuration → Policies → Windows Settings → Security Settings → Local Policies → Security Options → Interactive logon: Machine inactivity limit | Automatically locks the workstation screen and requires re-authentication after 10 minutes of inactivity. | Prevents unauthorized access to unattended, logged-in workstations — a common physical-layer attack in shared office environments. Chosen over relying on users to manually press Win+L because policy-enforced lockout is guaranteed and auditable, whereas user behavior is not. More targeted than a full screensaver GPO because it uses the OS-level inactivity timer which works even when screensavers are disabled. |

---

**Why these three specifically, over other candidates?**

The key principle is **layered defense across different attack surfaces**:

* **Account Lockout** covers the *network/authentication layer* — stops remote credential attacks  
* **USB Disable** covers the *physical layer* — stops walk-up media attacks and insider data theft  
* **Screen Lock** covers the *physical access layer* — stops unattended session hijacking

Other common candidates like "Disable Command Prompt," "Block .exe from Downloads," or "Audit Logon Events" are valid but more narrow — they address a single vector or are detective rather than preventive. These three are **preventive controls** that block entire classes of attack rather than specific techniques, which gives maximum coverage for a domain that already has password policy and access controls in place.

# Tab 17

for the closest WSC2024/WSA2025 setup, use this:

MikroTik RB2011 \= outer internet / management router only  
D-Link DGS-1210-28 \= VLAN switch  
pfSense VM \= competition firewall, DHCP, rules, VPN, Snort  
ESXi \= runs all competition VMs

**Direct Answer**  
Should we disable MikroTik firewall for the competition VLANs?  
**Yes, for the competition VLANs.**  
Do not let MikroTik act as firewall/gateway for:  
VLAN 10 DMZ  
VLAN 20 SERVERS  
VLAN 30 LAN  
VLAN 40 INTERNET-LAB  
Let **pfSense** handle those.

Should MikroTik still do DHCP?  
**Only for Management VLAN 99, not for the competition VLANs.**  
Official-style MA2 expects:  
pfSense \= DHCP for LAN clients  
WinSRV1 \= DNS server  
So for competition alignment:  
CLIENT1/CLIENT2 DHCP should come from pfSense  
DNS given by DHCP should be 192.168.2.10  
Gateway given by DHCP should be pfSense LAN IP

**Recommended Final Design**  
ISP / Venue Internet  
       |  
MikroTik RB2011  
       |  
D-Link DGS-1210-28  
       |  
ESXi Lab Host  
       |  
pfSense VM  
WINSRV1  
WINSRV3  
WINSRV4  
LINSRV1  
CLIENT1  
CLIENT2  
CLIENT3

Device Roles

Device  
Role  
MikroTik RB2011  
Internet upstream, NAT to venue ISP, management VLAN only  
D-Link DGS-1210-28  
VLAN trunk/access switch  
pfSense VM  
Main competition firewall/router/DHCP/VPN/Snort  
WINSRV1  
AD, DNS, GPO, file services  
LINSRV1  
Linux web server  
CLIENT1/CLIENT2  
Internal LAN clients  
CLIENT3  
External/Internet test client

Best Setup for Competition Mode  
MikroTik keeps only Management \+ WAN  
Keep MikroTik active for:

Network  
MikroTik Role  
WAN / Internet  
Yes  
VLAN 99 Management  
Yes  
VLAN 10 DMZ  
No  
VLAN 20 SERVERS  
No  
VLAN 30 LAN  
No  
VLAN 40 INTERNET-LAB  
No  
MikroTik management:  
MikroTik Management IP: 192.168.99.254  
D-Link Management IP:  192.168.99.2  
ESXi Management IP:    192.168.99.10

pfSense VM Interfaces  
Create pfSense with these NICs:

pfSense Interface  
ESXi Port Group  
IP  
WAN  
`WAN-UPSTREAM` or `MGMT/WAN`  
DHCP from MikroTik  
LAN  
`LAN`  
`172.16.100.254/24`  
SERVERS  
`SERVERS`  
`192.168.2.254/24`  
DMZ  
`DMZ`  
`192.168.1.254/24`  
INTERNET-LAB  
`INTERNET-LAB`  
`10.40.0.254/24`, optional depending on design  
If CLIENT3 is supposed to simulate an external client, put CLIENT3 on the same “ISP/external” side as pfSense WAN, not behind pfSense LAN.

ESXi Port Groups  
Use these:

ESXi Port Group  
VLAN ID  
Purpose  
`Management Network`  
0 / untagged VLAN 99  
ESXi management  
`WAN-UPSTREAM`  
99 or dedicated upstream VLAN  
pfSense WAN to MikroTik  
`SERVERS`  
20  
WINSRV1, WINSRV3, WINSRV4  
`DMZ`  
10  
LINSRV1  
`LAN`  
30  
CLIENT1, CLIENT2  
`INTERNET-LAB`  
40  
CLIENT3 / external testing

What to Disable on MikroTik  
For competition mode, disable MikroTik services on VLAN 10/20/30/40.

Disable DHCP on competition VLANs  
On MikroTik:  
/ip dhcp-server print  
Disable DHCP servers for:  
LAN  
SERVERS  
DMZ  
INTERNET-LAB  
Example:  
/ip dhcp-server disable \[find name\~"LAN"\]  
/ip dhcp-server disable \[find name\~"SERVERS"\]  
/ip dhcp-server disable \[find name\~"DMZ"\]  
/ip dhcp-server disable \[find name\~"INET"\]  
Keep DHCP only for management if you use it:  
VLAN 99 Management

Remove or disable MikroTik gateway IPs on competition VLANs  
This is critical.  
If pfSense will use:  
192.168.1.254  
192.168.2.254  
172.16.100.254  
10.40.0.254  
then remove those IPs from MikroTik.  
Check:  
/ip address print  
Disable only the competition VLAN gateway IPs:  
/ip address disable \[find interface=vlan10-DMZ\]  
/ip address disable \[find interface=vlan20-SERVERS\]  
/ip address disable \[find interface=vlan30-LAN\]  
/ip address disable \[find interface=vlan40-INETLAB\]  
Keep this enabled:  
vlan99-MGMT \= 192.168.99.254/24  
Do **not** disable VLAN 99 management.

Keep MikroTik WAN Internet active  
Keep:  
ether1-WAN DHCP client  
NAT to ISP  
Management VLAN 99  
This lets pfSense WAN get Internet through MikroTik.

pfSense DHCP Setup  
On pfSense, configure DHCP for LAN:  
Services \> DHCP Server \> LAN  
Set:  
Range: 172.16.100.100 \- 172.16.100.199  
Gateway: 172.16.100.254  
DNS: 192.168.2.10  
Domain: manila.com  
This matches the Test Project requirement:  
pfSense provides DHCP  
WinSRV1 is the DNS server

VM IP Plan in Competition Mode

VM  
Network  
IP  
Gateway  
DNS  
WINSRV1  
SERVERS  
`192.168.2.10`  
`192.168.2.254`  
`192.168.2.10`  
WINSRV3  
SERVERS  
`192.168.2.30`  
`192.168.2.254`  
`192.168.2.10`  
WINSRV4  
SERVERS  
`192.168.2.50`  
`192.168.2.254`  
`192.168.2.10`  
LINSRV1  
DMZ  
`192.168.1.10`  
`192.168.1.254`  
`192.168.2.10`  
CLIENT1  
LAN  
DHCP  
`172.16.100.254`  
`192.168.2.10`  
CLIENT2  
LAN  
DHCP  
`172.16.100.254`  
`192.168.2.10`  
CLIENT3  
External / ISP  
DHCP  
ISP/pfSense external design  
ISP DNS or lab DNS

Recommended Migration Steps  
Do this carefully.

Step 1 — Save backups  
On MikroTik:  
/system backup save name=before-pfsense-mode  
/export file=before-pfsense-mode  
On D-Link:  
Save current configuration backup  
On ESXi:  
Snapshot pfSense  
Snapshot WINSRV1  
Snapshot LINSRV1

Step 2 — Confirm pfSense interfaces  
In ESXi, pfSense should have NICs connected to:  
WAN-UPSTREAM  
LAN  
SERVERS  
DMZ  
INTERNET-LAB or ISP

Step 3 — Disable MikroTik DHCP on competition VLANs  
Disable DHCP on VLAN 10/20/30/40.

Step 4 — Disable MikroTik gateway IPs on competition VLANs  
Disable MikroTik IP addresses on VLAN 10/20/30/40.  
Keep VLAN 99 active.

Step 5 — Configure pfSense interface IPs  
On pfSense:  
LAN:     172.16.100.254/24  
SERVERS: 192.168.2.254/24  
DMZ:     192.168.1.254/24  
Optional:  
INTERNET-LAB: 10.40.0.254/24

Step 6 — Enable pfSense DHCP for LAN  
Set DHCP range:  
172.16.100.100 \- 172.16.100.199  
DNS: 192.168.2.10

Step 7 — Test connectivity  
From WINSRV1:  
ping 192.168.2.254  
ping 172.16.100.254  
From LINSRV1:  
ping \-c 4 192.168.1.254  
ping \-c 4 192.168.2.10  
From CLIENT1:  
ipconfig /all  
ping 172.16.100.254  
ping 192.168.2.10  
nslookup manila.com 192.168.2.10

Strong Recommendation  
Use **pfSense for competition mode**.  
Use **MikroTik for management and upstream only**.  
Do not use MikroTik DHCP for the competition LAN if you want the closest Test Project alignment.  
Final clean model:  
MikroTik \= outer internet \+ management  
D-Link \= VLAN switching  
pfSense \= competition firewall/router/DHCP/VPN/Snort  
WINSRV1 \= DNS/AD/GPO  
ESXi \= all VMs  
This will remove the conflict and align better with the actual Test Project.

# Updated Windows Server

For the **first VM on your new ESXi Lab Host**, build **WINSRV1** first.

This is the correct first VM because it becomes your lab’s control plane:

WINSRV1 \= Active Directory \+ DNS \+ GPO \+ File Services  
Domain  \= manila.com  
Network \= SERVERS VLAN 20  
IP      \= 192.168.2.10

Do **not** start with Kali, pfSense, or Linux. Start with **WINSRV1** because almost every other task depends on domain/DNS.

---

# **Phase 1 — Confirm ESXi Network Before Creating the VM**

On ESXi, confirm these port groups already exist:

| ESXi Port Group | VLAN ID | Purpose |
| ----- | ----- | ----- |
| `Management Network` | `0 / None` | ESXi management on VLAN 99 |
| `SERVERS` | `20` | WINSRV1, WINSRV3, WINSRV4 |
| `DMZ` | `10` | LINSRV1 |
| `LAN` | `30` | Client VMs |
| `INTERNET-LAB` | `40` | External test VMs |

Your ESXi host should be:

LABHOST1  
Management IP: 192.168.99.10  
Gateway:       192.168.99.254  
DNS:           192.168.2.10 later  
Switch Port:   D-Link Port 2  
---

# **Phase 2 — Upload the Windows Server ISO**

In ESXi web interface:

Storage \> Datastores \> DATASTORE-VMs \> Datastore browser

Create folders:

ISO  
VM-Templates  
Backups

Upload:

Windows Server 2022 Evaluation ISO

Recommended ISO:

Windows Server 2022 Standard Evaluation \- Desktop Experience

Use **Desktop Experience**, not Server Core. It is faster for training because students need ADUC, DNS Manager, Group Policy Management, Event Viewer, and File Services GUI tools.

---

# **Phase 3 — Create WINSRV1 VM**

Go to:

Virtual Machines \> Create / Register VM

Select:

Create a new virtual machine

## **VM Settings**

| Setting | Value |
| ----- | ----- |
| VM Name | `WINSRV1` |
| Compatibility | ESXi 8.0 virtual machine |
| Guest OS Family | Windows |
| Guest OS Version | Microsoft Windows Server 2022 |
| Datastore | `DATASTORE-VMs` |
| vCPU | 2 minimum, 4 preferred |
| RAM | 8 GB preferred |
| Hard Disk | 100 GB |
| Disk Provisioning | Thin provision |
| Network Adapter | `SERVERS` port group |
| Adapter Type | VMXNET3 |
| CD/DVD Drive | Windows Server 2022 ISO |
| Firmware | UEFI |

Important: network must be:

SERVERS port group \= VLAN 20  
---

# **Phase 4 — Install Windows Server 2022**

Power on the VM.

Select:

Windows Server 2022 Standard Evaluation (Desktop Experience)

Installation type:

Custom: Install Microsoft Server Operating System only

Install on the 100 GB disk.

Set local Administrator password:

Username: Administrator  
Password: P@ssw0rd

For final competition use, change to a stronger password. For training, this keeps all setup consistent.

---

# **Phase 5 — Install VMware Tools**

After first login:

Actions \> Guest OS \> Install VMware Tools

Inside Windows:

1. Open the mounted VMware Tools drive.  
2. Run installer.  
3. Choose **Typical**.  
4. Reboot.

This improves network, mouse, display, and storage drivers.

---

# **Phase 6 — Rename Server and Set Static IP**

Open **PowerShell as Administrator**.

## **Rename Server**

Rename-Computer \-NewName "WINSRV1" \-Restart

After reboot, login again as local Administrator.

---

## **Set Static IP**

Check adapter name:

Get-NetAdapter

Assume it is named `Ethernet`.

Set IP:

New-NetIPAddress \`  
\-InterfaceAlias "Ethernet" \`  
\-IPAddress 192.168.2.10 \`  
\-PrefixLength 24 \`  
\-DefaultGateway 192.168.2.254

Set DNS to itself:

Set-DnsClientServerAddress \`  
\-InterfaceAlias "Ethernet" \`  
\-ServerAddresses 192.168.2.10

Validate:

ipconfig /all  
ping 192.168.2.254  
ping 192.168.99.254

Expected:

IP Address:      192.168.2.10  
Subnet Mask:     255.255.255.0  
Default Gateway: 192.168.2.254  
DNS Server:      192.168.2.10  
---

# **Phase 7 — Take First Snapshot**

Before installing roles, take a snapshot.

In ESXi:

WINSRV1 \> Snapshots \> Take Snapshot

Snapshot name:

WINSRV1\_CLEAN\_INSTALLED\_STATIC\_IP  
---

# **Phase 8 — Install Required Roles**

On WINSRV1, open PowerShell as Administrator:

Install-WindowsFeature AD-Domain-Services,DNS,FS-FileServer,GPMC \`  
\-IncludeManagementTools

Validate:

Get-WindowsFeature AD-Domain-Services,DNS,FS-FileServer,GPMC

Do **not** install Certificate Authority on WINSRV1. In the WorldSkills-style structure, Certificate Services should be on:

WINSRV3 \= Issuing CA  
WINSRV4 \= Offline Root CA

WINSRV1 should focus on:

AD DS  
DNS  
GPO  
File Services  
---

# **Phase 9 — Promote WINSRV1 to Domain Controller**

Run:

Install-ADDSForest \`  
\-DomainName "manila.com" \`  
\-DomainNetbiosName "MANILA" \`  
\-InstallDNS \`  
\-SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd" \-AsPlainText \-Force) \`  
\-Force

The server will reboot.

After reboot, login as:

MANILA\\Administrator  
Password: P@ssw0rd  
---

# **Phase 10 — Validate AD and DNS**

Run PowerShell as Administrator:

Get-ADDomain  
Get-ADForest  
dcdiag /test:dns  
nslookup manila.com

Expected:

Domain: manila.com  
DNS test passes  
manila.com resolves to 192.168.2.10  
---

# **Phase 11 — Add DNS Forwarder for Internet**

Because your MikroTik WAN will use different ISP DHCP when you travel, use MikroTik as the DNS forwarder.

On WINSRV1:

Set-DnsServerForwarder \-IPAddress 192.168.2.254 \-UseRootHint $false  
Clear-DnsServerCache \-Force  
Get-DnsServerForwarder

Test:

Resolve-DnsName google.com  
Resolve-DnsName google.com \-Server 192.168.2.10

If this works, domain DNS and Internet DNS are both aligned.

---

# **Phase 12 — Create Core DNS Records**

Still on WINSRV1:

Add-DnsServerResourceRecordA \-Name "winsrv1" \-ZoneName "manila.com" \-IPv4Address "192.168.2.10"  
Add-DnsServerResourceRecordA \-Name "winsrv3" \-ZoneName "manila.com" \-IPv4Address "192.168.2.30"  
Add-DnsServerResourceRecordA \-Name "winsrv4" \-ZoneName "manila.com" \-IPv4Address "192.168.2.50"  
Add-DnsServerResourceRecordA \-Name "linsrv1" \-ZoneName "manila.com" \-IPv4Address "192.168.1.10"  
Add-DnsServerResourceRecordA \-Name "w3" \-ZoneName "manila.com" \-IPv4Address "192.168.1.10"

Validate:

nslookup winsrv1.manila.com  
nslookup linsrv1.manila.com  
nslookup w3.manila.com  
---

# **Phase 13 — Create OUs, Groups, and Users**

## **Create OUs**

New-ADOrganizationalUnit \-Name "Manila" \-Path "DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Accounting" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "IT" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Graphics" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Customer Service" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Executives" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Servers" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Workstations" \-Path "OU=Manila,DC=manila,DC=com"

## **Create Groups**

New-ADGroup \-Name "Accounting Users" \-GroupScope Global \-Path "OU=Accounting,OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "IT" \-GroupScope Global \-Path "OU=IT,OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "Graphics" \-GroupScope Global \-Path "OU=Graphics,OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "Customer Service" \-GroupScope Global \-Path "OU=Customer Service,OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "Executive" \-GroupScope Global \-Path "OU=Executives,OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "VPN Users" \-GroupScope Global \-Path "OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "WebUsers" \-GroupScope Global \-Path "OU=Manila,DC=manila,DC=com"

## **Create Users**

$Password \= ConvertTo-SecureString "P@ssw0rd" \-AsPlainText \-Force

New-ADUser \-Name "C1" \-SamAccountName "C1" \-AccountPassword $Password \-Enabled $true \-Path "OU=IT,OU=Manila,DC=manila,DC=com"  
New-ADUser \-Name "C2" \-SamAccountName "C2" \-AccountPassword $Password \-Enabled $true \-Path "OU=IT,OU=Manila,DC=manila,DC=com"

New-ADUser \-Name "AcctUser1" \-SamAccountName "acctuser1" \-AccountPassword $Password \-Enabled $true \-Path "OU=Accounting,OU=Manila,DC=manila,DC=com"  
New-ADUser \-Name "GraphicsUser1" \-SamAccountName "graphics1" \-AccountPassword $Password \-Enabled $true \-Path "OU=Graphics,OU=Manila,DC=manila,DC=com"  
New-ADUser \-Name "CustomerUser1" \-SamAccountName "cust1" \-AccountPassword $Password \-Enabled $true \-Path "OU=Customer Service,OU=Manila,DC=manila,DC=com"  
New-ADUser \-Name "ExecutiveUser1" \-SamAccountName "exec1" \-AccountPassword $Password \-Enabled $true \-Path "OU=Executives,OU=Manila,DC=manila,DC=com"

## **Add Members**

Add-ADGroupMember "IT" C1,C2  
Add-ADGroupMember "VPN Users" C1,C2  
Add-ADGroupMember "WebUsers" C1,C2  
Add-ADGroupMember "Accounting Users" acctuser1  
Add-ADGroupMember "Graphics" graphics1  
Add-ADGroupMember "Customer Service" cust1  
Add-ADGroupMember "Executive" exec1  
---

# **Phase 14 — Take Second Snapshot**

After AD/DNS/users are ready:

Snapshot Name: WINSRV1\_AD\_DNS\_USERS\_READY

This is your first major competition baseline.

---

# **Phase 15 — Final Validation**

From WINSRV1:

ipconfig /all  
ping 192.168.2.254  
ping 192.168.99.254  
ping 8.8.8.8  
Resolve-DnsName google.com  
dcdiag /test:dns  
Get-ADUser \-Filter \* | Select Name,SamAccountName

From MikroTik:

/ping 192.168.2.10

From a LAN client or future CLIENT1:

ping 192.168.2.10  
nslookup manila.com 192.168.2.10  
nslookup google.com 192.168.2.10  
---

# **WINSRV1 Final State**

At the end, WINSRV1 should be:

VM Name:        WINSRV1  
OS:             Windows Server 2022 Standard Evaluation Desktop Experience  
Port Group:     SERVERS  
VLAN:           20  
IP:             192.168.2.10/24  
Gateway:        192.168.2.254  
DNS:            192.168.2.10  
Domain:         manila.com  
NetBIOS:        MANILA  
Roles:  
\- Active Directory Domain Services  
\- DNS Server  
\- Group Policy Management  
\- File Services

Next VM after this should be **CLIENT1 on VLAN 30 / LAN** so you can validate domain join, DNS, DHCP, GPO, and routing.

# Windows Server GUI Version

## **Phase 6 — Rename Server and Set Static IP**

### **1\. Rename the Server**

1. Open **Server Manager** (it usually opens automatically on login).  
2. On the left menu, click **Local Server**.  
3. Click on the current computer name (e.g., WIN-XXXXXXX) next to the **Computer name** field.  
4. In the *System Properties* window that pops up, click the **Change...** button.  
5. In the **Computer name:** field, type WINSRV1.  
6. Click **OK**, then **OK** again. When prompted to restart, select **Restart Later** so we can finish the network setup first, or restart now if you prefer clean transitions.

### **2\. Set Static IP & DNS**

1. Right-click the network icon in the taskbar system tray and select **Open Network & Internet settings** (or run ncpa.cpl to jump straight there).  
2. Click **Change adapter options**.  
3. Right-click your network adapter (usually named *Ethernet*) and select **Properties**.  
4. Double-click **Internet Protocol Version 4 (TCP/IPv4)**.  
5. Select **Use the following IP address:** and enter:  
   * **IP address:** 192.168.2.10  
   * **Subnet mask:** 255.255.255.0  
   * **Default gateway:** 192.168.2.254  
6. Select **Use the following DNS server addresses:** and enter:  
   * **Preferred DNS server:** 192.168.2.10 *(pointing to itself)*  
7. Click **OK**, then **OK** again to apply.  
8. *Now, perform the system restart if you didn't earlier.*

## **Phase 7 — Take First Snapshot**

1. Log into your **ESXi Web Client**.  
2. In the left navigator pane, click on **Virtual Machines**.  
3. Right-click your **WINSRV1** VM.  
4. Navigate to **Snapshots** \> **Take snapshot**.  
5. Give it the exact name: WINSRV1\_CLEAN\_INSTALLED\_STATIC\_IP.  
6. Click **Take snapshot**.

## **Phase 8 — Install Required Roles**

1. Open **Server Manager**, and click **Manage** (top right) \> **Add Roles and Features**.  
2. Click **Next** on the Before You Begin screen.  
3. Choose **Role-based or feature-based installation** and click **Next**.  
4. Ensure **WINSRV1** is selected from the server pool, then click **Next**.  
5. On the **Server Roles** checklist, check the boxes for:  
   * **Active Directory Domain Services** (Click *Add Features* on the popup window that appears).  
   * **DNS Server** (Click *Add Features* on the popup window).  
   * **File and Storage Services** (Expand this; **File Server** is usually checked by default).  
6. Click **Next**.  
7. On the **Features** page, scroll down and check **Group Policy Management** (it should already be auto-selected, but double-check). Click **Next**.  
8. Click **Next** through the AD DS and DNS informational screens.  
9. On the final confirmation screen, click **Install**. Once finished, click **Close**.

## **Phase 9 — Promote WINSRV1 to Domain Controller**

1. In **Server Manager**, look at the top right corner for a yellow warning triangle notification flag. Click it.  
2. Click **Promote this server to a domain controller**.  
3. In the Deployment Configuration deployment operation window:  
   * Select **Add a new forest**.  
   * **Root domain name:** Type manila.com. Click **Next**.  
4. In Domain Controller Options:  
   * Leave functional levels as default.  
   * Ensure **Domain Name System (DNS) server** and **Global Catalog (GC)** are checked.  
   * Type P@ssw0rd into both **Directory Services Restore Mode (DSRM)** password boxes. Click **Next**.  
5. Skip DNS Options by clicking **Next**.  
6. Let the NetBIOS name auto-fill as MANILA, then click **Next**.  
7. Leave Paths as default and click **Next**.  
8. Review options, click **Next**.  
9. The wizard will run a prerequisites check. When the green checkmark appears at the top, click **Install**.  
10. The server will automatically reboot. Log back in using the credentials **MANILA\\Administrator** / P@ssw0rd.

## **Phase 10 — Validate AD and DNS**

1. Click **Start** \> **Windows Administrative Tools**.  
2. Verify you can see and open **Active Directory Users and Computers** and **DNS**.  
3. Open a standard command prompt (cmd) and type nslookup manila.com to confirm it resolves back to 192.168.2.10.

## **Phase 11 — Add DNS Forwarder for Internet**

1. Open **Server Manager** \> **Tools** (top right) \> **DNS**.  
2. In the DNS Manager window, click on your server name **WINSRV1** to expand it.  
3. Right-click **WINSRV1** and select **Properties**.  
4. Navigate to the **Forwarders** tab.  
5. Click **Edit...**.  
6. Click where it says *"Click here to add an IP address or DNS name"*, and type 192.168.2.254 (the MikroTik gateway).  
7. Click **OK**, then click **Apply** and **OK**.

## **Phase 12 — Create Core DNS Records**

1. Still inside **DNS Manager**, expand **WINSRV1** \> **Forward Lookup Zones**.  
2. Click on the **manila.com** zone folder.  
3. Right-click an empty space in the right pane and select **New Host (A or AAAA)...**.  
4. Create your records using this map:

| Name | IP Address |
| :---- | :---- |
| winsrv1 | 192.168.2.10 |
| winsrv3 | 192.168.2.30 |
| winsrv4 | 192.168.2.50 |
| linsrv1 | 192.168.1.10 |
| w3 | 192.168.1.10 |

5.   
   Click **Add Host** after typing each entry, then click **Done** when finished.

## **Phase 13 — Create OUs, Groups, and Users**

### **1\. Create Organizational Units (OUs)**

1. Open **Server Manager** \> **Tools** \> **Active Directory Users and Computers**.  
2. Click to expand **manila.com**.  
3. Right-click **manila.com** \> **New** \> **Organizational Unit**.  
   * Name: Manila  
4. Now, right-click your newly created **Manila** OU folder \> **New** \> **Organizational Unit**. Create these nested sub-OUs:  
   * Accounting  
   * IT  
   * Graphics  
   * Customer Service  
   * Executives  
   * Servers  
   * Workstations

### **2\. Create Security Groups**

Right-click inside each specified OU \> **New** \> **Group**. (Keep Group Scope as *Global* and Group Type as *Security*).

* Inside **Manila** OU $\\rightarrow$ Create groups named: VPN Users and WebUsers  
* Inside **Accounting** OU $\\rightarrow$ Create group named: Accounting Users  
* Inside **IT** OU $\\rightarrow$ Create group named: IT  
* Inside **Graphics** OU $\\rightarrow$ Create group named: Graphics  
* Inside **Customer Service** OU $\\rightarrow$ Create group named: Customer Service  
* Inside **Executives** OU $\\rightarrow$ Create group named: Executive

### **3\. Create Users**

Right-click inside the target OU \> **New** \> **User**. Use P@ssw0rd for passwords, and uncheck "User must change password at next logon" while checking "Password never expires".

* Inside **IT** OU $\\rightarrow$ Create users: C1 and C2  
* Inside **Accounting** OU $\\rightarrow$ Create user: acctuser1  
* Inside **Graphics** OU $\\rightarrow$ Create user: graphics1  
* Inside **Customer Service** OU $\\rightarrow$ Create user: cust1  
* Inside **Executives** OU $\\rightarrow$ Create user: exec1

### **4\. Assign Memberships**

1. Double-click a group (e.g., the **IT** group in the IT OU).  
2. Go to the **Members** tab and click **Add...**.  
3. Type the user accounts comma-separated (e.g., C1, C2), click **Check Names**, then click **OK**.  
4. Repeat this process for all groups according to your map:  
   * Group **IT** $\\rightarrow$ add C1, C2  
   * Group **VPN Users** $\\rightarrow$ add C1, C2  
   * Group **WebUsers** $\\rightarrow$ add C1, C2  
   * Group **Accounting Users** $\\rightarrow$ add acctuser1  
   * Group **Graphics** $\\rightarrow$ add graphics1  
   * Group **Customer Service** $\\rightarrow$ add cust1  
   * Group **Executive** $\\rightarrow$ add exec1

## **Phase 14 — Take Second Snapshot**

1. Head back over to your **ESXi Web Client**.  
2. Right-click **WINSRV1** VM \> **Snapshots** \> **Take snapshot**.  
3. Name it: WINSRV1\_AD\_DNS\_USERS\_READY.  
4. Click **Take snapshot**.

## **Phase 15 — Final Validation**

To check your work visually without typing commands:

* Open **DNS Manager** and verify your zones and forwarders look clean.  
* Open **Active Directory Users and Computers**, navigate your tree layout, and double-click groups to verify their members are listed.  
* Open an elevated Command Prompt or PowerShell terminal just to verify end-to-end routing with basic test tools (ping 8.8.8.8 and nslookup google.com).

# MicroTik setup

### **How the Setup Looks Inside ESXi**

Inside your single VMware ESXi host, you will have multiple VMs plugged into virtual switches. Think of it like this:

\[ VMware ESXi Host \]  
 ├── VM 1: WINSRV1 (Windows Server \- 192.168.2.10)  
 └── VM 2: MikroTik CHR (The Router \- 192.168.2.254)

The MikroTik VM acts as the bridge. It connects to your physical network to get internet access, and it simultaneously connects to your internal **SERVERS** port group to provide that internet access (and routing) to `WINSRV1`.

### **High-Level Steps to Add the MikroTik Router**

To get this working, you need to deploy the MikroTik router as a VM using these steps:

#### **1\. Download the MikroTik Template**

You don't install MikroTik from an ISO like Windows. Instead, you download a pre-made virtual machine template from the official MikroTik download page:

* Look for the **Cloud Hosted Router (CHR)** section.  
* Download the **OVA template** (usually the stable version). The OVA file is a complete package that ESXi can deploy in seconds.

#### **2\. Deploy the Router VM in ESXi**

1. Log into your **ESXi Web Client**.  
2. Click **Create / Register VM**.  
3. Select **Deploy a virtual machine from an OVF or OVA file** and click **Next**.  
4. Name it (e.g., `MikroTik-Gateway`) and upload the `.ova` file you just downloaded.

#### **3\. Map the Network Interfaces (Crucial\!)**

During the deployment wizard, ESXi will ask you to map the router's virtual network adapters. A standard WorldSkills MikroTik router setup needs at least two network adapters:

* **Network Adapter 1 (WAN):** Map this to **VM Network** (or whatever port group connects directly to your physical home/school router so it can grab real internet).  
* **Network Adapter 2 (LAN/SERVERS):** Map this to your **SERVERS** port group. This is the side that talks to your Windows Server.

#### **4\. Configure the MikroTik IP**

Once the MikroTik VM is powered on, you log into it (via the ESXi console or an app called WinBox) and assign the IP address `192.168.2.254/24` to the interface connected to the **SERVERS** port group.

The moment you do that, your Windows Server's persistent pings will immediately start succeeding\!

---

Awesome\! If you can successfully ping 8.8.8.8 from your Windows Server, that means your network routing is completely fixed. The virtual cables are plugged in correctly, and your server has a clear path to the internet.  
So, why is nslookup google.com still timing out?  
Since basic internet connectivity (Layer 3\) is working, the problem is strictly a **DNS configuration issue (Layer 7\)**. Your Windows Server is sending a DNS request to the MikroTik (192.168.2.254), but the MikroTik is ignoring or dropping it.  
In MikroTik routers, there is a security feature that blocks local devices from using it as a DNS server by default. Here is how to fix it using the GUI.

### **Step 1: Enable Remote Requests on the MikroTik (WinBox)**

This is the most common reason for a DNS timeout in a MikroTik lab environment.

1. Open **WinBox** and log into your MikroTik router.  
2. On the left-hand menu, click on **IP** and select **DNS**.  
3. In the *DNS Settings* window:  
   * Look for the checkbox that says **Allow Remote Requests**.  
   * **Check this box.** (If this is unchecked, the MikroTik will ignore all DNS requests coming from your Windows Server).  
4. In the **Servers** field at the top, make sure an upstream DNS server is listed (like 8.8.8.8 or your actual home router's IP). If it's empty, click the downward arrow and add 8.8.8.8.  
5. Click **Apply** and **OK**.

### **Step 2: Clear the DNS Cache on Windows Server**

Now that the router is allowed to answer, you need to clear out the old "timeout" errors stuck in your Windows Server's memory.

1. On **WINSRV1**, open Command Prompt or PowerShell.  
2. Type the following command to flush the local DNS resolver cache:  
3. DOS

ipconfig /flushdns

4.   
5.   
6. Next, restart the Windows DNS server service so it attempts to connect to the forwarder fresh. Open **Server Manager** \> **Tools** \> **DNS**.  
7. Right-click your server name (**WINSRV1**), select **All Tasks**, and click **Restart**.

### **Step 3: Test it Again**

Go back to your Command Prompt on the Windows Server and run the test command:  
DOS  
nslookup google.com

**What you should expect to see now:**  
Instead of a timeout, the MikroTik should successfully handle the request. The output will look something like this:  
DOS  
Server:  UnKnown  
Address:  192.168.2.10

Non\-authoritative answer:  
Name:    google.com  
Addresses:  142.250.190.46

*(The "Server: UnKnown" text is still fine for now because we haven't built a reverse lookup zone—the important part is that it successfully returns Google's IP addresses at the bottom\!)*  
Did checking the "Allow Remote Requests" box on the MikroTik clear up the timeout error?

# Next steps after winsrv1

Based on the topology from [WSC2024\_TP54\_MA2\_actual\_en.pdf](https://www.google.com/search?q=file:///C:/wsc/Cyber%2520Security/4.%2520Test%2520Project/WSC2024_TP54_MA2_actual_en.pdf) and your network details from the [Google Docs Training Plan](https://docs.google.com/document/d/1rx8H1-KVoEEf_a0YoP_Zusg2b6PtUS1PEtEt_BmeHZo/edit?tab=t.i51iyj5km50g#heading=h.ovwl6o6re2ue), your **WINSRV1** is up and running, and your **MikroTik-Gateway** is powered on but needs its interface/routing rules configured.

Since the remaining virtual machines (**WINSRV3** and **LINSRV1**) are missing from your ESXi inventory list, you must first register/deploy them, configure the network infrastructure, and then harden the individual operating systems.

Here is the step-by-step process to configure the remaining instances in your ESXi environment:

---

## **Step 1: Deploy and Register the Missing VMs in ESXi**

Before configuring guest operating systems, you need to bring **WINSRV3** and **LINSRV1** into your ESXi environment.

1. Click **Virtual Machines** in the left Navigator panel.  
2. Select **Create / Register VM** from the top menu bar.  
3. Choose **Register an existing virtual machine** (or *Deploy a virtual machine from an OVF/OVA file* depending on your lab's provided templates).  
4. Browse your datastore to locate the directories for **WINSRV3** and **LINSRV1**, select their .vmx files, and finish the wizard.  
5. **Adjust Network Adaptors:** Right-click each newly added VM, select **Edit Settings**, and map their network adapters to the correct ESXi virtual switches/port groups matching their logical subnets:  
   * **WINSRV3:** Map to the **SERVERS** network port group.  
   * **LINSRV1:** Map to the **DMZ** network port group.

---

## **Step 2: Configure the MikroTik Gateway (Firewall)**

Your MikroTik instance is already running; it now needs to act as the router/firewall dividing your subnets. Access the MikroTik terminal console via ESXi or WinBox to set up interfaces and firewall rules:

### **1\. Assign IP Addresses to Interfaces**

Match the gateways outlined in your training documentation:

Code snippet

/ip address  
add address=192.168.2.254/24 interface=ether3 comment="SERVERS Network"  
add address=192.168.1.254/24 interface=ether4 comment="DMZ Network"  
add address=172.16.100.254/24 interface=ether2 comment="LAN Network"

*(Note: interface mappings like ether3 or ether4 should be verified against your explicit port assignments).*

### **2\. Configure LAN DHCP Relay**

Instead of running DHCP directly on the firewall, configure it to forward requests to **WINSRV1 (192.168.2.10)**:

Code snippet

/ip dhcp-relay  
add name=LAN-Relay interface=ether2 dhcp-server=192.168.2.10 local-address=172.16.100.254 disabled=no

### **3\. Establish Firewall Rules (Table 1 Security Policy)**

Drop all traffic by default and explicitly permit required traffic profiles:

Code snippet

\# Allow LAN to Internet  
/ip firewall filter add action=accept chain=forward in-interface=ether2 out-interface=ether1

\# Allow LAN to DMZ (SSH, HTTP, HTTPS, DNS to LINSRV1)  
/ip firewall filter add action=accept chain=forward protocol=tcp dst-address=192.168.1.10 dst-port=2022,80,443,53 in-interface=ether2

\# Allow LAN and DMZ to Servers (Active Directory Essential Ports)  
/ip firewall filter add action=accept chain=forward protocol=tcp dst-address=192.168.2.0/24 dst-port=53,88,135,389,445,464,636,3268,3269,9389  
/ip firewall filter add action=accept chain=forward protocol=udp dst-address=192.168.2.0/24 dst-port=53,88,137,138,389,464

\# Drop all other cross-subnet forward traffic  
/ip firewall filter add action=drop chain=forward

---

## **Step 3: Configure the 2-Tier PKI Subordinate CA (WINSRV3)**

Power on **WINSRV3** and configure it to handle your network's automated certificate lifecycle.

### **1\. Network & Domain Join**

1. Log in (Default credentials: Administrator / P@ssword).  
2. Assign static IP configurations:  
   * **IP Address:** 192.168.2.30  
   * **Subnet Mask:** 255.255.255.0  
   * **Gateway:** 192.168.2.254  
   * **Preferred DNS:** 192.168.2.10 *(pointing to WINSRV1)*  
3. Open System Properties and **Join the machine to the domain lumiere.com** (or manila.com based on your active AD space). Reboot the server.

### **2\. Complete Certificate Services Configuration**

1. Open **Server Manager**, click the notification flag, and select **Configure Active Directory Certificate Services**.  
2. On the Role Services setup, select **Enterprise Subordinate CA** (since your Offline Root CA, WINSRV4, is already completed).  
3. Generate a certificate request, sign it using your Root CA architecture if required, and finalize the installation so the Subordinate CA can issue valid certificates.

### **3\. Deploy the "certenroll" GPO on WINSRV1**

Switch back to **WINSRV1** to force domain clients to trust and auto-enroll with WINSRV3:

1. Open **Group Policy Management** (gpmc.msc).  
2. Create a new GPO named certenroll and link it to the root domain.  
3. Edit the GPO and navigate to: Computer Configuration \-\> Policies \-\> Windows Settings \-\> Security Settings \-\> Public Key Policies.  
4. Open **Certificate Services Client \- Auto-Enrollment**. Set Configuration Model to **Enabled**, and check both:  
   * *Renew expired certificates, update pending certificates, and remove revoked certificates.*  
   * *Update certificates that use certificate templates.*

---

## **Step 4: Harden the Linux Web Server (LINSRV1)**

Power on **LINSRV1** via your ESXi interface and open the VM console.

### **1\. Network Integration & Domain Join**

Verify the machine uses static IP 192.168.1.10, subnet mask 255.255.255.0, and points to gateway 192.168.1.254 and DNS 192.168.2.10.

Join the system to your Active Directory domain utilizing realmd or SSSD:

Bash

sudo realm join lumiere.com \-U Administrator

### **2\. Secure and Restrict SSH Access**

1. Edit the SSH server configuration script:  
2. Bash

sudo nano /etc/ssh/sshd\_config

3.   
4.   
5. Modify or add the following directive lines to isolate control access:  
6. Plaintext

Port 2022  
PermitRootLogin no  
AllowUsers C1 C2

7.   
8.   
9. Save the file and restart the daemon system:  
10. Bash

sudo systemctl restart sshd

11.   
12. 

### **3\. Grant Sudo Privileges to Authorized Accounts**

Ensure users C1 and C2 can execute root commands securely:

Bash

sudo visudo

Add the following configuration lines at the bottom of the file:

Plaintext

C1    ALL=(ALL)    ALL  
C2    ALL=(ALL)    ALL

### **4\. Configure Local Firewalld and SELinux**

1. Ensure SELinux is set to protecting status inside /etc/selinux/config:  
2. Plaintext

SELINUX=enforcing

3.   
4.   
5. Open ports for web traffic and your newly adjusted SSH listener:  
6. Bash

sudo firewall-cmd \--permanent \--add-service=http  
sudo firewall-cmd \--permanent \--add-service=https  
sudo firewall-cmd \--permanent \--add-port=2022/tcp  
sudo firewall-cmd \--reload

7.   
8. 

### **5\. Establish HTTPS via Apache (httpd)**

1. Bind Apache to host your target site resources located inside /var/www/lum.  
2. Request a web server certificate from your active **WINSRV3** issuing infrastructure.  
3. Configure your /etc/httpd/conf.d/ssl.conf file to point directly to your key pair (SSLCertificateFile and SSLCertificateKeyFile) to clear browser certificate warnings across internal LAN systems.

# ✅ Based on 2022

The 2022 structure is more **operations-platform driven**, with heavier emphasis on evidence submission, preconfigured environments, incident response, DFIR, PKI, IAM, Python code review, Red Team flags, and Blue Team traffic analysis.

The updated winning strategy should be:

> **Week 1: Build the PC-only training arena**  
> **Week 2: Enterprise infrastructure hardening**  
> **Week 3: Incident response, DFIR, crypto, IAM, and code review**  
> **Week 4: Red Team \+ Blue Team CTF \+ full mock competition**

The 2022 marking scheme still uses 25-mark competition blocks for infrastructure, incident response/forensics/application security, Red Team CTF, and Blue Team CTF, so the training must be score-balanced across all four areas.

---

# **Major 2022 Updates to Include**

## **1\. Infrastructure training must include AD, Linux hardening, VPN, Apache, ModSecurity, and SIEM**

The 2022 marking scheme includes infrastructure checks for:

* AD domain creation  
* DNS configuration  
* Computers joined to domain  
* OUs and users  
* GPOs  
* Account and audit policies  
* Splunk forwarder  
* Linux user creation  
* Sudo restrictions  
* firewalld zones and ports  
* SELinux enforcing mode  
* auditd rules  
* VPN setup  
* Firewall rules  
* Apache virtual hosts  
* HTTP-to-HTTPS redirect  
* ModSecurity blocking `/etc/passwd`  
* SIEM log validation

This is a shift from pure device configuration into **enterprise security validation**.

---

## **2\. Module B in 2022 is broader than 2019**

The 2022 Module B project includes:

* Vulnerability Detection and Repair  
* Incident Response  
* Identity and Access Management  
* Digital Forensic Investigation  
* Cryptography and PKI  
* Python Code Review

The answer sheet confirms the exact expected outputs: exploit type, exploit URL, commands, first attack time, infected file path, webshell code, reverse-shell command, attacker credentials, download URL, malicious autorun path, mutex string, registry key, process parameters, memory-dump findings, credit-card PCAP findings, TLS certificate evidence, IAM changes, and Python secure-code fixes.

---

## **3\. Red Team CTF is reconnaissance-first**

The 2022 Module C file states that the infrastructure layout is intentionally excluded because reconnaissance is part of the exercise. It also covers database, enumeration, privilege escalation, web-based attacks, and Windows-based attacks, with flags formatted as `icc{...}`.

Training implication: the student must practice **structured enumeration**, not random scanning.

---

## **4\. Blue Team CTF is traffic-analysis driven**

The 2022 Module D file states that flags are found by inspecting traffic through the inside and outside firewall interfaces, with benign and malicious traffic injected by a traffic generator. It also includes Splunk SIEM and firewall access, plus several PAT service mappings such as PHP/nginx, Apache, and Django services.

Training implication: the student must practice **Splunk, firewall logs, PCAP review, DLP detection, beaconing, malicious URL analysis, and payload extraction**.

---

# **PC-Only Simulation Recommendation**

Use **EVE-NG as the primary platform** if your PCs can handle it. EVE-NG officially supports many relevant images including Windows, Windows Server, Linux, Kali, pfSense, VyOS, Ubuntu, CentOS/RHEL, and Debian, which matches your WorldSkills-style mixed infrastructure requirement. ([EVE-NG](https://www.eve-ng.net/index.php/documentation/supported-images/?utm_source=chatgpt.com))

Use **GNS3** as an alternative if the team is already familiar with it. Official GNS3 documentation supports using the GNS3 VM with VMware or VirtualBox, with VMware documented as the faster option due to nested virtualization support. ([GNS3 Documentation](https://docs.gns3.com/docs/getting-started/installation/download-gns3-vm/?utm_source=chatgpt.com))

For the firewall, use **pfSense or OPNsense** instead of waiting for physical firewall hardware. Netgate documents pfSense virtualization across Proxmox, VirtualBox, KVM, Hyper-V, VMware, and other environments. ([Netgate Documentation](https://docs.netgate.com/pfsense/en/latest/virtualization/index.html?utm_source=chatgpt.com))

For SIEM training, Splunk Free is enough for lab use because Splunk documents the free license as standalone, non-expiring, and limited to 500 MB/day indexing. ([Splunk Docs](https://help.splunk.com/en/splunk-enterprise/administer/admin-manual/9.4/configure-splunk-licenses/about-splunk-free?utm_source=chatgpt.com))

---

# **Rebuilt 1-Month WorldSkills Training Process**

## **Training Goal**

Train the student to perform like a competitor, not just like a technician.

The student must be able to:

1. Read tasks fast.  
2. Identify scoring items.  
3. Configure only what earns marks.  
4. Validate every requirement.  
5. Capture screenshots.  
6. Save configurations.  
7. Submit clean answers.  
8. Recover from misconfiguration.  
9. Work under time pressure.

---

# **WEEK 1 — PC-Only Competition Lab Build**

## **Weekly Objective**

Build the complete simulation environment using PCs only.

This week is not about winning marks yet. It is about creating a stable training arena that can be reused for infrastructure, forensics, Red Team, and Blue Team practice.

---

## **Target Skills**

* Virtualization  
* EVE-NG or GNS3 setup  
* Network segmentation  
* Windows/Linux VM deployment  
* Firewall replacement using pfSense/OPNsense  
* Snapshot and rollback discipline  
* Evidence folder structure

---

## **Lab Components**

| Competition Function | Simulation Replacement |
| ----- | ----- |
| Cisco router | VyOS / Cisco IOSv / Linux router |
| ASA / Palo Alto / Fortinet firewall | pfSense / OPNsense / virtual firewall |
| Cisco switch | EVE-NG switch / Open vSwitch / Linux bridge |
| Windows Server | Windows Server VM |
| Windows client | Windows 10/11 VM |
| Linux servers | Ubuntu / Rocky / Alma / CentOS-compatible VM |
| Kali workstation | Kali VM |
| FlareVM | Windows analysis VM |
| Splunk SIEM | Splunk Free VM |
| IDS | Snort or Suricata VM |
| Web server | Apache/Nginx Linux VM |
| Vulnerable apps | DVWA / OWASP Juice Shop / WordPress lab |

---

## **Day 1 — Platform Setup**

### **Objective**

Install and validate the simulation platform.

### **Tasks**

* Install EVE-NG or GNS3.  
* Install VMware Workstation / VirtualBox / Proxmox.  
* Create folder structure:

WorldSkills-Training/  
├── Week1-Lab-Build/  
├── Week2-Infrastructure/  
├── Week3-IR-DFIR/  
├── Week4-CTF/  
├── Evidence/  
├── Screenshots/  
├── Config-Backups/  
└── Answer-Sheets/

* Create baseline VMs:  
  * Windows Server  
  * Windows Client  
  * Linux Server  
  * Kali  
  * FlareVM or Windows analysis VM  
  * Splunk  
  * pfSense/OPNsense

### **Deliverable**

* Screenshot of all VMs created.  
* Snapshot named `DAY1_CLEAN_BASELINE`.  
* Inventory sheet with hostname, IP, role, username, password.

---

## **Day 2 — Network Segmentation**

### **Objective**

Build the base network that will support both 2019 and 2022 tasks.

### **Tasks**

Create these virtual networks:

| Segment | Purpose |
| ----- | ----- |
| Management | Admin access |
| Inside LAN | AD, users, internal systems |
| DMZ | Web, FTP, IDS |
| Outside | Internet/attacker simulation |
| Forensics | Isolated analysis |
| CTF Red | Attack lab |
| CTF Blue | Monitoring lab |

### **Deliverable**

* Network topology screenshot.  
* IP addressing plan.  
* Routing test results.  
* Snapshot named `DAY2_NETWORK_BASE`.

---

## **Day 3 — Firewall and Routing**

### **Objective**

Create a working firewall and routing backbone.

### **Tasks**

* Deploy pfSense/OPNsense.  
* Configure interfaces:  
  * WAN / Outside  
  * LAN / Inside  
  * DMZ  
* Configure NAT.  
* Configure baseline firewall rules.  
* Confirm:  
  * Inside can reach DMZ.  
  * Inside can reach Outside.  
  * DMZ cannot freely access Inside.  
  * Outside cannot access Inside unless explicitly allowed.

### **Deliverable**

* Firewall interface screenshot.  
* Firewall rule screenshot.  
* Ping/connectivity matrix.  
* Snapshot named `DAY3_FIREWALL_BASE`.

---

## **Day 4 — Core Servers**

### **Objective**

Prepare reusable enterprise services.

### **Tasks**

* Deploy Windows Server.  
* Install AD DS role, but do not overconfigure yet.  
* Deploy Linux Web Server.  
* Deploy Linux SIEM/logging server.  
* Deploy Linux IDS server.  
* Deploy Windows analysis VM.  
* Deploy Kali.

### **Deliverable**

* VM boot validation.  
* Remote access validation.  
* Hostname/IP table.  
* Snapshot named `DAY4_SERVER_BASE`.

---

## **Day 5 — Evidence Workflow and Baseline Mock**

### **Objective**

Train the student to behave like a competitor.

### **Tasks**

* Create screenshot naming rules:

W2D6\_AD\_DomainCreated\_01.png  
W3D11\_WebShell\_Path\_02.png  
W4D19\_Splunk\_MaliciousURL\_03.png

* Create config backup rules:

R1\_showrun\_Day5.txt  
pfsense\_rules\_Day5.pdf  
apache\_ssl\_config\_Day8.conf  
splunk\_searches\_Day10.txt

* Reboot all VMs.  
* Confirm everything still works.  
* Restore from snapshot once as a drill.

### **Deliverable**

* Week 1 baseline report.  
* Working simulation environment.  
* Snapshot named `WEEK1_COMPETITION_BASELINE`.

---

# **WEEK 2 — Enterprise Infrastructure Security**

## **Weekly Objective**

Train on the infrastructure-hardening tasks common to 2019 and 2022\.

This week is the highest-value technical foundation. It should cover 2019 Module A and the 2022 infrastructure marking scheme.

The 2019 Module A actual project breaks infrastructure into logon/password policies, network equipment hardening, public services protection, events monitoring, and firewall policy.

---

## **Week 2 Target Deliverables**

By the end of Week 2, the student must produce:

* Domain joined client  
* AD users and OUs  
* GPOs  
* Account policy  
* Audit policy  
* Linux hardening  
* firewalld rules  
* SELinux enforcing  
* auditd rule  
* Apache HTTPS virtual hosts  
* HTTP-to-HTTPS redirect  
* ModSecurity rule  
* Splunk log ingestion  
* Firewall rules  
* VPN simulation  
* Evidence PDF

---

## **Day 6 — Active Directory, DNS, OU, GPO**

### **Objective**

Train the student to complete Windows infrastructure scoring items fast.

### **Tasks**

* Create domain.  
* Configure DNS.  
* Join Windows client to domain.  
* Create OUs.  
* Create users.  
* Apply GPOs:  
  * Password complexity  
  * Minimum password length  
  * Account lockout  
  * Login banner  
  * Audit policy  
  * RDP security  
* Validate with login tests.

### **Deliverable**

* Domain creation screenshot.  
* DNS lookup screenshot.  
* Domain-joined computer screenshot.  
* GPO result screenshot.  
* Account lockout test screenshot.

---

## **Day 7 — Linux Hardening**

### **Objective**

Train Linux security policy and host hardening.

### **Tasks**

* Create required users.  
* Configure password policy.  
* Configure sudo restrictions.  
* Configure firewalld active zone.  
* Configure persistent firewall ports.  
* Enable SELinux enforcing.  
* Relabel filesystem if needed.  
* Add auditd rule.  
* Disable root SSH where required.  
* Configure login banner.

### **Deliverable**

* `/etc/passwd` evidence.  
* `/etc/shadow` evidence where safe for mentor review only.  
* `sudo -l` screenshots.  
* `firewall-cmd --list-all` screenshot.  
* `sestatus` screenshot.  
* `auditctl -l` screenshot.  
* SSH root-deny test screenshot.

---

## **Day 8 — Web, TLS, PKI, ModSecurity**

### **Objective**

Train public-service protection.

### **Tasks**

* Install Apache/Nginx.  
* Create virtual hosts:  
  * Primary site  
  * Secondary test site  
* Create self-signed or internal CA certificate.  
* Configure HTTPS.  
* Redirect HTTP to HTTPS.  
* Configure ModSecurity.  
* Add detection/blocking rule for `/etc/passwd`.  
* Validate browser access.

The 2022 marking scheme specifically checks web access to `www.shenghai.org`, `www.testshenghai.com`, HTTP redirect to HTTPS, and ModSecurity blocking `/etc/passwd`.

### **Deliverable**

* HTTPS browser screenshot.  
* HTTP redirect screenshot.  
* Certificate screenshot.  
* ModSecurity block screenshot.  
* Apache config backup.

---

## **Day 9 — SIEM, Logs, and IDS**

### **Objective**

Train high-value monitoring tasks.

### **Tasks**

* Install Splunk.  
* Install Splunk Universal Forwarder or simulate log forwarding.  
* Forward Windows security logs.  
* Forward Linux logs.  
* Configure Snort/Suricata.  
* Generate test events:  
  * Failed login  
  * Successful login  
  * FTP traffic  
  * ICMP traffic  
  * Payload containing `malware`  
* Search logs in Splunk.

The 2019 Module A actual requires Domain Controller security logs to be sent to Splunk, Splunk to receive logs on a specified port, audit policies for Windows events, and IDS alerts for FTP, ICMP, and `malware` payload traffic.

### **Deliverable**

* Splunk receiving logs.  
* Search query evidence.  
* IDS alert evidence.  
* Test traffic evidence.

---

## **Day 10 — Firewall Policy, VPN, and Reboot Validation**

### **Objective**

Finish Week 2 by validating infrastructure like judges would.

### **Tasks**

* Configure least-privilege firewall rules.  
* Configure simulated site-to-site VPN or remote access VPN.  
* Validate:  
  * AD login  
  * DNS  
  * HTTPS  
  * ModSecurity  
  * Splunk logs  
  * IDS alerts  
  * Firewall blocking  
  * VPN connectivity  
* Reboot all systems.  
* Retest.

### **Deliverable**

* Full Week 2 scoring checklist.  
* Evidence PDF.  
* Snapshot named `WEEK2_INFRA_READY`.

---

# **WEEK 3 — Incident Response, DFIR, Crypto, IAM, Code Review**

## **Weekly Objective**

Train the student for 2022 Module B and 2019 Module B.

The 2022 Module B tasks include web incident analysis, Windows server malware analysis, vulnerability repair, memory dump analysis, credit-card PCAP analysis, cryptography, IAM, and Python code review.

---

## **Week 3 Target Deliverables**

By the end of Week 3, the student must produce:

* Web incident report  
* Windows malware report  
* Vulnerability repair evidence  
* Memory dump answer sheet  
* PCAP evidence  
* TLS certificate evidence  
* IAM evidence  
* Python code review answers  
* Completed answer sheet

---

## **Day 11 — Web Server Incident Response**

### **Objective**

Identify web compromise artifacts.

### **Tasks**

* Review web logs.  
* Review WordPress files.  
* Search for webshells.  
* Identify exploit URL.  
* Identify commands and parameters.  
* Identify first successful attack time.  
* Identify infected file absolute path.  
* Extract webshell code.  
* Identify first reverse-shell command.  
* Identify attacker credentials.  
* Identify download URL.

### **Deliverable**

Completed Web Server IR answer section.

---

## **Day 12 — Windows Server Malware Investigation**

### **Objective**

Train autorun, registry, process, and malware analysis basics.

### **Tasks**

* Inspect startup folders.  
* Inspect registry autorun keys.  
* Inspect scheduled tasks.  
* Inspect services.  
* Identify malicious autorun path.  
* Identify mutex string.  
* Identify registry key written by malware.  
* Identify file created by malware.  
* Identify process name and parameters.  
* Identify registry API functions used.

### **Deliverable**

Completed Windows Server IR answer section.

---

## **Day 13 — Vulnerability Detection and Repair \+ IAM**

### **Objective**

Repair the vulnerable web server and lock down identity access.

### **Tasks**

* Fix discovered vulnerable code.  
* Provide before/after test evidence.  
* Disable root SSH login.  
* Change root password.  
* Allow approved user to sudo to root using their own password.  
* Reboot server.  
* Prove root SSH is blocked.

The 2022 Module B answer sheet requires before/after bug-fix evidence, root SSH prevention, root password change documentation, sudo configuration for the `ixia` user, reboot validation, and screenshots proving root SSH failure.

### **Deliverable**

* Vulnerability repair table.  
* IAM evidence.  
* Before/after screenshots.

---

## **Day 14 — Memory Dump and PCAP Analysis**

### **Objective**

Train DFIR speed.

### **Tasks**

For `dump.vmem` practice:

* Identify malicious process name.  
* Identify PID and PPID.  
* Identify attempted connection IPs and destination port.  
* Identify user who executed malware.  
* Identify related malware that must also be deleted.

For `creditcard.pcap` practice:

* Extract three credit card details.  
* Identify protocol used.  
* Identify exfiltration path.  
* Document filters used.

The 2022 Module B practical requires memory-dump answers with commands and relevant output, plus extraction of three stolen credit-card details from PCAP files.

### **Deliverable**

* Memory dump worksheet.  
* PCAP worksheet.  
* Tool command log.

---

## **Day 15 — Cryptography and Python Code Review**

### **Objective**

Train fast completion of crypto, PKI, and secure coding tasks.

### **Tasks**

Crypto:

* Generate 4096-bit RSA self-signed certificate.  
* Configure web server for HTTP and HTTPS.  
* Redirect HTTP to HTTPS.  
* Save key and certificate as evidence.

Code review:

* Identify unsafe line.  
* Explain why unsafe.  
* Explain secure approach.  
* Provide corrected line only.  
* Practice Python risks:  
  * command injection  
  * unsafe deserialization  
  * SQL injection  
  * path traversal  
  * weak randomness  
  * hardcoded secrets

The 2022 Module B requires a self-signed 4K RSA certificate, SNI behavior for a URI including the IP address, HTTP and HTTPS configuration, HTTP-to-HTTPS redirect, and Python code review answers.

### **Deliverable**

* Certificate evidence.  
* Web config evidence.  
* Python code review answer sheet.  
* Snapshot named `WEEK3_IR_DFIR_READY`.

---

# **WEEK 4 — Red Team, Blue Team, and Final Mock Competition**

## **Weekly Objective**

Convert skills into competition speed.

2019 and 2022 both treat Red/Blue CTF as major scoring areas. In 2019, Module C covers enumeration, web attacks, database attacks, Windows attacks, root access, cryptography, and steganography.  
2019 Module D covers reconnaissance/application detection, malicious URL, exploits/malware, botnet, data leakage, reverse engineering, and forensics.

---

## **Week 4 Target Deliverables**

* Red Team enumeration checklist  
* Blue Team Splunk checklist  
* Flag tracking sheet  
* PCAP analysis worksheet  
* Mock competition score  
* Final improvement plan

---

## **Day 16 — Red Team Enumeration**

### **Objective**

Train structured recon.

### **Tasks**

* Discover live hosts.  
* Identify open ports.  
* Identify services.  
* Identify web directories.  
* Identify technologies.  
* Build target notes.  
* Avoid management network ranges.  
* Log every command and result.

### **Deliverable**

* Enumeration matrix.  
* Service list.  
* Attack-path hypothesis.

---

## **Day 17 — Red Team Web, Database, Windows**

### **Objective**

Train common CTF attack categories in an isolated lab.

### **Tasks**

* Web testing:  
  * authentication issues  
  * file upload  
  * directory traversal  
  * input validation  
* Database testing:  
  * basic SQL injection detection in legal lab only  
  * weak credentials  
  * exposed backups  
* Windows testing:  
  * SMB enumeration  
  * weak shares  
  * credential reuse  
  * local privilege review

### **Deliverable**

* Flags captured.  
* Exploitation notes.  
* Remediation notes.

---

## **Day 18 — Crypto, Stego, Privilege Escalation**

### **Objective**

Train tie-breaker categories.

### **Tasks**

* Decode common encodings.  
* Identify hashes.  
* Crack only provided lab hashes.  
* Inspect metadata.  
* Use strings/binwalk/file.  
* Search hidden files.  
* Linux privilege escalation checklist.  
* Windows privilege escalation checklist.

### **Deliverable**

* Crypto/stego checklist.  
* Privilege escalation checklist.  
* Flag log.

---

## **Day 19 — Blue Team Traffic and SIEM**

### **Objective**

Train SOC-style flag finding.

### **Tasks**

* Analyze firewall logs.  
* Analyze Splunk logs.  
* Identify:  
  * reconnaissance  
  * malicious URL  
  * exploit attempt  
  * drive-by malware  
  * botnet traffic  
  * data leakage  
  * suspicious file transfer  
* Use PCAP filters.  
* Extract suspicious indicators.  
* Decode obfuscated flags.

The 2022 Module D file specifically says participants find flags by inspecting traffic flowing through firewall interfaces, and the lab includes Splunk SIEM, firewall access, and mapped services behind PAT.

### **Deliverable**

* SOC triage report.  
* Splunk search list.  
* PCAP evidence.  
* Blue Team flag log.

---

## **Day 20 — Full Mock Competition**

### **Objective**

Run the student under pressure.

### **Morning: Infrastructure Mock**

* AD/DNS/GPO  
* Linux hardening  
* Web TLS  
* ModSecurity  
* Firewall rules  
* Splunk logging  
* VPN validation

### **Midday: Incident Response Mock**

* Web compromise  
* Windows autorun malware  
* Memory dump  
* PCAP  
* IAM  
* Crypto  
* Code review

### **Afternoon: CTF Mini-Round**

* 90 minutes Red Team  
* 90 minutes Blue Team

### **Deliverable**

* Final scorecard.  
* Screenshot package.  
* Answer sheet package.  
* Mentor review.

---

# **Updated Weekly Objectives**

## **Week 1 Objective — Build the Arena**

**Success criteria:**

* EVE-NG/GNS3 working.  
* Firewall VM working.  
* Windows/Linux/Kali/Splunk VMs working.  
* Snapshots created.  
* Evidence workflow established.

---

## **Week 2 Objective — Infrastructure Score**

**Success criteria:**

* Domain created.  
* DNS works.  
* GPO applies.  
* Linux users and sudo rules work.  
* firewalld, SELinux, auditd configured.  
* Apache HTTPS and redirect work.  
* ModSecurity blocks malicious path.  
* Splunk receives logs.  
* Firewall rules are minimal.  
* VPN simulation works.

---

## **Week 3 Objective — IR/DFIR/Application Security Score**

**Success criteria:**

* Student can complete answer sheet fields accurately.  
* Student can identify paths, timestamps, hashes, process names, registry keys, and exploit URLs.  
* Student can repair vulnerabilities and prove before/after state.  
* Student can analyze memory and PCAP artifacts.  
* Student can create TLS certificates.  
* Student can complete Python code review answers.

---

## **Week 4 Objective — Speed and Competition Execution**

**Success criteria:**

* Student can enumerate without guidance.  
* Student can find flags across multiple categories.  
* Student can use Splunk and PCAP filters quickly.  
* Student avoids invalid targets and unsafe activity.  
* Student submits clean evidence.  
* Student can recover from broken services.

---

# **Practical Team Setup With Only PCs**

## **Minimum Setup**

For one student:

| Item | Minimum |
| ----- | ----- |
| CPU | 6 cores |
| RAM | 32 GB |
| Storage | 500 GB SSD |
| Network | Isolated virtual networks |
| Hypervisor | VMware / Proxmox / VirtualBox |
| Simulator | EVE-NG or GNS3 |

## **Low-Spec Setup**

For 16 GB RAM PCs, split the labs:

| Lab | Run Only These |
| ----- | ----- |
| Infrastructure Lab | AD \+ Linux \+ Firewall \+ Splunk |
| IR Lab | Web Server \+ Windows Server \+ Analyst VM |
| Red Lab | Kali \+ 2 vulnerable targets |
| Blue Lab | Splunk \+ firewall logs \+ PCAP files |

Do not force the full topology on weak hardware. It will slow training and create operational noise.

---

# **Winning Operating Rhythm**

The student must follow this process every day:

1. Read all tasks first.  
2. Highlight scoring verbs: create, configure, submit, prove, screenshot, explain.  
3. Configure the highest-mark items first.  
4. Validate immediately.  
5. Screenshot immediately.  
6. Save configuration.  
7. Reboot-test when required.  
8. Write answer-sheet entries in complete form.  
9. Backup evidence.  
10. Reset lab using snapshot.

# Week 1

The goal of Week 1 is not yet to complete hardening. The goal is to create a **stable, reusable, snapshot-based competition lab** that can support:

* Infrastructure setup and hardening  
* Incident response  
* Digital forensics  
* Red Team CTF  
* Blue Team monitoring  
* Evidence submission practice

WorldSkills-style tasks require the competitor to confirm devices are working, recover locked or inaccessible devices, test all functionality before submission, and save evidence/configurations properly. That operating discipline starts in Week 1\.

---

# **WEEK 1 LAB BUILD PLAN**

## **Theme: Build the Competition Arena**

## **Weekly Objective**

By the end of Week 1, the student must have a working simulated cyber range using only PCs.

The lab must support the 2019-style infrastructure topology, which includes Windows Server, CentOS/Linux servers, Windows clients, router, firewall, switch, DMZ, inside network, outside network, Splunk, Snort/IDS, Apache, FTP, DNS, and VPN-related components.

The lab must also support the 2022-style workflow, where competitors work from analyst machines, investigate a compromised web server and Windows server, submit evidence, perform memory/PCAP analysis, configure TLS/PKI, perform IAM hardening, and review Python code.

---

# **Week 1 Success Criteria**

The student passes Week 1 only if the following are complete:

| Area | Required Output |
| ----- | ----- |
| Simulation platform | EVE-NG or GNS3 working |
| Network topology | Inside, DMZ, Outside, Management, IR, Red, Blue networks created |
| Firewall | pfSense/OPNsense or equivalent firewall deployed |
| Router | VyOS/Linux router or Cisco image if legally available |
| Core servers | Windows Server, Linux Web, Linux SIEM, Linux IDS, Kali, Analyst VM deployed |
| Connectivity | Ping and service tests completed |
| Evidence workflow | Screenshot, config backup, and daily report folders created |
| Snapshots | Clean rollback points created |
| Student discipline | Can explain topology without reading notes |

---

# **Tooling Decision**

Use **one primary simulator**, not both, unless you have extra time.

## **Recommended Option: EVE-NG**

EVE-NG is preferred for this program because it is cleaner for multi-device cyber range layouts and supports many image types used in mixed security labs. Its supported-image documentation includes Cisco/network images and QEMU-based images, making it appropriate for routers, firewalls, Linux, Windows, Kali-style systems, and security appliances. ([EVE-NG](https://www.eve-ng.net/index.php/documentation/supported-images/?utm_source=chatgpt.com))

## **Alternative Option: GNS3**

Use GNS3 if the students are already familiar with it. GNS3 officially supports using a GNS3 VM locally on VMware, VirtualBox, or Hyper-V; GNS3 documentation also notes VMware is generally faster because of nested virtualization acceleration. ([GNS3 Documentation](https://docs.gns3.com/docs/?utm_source=chatgpt.com))

## **Firewall Replacement**

Use **pfSense** or **OPNsense** as the firewall replacement. pfSense documentation confirms it can run in major virtual environments such as Proxmox, VirtualBox, KVM, Hyper-V, VMware, and others. ([Netgate Documentation](https://docs.netgate.com/pfsense/en/latest/virtualization/index.html?utm_source=chatgpt.com))

---

# **Week 1 Lab Architecture**

## **Core Topology**

                        OUTSIDE / ATTACK SIMULATION  
                              130.18.30.0/24  
                                      |  
                                  R1 / VyOS  
                                      |  
                         TRANSIT-OUTSIDE 209.165.200.232/30  
                                      |  
                              CENTOS-04 / TRANSIT  
                                      |  
                         TRANSIT-FW 209.165.200.224/30  
                                      |  
                              FW-EDGE / pfSense  
                       /                              \\  
          INSIDE LAN 192.168.10.0/24          DMZ 192.168.2.0/24  
          AD, Client, SIEM                     Web, FTP, IDS, RADIUS

## **Additional Training Networks**

MGMT-NET        100.100.128.0/24     Admin/RDP/SSH simulation  
IR-NET          192.168.1.0/24       2019/2022 incident-response lab  
RED-CTF-NET     10.10.10.0/24        Red Team target lab  
BLUE-MON-NET    172.16.10.0/24       Splunk/traffic-analysis lab  
FORENSICS-NET   10.50.50.0/24        Offline malware/forensic analysis

The 2022 Module C Red Team task intentionally excludes infrastructure layout because reconnaissance is part of the exercise, and it states there are no flags on the `100.100.0.0/16` management segment. That management segment must be treated as **off-limits** in training.

The 2022 Module D Blue Team task focuses on inspecting traffic through inside and outside firewall interfaces, with Splunk SIEM and firewall access used for traffic-based flag discovery. Week 1 must therefore prepare a firewall \+ SIEM \+ packet-capture workflow.

---

# **Standard Student Rules for Week 1**

These rules are mandatory.

## **Operational Rules**

1. **Do not scan or attack anything outside the lab.**  
2. **Do not scan the management network.**  
3. **Do not run denial-of-service tools.**  
4. **Do not modify simulator host settings unless instructed.**  
5. **Do not delete snapshots.**  
6. **Every change must have evidence.**  
7. **Every daily output must be reproducible after reboot.**  
8. **Save configurations before every major change.**  
9. **Use the assigned hostnames and IP plan.**  
10. **Log commands, errors, and fixes.**

The WorldSkills documents repeatedly emphasize testing functionality, saving configurations, and ensuring systems remain accessible for marking. Treat this as competition muscle memory.

---

# **Required Folder Structure**

Create this on the training PC or shared drive:

WorldSkills-Cyber-Training/  
├── 00-Admin/  
│   ├── PC-Inventory.xlsx  
│   ├── Credentials-Lab-Only.xlsx  
│   └── Training-Rules.pdf  
├── 01-Week1-Lab-Build/  
│   ├── Day1-Platform/  
│   ├── Day2-Networks/  
│   ├── Day3-Firewall-Routing/  
│   ├── Day4-VMs-Services/  
│   └── Day5-Validation/  
├── 02-Week2-Infrastructure/  
├── 03-Week3-IR-DFIR-AppSec/  
├── 04-Week4-CTF/  
├── Evidence/  
│   ├── Screenshots/  
│   ├── Config-Backups/  
│   ├── Command-Logs/  
│   ├── Reports/  
│   └── PCAPs/  
└── Snapshots/

## **Screenshot Naming Standard**

W1D1\_EVE\_Login\_01.png  
W1D2\_Topology\_Full\_01.png  
W1D3\_Firewall\_Interfaces\_01.png  
W1D4\_ADSVR\_IPConfig\_01.png  
W1D5\_PingMatrix\_01.png

## **Config Backup Naming Standard**

W1D3\_R1\_VyOS\_Config.txt  
W1D3\_FWEDGE\_Rules.pdf  
W1D4\_WEB01\_IPConfig.txt  
W1D5\_Final\_PingMatrix.xlsx

---

# **DAY 1 — Platform Preparation**

## **Daily Objective**

Install and validate the virtualization/simulation platform.

## **Student Outcome**

By the end of Day 1, the student must be able to start the simulator, open the lab workspace, boot at least one virtual node, and take a clean baseline snapshot.

---

## **Day 1 Step-by-Step Instructions**

### **Step 1 — Verify PC Readiness**

Each student PC must be checked.

Minimum usable specification:

| Component | Minimum | Recommended |
| ----- | ----- | ----- |
| CPU | 4 cores | 8+ cores |
| RAM | 16 GB | 32–64 GB |
| Storage | 250 GB SSD | 500 GB–1 TB SSD |
| Virtualization | Required | Required |
| Network | Isolated lab only | Isolated lab only |

Student must verify:

BIOS virtualization enabled  
Enough free disk space  
Host OS stable  
No corporate production network bridged into lab  
Antivirus exclusions approved for lab VM folder, if required

---

### **Step 2 — Select Platform**

Choose one:

| Scenario | Platform |
| ----- | ----- |
| Strong PC / dedicated lab server | EVE-NG |
| Students already know GNS3 | GNS3 |
| Weak PCs | VMware/VirtualBox-only segmented lab |
| Multiple students sharing one large server | EVE-NG on Proxmox/VMware |

---

### **Step 3A — EVE-NG Setup Path**

Use this when EVE-NG is the primary platform.

1. Install EVE-NG VM on VMware.  
2. Assign resources:  
   * CPU: 4–8 vCPU  
   * RAM: 16–32 GB  
   * Disk: 150 GB minimum  
3. Configure EVE management IP.  
4. Log in to EVE web UI.  
5. Create lab folder:

WorldSkills-Week1-Foundation

6. Upload/import only legally obtained images:  
   * pfSense/OPNsense  
   * VyOS  
   * Ubuntu/Rocky/Alma/CentOS-compatible Linux  
   * Kali  
   * Windows Server  
   * Windows Client  
   * Splunk VM or Linux VM for Splunk  
7. Create test node and confirm it boots.

---

### **Step 3B — GNS3 Setup Path**

Use this when GNS3 is the primary platform.

1. Install GNS3 client.  
2. Import GNS3 VM into VMware/VirtualBox.  
3. Connect GNS3 client to GNS3 VM.  
4. Create new project:

WorldSkills-Week1-Foundation

5. Add one Ethernet switch.  
6. Add one Linux router or VyOS node.  
7. Start the node.  
8. Confirm console access works.  
9. Add VMware VMs to GNS3 topology as needed.

---

### **Step 4 — Create Image Inventory**

Create a table:

| Image | Version | Role | Source | License Status | Imported |
| ----- | ----- | ----- | ----- | ----- | ----- |
| pfSense/OPNsense |  | Firewall |  |  |  |
| VyOS |  | Router |  |  |  |
| Windows Server |  | AD/DNS/NPS |  |  |  |
| Windows Client |  | Competitor/Client |  |  |  |
| Linux Server |  | Web/SIEM/IDS |  |  |  |
| Kali |  | Red Team |  |  |  |
| FlareVM/Windows Tools VM |  | Forensics |  |  |  |

Do not use pirated Cisco, Palo Alto, Fortinet, or Windows images. Use legal eval images, open-source substitutes, or licensed academic/vendor images.

---

### **Step 5 — Create Evidence Baseline**

Student must create:

Evidence/Screenshots/W1D1/  
Evidence/Command-Logs/W1D1/  
Evidence/Config-Backups/W1D1/  
Evidence/Reports/W1D1/

Create the first command log:

W1D1\_CommandLog.txt

Required format:

Time:  
Device:  
Command:  
Purpose:  
Result:  
Screenshot file:  
Issue encountered:  
Fix applied:

---

### **Step 6 — Create First Snapshot**

Snapshot name:

W1D1\_CLEAN\_PLATFORM\_BASELINE

For EVE-NG:

* Backup lab file.  
* Export node configs if available.  
* Snapshot VM in VMware/Proxmox.

For GNS3:

* Save project.  
* Snapshot GNS3 VM.  
* Snapshot external VMware/VirtualBox VMs.

---

## **Day 1 Deliverables**

Student must submit:

| Deliverable | Required Evidence |
| ----- | ----- |
| Simulator installed | Screenshot of EVE/GNS3 dashboard |
| VM inventory | Completed image inventory table |
| Test node booted | Screenshot of console login |
| Folder structure created | Screenshot of folder tree |
| Baseline snapshot | Screenshot of snapshot list |
| Command log | `W1D1_CommandLog.txt` |

---

## **Day 1 Mentor Check**

Mentor asks the student:

1. What simulator are you using and why?  
2. Where are your snapshots stored?  
3. What networks will be isolated?  
4. What images are legal and ready?  
5. How will you recover if the lab breaks?

---

# **DAY 2 — Network Design and Topology Creation**

## **Daily Objective**

Create the WorldSkills-style network skeleton.

## **Student Outcome**

By end of Day 2, the student must have all network segments created and all placeholder devices connected.

---

# **Day 2 IP Plan**

## **2019 Infrastructure-Style Network**

| Segment | Subnet | Gateway |
| ----- | ----- | ----- |
| Inside | `192.168.10.0/24` | `192.168.10.1` |
| DMZ | `192.168.2.0/24` | `192.168.2.1` |
| Outside Client | `130.18.30.0/24` | `130.18.30.1` |
| Firewall Transit | `209.165.200.224/30` | FW: `.226`, Transit: `.225` |
| Router Transit | `209.165.200.232/30` | R1: `.234`, Transit: `.233` |

These values match the 2019-style Module A network plan for ASA outside/inside/DMZ, R1, CentOS-04, and outside host placement.

## **2022 Management / IR-Style Network**

| Segment | Subnet | Purpose |
| ----- | ----- | ----- |
| Management | `100.100.128.0/24` | Simulated RDP/SSH/admin access |
| IR Lab | `192.168.1.0/24` | Compromised web/file/forensics lab |
| Red CTF | `10.10.10.0/24` | Red Team target network |
| Blue Monitoring | `172.16.10.0/24` | SIEM and monitoring |

The 2022 Module B environment uses management IPs for competitor systems, compromised web server, and compromised Windows server, including `100.100.128.50`, `.40`, `.60`, and `.70`.

---

## **Day 2 Step-by-Step Instructions**

### **Step 1 — Create Lab Project**

Project name:

WSC-Week1-Foundation-Lab

Description:

WorldSkills Cyber Security PC-only training topology for Infrastructure, IR, Red Team, and Blue Team modules.

---

### **Step 2 — Create Network Objects**

In EVE-NG or GNS3, create these networks/switches:

MGMT-SW  
INSIDE-SW  
DMZ-SW  
OUTSIDE-SW  
TRANSIT-FW-SW  
TRANSIT-R1-SW  
IR-SW  
RED-CTF-SW  
BLUE-MON-SW  
FORENSICS-SW

Use simple Ethernet switches first. Do not overcomplicate switching on Day 2\.

---

### **Step 3 — Add Placeholder Nodes**

Add these nodes even if not fully configured yet:

| Node Name | Role | Network |
| ----- | ----- | ----- |
| R1 | Outside router | Outside / Transit-R1 |
| CENTOS-04 or TRANSIT-RTR | Transit router | Transit-R1 / Transit-FW |
| FW-EDGE | Firewall | Transit-FW / Inside / DMZ |
| SW-1 | Inside switch placeholder | Inside |
| SW-2 | DMZ switch placeholder | DMZ |
| AD-SVR | Windows Server | Inside |
| WIN-CLI | Windows Client | Inside |
| SIEM01 | Splunk server | Inside or Blue |
| WEB-01 | Apache/WordPress server | DMZ |
| FTP-IDS01 | FTP/RADIUS/IDS | DMZ |
| KALI01 | Red Team workstation | Red / Outside |
| FLARE01 | Analyst workstation | Management / Forensics |
| IR-WEB | Compromised web practice server | IR |
| IR-WIN | Compromised Windows practice server | IR |
| BLUE-FW | Blue Team firewall practice | Blue |
| BLUE-WEB | Blue Team internal services | Blue |

---

### **Step 4 — Connect the Core Topology**

Connect:

OUTSIDE-SW \<-\> R1  
R1 \<-\> TRANSIT-R1-SW  
TRANSIT-R1-SW \<-\> CENTOS-04  
CENTOS-04 \<-\> TRANSIT-FW-SW  
TRANSIT-FW-SW \<-\> FW-EDGE  
FW-EDGE \<-\> INSIDE-SW  
FW-EDGE \<-\> DMZ-SW  
INSIDE-SW \<-\> AD-SVR  
INSIDE-SW \<-\> WIN-CLI  
INSIDE-SW \<-\> SIEM01  
DMZ-SW \<-\> WEB-01  
DMZ-SW \<-\> FTP-IDS01

Connect optional training networks:

MGMT-SW \<-\> FLARE01  
MGMT-SW \<-\> IR-WEB  
MGMT-SW \<-\> IR-WIN  
RED-CTF-SW \<-\> KALI01  
BLUE-MON-SW \<-\> BLUE-FW  
BLUE-MON-SW \<-\> SIEM01  
FORENSICS-SW \<-\> FLARE01

---

### **Step 5 — Add Labels**

Every link must be labeled.

Example:

R1 eth0 \-\> OUTSIDE 130.18.30.1/24  
R1 eth1 \-\> TRANSIT-R1 209.165.200.234/30  
CENTOS-04 eth0 \-\> R1 209.165.200.233/30  
CENTOS-04 eth1 \-\> FW 209.165.200.225/30  
FW WAN \-\> 209.165.200.226/30  
FW LAN \-\> 192.168.10.1/24  
FW DMZ \-\> 192.168.2.1/24

---

### **Step 6 — Save Topology Diagram**

Export or screenshot the full topology.

File name:

W1D2\_Topology\_Full\_01.png

---

## **Day 2 Deliverables**

| Deliverable | Required Evidence |
| ----- | ----- |
| Network objects created | Screenshot of simulator topology |
| Devices placed | Topology screenshot |
| Links labeled | Screenshot with labels visible |
| IP plan drafted | `W1D2_IPPlan.xlsx` |
| Project saved | Screenshot of project name |
| Snapshot created | `W1D2_NETWORK_SKELETON` |

---

## **Day 2 Mentor Check**

Student must explain:

1. Which network is Inside?  
2. Which network is DMZ?  
3. Which network is Outside?  
4. Which segment must not be attacked?  
5. Where will Splunk live?  
6. Where will the compromised servers live?  
7. Which devices replace physical firewall and router?

---

# **DAY 3 — Firewall and Routing Backbone**

## **Daily Objective**

Make the simulated topology route traffic.

## **Student Outcome**

By end of Day 3, Inside, DMZ, Outside, and Transit networks must be reachable according to firewall rules.

---

# **Day 3 Addressing**

| Device | Interface | IP |
| ----- | ----- | ----- |
| R1 | Outside LAN | `130.18.30.1/24` |
| R1 | Transit to CENTOS-04 | `209.165.200.234/30` |
| CENTOS-04 | To R1 | `209.165.200.233/30` |
| CENTOS-04 | To FW | `209.165.200.225/30` |
| FW-EDGE WAN | Outside | `209.165.200.226/30` |
| FW-EDGE LAN | Inside | `192.168.10.1/24` |
| FW-EDGE DMZ | DMZ | `192.168.2.1/24` |
| Outside Client | NIC | `130.18.30.10/24` |
| Test Inside Client | NIC | `192.168.10.200/24` |
| Test DMZ Host | NIC | `192.168.2.10/24` |

---

## **Day 3 Step-by-Step Instructions**

### **Step 1 — Configure R1**

Using VyOS or Linux router.

Example VyOS configuration:

configure  
set system host-name R1  
set interfaces ethernet eth0 address '130.18.30.1/24'  
set interfaces ethernet eth1 address '209.165.200.234/30'  
set protocols static route 0.0.0.0/0 next-hop '209.165.200.233'  
commit  
save  
exit

Verify:

show interfaces  
show ip route  
ping 209.165.200.233

---

### **Step 2 — Configure Transit Router**

Using VyOS:

configure  
set system host-name CENTOS-04  
set interfaces ethernet eth0 address '209.165.200.233/30'  
set interfaces ethernet eth1 address '209.165.200.225/30'  
set protocols static route 192.168.10.0/24 next-hop '209.165.200.226'  
set protocols static route 192.168.2.0/24 next-hop '209.165.200.226'  
set protocols static route 130.18.30.0/24 next-hop '209.165.200.234'  
commit  
save  
exit

Using Linux instead:

sudo hostnamectl set-hostname CENTOS-04  
sudo sysctl \-w net.ipv4.ip\_forward=1  
echo "net.ipv4.ip\_forward \= 1" | sudo tee \-a /etc/sysctl.conf  
sudo sysctl \-p

Set interface IPs using your OS network manager.

---

### **Step 3 — Configure Firewall Interfaces**

Using pfSense/OPNsense console:

WAN  \-\> 209.165.200.226/30  
LAN  \-\> 192.168.10.1/24  
DMZ  \-\> 192.168.2.1/24

Set WAN gateway:

209.165.200.225

Do not bridge WAN to the real office network.

---

### **Step 4 — Configure Firewall Rules**

Initial rules for training only:

| Interface | Rule |
| ----- | ----- |
| LAN | Allow LAN to any |
| DMZ | Allow DMZ to WAN only |
| WAN | Block inbound by default |
| LAN to DMZ | Allow only test ICMP/HTTP initially |
| DMZ to LAN | Block by default |

Later weeks will harden this. Day 3 is about stable routing.

---

### **Step 5 — Configure NAT**

On FW-EDGE:

Source: 192.168.10.0/24  
Destination: Any  
Translation: WAN address

Optional:

Source: 192.168.2.0/24  
Destination: Any  
Translation: WAN address

---

### **Step 6 — Configure Test Clients**

Inside test client:

IP: 192.168.10.200  
Mask: 255.255.255.0  
Gateway: 192.168.10.1  
DNS: 192.168.10.10 later

DMZ test server:

IP: 192.168.2.10  
Mask: 255.255.255.0  
Gateway: 192.168.2.1

Outside client:

IP: 130.18.30.10  
Mask: 255.255.255.0  
Gateway: 130.18.30.1

---

### **Step 7 — Validate Connectivity**

Run tests:

From Inside Client:

ping 192.168.10.1  
ping 192.168.2.1  
ping 192.168.2.10  
ping 130.18.30.10

From DMZ Host:

ping 192.168.2.1  
ping 209.165.200.225  
ping 130.18.30.10

From R1:

ping 130.18.30.10  
ping 209.165.200.233  
ping 209.165.200.226

From firewall:

Ping LAN host  
Ping DMZ host  
Ping R1 transit

---

### **Step 8 — Complete Ping Matrix**

| Source | Destination | Expected | Actual | Evidence |
| ----- | ----- | ----- | ----- | ----- |
| Inside client | Inside gateway | Pass |  |  |
| Inside client | DMZ gateway | Pass |  |  |
| Inside client | DMZ server | Pass |  |  |
| Inside client | Outside client | Pass if allowed |  |  |
| DMZ server | Inside client | Block by default |  |  |
| Outside client | Inside client | Block |  |  |
| Outside client | DMZ server | Block unless NAT/port-forward exists |  |  |

---

## **Day 3 Deliverables**

| Deliverable | Required Evidence |
| ----- | ----- |
| R1 configured | Config backup |
| Transit router configured | Config backup |
| Firewall interfaces configured | Screenshot |
| NAT configured | Screenshot |
| Firewall rules configured | Screenshot |
| Ping matrix completed | Spreadsheet/PDF |
| Snapshot | `W1D3_ROUTING_FIREWALL_BASE` |

---

## **Day 3 Mentor Check**

Mentor breaks one route or firewall rule and asks student to troubleshoot. Student must identify the issue using:

ipconfig /all or ip a  
ping  
traceroute / tracert  
route print / ip route  
firewall logs  
interface status

---

# **DAY 4 — Core VM Deployment and Base Services**

## **Daily Objective**

Deploy the servers and workstations needed for Week 2 to Week 4\.

## **Student Outcome**

By end of Day 4, all core VMs must boot, have correct IP addresses, and be reachable from the correct networks.

---

# **Day 4 VM Placement**

## **Infrastructure Lab**

| VM | Role | Network | IP |
| ----- | ----- | ----- | ----- |
| AD-SVR | AD/DNS/GPO/NPS | Inside | `192.168.10.10` |
| WIN-CLI | Domain client | Inside | `192.168.10.200` |
| SIEM01 | Splunk | Inside/Blue | `192.168.10.20` |
| WEB-01 | Apache/HTTPS/ModSecurity | DMZ | `192.168.2.10` |
| FTP-IDS01 | FTP/RADIUS/Snort | DMZ | `192.168.2.11` |
| KALI01 | Red Team workstation | Red/Outside | `10.10.10.10` |
| FLARE01 | Analyst/forensics | Mgmt/Forensics | `100.100.128.50` |

## **Incident Response Lab**

| VM | Role | Network | IP |
| ----- | ----- | ----- | ----- |
| IR-WEB | Compromised Linux web practice | IR | `192.168.1.55` or `100.100.128.60` |
| IR-WIN | Compromised Windows practice | IR | `192.168.1.58` or `100.100.128.70` |
| IR-LINUXM | Linux malware practice | IR | `192.168.1.59` |
| IR-ANALYST | Competitor workstation | IR | `192.168.1.252` |

The 2019 incident-response topology uses Web Server `192.168.1.55`, File Server `192.168.1.58`, LinuxM Server `192.168.1.59`, and competitor workstations on the same virtual switch.

---

## **Day 4 Step-by-Step Instructions**

### **Step 1 — Deploy Windows Server**

VM settings:

Name: AD-SVR  
CPU: 2 vCPU  
RAM: 4–6 GB  
Disk: 60 GB  
Network: INSIDE-SW  
IP: 192.168.10.10/24  
Gateway: 192.168.10.1  
DNS: 127.0.0.1 initially or 192.168.10.10 after DNS role

Do not configure AD yet unless time allows. Week 2 owns AD hardening.

Day 4 goal is boot, IP, hostname, remote access.

---

### **Step 2 — Deploy Windows Client**

Name: WIN-CLI  
Network: INSIDE-SW  
IP: 192.168.10.200/24  
Gateway: 192.168.10.1  
DNS: 192.168.10.10 later

Install:

RSAT tools if available  
Browser  
Wireshark  
Notepad++  
7-Zip  
Remote Desktop enabled for lab use

---

### **Step 3 — Deploy Linux Web Server**

Name: WEB-01  
Network: DMZ-SW  
IP: 192.168.2.10/24  
Gateway: 192.168.2.1

Install base packages:

sudo dnf install \-y httpd openssl mod\_ssl firewalld audit

or Ubuntu equivalent:

sudo apt update  
sudo apt install \-y apache2 openssl ufw auditd

Create placeholder page:

echo "WEB-01 WorldSkills Training Lab" | sudo tee /var/www/html/index.html

Start service:

sudo systemctl enable \--now httpd

or:

sudo systemctl enable \--now apache2

---

### **Step 4 — Deploy SIEM Server**

Name: SIEM01  
Network: INSIDE-SW or BLUE-MON-SW  
IP: 192.168.10.20/24  
Gateway: 192.168.10.1

Install base tools:

sudo apt update  
sudo apt install \-y curl wget net-tools tcpdump rsyslog

Splunk may be installed on Day 4 if the machine can handle it; otherwise prepare the VM and install Splunk in Week 2\.

The 2022 marking scheme explicitly checks Splunk installation and whether logs exist for a named user, so SIEM readiness must be built into the lab early.

---

### **Step 5 — Deploy IDS/FTP Server**

Name: FTP-IDS01  
Network: DMZ-SW  
IP: 192.168.2.11/24  
Gateway: 192.168.2.1

Install base packages:

sudo apt update  
sudo apt install \-y tcpdump wireshark-common rsyslog

Optional:

sudo apt install \-y snort

or:

sudo apt install \-y suricata

Do not tune detection rules yet. That is Week 2\.

---

### **Step 6 — Deploy Kali**

Name: KALI01  
Network: RED-CTF-SW or OUTSIDE-SW  
IP: 10.10.10.10/24 or 130.18.30.20/24

Student instruction:

Kali must only scan lab targets.  
Kali must never scan host office networks.  
Kali must never scan 100.100.0.0/16 management segment.  
Kali must not run DoS tools.

The 2022 Red Team document warns that attacking/scanning the competition operations platform or management network can lead to penalties or disqualification. Apply the same rule in training.

---

### **Step 7 — Deploy Analyst / Forensics VM**

Name: FLARE01 or ANALYST01  
Network: MGMT-SW / FORENSICS-SW  
IP: 100.100.128.50/24

Install or prepare:

Wireshark  
7-Zip  
Sysinternals Suite  
Procmon  
Autoruns  
Process Explorer  
HxD  
CyberChef offline if available  
Python 3  
VS Code  
Autopsy  
Volatility 3

The 2022 Module B workflow includes Competitor Windows machines with analysis tools and Kali under WSL, used to access compromised Linux and Windows systems.

---

### **Step 8 — Deploy Incident Response Practice Servers**

Create optional Week 3 prep nodes now:

IR-WEB      192.168.1.55  
IR-WIN      192.168.1.58  
IR-LINUXM   192.168.1.59  
IR-ANALYST  192.168.1.252

Connect all to `IR-SW`.

Do not infect or intentionally compromise yet. Week 1 is clean baseline only.

---

### **Step 9 — Validate Remote Access**

From FLARE01/ANALYST01:

RDP to Windows Server  
RDP to Windows Client  
SSH to WEB-01  
SSH to FTP-IDS01  
Browser to WEB-01  
Ping SIEM01  
Ping firewall LAN IP

---

## **Day 4 Deliverables**

| Deliverable | Required Evidence |
| ----- | ----- |
| AD-SVR deployed | IP config screenshot |
| WIN-CLI deployed | IP config screenshot |
| WEB-01 deployed | Browser screenshot |
| SIEM01 deployed | Login/IP screenshot |
| FTP-IDS01 deployed | IP screenshot |
| Kali deployed | IP screenshot |
| Analyst VM deployed | Tool list screenshot |
| IR practice network created | Topology screenshot |
| Snapshot | `W1D4_CORE_VM_BASELINE` |

---

## **Day 4 Mentor Check**

Student must demonstrate:

1. SSH to Linux server.  
2. RDP to Windows server/client.  
3. Browser access to WEB-01.  
4. Ping from Inside to DMZ.  
5. Firewall logs showing allowed/blocked traffic.  
6. VM snapshots exist.

---

# **DAY 5 — Evidence Workflow, Validation, and Recovery Drill**

## **Daily Objective**

Make the Week 1 lab stable, documented, and reusable.

## **Student Outcome**

By end of Day 5, the student must deliver a complete Week 1 baseline package and prove the lab survives reboot.

---

# **Day 5 Step-by-Step Instructions**

## **Step 1 — Create Local Evidence Portal**

Since the real Keysight/WorldSkills evidence portal is not available in your local lab, create a local substitute.

Option A: Windows shared folder

\\\\AD-SVR\\Evidence

Option B: Linux web upload folder

http://SIEM01/evidence

Option C: Simple shared folder on host PC

WorldSkills-Cyber-Training/Evidence/

Required evidence categories:

Screenshots  
Configs  
Command Logs  
Reports  
PCAPs  
Hash Values  
Answer Sheets

The 2022 Module B workflow expects evidence upload and answer sheet submission, so Week 1 should simulate that submission habit even without the official portal.

---

## **Step 2 — Create Week 1 Validation Checklist**

Student completes:

| Check | Pass/Fail | Evidence |
| ----- | ----- | ----- |
| EVE/GNS3 opens successfully |  |  |
| All core nodes boot |  |  |
| FW interfaces are correct |  |  |
| R1 routes are correct |  |  |
| Inside gateway reachable |  |  |
| DMZ gateway reachable |  |  |
| Outside client reachable |  |  |
| Inside to DMZ works as allowed |  |  |
| DMZ to Inside blocked by default |  |  |
| Analyst VM can manage lab systems |  |  |
| Kali is isolated |  |  |
| Evidence folder works |  |  |
| Snapshots exist |  |  |

---

## **Step 3 — Run Full Connectivity Test**

From Inside client:

ping 192.168.10.1  
ping 192.168.2.1  
ping 192.168.2.10  
ping 192.168.10.20

From Web server:

ping 192.168.2.1  
ping 209.165.200.225

From firewall:

Ping 192.168.10.200  
Ping 192.168.2.10  
Ping 209.165.200.225

From Kali:

ping assigned lab target only

No scans against management network.

---

## **Step 4 — Confirm Services**

Check WEB-01:

curl http://192.168.2.10

Check SIEM01:

systemctl status rsyslog

Check Linux SSH:

ssh user@192.168.2.10

Check firewall rule logs:

Open firewall GUI  
Check logs  
Confirm allowed/blocked events

---

## **Step 5 — Save Configurations**

Export:

Firewall rules  
R1 configuration  
Transit router configuration  
IP addressing table  
Topology screenshot  
VM inventory

Config backup names:

W1D5\_FWEDGE\_Config.pdf  
W1D5\_R1\_Config.txt  
W1D5\_CENTOS04\_Config.txt  
W1D5\_IP\_Addressing.xlsx  
W1D5\_Topology.png

---

## **Step 6 — Reboot Test**

Reboot in this order:

1. R1  
2. Transit router  
3. Firewall  
4. Linux servers  
5. Windows servers  
6. Clients  
7. Kali/Analyst VM

After reboot, validate:

IP addresses retained  
Routes retained  
Firewall rules retained  
Web page still reachable  
SSH still reachable  
RDP still reachable

No reboot validation means the lab is not competition-ready.

---

## **Step 7 — Recovery Drill**

Mentor intentionally breaks one item:

| Break | Student Must Fix |
| ----- | ----- |
| Wrong default gateway | Identify and correct |
| Firewall rule disabled | Locate and restore |
| Server NIC disconnected | Reconnect |
| Wrong DNS configured | Diagnose and correct |
| Route removed | Restore route |
| Web service stopped | Restart and enable |

Student must document:

What failed:  
How detected:  
Command/tool used:  
Fix applied:  
Proof after fix:

---

## **Step 8 — Create Final Week 1 Snapshot**

Snapshot name:

WEEK1\_COMPETITION\_BASELINE\_READY

This snapshot becomes the reset point for Week 2\.

---

# **Day 5 Deliverables**

| Deliverable | Required Evidence |
| ----- | ----- |
| Final topology | Screenshot |
| Full IP table | Spreadsheet |
| Full VM inventory | Spreadsheet |
| Ping matrix | Spreadsheet/PDF |
| Firewall rules | Screenshot/PDF |
| Router config | Text backup |
| Server access proof | Screenshots |
| Evidence portal/folder | Screenshot |
| Reboot validation | Checklist |
| Recovery drill report | Short report |
| Final snapshot | Screenshot |

---

# **Week 1 Final Student Report Format**

Student must submit a short PDF:

Title: Week 1 WorldSkills Cyber Lab Build Report

1\. Lab Platform Used  
2\. PC Specifications  
3\. Network Topology  
4\. IP Addressing Table  
5\. Device and VM Inventory  
6\. Firewall and Routing Summary  
7\. Evidence Folder Structure  
8\. Connectivity Test Results  
9\. Issues Encountered  
10\. Fixes Applied  
11\. Snapshot List  
12\. Readiness Statement

Readiness statement:

I confirm that the Week 1 lab baseline is functional, documented, reboot-tested, and ready for Week 2 Infrastructure Security training.

---

# **Final Week 1 Checklist**

| Requirement | Status |
| ----- | ----- |
| EVE-NG or GNS3 installed | ☐ |
| Legal images imported | ☐ |
| Inside network created | ☐ |
| DMZ network created | ☐ |
| Outside network created | ☐ |
| Management network created | ☐ |
| IR network created | ☐ |
| Red CTF network created | ☐ |
| Blue monitoring network created | ☐ |
| Firewall deployed | ☐ |
| Router deployed | ☐ |
| AD-SVR deployed | ☐ |
| WIN-CLI deployed | ☐ |
| WEB-01 deployed | ☐ |
| SIEM01 deployed | ☐ |
| FTP/IDS server deployed | ☐ |
| Kali deployed | ☐ |
| Analyst VM deployed | ☐ |
| Firewall rules tested | ☐ |
| Routing tested | ☐ |
| Evidence workflow tested | ☐ |
| Config backups saved | ☐ |
| Reboot test passed | ☐ |
| Recovery drill completed | ☐ |
| Final snapshot created | ☐ |

---

# **Student Performance Standard for Week 1**

The student is not allowed to move to Week 2 until they can do these without assistance:

1. Explain the full topology.  
2. Identify every subnet.  
3. Identify every gateway.  
4. Explain which systems are in Inside, DMZ, Outside, Management, IR, Red, and Blue networks.  
5. Restore the lab from snapshot.  
6. Fix a basic routing issue.  
7. Fix a basic firewall rule issue.  
8. Capture clean evidence.  
9. Save configurations.  
10. Produce a one-page status report.

---

# **Mentor Notes**

Do not help too early. Week 1 is where troubleshooting discipline is built.

Use this coaching model:

| Student Behavior | Mentor Response |
| ----- | ----- |
| Student asks “Why no ping?” | Ask for IP, gateway, route, firewall log |
| Student says “It worked earlier” | Ask for snapshot/config evidence |
| Student forgets screenshot | Mark as incomplete |
| Student changes many things at once | Stop and force rollback |
| Student cannot explain topology | Repeat Day 2 |
| Student cannot recover broken route | Repeat Day 3 |

Strong recommendation: **Week 1 must be strict.** A weak lab foundation will destroy Week 2 and Week 3 productivity.

# Week 2

Week 2 is the scoring backbone. In the 2019 Module A actual project, the infrastructure module is worth **25 marks**, divided into **logon/password policies, network equipment hardening, public services protection, events monitoring, and firewall policy**. The 2022 materials also reinforce the same operational mindset: read all tasks first, confirm all systems work, test before submission, and recover any locked or misconfigured system.

---

# **WEEK 2 — ENTERPRISE INFRASTRUCTURE SECURITY**

## **Weekly Objective**

By the end of Week 2, the student must have a hardened enterprise simulation environment with:

* Active Directory domain  
* DNS  
* Domain-joined client  
* GPO security policies  
* Linux password/security hardening  
* Apache HTTPS website  
* HTTP-to-HTTPS redirect  
* ModSecurity blocking test  
* Secure FTP / FTPS service  
* Splunk log ingestion  
* IDS/Snort/Suricata alerts  
* Firewall least-privilege rules  
* VPN simulation  
* Reboot-tested configuration  
* Evidence package

This maps directly to the WorldSkills expectation that services are functional but security is not yet fully implemented, so competitors must apply enterprise security policies and hardening controls.

---

# **Student Rules for Week 2**

1. Start every day from the `WEEK1_COMPETITION_BASELINE_READY` snapshot.  
2. Do not change IP addresses unless instructed.  
3. Do not disable the firewall permanently to “make things work.”  
4. Every configuration must have:  
   * screenshot  
   * command/config backup  
   * validation proof  
5. Reboot-test before the end of every day.  
6. Use a daily command log.  
7. Do not proceed to the next task until the current task is validated.  
8. Save configuration after every major change.  
9. Any issue must be documented as:  
   * problem  
   * cause  
   * fix  
   * proof

WorldSkills explicitly scores implemented and working tasks only; the project warns that one configuration change may break a previous requirement, so testing must be continuous.

---

# **Week 2 Lab Devices**

Use the Week 1 lab devices:

| Device | Role | IP |
| ----- | ----- | ----- |
| `AD-SVR` | Windows Server / AD / DNS / GPO | `192.168.10.10` |
| `WIN-CLI` | Windows client | `192.168.10.200` |
| `SIEM01` | Splunk / log server | `192.168.10.20` |
| `WEB-01` | Apache / HTTPS / ModSecurity | `192.168.2.10` |
| `FTP-IDS01` | FTP / IDS / Snort or Suricata | `192.168.2.11` |
| `FW-EDGE` | pfSense / OPNsense firewall | `192.168.10.1`, `192.168.2.1` |
| `R1` | Router / VyOS / IOSv | lab dependent |
| `KALI01` | Test workstation only | lab dependent |

---

# **Daily Training Structure**

Use the same operating rhythm daily:

| Time | Activity |
| ----- | ----- |
| 08:30–09:00 | Review previous day / restore snapshot if needed |
| 09:00–10:30 | Concept \+ mentor demo |
| 10:30–10:45 | Break |
| 10:45–12:30 | Hands-on block 1 |
| 12:30–13:30 | Lunch |
| 13:30–15:30 | Hands-on block 2 |
| 15:30–15:45 | Break |
| 15:45–17:00 | Validation and troubleshooting |
| 17:00–17:30 | Screenshots, config backup, report |
| 17:30–18:00 | Mentor scoring |

---

# **DAY 6 — Active Directory, DNS, Users, GPO**

## **Daily Objective**

Build the identity foundation: domain, DNS, users, groups, client join, and baseline security GPO.

The 2019 Module A actual project lists Active Directory, DNS, DHCP, and Network Policy Server as core DC services, while the 2022 marking scheme checks domain creation, DNS correctness, joined computers, OUs, GPOs, account policy, audit policy, and Splunk forwarder presence.

---

## **Step 1 — Pre-check**

On `AD-SVR`:

hostname  
ipconfig /all  
ping 192.168.10.1  
ping 192.168.10.200

Expected:

Hostname: AD-SVR  
IP: 192.168.10.10  
Gateway: 192.168.10.1  
DNS: 192.168.10.10 after DNS role is installed

Evidence:

W2D6\_ADSVR\_IPConfig\_01.png

---

## **Step 2 — Install AD DS and DNS**

On Windows Server PowerShell as Administrator:

Install-WindowsFeature AD-Domain-Services,DNS \-IncludeManagementTools

Promote server to domain controller:

Install-ADDSForest \`  
\-DomainName "shenghai.ws" \`  
\-DomainNetbiosName "SHENGHAI" \`  
\-InstallDNS \`  
\-SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd1\!" \-AsPlainText \-Force) \`  
\-Force

The server will reboot.

Lab note: Use `shenghai.ws` for 2022-style training. If you want a 2019-only variant, use `RUSSIA.LOCAL` or `wsc2019.cloud`, but do not mix domains inside one active lab.

---

## **Step 3 — Validate domain**

After reboot:

Get-WmiObject \-Class Win32\_ComputerSystem | Select-Object Domain  
nslookup shenghai.ws  
dcdiag /test:dns

Deliverable:

W2D6\_DomainCreated\_01.png  
W2D6\_DNSLookup\_01.png

---

## **Step 4 — Create OUs**

Create the base OU structure:

New-ADOrganizationalUnit \-Name "Accounting" \-Path "DC=shenghai,DC=ws"  
New-ADOrganizationalUnit \-Name "IT" \-Path "DC=shenghai,DC=ws"  
New-ADOrganizationalUnit \-Name "Servers" \-Path "DC=shenghai,DC=ws"  
New-ADOrganizationalUnit \-Name "Workstations" \-Path "DC=shenghai,DC=ws"

Validate:

Get-ADOrganizationalUnit \-Filter \* | Select-Object Name

---

## **Step 5 — Create users and groups**

Create groups:

New-ADGroup \-Name "Accounting Users" \-GroupScope Global \-Path "OU=Accounting,DC=shenghai,DC=ws"  
New-ADGroup \-Name "IT Admins" \-GroupScope Global \-Path "OU=IT,DC=shenghai,DC=ws"

Create test users:

$pwd \= ConvertTo-SecureString "P@ssw0rd1\!" \-AsPlainText \-Force

New-ADUser \-Name "jfrank" \-SamAccountName "jfrank" \-AccountPassword $pwd \-Enabled $true \-Path "OU=Accounting,DC=shenghai,DC=ws"  
New-ADUser \-Name "maggie" \-SamAccountName "maggie" \-AccountPassword $pwd \-Enabled $true \-Path "OU=IT,DC=shenghai,DC=ws"  
New-ADUser \-Name "lionel" \-SamAccountName "lionel" \-AccountPassword $pwd \-Enabled $true \-Path "OU=IT,DC=shenghai,DC=ws"

Add users to groups:

Add-ADGroupMember "Accounting Users" jfrank  
Add-ADGroupMember "IT Admins" maggie,lionel

Validate:

Get-ADUser \-Filter {Name \-like "j\*"} \-Properties \* | Select-Object Name,SamAccountName

The 2022 marking scheme specifically references checking logs for `jfrank`, so create that user now for later Splunk validation.

---

## **Step 6 — Configure DNS records**

Open DNS Manager or use PowerShell:

Add-DnsServerResourceRecordA \-Name "www" \-ZoneName "shenghai.ws" \-IPv4Address "192.168.2.10"  
Add-DnsServerResourceRecordA \-Name "testshenghai" \-ZoneName "shenghai.ws" \-IPv4Address "192.168.2.10"  
Add-DnsServerResourceRecordA \-Name "siem01" \-ZoneName "shenghai.ws" \-IPv4Address "192.168.10.20"  
Add-DnsServerResourceRecordA \-Name "ftp" \-ZoneName "shenghai.ws" \-IPv4Address "192.168.2.11"

Validate from `WIN-CLI` later:

nslookup www.shenghai.ws  
nslookup testshenghai.shenghai.ws

---

## **Step 7 — Join Windows client to domain**

On `WIN-CLI`, set DNS to `192.168.10.10`.

PowerShell as Administrator:

Rename-Computer \-NewName "WIN-CLI" \-Force  
Add-Computer \-DomainName "shenghai.ws" \-Credential "SHENGHAI\\Administrator" \-Restart

Validate after reboot:

whoami  
gpresult /r

Evidence:

W2D6\_WINCLI\_DomainJoined\_01.png

---

## **Step 8 — Create GPO: password and account policy**

On `AD-SVR`, open:

Group Policy Management

Create GPO:

GPO Name: WSC-Security-Baseline  
Link: Domain root or Accounting OU

Configure:

Computer Configuration  
 \> Policies  
  \> Windows Settings  
   \> Security Settings  
    \> Account Policies

Recommended lab settings:

| Policy | Value |
| ----- | ----- |
| Minimum password length | 10 |
| Password complexity | Enabled |
| Account lockout threshold | 3 invalid attempts |
| Account lockout duration | 1 minute |
| Reset account lockout counter | 1 minute |

The 2019 Module A actual project explicitly requires password length, password complexity, failed-login blocking, and inactivity timeout across Windows, Linux, and network devices.

---

## **Step 9 — Create GPO: login banner and audit policy**

Configure login banner:

Computer Configuration  
 \> Policies  
  \> Windows Settings  
   \> Security Settings  
    \> Local Policies  
     \> Security Options

Set:

Interactive logon: Message title \= Warning  
Interactive logon: Message text \= For authorized users only

Configure audit policy:

Computer Configuration  
 \> Policies  
  \> Windows Settings  
   \> Security Settings  
    \> Advanced Audit Policy Configuration

Enable Success and Failure for:

Credential Validation  
Logon  
Account Lockout  
Computer Account Management  
Security Group Management  
Sensitive Privilege Use

The 2019 Module A actual project requires DC security logs to be forwarded to Splunk and audit policies for credential validation, logon, account lockout, security group management, and sensitive privilege use.

---

## **Step 10 — Validate GPO**

On `WIN-CLI`:

gpupdate /force  
gpresult /r

Test:

* login banner appears  
* wrong password causes lockout  
* policy is visible in `gpresult`

Evidence:

W2D6\_GPO\_Result\_01.png  
W2D6\_LoginBanner\_01.png  
W2D6\_AccountLockout\_01.png

---

## **Day 6 Deliverables**

| Deliverable | Evidence |
| ----- | ----- |
| Domain created | Screenshot |
| DNS working | `nslookup` screenshot |
| Users/OUs created | ADUC screenshot |
| Client joined | System/domain screenshot |
| GPO created | GPMC screenshot |
| Password policy validated | Screenshot |
| Audit policy configured | Screenshot |
| Login banner visible | Screenshot |
| Config notes | Daily report |

Snapshot:

W2D6\_AD\_GPO\_READY

---

# **DAY 7 — Linux Security Hardening**

## **Daily Objective**

Harden Linux systems: `WEB-01`, `FTP-IDS01`, and `SIEM01`.

The 2019 infrastructure project requires Linux password policy, login banners, failed-login blocking, and inactivity timeout. The 2022 marking scheme checks Linux user creation, sudo permissions, firewalld, SELinux enforcing, and auditd rules.

---

## **Step 1 — Pre-check all Linux servers**

Run on each Linux VM:

hostname  
ip a  
ip route  
ping \-c 3 192.168.10.1  
ping \-c 3 192.168.2.1

Evidence:

W2D7\_Linux\_Precheck\_WEB01\_01.png

---

## **Step 2 — Create local users**

On `WEB-01`:

sudo useradd maggie  
sudo passwd maggie

sudo useradd lionel  
sudo passwd lionel

sudo useradd ixia  
sudo passwd ixia

Use lab password:

P@ssw0rd1\!

Validate:

cat /etc/passwd | egrep 'maggie|lionel|ixia'

---

## **Step 3 — Configure sudo access**

Allow `maggie` full sudo:

sudo usermod \-aG wheel maggie

Allow `lionel` to run only package commands:

sudo visudo

Add:

lionel ALL=(root) /usr/bin/yum, /usr/bin/rpm, /usr/bin/dnf

For Ubuntu-based systems, adapt paths:

lionel ALL=(root) /usr/bin/apt, /usr/bin/dpkg

Validate:

sudo \-u maggie sudo \-l  
sudo \-u lionel sudo \-l

Evidence:

W2D7\_Sudo\_Maggie\_01.png  
W2D7\_Sudo\_Lionel\_01.png

---

## **Step 4 — Configure password policy**

Install required modules:

CentOS/Rocky/Alma:

sudo dnf install \-y libpwquality pam

Ubuntu:

sudo apt install \-y libpam-pwquality

Edit:

sudo nano /etc/security/pwquality.conf

Set:

minlen \= 10  
ucredit \= \-1  
lcredit \= \-1  
dcredit \= \-1  
ocredit \= \-1  
retry \= 3

Configure password aging:

sudo chage \-M 30 maggie  
sudo chage \-M 30 lionel  
sudo chage \-M 30 ixia

Validate:

chage \-l maggie

---

## **Step 5 — Configure failed-login lockout**

CentOS/Rocky/Alma:

sudo authselect current  
sudo authselect enable-feature with-faillock  
sudo authselect apply-changes

Edit:

sudo nano /etc/security/faillock.conf

Set:

deny \= 3  
unlock\_time \= 60  
fail\_interval \= 3600

Validate:

faillock \--user maggie

---

## **Step 6 — Configure SSH hardening**

Edit:

sudo nano /etc/ssh/sshd\_config

Set:

PermitRootLogin no  
PasswordAuthentication yes  
ClientAliveInterval 60  
ClientAliveCountMax 1  
Banner /etc/issue.net

Create banner:

echo "For authorized users only" | sudo tee /etc/issue.net

Restart SSH:

sudo systemctl restart sshd

Validate:

ssh root@192.168.2.10  
ssh maggie@192.168.2.10

Root SSH must fail. User SSH should work.

---

## **Step 7 — Configure firewalld**

Install and enable:

sudo dnf install \-y firewalld  
sudo systemctl enable \--now firewalld

On `WEB-01`:

sudo firewall-cmd \--set-default-zone=dmz  
sudo firewall-cmd \--permanent \--zone=dmz \--add-service=http  
sudo firewall-cmd \--permanent \--zone=dmz \--add-service=https  
sudo firewall-cmd \--reload  
sudo firewall-cmd \--list-all

On `FTP-IDS01`:

sudo firewall-cmd \--permanent \--zone=dmz \--add-service=ftp  
sudo firewall-cmd \--permanent \--zone=dmz \--add-port=990/tcp  
sudo firewall-cmd \--reload  
sudo firewall-cmd \--list-all

The 2022 marking scheme includes checks for active firewall zone, persistent ports, and HTTP/HTTPS services enabled in firewall.

---

## **Step 8 — Enable SELinux enforcing**

Check:

sestatus

Edit:

sudo nano /etc/selinux/config

Set:

SELINUX=enforcing

Apply:

sudo setenforce 1  
sestatus

Evidence:

W2D7\_SELinux\_Enforcing\_01.png

---

## **Step 9 — Configure auditd**

Install:

sudo dnf install \-y audit audit-libs  
sudo systemctl enable \--now auditd

Create audit rule:

sudo auditctl \-w /etc/passwd \-p wa \-k identity\_watch  
sudo auditctl \-w /etc/ssh/sshd\_config \-p wa \-k ssh\_config\_watch  
sudo auditctl \-l

Make persistent:

echo "-w /etc/passwd \-p wa \-k identity\_watch" | sudo tee \-a /etc/audit/rules.d/wsc.rules  
echo "-w /etc/ssh/sshd\_config \-p wa \-k ssh\_config\_watch" | sudo tee \-a /etc/audit/rules.d/wsc.rules  
sudo augenrules \--load

The 2022 marking scheme includes an auditd rule check.

---

## **Day 7 Deliverables**

| Deliverable | Evidence |
| ----- | ----- |
| Linux users created | `/etc/passwd` screenshot |
| Password policy set | Config screenshot |
| Failed login lockout | Test screenshot |
| SSH root disabled | Failed root SSH screenshot |
| Sudo rules configured | `sudo -l` screenshots |
| firewalld configured | `firewall-cmd --list-all` |
| SELinux enforcing | `sestatus` screenshot |
| auditd configured | `auditctl -l` screenshot |

Snapshot:

W2D7\_LINUX\_HARDENED

---

# **DAY 8 — Public Services: HTTPS, ModSecurity, FTPS**

## **Daily Objective**

Secure public services in the DMZ.

The 2019 Module A actual project requires the corporate website to use HTTPS only and redirect HTTP to HTTPS, and requires FTP to accept explicit SSL/TLS only. The 2022 marking scheme checks access to `www.shenghai.org`, `www.testshenghai.com`, HTTP-to-HTTPS redirect, and ModSecurity blocking `/etc/passwd`.

---

## **Step 1 — Configure DNS records**

On `AD-SVR`, add DNS records:

Add-DnsServerResourceRecordA \-Name "www" \-ZoneName "shenghai.ws" \-IPv4Address "192.168.2.10"  
Add-DnsServerResourceRecordA \-Name "www.testshenghai" \-ZoneName "shenghai.ws" \-IPv4Address "192.168.2.10"

Optional if using local hosts file on `WIN-CLI`:

192.168.2.10 www.shenghai.org  
192.168.2.10 www.testshenghai.com

For training, either DNS method is acceptable, but document which one was used.

---

## **Step 2 — Install Apache and SSL module**

On `WEB-01`:

CentOS/Rocky/Alma:

sudo dnf install \-y httpd mod\_ssl openssl  
sudo systemctl enable \--now httpd

Ubuntu:

sudo apt install \-y apache2 openssl  
sudo systemctl enable \--now apache2

---

## **Step 3 — Create website folders**

sudo mkdir \-p /var/www/shenghai  
sudo mkdir \-p /var/www/testshenghai

echo "www.shenghai.org secure site" | sudo tee /var/www/shenghai/index.html  
echo "www.testshenghai.com secure site" | sudo tee /var/www/testshenghai/index.html

---

## **Step 4 — Create self-signed certificate**

sudo mkdir \-p /etc/pki/tls/wsc  
sudo openssl req \-x509 \-nodes \-newkey rsa:4096 \\  
\-keyout /etc/pki/tls/wsc/shenghai.key \\  
\-out /etc/pki/tls/wsc/shenghai.crt \\  
\-days 365 \\  
\-subj "/C=PH/O=WorldSkills Training/CN=www.shenghai.org"

For Ubuntu path:

sudo mkdir \-p /etc/ssl/wsc

The 2022 Module B practical task requires a self-signed 4K RSA certificate, HTTP and HTTPS service, and redirect from HTTP to HTTPS.

---

## **Step 5 — Configure Apache virtual hosts**

CentOS example:

sudo nano /etc/httpd/conf.d/shenghai.conf

Add:

\<VirtualHost \*:80\>  
    ServerName www.shenghai.org  
    ServerAlias www.testshenghai.com  
    Redirect permanent / https://www.shenghai.org/  
\</VirtualHost\>

\<VirtualHost \*:443\>  
    ServerName www.shenghai.org  
    DocumentRoot /var/www/shenghai

    SSLEngine on  
    SSLCertificateFile /etc/pki/tls/wsc/shenghai.crt  
    SSLCertificateKeyFile /etc/pki/tls/wsc/shenghai.key

    ErrorLog logs/shenghai-error.log  
    CustomLog logs/shenghai-access.log combined  
\</VirtualHost\>

\<VirtualHost \*:443\>  
    ServerName www.testshenghai.com  
    DocumentRoot /var/www/testshenghai

    SSLEngine on  
    SSLCertificateFile /etc/pki/tls/wsc/shenghai.crt  
    SSLCertificateKeyFile /etc/pki/tls/wsc/shenghai.key

    ErrorLog logs/testshenghai-error.log  
    CustomLog logs/testshenghai-access.log combined  
\</VirtualHost\>

Test config:

sudo apachectl configtest  
sudo systemctl restart httpd

---

## **Step 6 — Validate HTTPS and redirect**

From `WIN-CLI` browser:

https://www.shenghai.org  
https://www.testshenghai.com  
http://www.shenghai.org  
http://www.testshenghai.com

Expected:

* HTTPS loads.  
* HTTP redirects to HTTPS.

Evidence:

W2D8\_HTTPS\_shenghai\_01.png  
W2D8\_HTTPS\_testshenghai\_01.png  
W2D8\_HTTP\_Redirect\_01.png

---

## **Step 7 — Install and configure ModSecurity**

CentOS/Rocky/Alma:

sudo dnf install \-y mod\_security mod\_security\_crs

Ubuntu:

sudo apt install \-y libapache2-mod-security2  
sudo a2enmod security2

Enable engine:

sudo nano /etc/httpd/conf.d/mod\_security.conf

or Ubuntu:

sudo nano /etc/modsecurity/security2.conf

Set:

SecRuleEngine On

Add custom rule:

sudo nano /etc/httpd/conf.d/wsc-modsec.conf

Add:

\<IfModule security2\_module\>  
SecRule REQUEST\_URI "@contains /etc/passwd" "id:1000001,phase:1,deny,status:403,msg:'Blocked /etc/passwd path access'"  
SecRule ARGS "@contains /etc/passwd" "id:1000002,phase:2,deny,status:403,msg:'Blocked /etc/passwd parameter access'"  
\</IfModule\>

Restart:

sudo systemctl restart httpd

Validate from browser:

https://www.shenghai.org/etc/passwd  
https://www.shenghai.org/?testparameter=/etc/passwd  
https://www.testshenghai.com/etc/passwd

Expected:

403 Forbidden

Evidence:

W2D8\_ModSecurity\_Block\_Path\_01.png  
W2D8\_ModSecurity\_Block\_Parameter\_01.png

---

## **Step 8 — Configure secure FTP / FTPS**

On `FTP-IDS01`:

sudo dnf install \-y vsftpd openssl  
sudo systemctl enable \--now vsftpd

Create user:

sudo groupadd ftpgroup  
sudo useradd \-g ftpgroup \-d /files/users/skill54-ftp \-s /sbin/nologin ftpuser  
sudo mkdir \-p /files/users/skill54-ftp  
sudo chown \-R ftpuser:ftpgroup /files/users/skill54-ftp

Create certificate:

sudo openssl req \-x509 \-nodes \-newkey rsa:2048 \\  
\-keyout /etc/pki/tls/private/vsftpd.key \\  
\-out /etc/pki/tls/certs/vsftpd.crt \\  
\-days 365 \\  
\-subj "/C=PH/O=WorldSkills Training/CN=ftp.shenghai.ws"

Edit:

sudo nano /etc/vsftpd/vsftpd.conf

Core settings:

anonymous\_enable=NO  
local\_enable=YES  
write\_enable=YES  
chroot\_local\_user=YES  
ssl\_enable=YES  
allow\_anon\_ssl=NO  
force\_local\_data\_ssl=YES  
force\_local\_logins\_ssl=YES  
rsa\_cert\_file=/etc/pki/tls/certs/vsftpd.crt  
rsa\_private\_key\_file=/etc/pki/tls/private/vsftpd.key

Restart:

sudo systemctl restart vsftpd

Validate using FileZilla or `lftp`:

lftp \-u ftpuser ftps://192.168.2.11

The 2019 pre-release specified explicit SSL/TLS-only FTP, one active concurrent session, no renaming, and virtual-user design for the FTP server.

---

## **Day 8 Deliverables**

| Deliverable | Evidence |
| ----- | ----- |
| HTTPS website works | Browser screenshot |
| Test website works | Browser screenshot |
| HTTP redirects to HTTPS | Browser screenshot |
| Certificate created | Command screenshot |
| ModSecurity blocks `/etc/passwd` | 403 screenshot |
| FTPS works | Client screenshot |
| Plain FTP rejected | Failed connection screenshot |
| Config backups | Apache, ModSecurity, vsftpd configs |

Snapshot:

W2D8\_PUBLIC\_SERVICES\_SECURED

---

# **DAY 9 — SIEM, Logging, IDS**

## **Daily Objective**

Implement monitoring and detection.

Events monitoring has the highest mark weight in 2019 Module A: **6.75 out of 25**, higher than any other infrastructure subcategory.

---

## **Step 1 — Install or validate Splunk on `SIEM01`**

If Splunk is already installed, start it:

sudo /opt/splunk/bin/splunk start \--accept-license  
sudo /opt/splunk/bin/splunk enable boot-start

Access:

http://192.168.10.20:8000

Create admin account for lab use.

Evidence:

W2D9\_Splunk\_Login\_01.png

---

## **Step 2 — Configure Splunk receiving port**

In Splunk Web:

Settings \> Forwarding and receiving \> Configure receiving \> New Receiving Port

Set:

8090

Or CLI:

sudo /opt/splunk/bin/splunk enable listen 8090

The 2019 Module A actual project requires Splunk to receive logs from the Domain Controller on port `8090`.

---

## **Step 3 — Install Splunk Universal Forwarder on `AD-SVR`**

Install the Windows Splunk Universal Forwarder.

Configure forward server:

cd "C:\\Program Files\\SplunkUniversalForwarder\\bin"  
.\\splunk.exe add forward-server 192.168.10.20:8090

Add Windows Security log input:

.\\splunk.exe add monitor WinEventLog://Security  
.\\splunk.exe restart

If using GUI install, configure:

Receiving indexer: 192.168.10.20  
Port: 8090  
Logs: Security Event Log

---

## **Step 4 — Generate Windows events**

On `WIN-CLI`:

Log in as jfrank.  
Fail login 3 times.  
Log off.  
Log in successfully.

On `AD-SVR`, run:

Get-EventLog \-LogName Security \-Newest 20

---

## **Step 5 — Validate in Splunk**

In Splunk search:

index=main jfrank

Also test:

index=main EventCode=4624  
index=main EventCode=4625  
index=main EventCode=4740

Evidence:

W2D9\_Splunk\_jfrank\_01.png  
W2D9\_Splunk\_FailedLogon\_01.png

The 2022 marking scheme includes checking whether logs for `jfrank` exist in Splunk.

---

## **Step 6 — Configure Linux syslog forwarding**

On Linux servers:

sudo nano /etc/rsyslog.d/99-siem.conf

Add:

\*.\* @@192.168.10.20:514

Restart:

sudo systemctl restart rsyslog

On `SIEM01`, enable syslog receive:

sudo nano /etc/rsyslog.conf

Uncomment or add:

module(load="imtcp")  
input(type="imtcp" port="514")

Restart:

sudo systemctl restart rsyslog

Validate with test log:

logger "WSC test log from WEB-01"

---

## **Step 7 — Install IDS**

On `FTP-IDS01`:

sudo dnf install \-y suricata

or:

sudo apt install \-y suricata

For Snort if available:

sudo dnf install \-y snort

---

## **Step 8 — Add detection rules**

For Snort-style rules:

sudo nano /etc/snort/rules/local.rules

Add:

alert tcp any any \-\> any 21 (msg:"WSC FTP traffic detected"; sid:100001;)  
alert icmp any any \-\> any any (msg:"WSC ICMP traffic detected"; sid:100002;)  
alert tcp any any \-\> any any (msg:"WSC malware keyword detected"; content:"malware"; sid:100003;)

For Suricata:

sudo nano /var/lib/suricata/rules/local.rules

Add equivalent rules and include local rules in `suricata.yaml`.

The 2019 task requires alerting/logging for external FTP traffic, pings, and payload containing `malware`.

---

## **Step 9 — Generate test traffic**

From `KALI01` or Outside client:

ping 192.168.2.11

FTP test:

ftp 192.168.2.11

Malware keyword test:

echo "malware" | nc 192.168.2.11 8080

Check IDS logs:

sudo tail \-f /var/log/suricata/fast.log

or:

sudo tail \-f /var/log/snort/alert

---

## **Day 9 Deliverables**

| Deliverable | Evidence |
| ----- | ----- |
| Splunk running | Login screenshot |
| Port 8090 configured | Splunk receiving screenshot |
| AD logs forwarded | Splunk search screenshot |
| `jfrank` logs visible | Search screenshot |
| Linux logs forwarded | Syslog evidence |
| IDS installed | Service status screenshot |
| FTP alert | Alert screenshot |
| ICMP alert | Alert screenshot |
| Malware alert | Alert screenshot |

Snapshot:

W2D9\_MONITORING\_READY

---

# **DAY 10 — Firewall Policy, VPN Simulation, Final Validation**

## **Daily Objective**

Finish the infrastructure module by enforcing least-privilege firewall rules, validating VPN, and producing a Week 2 evidence package.

The 2019 Module A actual requires all server/client/network firewalls to be enabled, Domain Controller firewall rules to allow Splunk forwarding on port 8090, and minimal permission rules only for required traffic.

---

## **Step 1 — Create firewall rule matrix**

Before touching the firewall, the student must complete this:

| Source | Destination | Port | Action | Reason |
| ----- | ----- | ----- | ----- | ----- |
| WIN-CLI | AD-SVR | DNS/LDAP/Kerberos/SMB | Allow | Domain services |
| AD-SVR | SIEM01 | 8090 | Allow | Splunk forwarding |
| Inside | WEB-01 | 443 | Allow | HTTPS |
| Inside | WEB-01 | 80 | Allow | Redirect test only |
| Outside | WEB-01 | 443 | Allow if testing public access |  |
| DMZ | Inside | Any | Block | Segmentation |
| Outside | Inside | Any | Block | Security |
| Analyst | Linux servers | 22 | Allow | Admin SSH |
| Analyst | Windows servers | 3389 | Allow | Admin RDP |

---

## **Step 2 — Configure pfSense/OPNsense rules**

On `FW-EDGE`:

### **LAN Rules**

Allow:

LAN net \-\> AD-SVR DNS/Kerberos/LDAP/SMB  
LAN net \-\> WEB-01 80/443  
LAN net \-\> SIEM01 8000/8090  
LAN net \-\> FTP-IDS01 required test ports

Block:

LAN net \-\> any unnecessary ports

### **DMZ Rules**

Allow:

DMZ net \-\> WAN only required outbound  
DMZ net \-\> SIEM01 syslog/Splunk if needed

Block:

DMZ net \-\> LAN net

### **WAN Rules**

Default:

Block all inbound

Optional lab-only:

WAN test client \-\> WEB-01 443

Evidence:

W2D10\_FW\_LANRules\_01.png  
W2D10\_FW\_DMZRules\_01.png  
W2D10\_FW\_WANRules\_01.png

---

## **Step 3 — Enable Windows Firewall**

On `AD-SVR` and `WIN-CLI`:

Set-NetFirewallProfile \-Profile Domain,Private,Public \-Enabled True  
Get-NetFirewallProfile | Select Name,Enabled

Allow Splunk forwarding:

New-NetFirewallRule \-DisplayName "Allow Splunk Forwarding 8090" \`  
\-Direction Outbound \`  
\-Protocol TCP \`  
\-RemoteAddress 192.168.10.20 \`  
\-RemotePort 8090 \`  
\-Action Allow

Evidence:

W2D10\_WindowsFirewall\_Profiles\_01.png

---

## **Step 4 — Enable Linux firewall rules**

On `WEB-01`:

sudo firewall-cmd \--list-all  
sudo firewall-cmd \--permanent \--remove-service=ssh  
sudo firewall-cmd \--permanent \--add-rich-rule='rule family="ipv4" source address="192.168.10.0/24" service name="ssh" accept'  
sudo firewall-cmd \--reload  
sudo firewall-cmd \--list-all

On `FTP-IDS01`:

sudo firewall-cmd \--list-all

Confirm only required ports/services are open.

---

## **Step 5 — VPN simulation**

Use one of the following depending on your lab.

### **Option A — pfSense site-to-site IPsec**

Create:

VPN A: FW-EDGE  
VPN B: R1 or second pfSense  
Encryption: AES256  
Hash: SHA256  
DH Group: 14 or higher  
Authentication: PSK for training, certificate if advanced

### **Option B — WireGuard training substitute**

If IPsec is too heavy for Week 2:

* Install WireGuard on `FW-EDGE` or Linux router.  
* Create peer between Inside and Outside test host.  
* Route test subnet over tunnel.

### **Option C — OpenVPN remote access**

Use pfSense OpenVPN wizard:

Remote user: vpnuser  
Pool: 10.250.250.0/24  
Access: Inside subnet only

Validation:

VPN client connects.  
VPN client can ping AD-SVR.  
VPN client can SSH to WEB-01 only if rule allows.

The 2019 Module A actual scores network equipment hardening and includes encrypted inter-branch or teleworker traffic as a requirement.

---

## **Step 6 — Full validation test**

The student must complete this matrix:

| Test | Expected Result | Evidence |
| ----- | ----- | ----- |
| WIN-CLI login to domain | Pass | Screenshot |
| `nslookup www.shenghai.ws` | Pass | Screenshot |
| `https://www.shenghai.org` | Pass | Screenshot |
| `http://www.shenghai.org` redirects | Pass | Screenshot |
| `/etc/passwd` URL blocked | 403 | Screenshot |
| Plain FTP | Fail | Screenshot |
| FTPS | Pass | Screenshot |
| Failed login appears in Splunk | Pass | Screenshot |
| `index=main jfrank` | Pass | Screenshot |
| IDS FTP alert | Pass | Screenshot |
| IDS ICMP alert | Pass | Screenshot |
| IDS malware alert | Pass | Screenshot |
| DMZ to LAN unrestricted access | Block | Screenshot |
| Outside to Inside access | Block | Screenshot |
| VPN connection | Pass | Screenshot |

---

## **Step 7 — Reboot validation**

Reboot in this order:

1. `AD-SVR`  
2. `SIEM01`  
3. `WEB-01`  
4. `FTP-IDS01`  
5. `FW-EDGE`  
6. `WIN-CLI`

After reboot, retest:

Domain login  
DNS  
HTTPS  
Redirect  
ModSecurity  
Splunk receiving  
IDS alert  
Firewall rules  
VPN

WorldSkills instructions require configurations to remain working after reboot.

---

## **Step 8 — Produce Week 2 evidence package**

Required folder:

WorldSkills-Cyber-Training/  
└── 02-Week2-Infrastructure/  
    ├── Day6-AD-DNS-GPO/  
    ├── Day7-Linux-Hardening/  
    ├── Day8-Public-Services/  
    ├── Day9-SIEM-IDS/  
    ├── Day10-Firewall-VPN/  
    └── Week2-Final-Report/

Required report:

Week2\_Infrastructure\_Security\_Report.pdf

Report sections:

1. Objective  
2. Topology  
3. AD/DNS/GPO implementation  
4. Linux hardening  
5. Public services protection  
6. SIEM and IDS monitoring  
7. Firewall rules  
8. VPN validation  
9. Reboot validation  
10. Issues and fixes  
11. Screenshots index  
12. Readiness statement

---

## **Day 10 Deliverables**

| Deliverable | Evidence |
| ----- | ----- |
| Firewall least privilege | Rules screenshot |
| Windows Firewall enabled | PowerShell screenshot |
| Linux firewall restricted | `firewall-cmd` screenshot |
| VPN working | Connection screenshot |
| Full validation matrix | Completed spreadsheet |
| Reboot validation | Checklist |
| Final report | PDF |
| Final snapshot | Screenshot |

Snapshot:

WEEK2\_INFRA\_SECURITY\_READY

---

# **Week 2 Final Checklist**

| Requirement | Status |
| ----- | ----- |
| AD domain created | ☐ |
| DNS records working | ☐ |
| Windows client joined | ☐ |
| OUs created | ☐ |
| Users created | ☐ |
| GPO password policy applied | ☐ |
| GPO lockout policy applied | ☐ |
| GPO login banner applied | ☐ |
| GPO audit policy applied | ☐ |
| Linux users created | ☐ |
| Linux sudo restrictions applied | ☐ |
| Linux password policy configured | ☐ |
| Root SSH disabled | ☐ |
| firewalld enabled | ☐ |
| SELinux enforcing | ☐ |
| auditd rules configured | ☐ |
| HTTPS website working | ☐ |
| HTTP redirects to HTTPS | ☐ |
| ModSecurity blocks `/etc/passwd` | ☐ |
| FTPS working | ☐ |
| Plain FTP blocked | ☐ |
| Splunk installed/running | ☐ |
| Splunk receives AD logs | ☐ |
| `jfrank` logs visible | ☐ |
| IDS FTP alert working | ☐ |
| IDS ICMP alert working | ☐ |
| IDS malware alert working | ☐ |
| Firewall rules documented | ☐ |
| VPN simulated | ☐ |
| Reboot test passed | ☐ |
| Evidence package completed | ☐ |

---

# **Mentor Scoring for Week 2**

Score every student out of 100:

| Area | Weight |
| ----- | ----- |
| AD/DNS/GPO | 20 |
| Linux hardening | 20 |
| Public services | 20 |
| SIEM/IDS | 20 |
| Firewall/VPN | 10 |
| Evidence quality | 10 |

Passing score:

80/100 minimum

Competition-ready score:

90/100+

---

# **Student Readiness Gate**

The student cannot proceed to Week 3 unless they can demonstrate these without mentor help:

1. Join a Windows client to the domain.  
2. Apply and validate GPO.  
3. Create Linux users and sudo rules.  
4. Disable root SSH.  
5. Configure firewalld.  
6. Confirm SELinux enforcing.  
7. Configure HTTPS and redirect.  
8. Block `/etc/passwd` with ModSecurity.  
9. Forward logs to Splunk.  
10. Trigger IDS alerts.  
11. Explain every firewall rule.  
12. Reboot and prove the lab still works.

# Week 3

This week is based mainly on **WorldSkills 2019 Module B** and **WorldSkills 2022 Module B**. The 2022 Module B scope includes **Vulnerability Detection and Repair, Incident Response, Identity and Access Management, Digital Forensic Investigation, Cryptography and PKI, and Code Review**. The marking scheme also confirms the scored outputs: web incident analysis, Windows autorun malware analysis, memory dump analysis, credit-card PCAP analysis, cryptography, IAM, and code review.

---

# **WEEK 3 — INCIDENT RESPONSE, DFIR, APPLICATION SECURITY**

## **Weekly Objective**

By the end of Week 3, the student must be able to:

1. Investigate a compromised Linux/Apache/WordPress-style web server.  
2. Investigate suspicious Windows persistence and autorun behavior.  
3. Repair vulnerable web application issues.  
4. Analyze memory dumps.  
5. Analyze PCAP files and extract indicators.  
6. Configure TLS using a 4096-bit RSA certificate.  
7. Harden SSH/IAM controls.  
8. Review vulnerable Python/PHP/C-style code.  
9. Complete an answer sheet with clear evidence.

WorldSkills expects the competitor to **read all tasks first, verify all devices, recover inaccessible systems, test before submission, and submit evidence**.

---

# **Week 3 Lab Design**

## **Required VMs**

Use your PC-only EVE-NG/GNS3/VMware lab.

| VM | Purpose | Suggested IP |
| ----- | ----- | ----- |
| `COMPETITOR-01` | Analyst workstation / FlareVM / tools | `100.100.128.50` |
| `COMPETITOR-02` | Backup analyst workstation | `100.100.128.40` |
| `IR-WEB` | Compromised Linux web server practice | `100.100.128.60` |
| `IR-WIN` | Compromised Windows server practice | `100.100.128.70` |
| `IR-LINUXM` | Linux malware/forensics practice server | `192.168.1.59` |
| `SIEM01` | Splunk / log storage | `192.168.10.20` |
| `KALI01` | Analysis helper only, not attacker mode | Isolated lab only |

The 2022 Module B topology uses competitor systems at `100.100.128.50` and `.40`, compromised web server at `.60`, and compromised Windows server at `.70`.

---

# **Required Tools**

Install or prepare these before Day 11:

## **On `COMPETITOR-01`**

* Wireshark  
* CyberChef  
* 7-Zip  
* Notepad++  
* VS Code  
* Sysinternals Suite  
* Autoruns  
* Process Explorer  
* Procmon  
* HashCalc / PowerShell hashing  
* Python 3  
* Volatility 3  
* Autopsy / FTK Imager  
* Browser  
* SSH client  
* RDP client

## **On Linux analyst VM**

sudo apt update  
sudo apt install \-y wireshark tshark tcpdump binwalk foremost strings grep ripgrep jq python3 python3-pip exiftool hashdeep yara

## **Week 3 Evidence Folder**

03-Week3-IR-DFIR-AppSec/  
├── Day11-Web-Incident-Response/  
├── Day12-Windows-Incident-Response/  
├── Day13-Vulnerability-Repair-IAM/  
├── Day14-Memory-PCAP-Forensics/  
├── Day15-Crypto-CodeReview-Mock/  
├── Evidence/  
│   ├── Screenshots/  
│   ├── Commands/  
│   ├── Hashes/  
│   ├── Logs/  
│   ├── PCAP/  
│   ├── Reports/  
│   └── Answer-Sheets/  
└── Week3-Final-Report/

---

# **Student Rules for Week 3**

1. **Do not run unknown malware. Analyze only.**  
2. **Do not attack external systems.**  
3. **Do not scan the office network.**  
4. **Do not delete evidence before copying it.**  
5. **Do not fix the system before collecting evidence.**  
6. **Every finding must include proof.**  
7. **Every answer must include exact path, filename, timestamp, hash, command, or screenshot where applicable.**  
8. **If no answer is found, write `Not found` and document the search path. Do not leave blanks.**  
9. **Use snapshots before remediation.**  
10. **Submit clean answer sheets, not raw notes.**

This matters because both 2019 and 2022 answer sheets require precise answers such as exploit URL, first successful attack time, infected file path, webshell code, autorun path, mutex string, process details, memory findings, PCAP findings, IAM proof, and code fixes.

---

# **Daily Training Structure**

| Time | Activity |
| ----- | ----- |
| 08:30–09:00 | Restore/check snapshot, open evidence folder |
| 09:00–10:30 | Mentor demo \+ methodology |
| 10:30–10:45 | Break |
| 10:45–12:30 | Hands-on investigation block 1 |
| 12:30–13:30 | Lunch |
| 13:30–15:30 | Hands-on investigation block 2 |
| 15:30–15:45 | Break |
| 15:45–17:00 | Remediation / validation / screenshots |
| 17:00–17:30 | Answer sheet writing |
| 17:30–18:00 | Mentor scoring and review |

---

# **DAY 11 — Web Server Incident Response**

## **Daily Objective**

Train the student to investigate a compromised Linux web server and produce exact incident-response answers.

The 2022 task requires the student to find exploit types, exploit URL, commands and parameters, first successful attack time, infected file path, webshell code, first reverse-shell command, attacker credentials, and file download URL.

---

## **Step 1 — Start from snapshot**

Snapshot to restore:

WEEK2\_INFRA\_SECURITY\_READY

Create new snapshot before investigation:

W3D11\_WEB\_IR\_START

Student instruction:

Do not remediate yet. Evidence first, repair later.

---

## **Step 2 — Confirm system access**

From `COMPETITOR-01`:

ping 100.100.128.60  
ssh root@100.100.128.60

If using 2019-style lab:

ping 192.168.1.55  
ssh root@192.168.1.55

Capture:

W3D11\_WebServer\_Access\_01.png

---

## **Step 3 — Preserve evidence**

On `IR-WEB`:

mkdir \-p /root/ir-evidence  
date | tee /root/ir-evidence/system\_time.txt  
hostnamectl | tee /root/ir-evidence/hostname.txt  
ip a | tee /root/ir-evidence/ip\_address.txt  
last \-a | tee /root/ir-evidence/last\_logins.txt  
who \-a | tee /root/ir-evidence/who.txt

Archive logs before changing anything:

cp \-a /var/log /root/ir-evidence/var-log-copy

Archive web directories:

cp \-a /var/www /root/ir-evidence/var-www-copy 2\>/dev/null  
cp \-a /home/wwwroot /root/ir-evidence/home-wwwroot-copy 2\>/dev/null

Evidence:

W3D11\_Evidence\_Preserved\_01.png

---

## **Step 4 — Review web logs**

Check common log locations:

ls \-lah /var/log/httpd/  
ls \-lah /var/log/apache2/  
ls \-lah /home/wwwlogs/

Search suspicious requests:

grep \-R "POST" /var/log/httpd/ /var/log/apache2/ /home/wwwlogs/ 2\>/dev/null | head \-50  
grep \-R "cmd=" /var/log/httpd/ /var/log/apache2/ /home/wwwlogs/ 2\>/dev/null  
grep \-R "shell" /var/log/httpd/ /var/log/apache2/ /home/wwwlogs/ 2\>/dev/null  
grep \-R "wget\\|curl\\|bash\\|python\\|nc\\|php" /var/log/httpd/ /var/log/apache2/ /home/wwwlogs/ 2\>/dev/null

Student must fill:

| Required Answer | Evidence |
| ----- | ----- |
| Exploit type | Log line |
| Exploit URL | Full URL |
| Commands and parameters | Log line |
| First successful attack time | Timestamp |
| File download URL | Request line |

---

## **Step 5 — Search for suspicious web files**

find /var/www /home/wwwroot \-type f \-mtime \-30 \-ls 2\>/dev/null  
find /var/www /home/wwwroot \-type f \\( \-name "\*.php" \-o \-name "\*.phtml" \-o \-name "\*.txt" \\) \-print 2\>/dev/null  
grep \-R "eval\\|base64\_decode\\|shell\_exec\\|system\\|passthru\\|assert\\|preg\_replace" /var/www /home/wwwroot 2\>/dev/null

Do not execute suspicious PHP.

Capture:

W3D11\_Suspicious\_WebFile\_Search\_01.png

---

## **Step 6 — Identify infected file**

For each suspicious file:

stat /path/to/suspicious\_file  
sha256sum /path/to/suspicious\_file  
sed \-n '1,120p' /path/to/suspicious\_file

Student must record:

Filename:  
Absolute path:  
File owner:  
Modified time:  
SHA256:  
Suspicious code:  
Related URL:

---

## **Step 7 — Review shell history and authentication logs**

cat \~/.bash\_history 2\>/dev/null  
grep \-i "Accepted\\|Failed\\|session opened\\|sudo" /var/log/secure /var/log/auth.log 2\>/dev/null  
last \-a  
lastlog

Student must identify:

| Required Answer | Source |
| ----- | ----- |
| Username used by attacker | auth log / shell history |
| Password evidence, if found | app config / logs / notes |
| Related actions | command history / process logs |
| First command after shell | history/log correlation |

Do not guess. If not found, document search evidence.

---

## **Step 8 — Build incident timeline**

Create:

W3D11\_Web\_Incident\_Timeline.xlsx

Columns:

Time | Source | Event | Evidence File | Confidence | Notes

Minimum entries:

1. First suspicious request  
2. Exploit request  
3. File write / webshell creation  
4. Reverse shell event, if visible  
5. Credential use  
6. Download request  
7. Remediation candidate

---

## **Day 11 Deliverables**

| Deliverable | Required Evidence |
| ----- | ----- |
| Web incident timeline | Spreadsheet |
| Exploit type and URL | Log screenshot |
| First attack time | Log screenshot |
| Infected file path | Terminal screenshot |
| Webshell code evidence | Screenshot/text copy |
| First reverse-shell command | Log/history evidence |
| Attacker username/password | Evidence or documented not found |
| Download URL | Log screenshot |
| Daily answer sheet section | Completed draft |

Snapshot:

W3D11\_WEB\_IR\_COMPLETE

---

# **DAY 12 — Windows Server Incident Response**

## **Daily Objective**

Investigate Windows persistence, autorun behavior, suspicious process parameters, registry artifacts, and malware indicators.

The 2022 task requires filename/path of malicious autorun, mutex string, registry key value, file created by malware, process name/parameters, and registry functions used. The 2019 task similarly asks for screen-locking malware path, SHA1, stager path, and attack steps.

---

## **Step 1 — Start clean**

Restore or snapshot:

W3D12\_WINDOWS\_IR\_START

From `COMPETITOR-01`:

ping 100.100.128.70  
mstsc /v:100.100.128.70

Evidence:

W3D12\_WindowsServer\_Access\_01.png

---

## **Step 2 — Collect volatile triage data**

On `IR-WIN`, open PowerShell as Administrator:

mkdir C:\\IR-Evidence

Get-Date | Out-File C:\\IR-Evidence\\system\_time.txt  
hostname | Out-File C:\\IR-Evidence\\hostname.txt  
ipconfig /all | Out-File C:\\IR-Evidence\\ipconfig.txt  
tasklist /v | Out-File C:\\IR-Evidence\\tasklist.txt  
netstat \-ano | Out-File C:\\IR-Evidence\\netstat.txt  
whoami /all | Out-File C:\\IR-Evidence\\whoami.txt

---

## **Step 3 — Check autorun locations**

Use Sysinternals Autoruns if available.

CLI checks:

reg query HKLM\\Software\\Microsoft\\Windows\\CurrentVersion\\Run  
reg query HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\Run  
reg query HKLM\\Software\\Microsoft\\Windows\\CurrentVersion\\RunOnce  
reg query HKCU\\Software\\Microsoft\\Windows\\CurrentVersion\\RunOnce

Check startup folders:

dir "$env:APPDATA\\Microsoft\\Windows\\Start Menu\\Programs\\Startup"  
dir "C:\\ProgramData\\Microsoft\\Windows\\Start Menu\\Programs\\Startup"

Check scheduled tasks:

schtasks /query /fo LIST /v \> C:\\IR-Evidence\\scheduled\_tasks.txt

Student must record:

Autorun path:  
Registry key:  
Filename:  
Timestamp:  
User context:  
Evidence screenshot:

---

## **Step 4 — Identify suspicious processes**

Get-Process | Sort-Object CPU \-Descending | Select-Object \-First 20  
wmic process get ProcessId,ParentProcessId,CommandLine,ExecutablePath \> C:\\IR-Evidence\\process\_cmdline.txt

Review:

notepad C:\\IR-Evidence\\process\_cmdline.txt

Student must identify:

| Required Answer | Evidence |
| ----- | ----- |
| Process name | Process list |
| Process parameters | CommandLine |
| Parent process | PPID |
| Executable path | ExecutablePath |

---

## **Step 5 — Hash suspicious files**

For suspicious file:

Get-FileHash "C:\\Path\\SuspiciousFile.exe" \-Algorithm SHA1  
Get-FileHash "C:\\Path\\SuspiciousFile.exe" \-Algorithm SHA256

Evidence:

W3D12\_SuspiciousFile\_SHA1\_01.png

---

## **Step 6 — Registry function analysis**

Using static analysis only:

* Open the suspicious executable in PEStudio, Detect It Easy, or strings.  
* Do not execute it.  
* Search strings:

strings.exe C:\\Path\\SuspiciousFile.exe \> C:\\IR-Evidence\\suspicious\_strings.txt  
notepad C:\\IR-Evidence\\suspicious\_strings.txt

Look for registry APIs:

RegCreateKeyEx  
RegSetValueEx  
RegOpenKeyEx  
RegQueryValueEx  
RegDeleteValue  
RegCloseKey

Student must record:

Registry function names:  
How found:  
Screenshot:

---

## **Step 7 — Create Windows incident timeline**

Create:

W3D12\_Windows\_Incident\_Timeline.xlsx

Columns:

Time | Artifact | Path/Key | Process | User | Evidence | Notes

---

## **Day 12 Deliverables**

| Deliverable | Required Evidence |
| ----- | ----- |
| Autorun path and filename | Registry screenshot |
| Mutex string | Static-analysis screenshot |
| Registry key/value | Registry screenshot |
| Created file name | Filesystem evidence |
| Process name/parameters | WMIC screenshot |
| Registry functions | Strings/PEStudio screenshot |
| SHA1/SHA256 | PowerShell screenshot |
| Timeline | Spreadsheet |
| Answer sheet section | Completed draft |

Snapshot:

W3D12\_WINDOWS\_IR\_COMPLETE

---

# **DAY 13 — Vulnerability Repair \+ IAM**

## **Daily Objective**

Repair identified weaknesses and provide before/after evidence. This is where students lose marks if they fix without proof.

The 2022 answer sheet requires **before/after bug-fix evidence**, while the IAM task requires blocking root SSH, changing root password, and allowing `ixia` to sudo using their own password. The 2019 Module B task also requires PHP hardening, MySQL import/export restrictions, management-tool removal, weak-password repair, Windows malware removal, admin password change, and blocking 3389 from the web server.

---

## **Step 1 — Snapshot before remediation**

W3D13\_PRE\_REMEDIATION

Student instruction:

Capture "before" proof before applying any fix.

---

## **Step 2 — Web bug repair worksheet**

Create:

W3D13\_Web\_Vulnerability\_Repair.xlsx

Columns:

Bug ID | Exploit Type | Vulnerable File | Vulnerable Code | Test Before | Fix Applied | Test After | Evidence

Minimum three bug categories to practice:

| Bug Type | What to look for |
| ----- | ----- |
| Command injection | unsafe shell calls |
| SQL injection | string-concatenated SQL |
| File upload/path issue | unsafe extension/path handling |
| Weak password | predictable/default credential |
| Insecure admin tool | exposed management directory |

---

## **Step 3 — PHP dangerous function hardening**

On `IR-WEB`:

php \--ini  
sudo cp /etc/php.ini /etc/php.ini.bak 2\>/dev/null || true  
sudo cp /etc/php/\*/apache2/php.ini /tmp/php.ini.bak 2\>/dev/null || true

Find PHP config:

php \-i | grep "Loaded Configuration File"

Edit loaded `php.ini`:

disable\_functions \= exec,passthru,shell\_exec,system,proc\_open,popen,curl\_exec,curl\_multi\_exec,parse\_ini\_file,show\_source

Restart Apache:

sudo systemctl restart httpd 2\>/dev/null || sudo systemctl restart apache2

Validate:

php \-i | grep disable\_functions

Evidence:

W3D13\_PHP\_DisableFunctions\_Before\_01.png  
W3D13\_PHP\_DisableFunctions\_After\_01.png

---

## **Step 4 — MySQL import/export restriction**

Check MySQL/MariaDB config:

mysql \--version  
grep \-R "secure\_file\_priv" /etc/my.cnf /etc/mysql/ 2\>/dev/null

Add under `[mysqld]`:

secure\_file\_priv \= /var/lib/mysql-files  
local\_infile \= 0

Restart:

sudo systemctl restart mariadb 2\>/dev/null || sudo systemctl restart mysql

Validate:

mysql \-e "SHOW VARIABLES LIKE 'secure\_file\_priv';"  
mysql \-e "SHOW VARIABLES LIKE 'local\_infile';"

Evidence:

W3D13\_MySQL\_Restriction\_01.png

---

## **Step 5 — Remove exposed management tools**

Search for exposed admin panels:

find /var/www /home/wwwroot \-type d \\( \-iname "\*phpmyadmin\*" \-o \-iname "\*adminer\*" \-o \-iname "\*manager\*" \-o \-iname "\*backup\*" \\) 2\>/dev/null

Do not delete immediately. First archive:

sudo mkdir \-p /root/ir-evidence/removed-management-tools  
sudo tar \-czf /root/ir-evidence/removed-management-tools/admin-tool-backup.tgz /path/to/tool

Then remove or restrict:

sudo rm \-rf /path/to/tool

Evidence:

W3D13\_ManagementTool\_Removed\_01.png

---

## **Step 6 — IAM: disable root SSH**

Check current config:

sudo grep \-i "^PermitRootLogin" /etc/ssh/sshd\_config

Edit:

sudo nano /etc/ssh/sshd\_config

Set:

PermitRootLogin no

Restart:

sudo systemctl restart sshd 2\>/dev/null || sudo systemctl restart ssh

Test from `COMPETITOR-01`:

ssh root@100.100.128.60

Expected:

Permission denied

Evidence:

W3D13\_RootSSH\_Blocked\_01.png

---

## **Step 7 — IAM: change root password**

On `IR-WEB` console or existing privileged session:

sudo passwd root

Use lab password policy. Document command used but **do not put the real password in screenshots** unless the competition task explicitly requires it.

---

## **Step 8 — IAM: allow `ixia` to sudo to root**

Create user if needed:

sudo useradd ixia 2\>/dev/null || true  
sudo passwd ixia

Use `visudo`:

sudo visudo

Add:

ixia ALL=(ALL) ALL

Validate:

su \- ixia  
sudo \-l  
sudo whoami

Expected:

root

Evidence:

W3D13\_ixia\_sudo\_01.png

The 2022 marking scheme checks that root cannot SSH, root password is changed, and `ixia` can sudo.

---

## **Step 9 — Windows remediation practice**

On `IR-WIN`:

1. Disable or remove malicious autorun entries after evidence.  
2. Change local admin password if required:

net user Administrator AppServer123\!@\#

3. Block RDP from web server IP:

New-NetFirewallRule \-DisplayName "Block RDP from Web Server" \`  
\-Direction Inbound \`  
\-Protocol TCP \`  
\-LocalPort 3389 \`  
\-RemoteAddress 100.100.128.60 \`  
\-Action Block

Validate:

Get-NetFirewallRule \-DisplayName "Block RDP from Web Server"

Evidence:

W3D13\_RDP\_Block\_3389\_01.png

---

## **Day 13 Deliverables**

| Deliverable | Required Evidence |
| ----- | ----- |
| PHP dangerous functions disabled | Before/after screenshot |
| MySQL import/export restricted | Config and query screenshot |
| Admin tool removed/restricted | Path evidence |
| Weak-password issue repaired | Before/after proof |
| Root SSH disabled | Failed SSH screenshot |
| Root password changed | Command proof |
| `ixia` sudo works | `sudo -l` screenshot |
| RDP block rule | Firewall screenshot |
| Remediation worksheet | Completed |

Snapshot:

W3D13\_REMEDIATION\_IAM\_COMPLETE

---

# **DAY 14 — Memory Dump and PCAP Forensics**

## **Daily Objective**

Train host and network forensic analysis using memory dumps and packet captures.

The 2022 task requires identifying the malicious program name, PID/PPID, attempted connection IPs and destination port, executing user, another malicious program to delete, plus extraction of three credit-card details from PCAPs. The marking scheme confirms these as scored items.

---

## **Step 1 — Prepare forensic workspace**

On `COMPETITOR-01` or Linux analyst VM:

W3D14-Forensics/  
├── memory/  
├── pcap/  
├── exports/  
├── screenshots/  
└── notes/

Student instruction:

Never analyze evidence from the original location. Work from a copy.

---

## **Step 2 — Copy evidence and calculate hashes**

Linux:

sha256sum dump.vmem | tee memory/dump\_vmem\_sha256.txt  
sha256sum creditcard1.pcap creditcard2.pcap creditcard3.pcap | tee pcap/pcap\_hashes.txt

Windows PowerShell:

Get-FileHash .\\dump.vmem \-Algorithm SHA256  
Get-FileHash .\\creditcard1.pcap \-Algorithm SHA256

Evidence:

W3D14\_Evidence\_Hashes\_01.png

---

## **Step 3 — Memory dump triage with Volatility 3**

Run:

vol \-f dump.vmem windows.info  
vol \-f dump.vmem windows.pslist  
vol \-f dump.vmem windows.pstree  
vol \-f dump.vmem windows.cmdline  
vol \-f dump.vmem windows.netscan  
vol \-f dump.vmem windows.envars

Student must identify:

| Required Answer | Volatility Source |
| ----- | ----- |
| Malicious process name | `pslist`, `pstree`, `cmdline` |
| PID and PPID | `pslist`, `pstree` |
| Connection IPs and ports | `netscan` |
| User who executed malware | `sessions`, `getsids`, profile artifacts |
| Related malware to delete | process/tree/file correlation |

---

## **Step 4 — Extract suspicious process artifacts**

Use only if needed:

vol \-f dump.vmem windows.filescan \> memory/filescan.txt  
vol \-f dump.vmem windows.dlllist \--pid \<PID\> \> memory/dlllist\_PID.txt  
vol \-f dump.vmem windows.handles \--pid \<PID\> \> memory/handles\_PID.txt

Do not run extracted malware. Hash it only.

sha256sum extracted\_file.bin  
strings extracted\_file.bin | head \-100

---

## **Step 5 — PCAP triage in Wireshark**

Open `creditcard1.pcap`.

Initial filters:

http  
tcp  
dns  
ip.addr \== x.x.x.x  
frame contains "4111"  
frame contains "card"  
frame contains "cc"  
frame contains "visa"  
frame contains "master"

Use:

File \> Export Objects \> HTTP  
Statistics \> Conversations  
Statistics \> Protocol Hierarchy  
Follow TCP Stream

Student must extract:

Creditcard1.pcap:  
\- Card number:  
\- Expiry:  
\- CVV if visible:  
\- Destination IP:  
\- Protocol:  
\- Evidence screenshot:

Repeat for `creditcard2.pcap` and `creditcard3.pcap`.

---

## **Step 6 — PCAP triage using tshark**

tshark \-r creditcard1.pcap \-Y "http" \-T fields \-e frame.time \-e ip.src \-e ip.dst \-e http.request.full\_uri  
tshark \-r creditcard1.pcap \-Y 'frame contains "card"'  
tshark \-r creditcard1.pcap \-Y 'frame contains "4111"'

Export streams if needed:

tshark \-r creditcard1.pcap \-q \-z follow,tcp,ascii,0

---

## **Day 14 Deliverables**

| Deliverable | Required Evidence |
| ----- | ----- |
| Evidence hashes | Screenshot/text file |
| Memory profile/info | Volatility screenshot |
| Malicious process name | Screenshot |
| PID/PPID | Screenshot |
| Destination IP/port | Screenshot |
| Executing user | Screenshot |
| Related malware | Evidence |
| Credit card details from 3 PCAPs | Wireshark/tshark proof |
| DFIR worksheet | Completed |

Snapshot:

W3D14\_DFIR\_COMPLETE

---

# **DAY 15 — Cryptography, PKI, Code Review, Final Mock**

## **Daily Objective**

Complete Week 3 with TLS/PKI implementation, IAM validation, secure-code review, and a timed mini-mock.

The 2022 task requires creating a **self-signed 4096-bit RSA certificate**, configuring the web server for HTTP and HTTPS, and redirecting HTTP to HTTPS. It also requires code review answers: vulnerable line, why unsafe, how to secure, and corrected code.

---

## **Step 1 — Create 4096-bit RSA certificate**

On `IR-WEB`:

sudo mkdir \-p /etc/ssl/wsc-week3  
sudo openssl req \-x509 \-nodes \-newkey rsa:4096 \\  
\-keyout /etc/ssl/wsc-week3/ir-web.key \\  
\-out /etc/ssl/wsc-week3/ir-web.crt \\  
\-days 365 \\  
\-subj "/C=PH/O=WorldSkills Training/CN=100.100.128.60"

Verify:

openssl x509 \-in /etc/ssl/wsc-week3/ir-web.crt \-text \-noout | grep \-E "Subject:|Public-Key"

Evidence:

W3D15\_RSA4096\_Cert\_01.png

---

## **Step 2 — Configure HTTP and HTTPS**

Apache CentOS/Rocky:

sudo nano /etc/httpd/conf.d/ir-web-ssl.conf

Example:

\<VirtualHost \*:80\>  
    ServerName 100.100.128.60  
    Redirect permanent / https://100.100.128.60/  
\</VirtualHost\>

\<VirtualHost \*:443\>  
    ServerName 100.100.128.60  
    DocumentRoot /var/www/html

    SSLEngine on  
    SSLCertificateFile /etc/ssl/wsc-week3/ir-web.crt  
    SSLCertificateKeyFile /etc/ssl/wsc-week3/ir-web.key  
\</VirtualHost\>

Restart:

sudo apachectl configtest  
sudo systemctl restart httpd 2\>/dev/null || sudo systemctl restart apache2

Validate:

curl \-I http://100.100.128.60  
curl \-k \-I https://100.100.128.60

Expected:

HTTP returns redirect to HTTPS  
HTTPS returns 200/301/302 depending site config

---

## **Step 3 — Upload/save evidence**

Save:

ir-web.crt  
ir-web.key  
Apache config file  
curl output  
browser screenshots

Note: In a real competition, key/cert upload may be requested. In training, keep them in the evidence folder.

---

## **Step 4 — Code review methodology**

Student must use this structure:

| Field | Required Content |
| ----- | ----- |
| Code snippet and line number | Exact line |
| Vulnerability identity | Example: SQL Injection, Command Injection, Buffer Overflow |
| Why unsafe | Specific reason |
| How to secure | Principle |
| Safe code | Only changed line(s) |

The 2022 answer sheet explicitly uses this format for Python Code Review.

---

## **Step 5 — Practice code review categories**

Use safe training examples only. The student must identify and rewrite, not exploit.

Practice topics:

1. SQL injection  
   * Fix using parameterized queries.  
2. Command injection  
   * Fix by avoiding shell calls or using argument arrays.  
3. Path traversal  
   * Fix by normalizing paths and allowlisting directories.  
4. Unsafe deserialization  
   * Fix by using safe formats and validation.  
5. Weak randomness  
   * Fix using cryptographic random functions.  
6. Hardcoded secrets  
   * Fix using environment variables or secret storage.  
7. Buffer overflow in C  
   * Fix unsafe input functions.

2019 Module B also includes code review tasks requiring the competitor to identify vulnerable lines, name the attack, explain how to secure it, and provide secure code.

---

## **Step 6 — Timed mini-mock**

Run a 3-hour mini simulation.

### **Hour 1 — Web \+ Windows IR**

Student must answer:

Exploit URL:  
First attack time:  
Infected file:  
Webshell code:  
Autorun path:  
Suspicious process:

### **Hour 2 — DFIR**

Student must answer:

Malicious process:  
PID:  
PPID:  
Destination IP:  
Destination port:  
Credit card data from PCAP:

### **Hour 3 — Remediation \+ Code Review**

Student must complete:

PHP hardening:  
MySQL restriction:  
Root SSH disabled:  
ixia sudo proof:  
RSA 4096 certificate proof:  
One code review answer:

---

## **Day 15 Deliverables**

| Deliverable | Required Evidence |
| ----- | ----- |
| 4096-bit certificate | OpenSSL screenshot |
| HTTP/HTTPS config | Config screenshot |
| HTTP redirects to HTTPS | curl/browser proof |
| IAM proof | root SSH fail \+ ixia sudo |
| Code review worksheet | Completed |
| Mini-mock answer sheet | Completed |
| Week 3 final report | PDF |

Snapshot:

WEEK3\_IR\_DFIR\_APPSEC\_READY

---

# **Week 3 Final Checklist**

| Requirement | Status |
| ----- | ----- |
| Web logs preserved | ☐ |
| Web exploit URL identified | ☐ |
| First successful attack time identified | ☐ |
| Infected file path identified | ☐ |
| Webshell code documented | ☐ |
| First post-shell command identified | ☐ |
| Attacker credentials documented | ☐ |
| Download URL identified | ☐ |
| Windows autorun path identified | ☐ |
| Mutex string identified | ☐ |
| Registry key/value identified | ☐ |
| Malicious process and parameters identified | ☐ |
| Registry functions identified | ☐ |
| PHP dangerous functions disabled | ☐ |
| MySQL import/export restricted | ☐ |
| Exposed management tool removed/restricted | ☐ |
| Root SSH blocked | ☐ |
| Root password changed | ☐ |
| ixia sudo validated | ☐ |
| Memory dump analyzed | ☐ |
| Malicious process PID/PPID identified | ☐ |
| Network connections identified | ☐ |
| PCAP credit-card details extracted | ☐ |
| RSA 4096 certificate created | ☐ |
| HTTP/HTTPS configured | ☐ |
| HTTP redirects to HTTPS | ☐ |
| Code review completed | ☐ |
| Answer sheet completed | ☐ |
| Final evidence package completed | ☐ |

---

# **Week 3 Mentor Scoring**

Score out of 100:

| Area | Weight |
| ----- | ----- |
| Web incident response | 20 |
| Windows incident response | 15 |
| Vulnerability repair | 15 |
| Memory forensics | 15 |
| PCAP analysis | 10 |
| Crypto/IAM | 10 |
| Code review | 10 |
| Evidence quality | 5 |

Passing score:

80/100

Competition-ready score:

90/100+

---

# **Student Readiness Gate**

The student is not allowed to proceed to Week 4 unless they can do these without mentor help:

1. Build a timeline from web logs.  
2. Identify suspicious PHP/web files without executing them.  
3. Find Windows autorun persistence.  
4. Hash suspicious files.  
5. Use Volatility to identify process, PID, PPID, and connections.  
6. Use Wireshark to follow TCP streams and extract evidence.  
7. Apply before/after remediation screenshots.  
8. Configure SSH root-blocking and sudo access.  
9. Create a 4096-bit RSA certificate.  
10. Complete code review answers cleanly.  
11. Submit a complete answer sheet with no blanks.

Week 4 is Red Team/Blue Team speed training. Week 3 must lock in the answer-sheet discipline: **exact evidence, exact wording, exact proof**.

# Week 4

# **WEEK 4 — RED TEAM, BLUE TEAM, FINAL MOCK COMPETITION**

## **Weekly Objective**

By the end of Week 4, the student must be able to:

1. Perform structured reconnaissance inside an isolated CTF lab.  
2. Identify web, database, Windows, Linux, crypto, and stego flags.  
3. Analyze firewall logs, Splunk logs, and PCAP traffic.  
4. Detect reconnaissance, malicious URLs, exploits, botnet-like traffic, and data leakage.  
5. Submit flags accurately.  
6. Work under time pressure.  
7. Avoid disqualification mistakes.

The 2022 Module C Red Team task uses CTF flags in `icc{...}` format and covers **database, enumeration, privilege escalation, web-based attacks, and Windows-based attacks**. It also states that the infrastructure layout is intentionally hidden because reconnaissance is part of the challenge.

The 2022 Module D Blue Team task requires competitors to find flags by inspecting traffic through the **inside and outside firewall interfaces**, with traffic generated by a benign/malicious traffic generator. It also provides firewall and Splunk access for investigation.

---

# **Week 4 Lab Design**

## **Lab A — Red Team CTF Network**

RED-CTF-NET: 10.10.10.0/24

KALI01        10.10.10.10     Red Team workstation  
RED-WEB01     10.10.10.20     Web vulnerable target  
RED-DB01      10.10.10.30     Database target  
RED-LINUX01   10.10.10.40     Linux privilege target  
RED-WIN01     10.10.10.50     Windows target  
RED-FILES01   10.10.10.60     Crypto / stego / file target

## **Lab B — Blue Team Monitoring Network**

BLUE-MON-NET: 172.16.10.0/24

FLARE01       172.16.10.10    Analyst workstation  
SPLUNK01      172.16.10.20    SIEM  
FW-BLUE       172.16.10.1     pfSense / OPNsense firewall  
TG-01         172.16.10.30    Traffic generator  
BLUE-WEB01    172.16.10.40    Internal web service  
BLUE-APP01    172.16.10.50    Application service  
BLUE-DB01     172.16.10.60    Database service

## **Management Network Rule**

100.100.0.0/16 \= Management network  
Do not scan.  
Do not attack.  
Do not brute force.  
Do not test exploits.

The 2022 Red Team rules explicitly state that no flags exist on `100.100.0.0/16`, and attacking/scanning the management or CySOP/scoring infrastructure can lead to penalty or disqualification.

---

# **Student Rules for Week 4**

1. Only attack assigned lab targets.  
2. Never scan `100.100.0.0/16`.  
3. Never scan the office network.  
4. No DDoS, stress testing, crash testing, or destructive payloads.  
5. Every command must be logged.  
6. Every flag must include source proof.  
7. If a flag is rejected, mark it as invalid and move on.  
8. Do not share flags with other teams.  
9. Use only lab wordlists and lab-provided credentials.  
10. Stop brute-force attempts if account lockout or service instability occurs.

The 2019 and 2022 CTF instructions both warn against DDoS or attacking scoring/management systems.

---

# **Week 4 Folder Structure**

04-Week4-CTF/  
├── Day16-Red-Recon/  
├── Day17-Red-Web-DB-Windows/  
├── Day18-Red-PrivEsc-Crypto-Stego/  
├── Day19-Blue-Splunk-Firewall-PCAP/  
├── Day20-Final-Mock/  
├── Evidence/  
│   ├── Screenshots/  
│   ├── Command-Logs/  
│   ├── Flags/  
│   ├── PCAPs/  
│   ├── Splunk-Searches/  
│   └── Reports/  
└── Week4-Final-Scorecard/

## **Flag Log Format**

Flag No:  
Flag Value:  
Category:  
Target IP:  
Where Found:  
Tool Used:  
Command/Search Used:  
Screenshot:  
Status: Accepted / Rejected / Pending  
Notes:

---

# **DAY 16 — Red Team Lab Build \+ Reconnaissance**

## **Daily Objective**

Build the Red Team CTF lab and train the student to perform structured reconnaissance without guessing.

2019 Module C scored Red Team categories include **Enumeration, Web Based Attacks, Database Attacks, Windows Attacks, Root Access, Cryptography, and Steganography**.

---

## **Step 1 — Restore Week 4 base snapshot**

Snapshot:

WEEK3\_IR\_DFIR\_APPSEC\_READY

Create new snapshot:

W4D16\_RED\_CTF\_START

---

## **Step 2 — Create Red CTF network**

In EVE-NG/GNS3, create:

RED-CTF-SW  
Subnet: 10.10.10.0/24

Connect:

KALI01  
RED-WEB01  
RED-DB01  
RED-LINUX01  
RED-WIN01  
RED-FILES01

---

## **Step 3 — Configure IP addresses**

| Node | IP | Gateway |
| ----- | ----- | ----- |
| KALI01 | `10.10.10.10/24` | none or firewall |
| RED-WEB01 | `10.10.10.20/24` | none |
| RED-DB01 | `10.10.10.30/24` | none |
| RED-LINUX01 | `10.10.10.40/24` | none |
| RED-WIN01 | `10.10.10.50/24` | none |
| RED-FILES01 | `10.10.10.60/24` | none |

Keep this network isolated. Internet access is not required.

---

## **Step 4 — Prepare target services**

### **RED-WEB01**

Install one or more intentionally vulnerable lab apps:

DVWA  
OWASP Juice Shop  
WordPress practice instance  
Simple PHP login form

Place sample flags in:

/var/www/html/robots.txt  
/var/www/html/backup/  
HTML comments  
HTTP headers  
Hidden admin page

Example training flag:

icc{red\_web\_recon\_flag01}

---

### **RED-DB01**

Prepare:

MariaDB/MySQL  
Test database  
Weak lab-only account  
Backup file containing fake flag

Example locations:

/backup/db\_backup.sql  
mysql table named flags

---

### **RED-LINUX01**

Prepare:

Normal user account  
Misconfigured sudo rule for lab only  
Readable backup file  
Flag under /home/user/  
Root-only flag under /root/

---

### **RED-WIN01**

Prepare:

SMB share  
Windows local users  
Hidden text file  
Scheduled task clue  
Registry clue

---

### **RED-FILES01**

Prepare:

Images with metadata clues  
ZIP file with weak lab password  
Base64/hex encoded strings  
Audio or image stego challenge  
Hash challenge

---

## **Step 5 — Student reconnaissance workflow**

On KALI01:

mkdir \-p \~/w4-red/{scans,notes,flags,screenshots}  
cd \~/w4-red

Identify interface:

ip a  
ip route

Host discovery:

nmap \-sn 10.10.10.0/24 \-oA scans/day16\_ping\_sweep

Service discovery:

nmap \-sV \-sC \-Pn 10.10.10.20-60 \-oA scans/day16\_services

Quick web check:

curl \-I http://10.10.10.20  
curl http://10.10.10.20/robots.txt

Directory discovery, lab only:

gobuster dir \-u http://10.10.10.20 \-w /usr/share/wordlists/dirb/common.txt \-o scans/web\_dirs.txt

SMB enumeration, lab only:

enum4linux-ng 10.10.10.50 | tee scans/win\_enum.txt

---

## **Step 6 — Create target matrix**

Student must complete:

| Target | Open Ports | Services | Possible Path | Priority |
| ----- | ----- | ----- | ----- | ----- |
| 10.10.10.20 |  |  | Web | High |
| 10.10.10.30 |  |  | DB | Medium |
| 10.10.10.40 |  |  | Linux | High |
| 10.10.10.50 |  |  | Windows | Medium |
| 10.10.10.60 |  |  | Crypto/Files | Medium |

---

## **Day 16 Deliverables**

| Deliverable | Evidence |
| ----- | ----- |
| Red CTF topology | Screenshot |
| Target IP table | Spreadsheet |
| Ping sweep | Command output |
| Service scan | Nmap result |
| Web directory result | Gobuster result |
| SMB result | Enum output |
| Target matrix | Completed |
| Snapshot | `W4D16_RED_RECON_READY` |

---

# **DAY 17 — Red Team Web, Database, and Windows Challenges**

## **Daily Objective**

Train controlled CTF exploitation workflows for web, database, and Windows targets inside the lab only.

The 2022 Red Team module expects participants to apply Red Team TTPs to find flags hidden around the infrastructure, with flags possibly obfuscated.

---

## **Step 1 — Web application testing workflow**

Student must follow this sequence:

1\. Identify technology  
2\. Check robots.txt and sitemap  
3\. Enumerate directories  
4\. Check page source  
5\. Check forms  
6\. Check cookies and headers  
7\. Check upload features  
8\. Check authentication logic  
9\. Check parameter handling  
10\. Document possible flag locations

Useful commands:

whatweb http://10.10.10.20  
curl \-i http://10.10.10.20  
curl http://10.10.10.20/robots.txt

Browser checks:

View Source  
Developer Tools  
Cookies  
Local Storage  
Response Headers

Burp Suite Community may be used only against:

10.10.10.20  
10.10.10.30  
10.10.10.50

---

## **Step 2 — Web flag locations to train**

Mentor should hide flags in:

HTML comment  
robots.txt  
backup.zip  
/admin page  
HTTP response header  
JavaScript file  
Error message  
Weak login page

Student must log each flag with proof.

---

## **Step 3 — Database enumeration workflow**

From Kali:

nmap \-sV \-p 3306 10.10.10.30

If lab credentials are provided:

mysql \-h 10.10.10.30 \-u labuser \-p

Inside MySQL:

SHOW DATABASES;  
USE training;  
SHOW TABLES;  
SELECT \* FROM flags;

Student must not brute force database credentials unless the mentor explicitly provides a small lab wordlist.

---

## **Step 4 — Windows enumeration workflow**

From Kali:

nmap \-sV \-sC \-p 135,139,445,3389 10.10.10.50 \-oA scans/win\_target  
smbclient \-L //10.10.10.50/ \-N

If lab credentials are provided:

smbclient //10.10.10.50/Public \-U labuser

Check:

SMB shares  
Readable files  
Backup files  
Notes files  
Registry clue files  
Scheduled-task clue files

---

## **Step 5 — Student flag discipline**

For every flag:

Screenshot before submission.  
Copy exact flag.  
If format is icc{abc}, submit only abc if scoring platform requires omission.  
If rejected, document rejected value.  
Do not keep retrying blindly.

The 2022 Red Team document states that flags should be submitted without the `icc{}` identifier in CySOP.

---

## **Day 17 Deliverables**

| Deliverable | Evidence |
| ----- | ----- |
| Web test notes | Completed |
| Database test notes | Completed |
| Windows test notes | Completed |
| Minimum 5 practice flags | Flag log |
| Screenshots for each flag | Evidence folder |
| Rejected flag list | If applicable |
| Snapshot | `W4D17_RED_WEB_DB_WIN_COMPLETE` |

---

# **DAY 18 — Privilege Escalation, Crypto, Stego, and File Challenges**

## **Daily Objective**

Train high-value tie-breaker categories: root/admin access, cryptography, steganography, and hidden files.

2019 Module C gives scoring categories for **Root Access, Cryptography, and Steganography**, with steganography carrying its own scoring section.

---

## **Step 1 — Linux privilege escalation practice**

On `RED-LINUX01`, mentor prepares:

One normal user  
One readable backup file  
One sudo misconfiguration  
One SUID training binary or safe misconfigured script  
One root-only flag

Student workflow:

whoami  
id  
hostname  
sudo \-l  
find / \-perm \-4000 \-type f 2\>/dev/null  
find / \-writable \-type d 2\>/dev/null | head  
ls \-la /home  
cat /etc/passwd

Student must document:

Initial user  
Privilege path  
Command used  
Root/admin proof  
Flag location  
Fix recommendation

---

## **Step 2 — Windows privilege review practice**

On `RED-WIN01`, mentor prepares:

Readable share clue  
Weak file permission clue  
Scheduled task clue  
Service path clue  
Admin-only flag

Student workflow:

Check shares  
Check user privileges  
Check writable folders  
Check scheduled tasks  
Check service paths  
Search for flag files

No destructive escalation. No malware. No persistence.

---

## **Step 3 — Crypto challenge workflow**

Student checks:

file challenge\*  
strings challenge\*  
cat challenge.txt

Common decoding tools:

base64 \-d file.txt  
xxd \-r \-p hex.txt  
python3 \- \<\<'PY'  
import codecs  
print(codecs.decode("uryyb", "rot\_13"))  
PY

Hash identification:

hashid hash.txt

Cracking rule:

Only crack mentor-provided lab hashes using mentor-provided wordlists.  
Never use company passwords or real leaked wordlists.

---

## **Step 4 — Stego challenge workflow**

Student checks:

file image.png  
exiftool image.png  
strings image.png | head  
binwalk image.png

If stego tools are installed:

steghide info image.jpg

Use only lab-provided passwords/hints.

---

## **Step 5 — Flag search workflow**

On file target:

grep \-R "icc{" .  
find . \-type f \-exec strings {} \\; | grep "icc{"

For encoded flags:

grep \-R "aWNj" .

---

## **Day 18 Deliverables**

| Deliverable | Evidence |
| ----- | ----- |
| Linux privilege checklist | Completed |
| Windows privilege checklist | Completed |
| Crypto worksheet | Completed |
| Stego worksheet | Completed |
| Minimum 5 practice flags | Flag log |
| Commands used | Command log |
| Snapshot | `W4D18_RED_PRIV_CRYPTO_STEGO_COMPLETE` |

---

# **DAY 19 — Blue Team Monitoring: Firewall, Splunk, PCAP**

## **Daily Objective**

Build and operate the Blue Team lab: traffic generator, firewall logs, Splunk searches, and PCAP review.

The 2022 Blue Team task states that flags are found by inspecting traffic flowing through the firewall’s inside and outside interfaces. It also provides Splunk SIEM and firewall access, plus service mappings behind PAT.

2019 Module D Blue Team categories include **Reconnaissance/Application Detection, Malicious URL, Exploits, Botnet, Data Leakage, Reverse Engineering, and Forensic**.

---

## **Step 1 — Build Blue Team topology**

TG-01 \-\> FW-BLUE \-\> BLUE-WEB01 / BLUE-APP01 / BLUE-DB01  
                     |  
                  SPLUNK01  
                     |  
                  FLARE01

Networks:

Outside/Traffic Generator: 60.1.10.0/24  
Internal Blue Network: 172.16.10.0/24

---

## **Step 2 — Configure firewall logging**

On `FW-BLUE`:

Enable logging for allowed traffic.  
Enable logging for blocked traffic.  
Enable NAT/PAT logs if available.  
Enable web interface access only from FLARE01.

Suggested NAT-style service mappings:

| External IP | Internal IP | Port | Service |
| ----- | ----- | ----- | ----- |
| `60.1.10.80` | `172.16.10.40` | `1080` | PHP/nginx |
| `60.1.10.81` | `172.16.10.40` | `1081` | Apache |
| `60.1.10.82` | `172.16.10.50` | `1082` | PHP/nginx |
| `60.1.10.86` | `172.16.10.50` | `1086` | Django-style app |

The 2022 Blue Team task includes similar PAT service mappings such as PHP/nginx, Apache, and Django services.

---

## **Step 3 — Configure Splunk ingestion**

On `SPLUNK01`, ingest:

Firewall logs  
Web access logs  
Syslog from Linux services  
Traffic-generator logs  
Optional Zeek/Suricata logs

Create Splunk index:

wsc\_blue

Suggested Splunk searches:

index=wsc\_blue

index=wsc\_blue "icc{"

index=wsc\_blue src\_ip=\* dest\_ip=\* | stats count by src\_ip dest\_ip dest\_port

index=wsc\_blue uri="\*/etc/passwd\*" OR uri="\*cmd=\*"

index=wsc\_blue "User-Agent" "curl"

index=wsc\_blue "credit" OR "card" OR "exfil"

---

## **Step 4 — Configure safe traffic generator**

On `TG-01`, create:

mkdir \-p \~/blue-traffic  
nano \~/blue-traffic/generate\_blue\_traffic.sh

Example safe training traffic:

\#\!/bin/bash

WEB1="60.1.10.80:1080"  
WEB2="60.1.10.81:1081"  
APP1="60.1.10.86:1086"

\# Benign traffic  
curl \-s "http://$WEB1/" \>/dev/null  
curl \-s "http://$WEB2/about" \>/dev/null

\# Recon-like traffic  
curl \-s "http://$WEB1/robots.txt" \>/dev/null  
curl \-s "http://$WEB1/admin" \>/dev/null

\# Exploit-like request, harmless  
curl \-s "http://$WEB1/?file=/etc/passwd\&flag=icc{blue\_path\_traversal\_01}" \>/dev/null

\# Malicious URL simulation, harmless  
curl \-A "WSC-Training-Agent" \-s "http://$WEB2/download/update.exe?campaign=icc{blue\_malicious\_url\_01}" \>/dev/null

\# Botnet-like beacon simulation, harmless  
for i in {1..5}; do  
  curl \-s "http://$APP1/beacon?id=bot-$i\&flag=icc{blue\_beacon\_01}" \>/dev/null  
  sleep 2  
done

\# Data leakage simulation using fake data only  
curl \-s \-X POST "http://$APP1/upload" \\  
  \-d "customer=training\&card=4111111111111111\&flag=icc{blue\_dlp\_01}" \>/dev/null

Make executable:

chmod \+x \~/blue-traffic/generate\_blue\_traffic.sh

Run:

\~/blue-traffic/generate\_blue\_traffic.sh

These are harmless simulations. No malware is generated.

---

## **Step 5 — PCAP capture**

On firewall or monitoring node:

sudo tcpdump \-i any \-w w4d19\_blue\_traffic.pcap

Run generator, then stop capture.

Analyze in Wireshark:

http  
tcp contains "icc{"  
frame contains "4111111111111111"  
http.request.uri contains "/etc/passwd"

---

## **Step 6 — Blue Team triage worksheet**

Student must complete:

| Category | Evidence Source | Indicator | Flag |
| ----- | ----- | ----- | ----- |
| Recon | Firewall/Web logs | `/robots.txt`, `/admin` |  |
| Malicious URL | HTTP logs | suspicious download URL |  |
| Exploit | URI/path | `/etc/passwd` |  |
| Botnet | Repeated beacon | `/beacon?id=` |  |
| Data Leakage | POST body | fake card value |  |
| Forensic | PCAP | extracted stream |  |
| Reverse/Decode | encoded string | decoded flag |  |

---

## **Day 19 Deliverables**

| Deliverable | Evidence |
| ----- | ----- |
| Blue topology | Screenshot |
| Firewall logging enabled | Screenshot |
| Splunk index/search working | Screenshot |
| Traffic generator script | Saved copy |
| PCAP captured | `.pcap` file |
| Flags found in logs | Flag log |
| Splunk queries | Saved searches |
| Blue triage worksheet | Completed |
| Snapshot | `W4D19_BLUE_MONITORING_COMPLETE` |

---

# **DAY 20 — Full Mock Competition**

## **Daily Objective**

Run a controlled competition simulation covering Module A, Module B, Module C, and Module D workflows.

This is the final pressure test.

---

# **Day 20 Schedule**

| Time | Activity |
| ----- | ----- |
| 08:30–09:00 | Rules briefing, snapshot restore |
| 09:00–10:30 | Module A mini hardening validation |
| 10:30–10:45 | Break |
| 10:45–12:15 | Module B mini incident response |
| 12:15–13:15 | Lunch |
| 13:15–14:45 | Red Team CTF round |
| 14:45–15:00 | Break |
| 15:00–16:30 | Blue Team CTF round |
| 16:30–17:15 | Evidence cleanup and submission |
| 17:15–18:00 | Mentor scoring and debrief |

---

## **Part 1 — Module A Mini Validation**

Student must prove:

AD login works  
GPO applies  
Linux root SSH blocked  
HTTPS works  
HTTP redirects  
ModSecurity blocks /etc/passwd  
Splunk receives logs  
IDS alert triggers  
Firewall blocks DMZ-to-Inside  
VPN simulation works if configured

Time limit:

90 minutes

---

## **Part 2 — Module B Mini Incident Response**

Student receives:

One web log sample  
One suspicious PHP file  
One Windows autorun clue  
One memory artifact  
One PCAP  
One vulnerable code snippet

Student must submit:

Exploit URL  
Attack timestamp  
Infected file path  
Suspicious Windows autorun  
Malicious process  
PCAP indicator  
Secure-code correction

Time limit:

90 minutes

---

## **Part 3 — Red Team CTF Round**

Scope:

10.10.10.0/24 only  
No management network  
No DoS  
No external scanning

Target:

Minimum 5 flags  
At least 3 categories

Categories:

Enumeration  
Web  
Database  
Windows  
Linux privilege  
Crypto  
Stego

---

## **Part 4 — Blue Team CTF Round**

Student must use:

Firewall logs  
Splunk logs  
PCAP  
Web access logs

Target:

Minimum 5 blue flags  
At least 4 categories

Categories:

Recon  
Malicious URL  
Exploit  
Botnet  
Data leakage  
Forensic  
Decode/reverse clue

---

# **Day 20 Deliverables**

| Deliverable | Evidence |
| ----- | ----- |
| Module A mini checklist | Completed |
| Module B answer sheet | Completed |
| Red Team flag log | Completed |
| Blue Team flag log | Completed |
| Screenshots | Evidence folder |
| PCAP evidence | Saved |
| Splunk queries | Saved |
| Final scorecard | Completed |
| Final report | PDF |

Snapshot:

WEEK4\_FINAL\_MOCK\_COMPLETE

---

# **Week 4 Final Checklist**

| Requirement | Status |
| ----- | ----- |
| Red CTF network created | ☐ |
| Kali connected only to lab network | ☐ |
| Management network excluded | ☐ |
| Red targets deployed | ☐ |
| Recon workflow completed | ☐ |
| Web challenge completed | ☐ |
| Database challenge completed | ☐ |
| Windows challenge completed | ☐ |
| Linux privilege challenge completed | ☐ |
| Crypto challenge completed | ☐ |
| Stego challenge completed | ☐ |
| Blue firewall deployed | ☐ |
| Splunk collecting logs | ☐ |
| Traffic generator working | ☐ |
| PCAP captured | ☐ |
| Blue flags found in logs | ☐ |
| Final mock completed | ☐ |
| Evidence package complete | ☐ |
| Student can explain findings | ☐ |

---

# **Week 4 Mentor Scoring**

Score out of 100:

| Area | Weight |
| ----- | ----- |
| Red Team reconnaissance | 15 |
| Red Team flags | 20 |
| Blue Team log analysis | 20 |
| Blue Team PCAP analysis | 15 |
| Competition discipline | 10 |
| Evidence quality | 10 |
| Time management | 10 |

Passing score:

80/100

Competition-ready score:

90/100+

---

# **Final Student Readiness Gate**

The student is competition-ready only if they can:

1. Build a target map from zero.  
2. Find flags without random guessing.  
3. Use Splunk searches efficiently.  
4. Decode or extract flags from logs and PCAP.  
5. Avoid management/scoring systems.  
6. Submit only validated flag values.  
7. Keep clear evidence per flag.  
8. Work under time pressure.  
9. Explain attack and defense findings.  
10. Recover from failed or rejected flags without losing focus.

# ✖️ 2019

Yes. I would **rebuild the 1-month plan around scoring**, not around topics. The student must train in the same order the competition rewards marks:

**Build → Harden → Investigate → Repair → Attack → Defend → Mock Competition**

A critical correction: for a WorldSkills-style win strategy, training only Module A and Module B is not enough. The 2019 marking scheme allocates **25 marks each** to Infrastructure Hardening, Incident Response/Forensics/AppSec, CTF Attack, and CTF Defence, for a total of 100 marks.

I only see the uploaded 2019 materials in this chat, not a 2022 test-project file. This plan is therefore calibrated from the uploaded 2019 pre-release, actual modules, and marking scheme, then adapted for a **PC-only simulation lab**.

---

# **1-Month WorldSkills Cyber Security Training Plan**

## **Competition Target**

By the end of 1 month, the student should be able to:

1. Build a virtual enterprise network.  
2. Harden Windows, Linux, firewall, router, switch, web, FTP, VPN, IDS, and logging services.  
3. Investigate a compromised WordPress/Linux/Windows environment.  
4. Analyze PCAP, memory, disk image, PDF payload, ELF, and PE artifacts.  
5. Perform controlled CTF attack enumeration and exploitation inside an isolated lab.  
6. Defend using logs, firewall alerts, IDS/IPS alerts, Splunk, and forensic evidence.  
7. Produce competition-grade documentation and screenshots.

The Module A actual project is focused on **logon/password policies, network equipment hardening, public services protection, event monitoring, and firewall policy**, with the largest scoring areas being event monitoring, logon/password policy, and firewall policy.  
Module B is focused on **incident response, vulnerability repair, digital forensics, network analysis, malicious document analysis, and code review**.

---

# **Recommended PC-Only Simulation Stack**

## **Preferred Setup**

Use this stack:

| Component | Recommended Tool |
| ----- | ----- |
| Main network simulator | **EVE-NG** or **GNS3** |
| VM platform | VMware Workstation / Proxmox / VirtualBox |
| Firewall replacement | pfSense / OPNsense / ASAv if legally available |
| Router replacement | VyOS / Cisco IOSv / CSR1000v / CML node |
| Switch replacement | EVE-NG L2 image / Open vSwitch / Linux bridge / Cisco CML |
| Windows lab | Windows Server \+ Windows Client VMs |
| Linux lab | Rocky Linux / AlmaLinux / Ubuntu / CentOS-compatible VM |
| SOC tools | Splunk, Snort/Suricata, Wireshark |
| Forensics tools | Autopsy, Volatility, FTK Imager, CyberChef, binwalk, strings, foremost |
| Attack/CTF tools | Kali Linux, Nmap, Burp Suite Community, Gobuster/Feroxbuster, SQLMap for legal labs only |

GNS3 supports a GNS3 VM workflow using VMware or VirtualBox, with VMware noted in official documentation as the faster option because it supports nested virtualization more effectively. ([GNS3 Documentation](https://docs.gns3.com/docs/getting-started/setup-wizard-gns3-vm/?utm_source=chatgpt.com))  
EVE-NG is a stronger fit for mixed WorldSkills-style labs because its supported image list includes Windows, Windows Server, Linux, Kali, pfSense, VyOS, and other network/security platforms. ([EVE-NG](https://www.eve-ng.net/index.php/documentation/supported-images/?utm_source=chatgpt.com))  
pfSense is a practical firewall substitute because Netgate documents pfSense virtualization across Proxmox, VirtualBox, Xen, KVM, Hyper-V, VMware, and other platforms. ([docs.netgate.com](https://docs.netgate.com/pfsense/en/latest/virtualization/index.html?utm_source=chatgpt.com))  
Where exact Cisco behavior matters, Cisco Modeling Labs is the cleanest legal path because Cisco states that CML provides access to Cisco VM images for labs; Cisco also notes that CML-provided images are licensed for use inside CML. ([Cisco DevNet](https://developer.cisco.com/docs/modeling-labs/vm-images-for-cml-labs/?utm_source=chatgpt.com))

---

# **High-Level 4-Week Roadmap**

| Week | Main Focus | Competition Alignment | Weekly Objective |
| ----- | ----- | ----- | ----- |
| Week 1 | Lab build \+ Module A foundation | Infrastructure Setup | Build the full simulated enterprise lab |
| Week 2 | Module A hardening | 25-mark Infrastructure module | Score heavily on policies, monitoring, firewall, public services |
| Week 3 | Module B IR/DFIR/AppSec | 25-mark Incident Response module | Investigate, repair, analyze artifacts, complete answer sheets |
| Week 4 | CTF Attack \+ Defence \+ Mock Competition | Module C \+ D \+ Final polish | Train speed, accuracy, flags, reporting, recovery |

---

# **Daily Training Structure**

Use this structure every training day:

| Time | Activity |
| ----- | ----- |
| 08:30–09:00 | Previous-day review and blocker cleanup |
| 09:00–10:30 | Technical concept \+ mentor demo |
| 10:30–10:45 | Break |
| 10:45–12:30 | Hands-on lab block 1 |
| 12:30–13:30 | Lunch |
| 13:30–15:30 | Hands-on lab block 2 |
| 15:30–15:45 | Break |
| 15:45–17:00 | Validation, troubleshooting, screenshots |
| 17:00–17:30 | Documentation, answer sheet, configuration backup |
| 17:30–18:00 | Mentor scoring and next-day prep |

WorldSkills actual instructions emphasize saving configurations frequently, ensuring devices remain accessible for marking, testing requirements before submission, and confirming configs still work after reboot. This must become a daily habit, not an afterthought.

---

# **Week 1 — Build the Competition Arena**

## **Weekly Objective**

Build a working simulated enterprise lab using only PCs. The student must understand the full topology, IP addressing, routing path, firewall zones, server placement, and snapshot process.

## **Competition Target**

This week maps to the 2019 Module A pre-release topology: Windows Server, CentOS servers, Windows client, Cisco router, Cisco switch, ASA firewall, wireless AP, and inside/DMZ/outside segmentation.

## **Week 1 Deliverables**

By Friday, the student must submit:

1. Working topology screenshot.  
2. IP address table.  
3. VM inventory.  
4. Firewall/router/switch mapping.  
5. Ping test matrix.  
6. Snapshot list.  
7. Known issues and fixes log.

---

## **Day 1 — Simulation Platform and VM Baseline**

### **Objective**

Prepare the PC-only lab environment.

### **Tasks**

* Install EVE-NG or GNS3.  
* Install VMware Workstation, VirtualBox, or Proxmox.  
* Create VM templates:  
  * Windows Server  
  * Windows Client  
  * Linux Server  
  * Kali / analyst workstation  
* Create isolated lab networks:  
  * Management  
  * Inside  
  * DMZ  
  * Outside  
  * Forensics network  
* Create snapshot naming standard:  
  * `DAY01_CLEAN`  
  * `WEEK1_BASE`  
  * `MODULEA_READY`  
  * `MODULEB_READY`

### **Deliverable**

Lab platform operational. All VMs boot. Snapshots created.

---

## **Day 2 — Network Topology Build**

### **Objective**

Build the simulated network skeleton.

### **Tasks**

* Create logical topology:  
  * Inside LAN  
  * DMZ  
  * Outside  
  * Transit network  
* Add simulated router.  
* Add simulated firewall.  
* Add virtual switches.  
* Add placeholder VMs:  
  * AD-Svr  
  * CentOS-01  
  * CentOS-02  
  * CentOS-03  
  * CentOS-04  
  * WIN-CLI  
  * Outside Client

### **Deliverable**

Topology matches the 2019 Module A diagram: inside network, ASA/firewall boundary, DMZ network, outside router/client path.

---

## **Day 3 — Routing and Firewall Zone Connectivity**

### **Objective**

Make traffic move between zones.

### **Tasks**

* Configure inside gateway.  
* Configure DMZ gateway.  
* Configure outside/transit routing.  
* Configure firewall interfaces:  
  * Inside  
  * DMZ  
  * Outside  
* Configure basic NAT/PAT.  
* Test:  
  * Inside to firewall  
  * DMZ to firewall  
  * Outside to router  
  * Inside to outside  
  * Inside to DMZ  
  * DMZ to outside

### **Deliverable**

Ping matrix completed. Routing table screenshots captured.

---

## **Day 4 — Server Placement and Base Services**

### **Objective**

Place all infrastructure systems in their correct zones.

### **Tasks**

* Configure IP addresses:  
  * AD server inside  
  * Linux monitoring/logging server inside  
  * Web/LDAP/DNS server in DMZ  
  * FTP/RADIUS/IDS server in DMZ  
  * Outside client outside  
* Install base services only:  
  * Windows Server role preparation  
  * Linux SSH  
  * Apache placeholder page  
  * Basic DNS package  
  * Syslog package  
  * Wireshark/Snort placeholder

### **Deliverable**

All servers reachable from correct zones. No security hardening yet.

---

## **Day 5 — Week 1 Integration Test**

### **Objective**

Freeze the first stable lab baseline.

### **Tasks**

* Perform full network validation.  
* Reboot all VMs and verify connectivity remains.  
* Export configurations.  
* Capture screenshots.  
* Create `WEEK1_WORKING_BASELINE` snapshot.  
* Student explains topology from memory.

### **Deliverable**

Week 1 lab baseline ready for Module A hardening.

---

# **Week 2 — Win Module A: Infrastructure Setup and Security Hardening**

## **Weekly Objective**

Turn the working infrastructure into a hardened, scored environment.

## **Competition Target**

Module A actual scoring is divided into:

| Area | Marks |
| ----- | ----- |
| Logon and password policies | 5.75 |
| Network equipment hardening | 4.25 |
| Public services protection | 2.75 |
| Events monitoring | 6.75 |
| Firewall policy | 5.50 |

Highest priority: **events monitoring, logon/password policies, firewall policy**.

## **Week 2 Deliverables**

1. Hardened Windows domain.  
2. Hardened Linux servers.  
3. Hardened router/firewall/switch.  
4. HTTPS-only web service.  
5. Explicit TLS-only FTP.  
6. Splunk/Snort/Syslog monitoring.  
7. Firewall policy evidence.  
8. Module A PDF report.  
9. Reboot-proof validation.

---

## **Day 6 — Windows Domain, GPO, and Authentication**

### **Objective**

Score the Windows policy items.

### **Tasks**

* Configure AD DS.  
* Create domain users/groups.  
* Configure password policy:  
  * Minimum length  
  * Complexity  
  * Lockout  
  * Password age where required  
* Configure login banner.  
* Disable guest access.  
* Configure audit policy.  
* Configure RDP security.  
* Join Windows client to domain.

### **Deliverable**

Screenshots of GPO settings, domain login, password policy test, account lockout test.

---

## **Day 7 — Linux Security Policy and Access Control**

### **Objective**

Score Linux login/password policy items.

### **Tasks**

* Configure Linux password complexity.  
* Configure password aging.  
* Configure account lockout.  
* Configure SSH banner.  
* Disable root SSH login.  
* Configure inactivity timeout.  
* Create required users.  
* Test failed login lockout.

### **Deliverable**

Screenshots of `/etc/security`, PAM, SSH config, login banner, and lockout test.

---

## **Day 8 — Public Services Protection**

### **Objective**

Score web and FTP protection.

### **Tasks**

* Configure HTTPS-only web access.  
* Redirect HTTP to HTTPS.  
* Configure internal CA certificate.  
* Configure FTPS explicit TLS only.  
* Disable insecure FTP behavior.  
* Harden Apache:  
  * Remove version disclosure.  
  * Disable directory listing.  
  * Restrict access.  
  * Configure logs.  
* Harden FTP:  
  * Chroot users.  
  * Disable anonymous access.  
  * Limit sessions.  
  * Restrict write/rename behavior.

### **Deliverable**

HTTPS redirect screenshot, certificate screenshot, FTPS connection screenshot, failed plain FTP test.

---

## **Day 9 — Monitoring, IDS, and Logging**

### **Objective**

Score the highest-value Module A area.

### **Tasks**

* Install/configure Splunk.  
* Configure log receiving.  
* Forward Domain Controller logs.  
* Configure Syslog forwarding from network devices.  
* Configure Snort/Suricata.  
* Create detection rules for:  
  * FTP traffic  
  * ICMP traffic  
  * Payload containing `malware`  
* Generate test traffic.  
* Validate alert visibility.

Module A actual requires DC security logs to be sent to Splunk, Splunk to receive logs on a specified port, audit policies to match required Windows events, and DMZ traffic to be mirrored to IDS with alerts for FTP, ICMP, and malware payload content.

### **Deliverable**

Splunk dashboard screenshot, IDS alert screenshots, event log forwarding proof.

---

## **Day 10 — Firewall, VPN, Reboot Validation**

### **Objective**

Finish Module A and practice judge validation.

### **Tasks**

* Configure firewall rules using least privilege.  
* Allow only required services.  
* Block unnecessary inbound traffic.  
* Configure NAT.  
* Configure site-to-site VPN or simulated VPN.  
* Validate:  
  * HTTPS access  
  * FTPS access  
  * DNS  
  * AD login  
  * log forwarding  
  * IDS alerts  
  * VPN traffic  
* Reboot all devices.  
* Recheck all scoring items.

### **Deliverable**

Module A scoring checklist completed. Student must achieve at least **80% Module A readiness** before moving to Week 3\.

---

# **Week 3 — Win Module B: Incident Response, Forensics, and Application Security**

## **Weekly Objective**

Train the student to answer precisely, not just investigate. In Module B, marks are tied to answer-sheet accuracy, evidence, filenames, hashes, paths, timestamps, commands, and remediation screenshots.

## **Competition Target**

The Module B actual project requires the student to investigate a hacked WordPress/Linux web server, compromised Windows file server, malicious programs, PCAPs, memory dump, PDF payload, disk images, and code exhibits.

## **Week 3 Deliverables**

1. Incident timeline.  
2. Web compromise report.  
3. Windows malware/stager report.  
4. Remediation evidence.  
5. Forensic findings sheet.  
6. Code review answers.  
7. Module B answer sheet PDF.

---

## **Day 11 — Web Server Incident Response**

### **Objective**

Find the attack path on a compromised web server.

### **Tasks**

* Review web access logs.  
* Review Apache/PHP logs.  
* Review shell history.  
* Review WordPress files.  
* Identify:  
  * First attack command  
  * Time of execution  
  * Infected file  
  * Webshell code  
  * Webshell name  
  * Function called by webshell  
  * HTTP tunnel target IP  
  * Credentials used by attacker

Module B actual specifically asks for attack commands/parameters, first execution time, infected file, webshell code, webshell name, function called by webshell, HTTP tunnel target IP, and username/password used by the attacker.

### **Deliverable**

Completed Web\_Server incident response section.

---

## **Day 12 — Windows File Server Incident Response**

### **Objective**

Investigate Windows malware and persistence.

### **Tasks**

* Review startup entries.  
* Review registry autoruns.  
* Review scheduled tasks.  
* Review services.  
* Review suspicious processes.  
* Calculate SHA1 hash.  
* Identify:  
  * Lock-screen malware path and filename  
  * Malware SHA1  
  * Stager path and filename  
  * Stager behavior steps

### **Deliverable**

Completed File\_Server incident response section with hashes and paths.

---

## **Day 13 — Vulnerability Detection and Repair**

### **Objective**

Repair exploited weaknesses and prove remediation.

### **Tasks**

* Harden PHP:  
  * Disable dangerous functions.  
* Harden MySQL:  
  * Restrict import/export behavior.  
* Remove exposed management tool directories.  
* Fix weak passwords.  
* Remove three malicious programs.  
* Change administrator password from CMD.  
* Block RDP/3389 from the web server using Windows Firewall.

Module B actual requires PHP hardening, MySQL import/export restriction, management tool removal, weak password fix, three malware removals, administrator password change, and blocking 3389 through Windows Firewall with screenshot evidence.

### **Deliverable**

Remediation screenshots and completed vulnerability repair answer sheet.

---

## **Day 14 — Host Forensics: Linux, Windows Image, Memory Dump**

### **Objective**

Train host-based forensic workflow.

### **Tasks**

* LinuxM\_Svr:  
  * Identify malicious process.  
  * Locate malware file.  
  * Analyze ELF behavior.  
  * Recover modified settings.  
* Windows image/memory:  
  * Identify malicious process.  
  * Find hidden malware location.  
  * Analyze PE behavior.  
  * Extract key from memory.  
  * Recover corrupted file.

### **Deliverable**

Forensic worksheet with process name, path, behavior, evidence, and recovery steps.

---

## **Day 15 — Network Forensics, PDF Payload, Code Review**

### **Objective**

Complete the remaining high-speed Module B tasks.

### **Tasks**

* Analyze PCAP:  
  * Identify key.  
  * Identify malicious process.  
  * Extract file.  
  * Calculate SHA1 where required.  
* Analyze malicious PDF/system image:  
  * Extract malicious file.  
  * Calculate MD5.  
  * Decrypt file.  
  * Analyze payload behavior.  
* Code review:  
  * Identify vulnerable line.  
  * Name attack type.  
  * Explain fix.  
  * Provide secure code line.

The Module B actual code review includes C and PHP examples, and the answer sheet asks for vulnerable line, attack name, secure explanation, and corrected code line.

### **Deliverable**

Full Module B answer sheet completed under time pressure.

---

# **Week 4 — CTF Attack, CTF Defence, and Final Mock Competition**

## **Weekly Objective**

Convert knowledge into speed. WorldSkills is not won by knowing the tools. It is won by fast execution, clean evidence, and low-error submissions.

## **Competition Target**

Module C focuses on CTF Attack categories: enumeration, web attacks, database attacks, Windows attacks, root access, cryptography, and steganography.  
Module D focuses on CTF Defence categories: reconnaissance/application detection, malicious URL, exploits/malware, botnet, data leakage, reverse engineering, and forensics.

## **Week 4 Deliverables**

1. Attack checklist.  
2. Defence checklist.  
3. Flag log.  
4. Splunk investigation notes.  
5. Final Module A mock score.  
6. Final Module B mock score.  
7. Final post-mortem report.

---

## **Day 16 — CTF Attack: Enumeration and Web**

### **Objective**

Train fast target discovery.

### **Tasks**

* Host discovery.  
* Port scanning.  
* Service versioning.  
* Web directory enumeration.  
* Web login testing.  
* Basic SQL injection detection in controlled lab.  
* Basic file upload/path traversal testing in controlled lab.

### **Deliverable**

Enumeration worksheet and at least 5 practice flags.

---

## **Day 17 — CTF Attack: Database, Windows, Privilege Escalation**

### **Objective**

Train deeper exploitation thinking inside an isolated lab.

### **Tasks**

* Database enumeration.  
* Credential reuse testing.  
* Windows service review.  
* Local privilege escalation checklist.  
* Linux privilege escalation checklist.  
* Root/admin proof capture.

### **Deliverable**

Privilege escalation checklist and at least 5 practice flags.

---

## **Day 18 — CTF Attack: Crypto, Stego, Reverse Basics**

### **Objective**

Cover categories that often decide tie-breakers.

### **Tasks**

* Hash identification.  
* Basic cracking workflow using provided wordlists only.  
* Encoding/decoding with CyberChef.  
* Image steganography checks.  
* Strings/binwalk/file metadata checks.  
* Simple binary strings/static analysis.

### **Deliverable**

Crypto/stego/reversing cheat sheet and at least 5 practice flags.

---

## **Day 19 — CTF Defence: SOC and Firewall Monitoring**

### **Objective**

Train blue-team detection speed.

### **Tasks**

* Review Splunk alerts.  
* Identify malicious URL.  
* Identify exploit traffic.  
* Identify botnet-like beaconing.  
* Identify data leakage.  
* Identify suspicious internal-to-external traffic.  
* Write short incident notes.

Module D describes the competitor role as SOC analyst and network security engineer responsible for identifying attacks targeted on the network.

### **Deliverable**

SOC triage sheet with alert, evidence, source, destination, impact, and recommended response.

---

## **Day 20 — Final Integrated Mock**

### **Objective**

Simulate competition pressure.

### **Tasks**

* Morning: Module A timed hardening replay.  
* Midday: Module B timed incident response.  
* Afternoon: CTF attack/defence mini-round.  
* End of day: mentor scoring.

### **Deliverable**

Final mock score and remediation plan.

---

# **Final 5-Day Competition Simulation Option**

If you have 25 training days instead of 20, use the last 5 days like this:

| Day | Simulation |
| ----- | ----- |
| Day 21 | Full Module A mock, 6–8 hours |
| Day 22 | Full Module B mock, 6–8 hours |
| Day 23 | CTF Attack mock |
| Day 24 | CTF Defence mock |
| Day 25 | Full final exam: A \+ B \+ C/D mini-round |

---

# **Weekly Objectives Summary**

## **Week 1 Objective — Build the Lab**

**Goal:** Working PC-only simulation environment.

**Pass Criteria:**

* All VMs boot.  
* All network zones communicate correctly.  
* Snapshots created.  
* Student can explain topology without guide.

**Failure Criteria:**

* No stable snapshots.  
* Routing not working.  
* Firewall zones unclear.  
* Student depends on mentor for every connection test.

---

## **Week 2 Objective — Score Module A**

**Goal:** Harden infrastructure according to WorldSkills scoring areas.

**Pass Criteria:**

* Password/logon policy working.  
* HTTPS/FTPS working.  
* Splunk receives logs.  
* IDS detects FTP, ICMP, and malware keyword traffic.  
* Firewall rules are minimal and documented.  
* Reboot test passes.

**Score Target:** 20/25 or higher.

---

## **Week 3 Objective — Score Module B**

**Goal:** Complete IR, vulnerability repair, forensics, and code review answer sheets.

**Pass Criteria:**

* Answers include exact filenames, paths, commands, timestamps, hashes.  
* Screenshots support remediation.  
* Student can explain attacker behavior.  
* Student completes answer sheet cleanly.

**Score Target:** 18/25 or higher.

---

## **Week 4 Objective — Win Under Pressure**

**Goal:** Improve speed and coverage across attack and defence categories.

**Pass Criteria:**

* Student finds flags across at least 5 categories.  
* Student can triage alerts quickly.  
* Student avoids rabbit holes.  
* Student submits clean evidence.  
* Student finishes with recovery and reporting discipline.

**Score Target:** 65–75 total points minimum after one month, with 80+ as competitive target.

---

# **PC Resource Plan**

## **Strong PC Setup**

Recommended for one student:

| Resource | Recommended |
| ----- | ----- |
| CPU | 8 cores or higher |
| RAM | 32 GB minimum, 64 GB preferred |
| Storage | 500 GB SSD minimum |
| Network | Isolated virtual networks |
| Hypervisor | VMware / Proxmox / VirtualBox |

## **Low-Spec PC Setup**

For 16 GB RAM machines:

* Do not run full Module A topology all at once.  
* Split labs:  
  * Lab A: Network \+ AD \+ 2 Linux servers.  
  * Lab B: Web/File/Forensics.  
  * Lab C/D: Kali \+ 2–3 vulnerable targets.  
* Shut down unused VMs.  
* Snapshot aggressively.  
* Use Linux containers where possible.

---

# **Simulation Replacement Matrix**

| Real Competition Device | PC-Only Replacement |
| ----- | ----- |
| ASA firewall | pfSense / OPNsense / ASAv in CML |
| Cisco router | VyOS / IOSv / CSR1000v / CML |
| Cisco switch | EVE-NG L2 / Open vSwitch / CML IOL-L2 |
| Wireless AP | Simulate DHCP/RADIUS flow; use hostapd only if needed |
| Physical clients | Windows/Linux VMs |
| Splunk server | Splunk VM or Docker |
| IDS | Snort/Suricata VM |
| Web server | Linux Apache/Nginx VM |
| File server | Windows Server VM |

Strong opinion: **Do not wait for hardware.** Hardware helps realism, but WorldSkills success is driven by speed, methodology, documentation, validation, and troubleshooting under time pressure.

---

# **Mentor Scoring Model**

Score the student every Friday:

| Area | Weight |
| ----- | ----- |
| Technical completion | 40% |
| Evidence/screenshots | 20% |
| Time management | 15% |
| Troubleshooting independence | 15% |
| Documentation quality | 10% |

Daily rule:

* No screenshot \= no evidence.  
* No saved config \= incomplete.  
* No reboot validation \= unstable.  
* No written answer \= no mark.  
* No snapshot \= operational risk.

---

# **Winning Strategy**

The student should not memorize one topology. They should master repeatable workflows:

1. **Read task first.**  
2. **Map scoring items.**  
3. **Configure high-value items first.**  
4. **Validate immediately.**  
5. **Screenshot proof.**  
6. **Save configuration.**  
7. **Reboot-test when possible.**  
8. **Write final answer cleanly.**

The fastest way to lose marks is not technical failure—it is missing evidence, unsaved configs, inaccessible devices, incomplete answer sheets, and broken services after reboot.

# IP Table Router and Switch

## **Mini PC 1 — WINSRV1**

Use this for Windows Server 2022\.

Hostname: WINSRV1  
Role: AD DS, DNS, GPO, File Services  
Domain: manila.com  
IP: 192.168.2.10  
Subnet: 255.255.255.0  
Gateway: 192.168.2.254  
DNS: 192.168.2.10  
Connected Port: ether2

## **Mini PC 2 — WINSRV3 / Issuing CA**

Use this later for certificate services.

Hostname: WINSRV3  
Role: Issuing CA  
IP: 192.168.2.30  
Subnet: 255.255.255.0  
Gateway: 192.168.2.254  
DNS: 192.168.2.10  
Connected Port: ether3  
Domain Join: manila.com

## **Mini PC 3 — LINSRV1**

Use this for Linux web server.

Hostname: LINSRV1  
Role: Linux Web Server / Apache / SSH / SELinux / firewalld  
IP: 192.168.1.10  
Subnet: 255.255.255.0  
Gateway: 192.168.1.254  
DNS: 192.168.2.10  
Connected Port: ether4

## **Mini PC 4 — Client / Competitor Workstation**

Use this for testing.

Hostname: CLIENT1 or COMPETITOR-PC  
Role: LAN test client  
IP: DHCP  
Expected IP Range: 172.16.100.100-172.16.100.199  
Gateway: 172.16.100.254  
DNS: 192.168.2.10  
Connected Port: ether5

# Rocky Linux Server

For your competition setup, LINSRV1 should act as the **DMZ Linux web server** with Apache/httpd, SSH hardening, firewalld, SELinux, domain integration, sudo controls, and HTTPS. This aligns with the 2024/2025 MA2 task where LINSRV1 is placed in the DMZ at `192.168.1.10/24`, hosts the website, must join the domain, allow only C1/C2 SSH access, use SSH port `2022`, run firewalld, and keep SELinux enabled while httpd remains operational.

---

# **1\. Target LINSRV1 Configuration**

## **Network Details**

Connect the Rocky Linux server to:

D-Link Port 7  
VLAN 10 — DMZ

Use this static IP:

Hostname: LINSRV1  
IP Address: 192.168.1.10  
Subnet Mask: 255.255.255.0  
Gateway: 192.168.1.254  
DNS: 192.168.2.10  
Domain: manila.com

Your MikroTik gateway for the DMZ is:

192.168.1.254

Your Windows DNS / Domain Controller is:

WINSRV1 \= 192.168.2.10

---

# **2\. Recommended OS Install Settings**

Use:

OS: Rocky Linux 9.7 Minimal  
Installation Type: Minimal Install  
Disk: 60 GB minimum  
RAM: 4 GB minimum  
CPU: 2 cores  
Root password: P@ssw0rd for lab only  
Local user: competitor

Rocky’s official download page provides the latest Rocky Linux versions. Use the **Rocky Linux 9.x Minimal ISO**, not Rocky 10, for this training cycle. ([Rocky Linux](https://rockylinux.org/download))

---

# **3\. After Installation — Initial Setup**

Login as root or a sudo-capable user.

## **Set hostname**

sudo hostnamectl set-hostname LINSRV1.manila.com

Check:

hostnamectl

---

## **Configure static IP using `nmcli`**

First check your network interface name:

nmcli device status

Assume the interface is `ens160` or `enp1s0`. Replace it below if different.

sudo nmcli con show

Set static IP:

sudo nmcli con mod "System ens160" ipv4.addresses 192.168.1.10/24  
sudo nmcli con mod "System ens160" ipv4.gateway 192.168.1.254  
sudo nmcli con mod "System ens160" ipv4.dns 192.168.2.10  
sudo nmcli con mod "System ens160" ipv4.dns-search manila.com  
sudo nmcli con mod "System ens160" ipv4.method manual  
sudo nmcli con down "System ens160"  
sudo nmcli con up "System ens160"

If your connection name is not `System ens160`, use the name shown by:

nmcli con show

Validate:

ip a  
ip route  
cat /etc/resolv.conf  
ping \-c 4 192.168.1.254  
ping \-c 4 192.168.2.10  
nslookup manila.com 192.168.2.10

---

# **4\. Install Required Programs**

Run this first:

sudo dnf clean all  
sudo dnf makecache  
sudo dnf update \-y

Install the core competition packages:

sudo dnf install \-y \\  
httpd \\  
mod\_ssl \\  
openssl \\  
firewalld \\  
policycoreutils-python-utils \\  
setroubleshoot-server \\  
setools-console \\  
realmd \\  
sssd \\  
sssd-tools \\  
oddjob \\  
oddjob-mkhomedir \\  
adcli \\  
samba-common-tools \\  
krb5-workstation \\  
bind-utils \\  
chrony \\  
audit \\  
rsyslog \\  
nano \\  
vim \\  
wget \\  
curl \\  
unzip \\  
tar \\  
net-tools \\  
tcpdump \\  
lsof \\  
dnf-utils \\  
pam\_pwquality

## **What these are for**

| Package | Purpose |
| ----- | ----- |
| `httpd` | Apache web server |
| `mod_ssl` | HTTPS support |
| `openssl` | Certificate generation/testing |
| `firewalld` | Linux firewall |
| `policycoreutils-python-utils` | SELinux management tools like `semanage` |
| `realmd`, `sssd`, `adcli` | Join Linux to Active Directory |
| `krb5-workstation` | Kerberos authentication |
| `samba-common-tools` | AD/domain support tools |
| `bind-utils` | `nslookup`, `dig` |
| `chrony` | Time sync |
| `audit` | Audit logging |
| `rsyslog` | System logging |
| `tcpdump` | Network troubleshooting |
| `pam_pwquality` | Password complexity |

---

# **5\. Enable Core Services**

sudo systemctl enable \--now firewalld  
sudo systemctl enable \--now httpd  
sudo systemctl enable \--now sshd  
sudo systemctl enable \--now chronyd  
sudo systemctl enable \--now auditd  
sudo systemctl enable \--now rsyslog

Validate:

systemctl status firewalld \--no-pager  
systemctl status httpd \--no-pager  
systemctl status sshd \--no-pager  
sestatus

---

# **6\. Configure SELinux**

The task expects SELinux enabled while the website still works.

Set SELinux enforcing:

sudo setenforce 1  
sudo sed \-i 's/^SELINUX=.\*/SELINUX=enforcing/' /etc/selinux/config  
sestatus

Expected:

Current mode: enforcing  
Mode from config file: enforcing

---

# **7\. Create Website Directory**

The 2025-style version uses Manila naming, so use:

sudo mkdir \-p /var/www/manila  
echo "WorldSkills ASEAN Manila \- LINSRV1 Website" | sudo tee /var/www/manila/index.html

Set ownership and SELinux context:

sudo chown \-R apache:apache /var/www/manila  
sudo semanage fcontext \-a \-t httpd\_sys\_content\_t "/var/www/manila(/.\*)?"  
sudo restorecon \-Rv /var/www/manila

---

# **8\. Configure Apache HTTP Website**

Create the vhost file:

sudo nano /etc/httpd/conf.d/manila.conf

Paste:

\<VirtualHost \*:80\>  
    ServerName w3.manila.com  
    ServerAlias linsrv1.manila.com  
    DocumentRoot /var/www/manila

    \<Directory /var/www/manila\>  
        AllowOverride None  
        Require all granted  
        Options \-Indexes  
    \</Directory\>

    ErrorLog /var/log/httpd/manila-error.log  
    CustomLog /var/log/httpd/manila-access.log combined  
\</VirtualHost\>

Test and restart:

sudo apachectl configtest  
sudo systemctl restart httpd

---

# **9\. Configure HTTPS**

For now, use a self-signed certificate. Later, when WINSRV3 CA is ready, replace this with a CA-signed certificate.

sudo mkdir \-p /etc/pki/tls/wsc  
sudo openssl req \-x509 \-nodes \-newkey rsa:4096 \\  
\-keyout /etc/pki/tls/wsc/manila.key \\  
\-out /etc/pki/tls/wsc/manila.crt \\  
\-days 365 \\  
\-subj "/C=PH/ST=Metro Manila/L=Manila/O=WorldSkills/OU=Cyber/CN=w3.manila.com"

Secure the key:

sudo chmod 600 /etc/pki/tls/wsc/manila.key  
sudo restorecon \-Rv /etc/pki/tls/wsc

Edit the Apache config:

sudo nano /etc/httpd/conf.d/manila-ssl.conf

Paste:

\<VirtualHost \*:443\>  
    ServerName w3.manila.com  
    ServerAlias linsrv1.manila.com  
    DocumentRoot /var/www/manila

    SSLEngine on  
    SSLCertificateFile /etc/pki/tls/wsc/manila.crt  
    SSLCertificateKeyFile /etc/pki/tls/wsc/manila.key

    \<Directory /var/www/manila\>  
        AllowOverride None  
        Require all granted  
        Options \-Indexes  
    \</Directory\>

    ErrorLog /var/log/httpd/manila-ssl-error.log  
    CustomLog /var/log/httpd/manila-ssl-access.log combined  
\</VirtualHost\>

Restart:

sudo apachectl configtest  
sudo systemctl restart httpd

Test locally:

curl \-k https://localhost  
curl http://localhost

---

# **10\. Configure Firewall**

Open only required services:

sudo firewall-cmd \--set-default-zone=dmz  
sudo firewall-cmd \--permanent \--zone=dmz \--add-service=http  
sudo firewall-cmd \--permanent \--zone=dmz \--add-service=https  
sudo firewall-cmd \--permanent \--zone=dmz \--add-service=dns  
sudo firewall-cmd \--permanent \--zone=dmz \--add-port=2022/tcp  
sudo firewall-cmd \--reload  
sudo firewall-cmd \--list-all

For now, DNS service is opened because the WorldSkills-style task says LINSRV1 has public-facing DNS entries and website hosting. If you do not configure DNS service on LINSRV1 later, you may remove it.

---

# **11\. Change SSH to Port 2022**

The task requires SSH to listen on port `2022`, root SSH blocked, and only C1/C2 allowed.

## **Allow SSH port 2022 in SELinux**

sudo semanage port \-a \-t ssh\_port\_t \-p tcp 2022

If it already exists:

sudo semanage port \-m \-t ssh\_port\_t \-p tcp 2022

## **Edit SSH config**

sudo nano /etc/ssh/sshd\_config

Set or add:

Port 2022  
PermitRootLogin no  
PasswordAuthentication yes  
AllowUsers C1 C2 c1 c2 competitor

For now, keep `competitor` allowed so you do not lock yourself out. Later, remove it when C1/C2 testing is successful.

Restart SSH:

sudo systemctl restart sshd

Validate:

ss \-tulpn | grep ssh

Expected:

0.0.0.0:2022

Test from Client PC:

ssh competitor@192.168.1.10 \-p 2022

Root should fail:

ssh root@192.168.1.10 \-p 2022

---

# **12\. Create Local C1 and C2 Fallback Users**

Do this now even if you will later domain-join the server. The competition task allows creating matching local C1/C2 accounts if domain join fails.

sudo useradd C1  
sudo useradd C2  
sudo passwd C1  
sudo passwd C2

Use lab password:

P@ssw0rd

Give C1 and C2 sudo rights:

echo "C1 ALL=(ALL) ALL" | sudo tee /etc/sudoers.d/wsc-c1  
echo "C2 ALL=(ALL) ALL" | sudo tee /etc/sudoers.d/wsc-c2  
sudo chmod 440 /etc/sudoers.d/wsc-c1 /etc/sudoers.d/wsc-c2  
sudo visudo \-c

Validate:

su \- C1  
sudo whoami  
exit

su \- C2  
sudo whoami  
exit

Expected:

root

---

# **13\. Configure Local Password Policy**

For local fallback users C1/C2:

sudo nano /etc/security/pwquality.conf

Set:

minlen \= 10  
ucredit \= \-1  
lcredit \= \-1  
dcredit \= \-1  
ocredit \= \-1  
retry \= 3

Set password aging:

sudo chage \-M 30 \-m 25 \-W 25 C1  
sudo chage \-M 30 \-m 25 \-W 25 C2  
sudo chage \-M 30 \-m 25 \-W 25 competitor

Validate:

chage \-l C1  
chage \-l C2

This matches the 2025-style requirement for 10-character passwords, monthly password changes, and warning before expiration.

---

# **14\. Prepare for Domain Join to `manila.com`**

Before joining, make sure WINSRV1 has DNS records:

On **WINSRV1 DNS Manager** or PowerShell:

Add-DnsServerResourceRecordA \-Name "linsrv1" \-ZoneName "manila.com" \-IPv4Address "192.168.1.10"  
Add-DnsServerResourceRecordA \-Name "w3" \-ZoneName "manila.com" \-IPv4Address "192.168.1.10"

On Rocky, test:

nslookup manila.com 192.168.2.10  
nslookup winsrv1.manila.com 192.168.2.10  
nslookup linsrv1.manila.com 192.168.2.10

Check time:

timedatectl

Set timezone:

sudo timedatectl set-timezone Asia/Manila

Kerberos/AD join is sensitive to time drift. Keep server time close to WINSRV1.

---

# **15\. Join Rocky Linux to Active Directory**

Discover the domain:

realm discover manila.com

Join the domain:

sudo realm join \-U Administrator manila.com

Validate:

realm list  
id C1  
id C2  
getent passwd C1  
getent passwd C2

If domain usernames show as `C1@manila.com`, edit SSSD:

sudo nano /etc/sssd/sssd.conf

Make sure these are set under the domain section:

use\_fully\_qualified\_names \= False  
fallback\_homedir \= /home/%u

Restart:

sudo systemctl restart sssd

Enable automatic home directory creation:

sudo authselect enable-feature with-mkhomedir  
sudo authselect apply-changes  
sudo systemctl enable \--now oddjobd

---

# **16\. Give Domain C1 and C2 Sudo Rights**

If domain lookup works, create sudo rules:

echo "C1 ALL=(ALL) ALL" | sudo tee /etc/sudoers.d/domain-c1  
echo "C2 ALL=(ALL) ALL" | sudo tee /etc/sudoers.d/domain-c2  
sudo chmod 440 /etc/sudoers.d/domain-c1 /etc/sudoers.d/domain-c2  
sudo visudo \-c

Test:

su \- C1  
sudo whoami  
exit

Expected:

root

---

# **17\. Restrict SSH to C1 and C2 Only**

After C1/C2 login is confirmed, edit SSH:

sudo nano /etc/ssh/sshd\_config

Set:

Port 2022  
PermitRootLogin no  
PasswordAuthentication yes  
AllowUsers C1 C2 c1 c2  
		

Restart:

sudo systemctl restart sshd

Test from Client PC:

ssh C1@192.168.1.10 \-p 2022  
ssh C2@192.168.1.10 \-p 2022  
ssh root@192.168.1.10 \-p 2022

Expected:

C1 \= success  
C2 \= success  
root \= fail  
other users \= fail

---

# **18\. Enable Audit Logging**

sudo systemctl enable \--now auditd

Add audit rules:

sudo auditctl \-w /etc/ssh/sshd\_config \-p wa \-k ssh\_config\_watch  
sudo auditctl \-w /etc/sudoers \-p wa \-k sudoers\_watch  
sudo auditctl \-w /etc/httpd/conf.d/ \-p wa \-k httpd\_config\_watch  
sudo auditctl \-l

Make persistent:

sudo tee /etc/audit/rules.d/wsc.rules \> /dev/null \<\< 'EOF'  
\-w /etc/ssh/sshd\_config \-p wa \-k ssh\_config\_watch  
\-w /etc/sudoers \-p wa \-k sudoers\_watch  
\-w /etc/httpd/conf.d/ \-p wa \-k httpd\_config\_watch  
EOF

sudo augenrules \--load  
sudo auditctl \-l

---

# **19\. Optional: Configure HTTP to HTTPS Redirect**

This may be useful for later hardening practice.

Edit HTTP vhost:

sudo nano /etc/httpd/conf.d/manila.conf

Replace with:

\<VirtualHost \*:80\>  
    ServerName w3.manila.com  
    ServerAlias linsrv1.manila.com  
    Redirect permanent / https://w3.manila.com/  
\</VirtualHost\>

Restart:

sudo apachectl configtest  
sudo systemctl restart httpd

Validate:

curl \-I http://w3.manila.com  
curl \-k \-I https://w3.manila.com

---

# **20\. Final Validation Checklist**

Run these on LINSRV1:

hostnamectl  
ip a  
ip route  
cat /etc/resolv.conf  
sestatus  
firewall-cmd \--list-all  
ss \-tulpn | egrep '2022|80|443'  
systemctl status httpd \--no-pager  
systemctl status sshd \--no-pager  
realm list  
id C1  
id C2

Run these from Client1 / Competitor PC:

ping 192.168.1.10  
nslookup linsrv1.manila.com 192.168.2.10  
nslookup w3.manila.com 192.168.2.10

From a Linux/SSH client:

ssh C1@192.168.1.10 \-p 2022  
ssh C2@192.168.1.10 \-p 2022  
ssh root@192.168.1.10 \-p 2022  
curl \-k https://w3.manila.com

Expected result:

| Test | Expected |
| ----- | ----- |
| Ping LINSRV1 | Pass |
| DNS resolves `linsrv1.manila.com` | Pass |
| DNS resolves `w3.manila.com` | Pass |
| HTTP/HTTPS works | Pass |
| SSH on port 2022 | Pass |
| SSH root login | Fail |
| C1/C2 SSH login | Pass |
| C1/C2 sudo | Pass |
| firewalld running | Pass |
| SELinux enforcing | Pass |
| Apache still works with SELinux | Pass |

---

# **Final Server State**

Your Rocky Linux server should end as:

Hostname: LINSRV1.manila.com  
IP: 192.168.1.10/24  
Gateway: 192.168.1.254  
DNS: 192.168.2.10  
Website: /var/www/manila  
HTTP: enabled  
HTTPS: enabled  
SSH: port 2022 only  
Root SSH: disabled  
Allowed SSH users: C1 and C2  
C1/C2 sudo: enabled  
firewalld: enabled  
SELinux: enforcing  
Domain: manila.com joined if AD is ready

Priority: get **network \+ Apache \+ SSH 2022 \+ firewalld \+ SELinux** working first. Domain join can follow once WINSRV1 DNS and AD are confirmed stable.

# Master Reference Sheet

This is based on your current design:

MikroTik RB2011 \= Router / Firewall / Gateway / DHCP  
D-Link DGS-1210-28 \= Managed VLAN Switch  
Mini PCs \= Temporary physical servers  
Future Lab Host \= ESXi / VMware host later

This also follows the 2024/2025 WorldSkills-style topology where `WINSRV1` is on the **Servers** network, `LINSRV1` is on the **DMZ**, and the firewall/router separates Servers, DMZ, LAN, and Internet networks.

---

# **1\. Network Summary**

| Network | VLAN | Subnet | Gateway |
| ----- | ----- | ----- | ----- |
| DMZ | 10 | `192.168.1.0/24` | `192.168.1.254` |
| SERVERS | 20 | `192.168.2.0/24` | `192.168.2.254` |
| LAN / CLIENTS | 30 | `172.16.100.0/24` | `172.16.100.254` |
| INTERNET-LAB | 40 | `10.40.0.0/24` | `10.40.0.254` |

---

# **2\. MikroTik RB2011 Router IPs by Port**

## **Router Identity**

| Item | Value |
| ----- | ----- |
| Router Model | MikroTik RB2011UiAS-IN |
| RouterOS | 7.2.3 stable |
| Router Name | `RB2011-WSC-LAB` |
| Main LAN Management IP | `172.16.100.254` |
| Servers Management IP | `192.168.2.254` |

---

## **MikroTik Port Assignment**

| MikroTik Port | Purpose | Connected To | Network | Router IP |
| ----- | ----- | ----- | ----- | ----- |
| `ether1-WAN` | Optional Internet / ISP | ISP Router or upstream | WAN | DHCP from ISP |
| `ether2-WINSRV1` | SERVERS uplink | D-Link Port 1 | VLAN 20 / SERVERS | `192.168.2.254` |
| `ether3-WINSRV3` | Spare server port | Not used for now | SERVERS | Same bridge as SERVERS |
| `ether4-LINSRV1` | DMZ uplink | D-Link Port 2 | VLAN 10 / DMZ | `192.168.1.254` |
| `ether5-LAN-CLIENT1` | LAN uplink | D-Link Port 3 | VLAN 30 / LAN | `172.16.100.254` |
| `ether6-LAN-CLIENT2` | Optional direct LAN client | Optional | LAN | Same bridge as LAN |
| `ether7-LAN-SPARE` | Spare LAN | Optional | LAN | Same bridge as LAN |
| `ether8-LAN-SPARE` | Spare LAN | Optional | LAN | Same bridge as LAN |
| `ether9-INETLAB-CLIENT3` | Internet-Lab uplink | D-Link Port 4 | VLAN 40 / INTERNET-LAB | `10.40.0.254` |
| `ether10-INETLAB-SPARE` | Spare Internet-Lab | Optional | INTERNET-LAB | Same bridge as Internet-Lab |

Important: **Do not connect both MikroTik ether2 and ether3 to the same D-Link VLAN at the same time**, because they are in the same bridge and may create a loop.

---

# **3\. D-Link DGS-1210-28 Switch IP and VLANs**

## **Switch Management**

| Item | Value |
| ----- | ----- |
| Switch Model | D-Link DGS-1210-28 |
| Switch Name | `DGS-WSC-SW1` |
| Management VLAN | VLAN 30 / LAN |
| Management IP | `172.16.100.253` |
| Subnet Mask | `255.255.255.0` |
| Gateway | `172.16.100.254` |
| Access URL | `http://172.16.100.253` |

---

## **D-Link Port Assignment**

| D-Link Port | Connected Device | VLAN | Mode | PVID |
| ----- | ----- | ----- | ----- | ----- |
| 1 | MikroTik `ether2` | 20 SERVERS | Untagged Access | 20 |
| 2 | MikroTik `ether4` | 10 DMZ | Untagged Access | 10 |
| 3 | MikroTik `ether5` | 30 LAN | Untagged Access | 30 |
| 4 | MikroTik `ether9` | 40 INTERNET-LAB | Untagged Access | 40 |
| 5 | Mini PC 1 / `WINSRV1` | 20 SERVERS | Untagged Access | 20 |
| 6 | Mini PC 2 / `WINSRV3` | 20 SERVERS | Untagged Access | 20 |
| 7 | Mini PC 3 / `LINSRV1` | 10 DMZ | Untagged Access | 10 |
| 8 | Mini PC 4 / Client1 / Competitor PC | 30 LAN | Untagged Access | 30 |
| 9 | Admin PC / Client2 | 30 LAN | Untagged Access | 30 |
| 10 | Client3 / External test client | 40 INTERNET-LAB | Untagged Access | 40 |
| 11–22 | Spare | Assign as needed | Disabled or unused | As needed |
| 23 | Future ESXi Lab Host | VLAN 30 untagged, VLAN 10/20/40 tagged | Hybrid/Trunk | 30 |
| 24 | Admin spare | 30 LAN | Untagged Access | 30 |
| 25–28 | SFP / uplink spare | Optional | Unused | — |

---

# **4\. Server and Client IP Address Assignment**

## **Mini PC 1 — WINSRV1**

| Item | Value |
| ----- | ----- |
| Hostname | `WINSRV1` |
| OS | Windows Server 2022 |
| Role | AD DS, DNS, GPO, File Services |
| Domain | `manila.com` |
| IP Address | `192.168.2.10` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `192.168.2.254` |
| DNS Server | `192.168.2.10` |
| D-Link Port | Port 5 |
| VLAN | 20 SERVERS |

---

## **Mini PC 2 — WINSRV3**

| Item | Value |
| ----- | ----- |
| Hostname | `WINSRV3` |
| OS | Windows Server 2022 |
| Role | Issuing CA / Certificate Services |
| IP Address | `192.168.2.30` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `192.168.2.254` |
| DNS Server | `192.168.2.10` |
| Domain | `manila.com` |
| D-Link Port | Port 6 |
| VLAN | 20 SERVERS |

---

## **Future Server — WINSRV4**

| Item | Value |
| ----- | ----- |
| Hostname | `WINSRV4` |
| OS | Windows Server 2022 |
| Role | Offline Root CA |
| IP Address | `192.168.2.50` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `192.168.2.254` |
| DNS Server | `192.168.2.10` |
| Domain | Usually not required if offline Root CA |
| VLAN | 20 SERVERS |
| Note | Configure, then keep powered off when not needed |

---

## **Mini PC 3 — LINSRV1**

| Item | Value |
| ----- | ----- |
| Hostname | `LINSRV1` |
| FQDN | `LINSRV1.manila.com` |
| OS | Rocky Linux 9.7 Minimal |
| Role | Linux Web Server / Apache / SSH / SELinux / firewalld |
| IP Address | `192.168.1.10` |
| Subnet Mask | `255.255.255.0` |
| Default Gateway | `192.168.1.254` |
| DNS Server | `192.168.2.10` |
| Website DNS | `w3.manila.com` |
| D-Link Port | Port 7 |
| VLAN | 10 DMZ |

---

## **Mini PC 4 — Client1 / Competitor PC**

| Item | Value |
| ----- | ----- |
| Hostname | `CLIENT1` or `COMPETITOR-PC` |
| OS | Windows 10/11 Pro |
| Role | Domain client / testing workstation |
| IP Address | DHCP |
| Expected IP Range | `172.16.100.100–172.16.100.199` |
| Gateway | `172.16.100.254` |
| DNS Server | `192.168.2.10` |
| Domain | `manila.com` |
| D-Link Port | Port 8 |
| VLAN | 30 LAN |

---

## **Admin PC / Client2**

| Item | Value |
| ----- | ----- |
| Hostname | `CLIENT2` or `ADMIN-PC` |
| OS | Windows 10/11 Pro |
| IP Address | DHCP |
| Expected IP Range | `172.16.100.100–172.16.100.199` |
| Gateway | `172.16.100.254` |
| DNS Server | `192.168.2.10` |
| D-Link Port | Port 9 |
| VLAN | 30 LAN |

---

## **Client3 — External Test Client**

| Item | Value |
| ----- | ----- |
| Hostname | `CLIENT3` |
| OS | Windows 10/11 Pro |
| Role | External / Internet test client |
| IP Address | DHCP |
| Expected IP Range | `10.40.0.100–10.40.0.199` |
| Gateway | `10.40.0.254` |
| DNS Server | `10.40.0.254` |
| D-Link Port | Port 10 |
| VLAN | 40 INTERNET-LAB |

---

# **5\. DNS Records to Create on WINSRV1**

Create these DNS A records in the `manila.com` zone:

| DNS Name | IP Address | Purpose |
| ----- | ----- | ----- |
| `winsrv1.manila.com` | `192.168.2.10` | Domain Controller |
| `winsrv3.manila.com` | `192.168.2.30` | Issuing CA |
| `winsrv4.manila.com` | `192.168.2.50` | Offline Root CA |
| `linsrv1.manila.com` | `192.168.1.10` | Linux server |
| `w3.manila.com` | `192.168.1.10` | Linux website |

---

# **6\. Password Master List**

Use this for the lab only. Do not use these passwords in production.

## **Network Devices**

| Device | Username | Password | Notes |
| ----- | ----- | ----- | ----- |
| MikroTik RB2011 | Existing admin account | Use your current MikroTik admin password | The script did **not** change the router admin password |
| MikroTik RB2011 recommended lab password | `admin` or your admin user | `P@ssw0rd` or stronger | Change manually if needed |
| D-Link DGS-1210-28 | `admin` | `P@ssw0rd` | If you set this during wizard |
| Future pfSense | `admin` | `pfsense` initially | Change to `P@ssw0rd` during task if required |

---

## **Windows Servers**

| Device / Account | Username | Password |
| ----- | ----- | ----- |
| WINSRV1 Local Administrator before domain | `Administrator` | `P@ssw0rd` |
| Domain Administrator | `MANILA\Administrator` | `P@ssw0rd` |
| WINSRV3 Local Administrator | `Administrator` | `P@ssw0rd` |
| WINSRV3 Domain Login | `MANILA\Administrator` | `P@ssw0rd` |
| WINSRV4 Local Administrator | `Administrator` | `P@ssw0rd` |

---

## **Domain Users**

| User | Group / Purpose | Password |
| ----- | ----- | ----- |
| `C1` | Linux SSH / sudo / VPN Users | `P@ssw0rd` |
| `C2` | Linux SSH / sudo / VPN Users | `P@ssw0rd` |
| `acctuser1` | Accounting Users | `P@ssw0rd` |
| `graphics1` | Graphics | `P@ssw0rd` |
| `cust1` | Customer Service | `P@ssw0rd` |
| `exec1` | Executive | `P@ssw0rd` |
| `Competitor0` | Competitor workstation login if used | `CharterDressing` |

The `Competitor0 / CharterDressing` credential appears in the 2024/2025 task documents for competitor laptops. Use it only if you want to mirror that setup.

---

## **Linux Server**

| Device | Username | Password | Notes |
| ----- | ----- | ----- | ----- |
| LINSRV1 | `root` | `P@ssw0rd` | Root SSH should be disabled after setup |
| LINSRV1 | `competitor` | `P@ssw0rd` | Temporary admin during setup |
| LINSRV1 | `C1` | `P@ssw0rd` | Local fallback or domain user |
| LINSRV1 | `C2` | `P@ssw0rd` | Local fallback or domain user |

---

# **7\. Physical Cable Map**

| Cable | From | To | Purpose |
| ----- | ----- | ----- | ----- |
| 1 | MikroTik `ether2` | D-Link Port 1 | SERVERS uplink |
| 2 | MikroTik `ether4` | D-Link Port 2 | DMZ uplink |
| 3 | MikroTik `ether5` | D-Link Port 3 | LAN uplink |
| 4 | MikroTik `ether9` | D-Link Port 4 | INTERNET-LAB uplink |
| 5 | WINSRV1 NIC | D-Link Port 5 | Server network |
| 6 | WINSRV3 NIC | D-Link Port 6 | Server network |
| 7 | LINSRV1 NIC | D-Link Port 7 | DMZ |
| 8 | Client1 / Competitor PC | D-Link Port 8 | LAN |
| 9 | Admin PC / Client2 | D-Link Port 9 | LAN |
| 10 | Client3 | D-Link Port 10 | Internet-Lab |
| 11 | Future Lab Host | D-Link Port 23 | ESXi trunk/hybrid |

---

# **8\. Validation Commands**

## **On MikroTik**

/ip address print  
/ip dhcp-server lease print  
/ip firewall filter print stats  
/ip route print  
/ping 192.168.2.10  
/ping 192.168.1.10  
/ping 172.16.100.253

---

## **On D-Link**

Check:

Management IP: 172.16.100.253  
Default Gateway: 172.16.100.254  
VLAN 10, 20, 30, 40 exist  
PVIDs are correct  
Configuration is saved

---

## **On Client1 / Competitor PC**

ipconfig /all  
ping 172.16.100.254  
ping 172.16.100.253  
ping 192.168.2.10  
ping 192.168.1.10  
nslookup manila.com 192.168.2.10  
nslookup linsrv1.manila.com 192.168.2.10  
nslookup w3.manila.com 192.168.2.10

---

## **On WINSRV1**

ipconfig /all  
ping 192.168.2.254  
ping 172.16.100.254  
ping 192.168.1.10  
nslookup manila.com  
dcdiag /test:dns

---

## **On LINSRV1**

ip a  
ip route  
cat /etc/resolv.conf  
ping \-c 4 192.168.1.254  
ping \-c 4 192.168.2.10  
nslookup manila.com 192.168.2.10  
nslookup w3.manila.com 192.168.2.10

---

# **9\. Current Recommended Build Order**

Follow this order to avoid confusion:

1\. Configure MikroTik  
2\. Configure D-Link VLANs and management IP  
3\. Connect cables based on cable map  
4\. Configure WINSRV1 static IP  
5\. Install AD DS \+ DNS on WINSRV1  
6\. Create manila.com domain  
7\. Create DNS records  
8\. Configure Client1 DHCP and join domain  
9\. Configure LINSRV1 static IP  
10\. Test routing from LAN to SERVERS and DMZ  
11\. Configure Apache/httpd on LINSRV1  
12\. Configure SSH/firewalld/SELinux on LINSRV1  
13\. Configure WINSRV3 CA later  
14\. Configure Lab Host later

---

# **10\. One-Page Quick Reference**

MikroTik LAN IP:        172.16.100.254  
MikroTik SERVERS IP:    192.168.2.254  
MikroTik DMZ IP:        192.168.1.254  
MikroTik INTERNET IP:   10.40.0.254

D-Link Switch IP:       172.16.100.253

WINSRV1:                192.168.2.10  
WINSRV3:                192.168.2.30  
WINSRV4:                192.168.2.50  
LINSRV1:                192.168.1.10

Client DHCP LAN:        172.16.100.100-199  
Client3 DHCP Internet:  10.40.0.100-199

Domain:                 manila.com  
Domain Admin:           MANILA\\Administrator  
Lab Password:           P@ssw0rd

D-Link Admin:           admin / P@ssw0rd  
MikroTik Admin:         your current admin password  
LINSRV1 root:           root / P@ssw0rd  
Competitor Login:       Competitor0 / CharterDressing

My recommendation: print this and place it beside the rack/table. This becomes your **network bible** for the current temporary competition lab.

# Rocky Server

Once the installation finishes and asks you to reboot, follow these steps to manually force the bootloader onto the Samsung SSD.

---

🛠️ Step 1: Boot Back Into Rescue Mode

1. Leave your Rocky Linux installation USB plugged into the Mac Mini.  
2. Restart the Mac and hold down the Option (Alt) key on your keyboard immediately until you see the boot menu.  
3. Select your USB drive to launch the installer menu again.  
4. On the initial black boot menu, use your arrow keys to select Troubleshooting, then select Rescue a Rocky Linux system.

---

📂 Step 2: Mount Your Installed System

The rescue environment will search your Samsung SSD for the newly installed OS.

1. When prompted with a menu asking to mount your system, type 1 and press Enter (to choose *Continue*).  
2. The system will tell you that your OS is mounted under /mnt/sysroot.  
3. Press Enter to open the terminal shell command line.

Type the following command to switch from the USB environment directly into your actual SSD file system:  
bash  
chroot /mnt/sysroot

4. Use code with caution.

---

💾 Step 3: Manually Install the Bootloader

Now that you are inside your system, run these commands to install the EFI files that the installer skipped:

Reinstall GRUB and EFI binaries:  
bash  
dnf reinstall grub2-efi-x64 shim-x64

1. Use code with caution.

Force GRUB onto the Mac boot path:  
bash  
grub2-install \--target=x86\_64-efi \--efi-directory=/boot/efi \--bootloader-id=rocky \--recheck

2. Use code with caution.

Generate the boot configuration file:  
bash  
grub2-mkconfig \-o /boot/efi/EFI/rocky/grub.cfg

3. Use code with caution.

Exit and reboot:  
bash  
exit  
reboot

4. Use code with caution.

Unplug your USB drive while the Mac restarts. Your Mac Mini should now boot directly into your Rocky Linux 9.7 minimal system.

# Windows Server 2022

**First Windows Server 2022**, configure it as **WinSRV1 / WINSRV1**, the core internal server for the 2025-style WorldSkills lab.

Based on the 2025 MA2 reference, **WINSRV1 is the Domain Controller for AD, DNS, and DHCP**, placed in the **SERVERS VLAN** with IP **192.168.2.10/24**. The same file also assigns **WinSRV3 as the Issuing CA** and **WinSRV4 as the Offline Root CA**, so do **not** install Certificate Authority on WinSRV1.

---

# **1\. WinSRV1 Target Role**

## **Server Name**

WINSRV1

## **Domain**

Use the 2025 ASEAN-style domain:

manila.com

## **IP Address**

IP Address: 192.168.2.10  
Subnet Mask: 255.255.255.0  
Default Gateway: 192.168.2.254  
Preferred DNS: 127.0.0.1 or 192.168.2.10

The firewall/server gateway in the 2025 topology is listed as **192.168.2.254**, while WINSRV1 is listed as **192.168.2.10/24**.

---

# **2\. Roles and Features to Install**

Install these on **WinSRV1**:

| Role / Feature | Required? | Purpose |
| ----- | ----- | ----- |
| **Active Directory Domain Services** | Yes | Domain controller for `manila.com` |
| **DNS Server** | Yes | Internal domain name resolution |
| **DHCP Server** | Conditional | Install for practice, but avoid conflict if pfSense handles LAN DHCP |
| **Group Policy Management** | Yes | GPO configuration |
| **File and Storage Services** | Yes | `pictures` shared folder |
| **RSAT AD Tools** | Yes | AD Users and Computers, DNS, GPO tools |
| **Windows Defender Firewall** | Yes | Server-side firewall policy |
| **Wireshark** | Optional | Troubleshooting; MA2 notes Wireshark is installed on WINSRV1 |

Important: The 2025 MA2 task says pfSense should provide DHCP for LAN clients and use WINSRV1 as the DNS server. So for competition alignment, **do not run overlapping DHCP scopes on both pfSense and WinSRV1**. Use pfSense for LAN DHCP, and WinSRV1 for DNS.

---

# **3\. Recommended VM Specification**

For WinSRV1:

| Component | Minimum |
| ----- | ----- |
| OS | Windows Server 2022 Standard/Desktop Experience |
| vCPU | 2–4 vCPU |
| RAM | 6–8 GB |
| Disk | 100 GB |
| NIC | Connected to `SERVERS` network/VLAN |
| IP | Static `192.168.2.10/24` |

Take a snapshot before promotion:

WINSRV1\_CLEAN\_BEFORE\_AD

---

# **4\. Step-by-Step Setup**

## **Step 1 — Rename the Server**

Open **PowerShell as Administrator**:

Rename-Computer \-NewName "WINSRV1" \-Restart

After reboot, confirm:

hostname

Expected:

WINSRV1

---

## **Step 2 — Configure Static IP**

Open PowerShell as Administrator:

Get-NetAdapter

Assume the adapter name is `Ethernet`.

New-NetIPAddress \`  
\-InterfaceAlias "Ethernet" \`  
\-IPAddress 192.168.2.10 \`  
\-PrefixLength 24 \`  
\-DefaultGateway 192.168.2.254

Set-DnsClientServerAddress \`  
\-InterfaceAlias "Ethernet" \`  
\-ServerAddresses 192.168.2.10

Validate:

ipconfig /all  
ping 192.168.2.254

If DNS is not installed yet, `192.168.2.10` may not fully resolve. That is normal at this stage.

---

## **Step 3 — Install Required Roles**

Run:

Install-WindowsFeature AD-Domain-Services,DNS,DHCP,FS-FileServer,GPMC \`  
\-IncludeManagementTools

Validate:

Get-WindowsFeature AD-Domain-Services,DNS,DHCP,FS-FileServer,GPMC

---

## **Step 4 — Promote WinSRV1 as Domain Controller**

Use:

Install-ADDSForest \`  
\-DomainName "manila.com" \`  
\-DomainNetbiosName "MANILA" \`  
\-InstallDNS \`  
\-SafeModeAdministratorPassword (ConvertTo-SecureString "P@ssw0rd" \-AsPlainText \-Force) \`  
\-Force

The server will reboot.

After reboot, log in as:

MANILA\\Administrator

---

## **Step 5 — Validate AD and DNS**

Run:

Get-ADDomain  
Get-ADForest  
dcdiag /test:dns  
nslookup manila.com

Expected:

Domain: manila.com  
DNS resolves successfully

Take snapshot:

WINSRV1\_AD\_DNS\_READY

---

# **5\. Create Core Organizational Units**

Open PowerShell as Administrator:

New-ADOrganizationalUnit \-Name "Manila" \-Path "DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Accounting" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "IT" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Graphics" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Customer Service" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Executives" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Servers" \-Path "OU=Manila,DC=manila,DC=com"  
New-ADOrganizationalUnit \-Name "Workstations" \-Path "OU=Manila,DC=manila,DC=com"

---

# **6\. Create Required Groups**

New-ADGroup \-Name "Accounting Users" \-GroupScope Global \-Path "OU=Accounting,OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "IT" \-GroupScope Global \-Path "OU=IT,OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "Graphics" \-GroupScope Global \-Path "OU=Graphics,OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "Customer Service" \-GroupScope Global \-Path "OU=Customer Service,OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "Executive" \-GroupScope Global \-Path "OU=Executives,OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "VPN Users" \-GroupScope Global \-Path "OU=Manila,DC=manila,DC=com"  
New-ADGroup \-Name "WebUsers" \-GroupScope Global \-Path "OU=Manila,DC=manila,DC=com"

These groups support the 2025 tasks for GPO targeting, file share permissions, VPN users, and Linux access testing.

---

# **7\. Create Test Users**

Use the lab password only:

$Password \= ConvertTo-SecureString "P@ssw0rd" \-AsPlainText \-Force

New-ADUser \-Name "C1" \-SamAccountName "C1" \-AccountPassword $Password \-Enabled $true \-Path "OU=IT,OU=Manila,DC=manila,DC=com"  
New-ADUser \-Name "C2" \-SamAccountName "C2" \-AccountPassword $Password \-Enabled $true \-Path "OU=IT,OU=Manila,DC=manila,DC=com"

New-ADUser \-Name "AcctUser1" \-SamAccountName "acctuser1" \-AccountPassword $Password \-Enabled $true \-Path "OU=Accounting,OU=Manila,DC=manila,DC=com"  
New-ADUser \-Name "GraphicsUser1" \-SamAccountName "graphics1" \-AccountPassword $Password \-Enabled $true \-Path "OU=Graphics,OU=Manila,DC=manila,DC=com"  
New-ADUser \-Name "CustomerUser1" \-SamAccountName "cust1" \-AccountPassword $Password \-Enabled $true \-Path "OU=Customer Service,OU=Manila,DC=manila,DC=com"  
New-ADUser \-Name "ExecutiveUser1" \-SamAccountName "exec1" \-AccountPassword $Password \-Enabled $true \-Path "OU=Executives,OU=Manila,DC=manila,DC=com"

Add group memberships:

Add-ADGroupMember "IT" C1,C2  
Add-ADGroupMember "VPN Users" C1,C2  
Add-ADGroupMember "WebUsers" C1,C2  
Add-ADGroupMember "Accounting Users" acctuser1  
Add-ADGroupMember "Graphics" graphics1  
Add-ADGroupMember "Customer Service" cust1  
Add-ADGroupMember "Executive" exec1

The 2025 LINSRV1 task specifically expects users **C1** and **C2** to be allowed SSH access and sudo rights after Linux integration.

---

# **8\. DNS Records to Create**

Create records for internal services:

Add-DnsServerResourceRecordA \-Name "winsrv1" \-ZoneName "manila.com" \-IPv4Address "192.168.2.10"  
Add-DnsServerResourceRecordA \-Name "linsrv1" \-ZoneName "manila.com" \-IPv4Address "192.168.1.10"  
Add-DnsServerResourceRecordA \-Name "w3" \-ZoneName "manila.com" \-IPv4Address "192.168.1.10"

Validate:

nslookup linsrv1.manila.com  
nslookup w3.manila.com

The MA2 task requires domain machines to resolve LINSRV1 by name.

---

# **9\. Configure Password Policy**

Open:

Group Policy Management  
Default Domain Policy  
Computer Configuration  
 \> Policies  
  \> Windows Settings  
   \> Security Settings  
    \> Account Policies  
     \> Password Policy

Set:

| Policy | Value |
| ----- | ----- |
| Minimum password length | 8 |
| Maximum password age | 30 days |
| Password complexity | Enabled |

The 2025 task requires domain user passwords to be **8 characters** and changed monthly.

Command alternative:

net accounts /minpwlen:8 /maxpwage:30 /domain

---

# **10\. Configure Fine-Grained Password Policy for Executives**

Run:

New-ADFineGrainedPasswordPolicy \`  
\-Name "Executive-Password-Policy" \`  
\-Precedence 1 \`  
\-MinPasswordLength 10 \`  
\-ComplexityEnabled $true \`  
\-MaxPasswordAge "30.00:00:00" \`  
\-LockoutThreshold 5 \`  
\-LockoutDuration "00:15:00" \`  
\-LockoutObservationWindow "00:15:00"

Apply to Executive group:

Add-ADFineGrainedPasswordPolicySubject \`  
\-Identity "Executive-Password-Policy" \`  
\-Subjects "Executive"

Validate:

Get-ADFineGrainedPasswordPolicy \-Filter \*

The MA2 task requires a fine-grained password policy for the executive group with a **10-character password** requirement.

---

# **11\. Configure Login Banner GPO**

Create a GPO:

New-GPO \-Name "login-banner" | New-GPLink \-Target "DC=manila,DC=com"

Configure manually in Group Policy Management:

Computer Configuration  
 \> Policies  
  \> Windows Settings  
   \> Security Settings  
    \> Local Policies  
     \> Security Options

Set:

Interactive logon: Message title for users attempting to log on  
\= WorldSkills ASEAN Manila

Interactive logon: Message text for users attempting to log on  
\= authorized access only

The required title is **WorldSkills ASEAN Manila**, and the required text is **authorized access only**.

---

# **12\. Create GPO: Control Panel Restriction**

Create GPO:

New-GPO \-Name "control"  
New-GPLink \-Name "control" \-Target "OU=Accounting,OU=Manila,DC=manila,DC=com"

Configure:

User Configuration  
 \> Policies  
  \> Administrative Templates  
   \> Control Panel  
    \> Prohibit access to Control Panel and PC settings  
\= Enabled

This GPO should apply only to Accounting users.

---

# **13\. Create GPO: Disable Registry Editing Tools**

Create GPO:

New-GPO \-Name "registry"  
New-GPLink \-Name "registry" \-Target "OU=Manila,DC=manila,DC=com"

Configure:

User Configuration  
 \> Policies  
  \> Administrative Templates  
   \> System  
    \> Prevent access to registry editing tools  
\= Enabled

The 2025 task says this is applicable to users in Manila.

---

# **14\. Chrome Enterprise Policy GPO**

The MA2 task says the Chrome Enterprise bundle is in the domain administrator’s Documents folder and should be used to create a GPO called **google**, forcing users to use `http://w3.manila.com` as the default homepage.

## **Steps**

1. Extract:

googleChromeEnterpriseBundle64.zip

2. Copy ADMX files to:

C:\\Windows\\PolicyDefinitions

3. Copy language files, usually `en-US`, to:

C:\\Windows\\PolicyDefinitions\\en-US

4. Create GPO:

New-GPO \-Name "google"  
New-GPLink \-Name "google" \-Target "DC=manila,DC=com"

5. Configure:

Computer Configuration or User Configuration  
 \> Policies  
  \> Administrative Templates  
   \> Google  
    \> Google Chrome  
     \> Home page

Set homepage:

http://w3.manila.com

Also configure:

Use New Tab Page as homepage \= Disabled  
Homepage URL \= http://w3.manila.com  
Action on startup \= Open a list of URLs  
URLs to open on startup \= http://w3.manila.com

---

# **15\. File Share: pictures**

The MA2 task requires a share on WinSRV1 at:

C:\\shares\\pictures

Shared as:

pictures

Permissions:

| Group | Permission |
| ----- | ----- |
| Customer Service | Read |
| Graphics | Modify |
| IT | Full Control |
| Everyone else | No access |

## **PowerShell Setup**

New-Item \-Path "C:\\shares\\pictures" \-ItemType Directory \-Force  
New-Item \-Path "C:\\shares\\pictures\\manila.jpg" \-ItemType File \-Force

Create SMB share:

New-SmbShare \`  
\-Name "pictures" \`  
\-Path "C:\\shares\\pictures" \`  
\-FullAccess "MANILA\\IT" \`  
\-ChangeAccess "MANILA\\Graphics" \`  
\-ReadAccess "MANILA\\Customer Service"

Remove inheritance and set NTFS permissions:

icacls "C:\\shares\\pictures" /inheritance:r  
icacls "C:\\shares\\pictures" /grant "MANILA\\IT:(OI)(CI)(F)"  
icacls "C:\\shares\\pictures" /grant "MANILA\\Graphics:(OI)(CI)(M)"  
icacls "C:\\shares\\pictures" /grant "MANILA\\Customer Service:(OI)(CI)(RX)"  
icacls "C:\\shares\\pictures" /remove "Users" "Authenticated Users" "Everyone"

Validate:

Get-SmbShare pictures  
Get-SmbShareAccess pictures  
icacls "C:\\shares\\pictures"

---

# **16\. Enable File Auditing for manila.jpg**

Enable audit policy:

auditpol /set /subcategory:"File System" /success:enable /failure:enable

Apply audit rule to `manila.jpg`:

$File \= "C:\\shares\\pictures\\manila.jpg"  
$Acl \= Get-Acl $File  
$AuditRule \= New-Object System.Security.AccessControl.FileSystemAuditRule(  
    "Everyone",  
    "ReadData",  
    "None",  
    "None",  
    "Success"  
)  
$Acl.AddAuditRule($AuditRule)  
Set-Acl $File $Acl

Validate:

(Get-Acl "C:\\shares\\pictures\\manila.jpg").Audit

Then test by reading the file from a client and checking:

Event Viewer  
 \> Windows Logs  
  \> Security

Look for file access audit events.

---

# **17\. Certificate Auto-Enrollment GPO Placeholder**

Do **not** configure this fully until WinSRV3 Issuing CA is ready.

But create the GPO now:

New-GPO \-Name "certenroll"  
New-GPLink \-Name "certenroll" \-Target "DC=manila,DC=com"

Later, after WinSRV3 CA is online, configure:

Computer Configuration  
 \> Policies  
  \> Windows Settings  
   \> Security Settings  
    \> Public Key Policies  
     \> Certificate Services Client \- Auto-Enrollment  
\= Enabled

The MA2 task requires a GPO named **certenroll** so domain computers automatically receive certificates from the issuing CA.

---

# **18\. Recommended Additional GPOs for the Appendix**

The task asks for three recommended GPOs to better secure the domain. Use these:

| GPO Name | Policy Path | Effect | Why Recommended |
| ----- | ----- | ----- | ----- |
| `screen-lock` | User Config → Admin Templates → Control Panel → Personalization | Locks workstation after inactivity | Prevents unattended access |
| `windows-firewall` | Computer Config → Windows Defender Firewall | Enforces host firewall | Reduces lateral movement risk |
| `disable-removable-storage` | Computer Config → Admin Templates → System → Removable Storage Access | Blocks USB storage access | Reduces malware and data leakage risk |

This maps cleanly to the required Table 2 format in the MA2 document.

---

# **19\. Final Validation Checklist**

Run these before moving to WinSRV3, pfSense, or LINSRV1.

## **AD / DNS**

dcdiag  
dcdiag /test:dns  
nslookup manila.com  
nslookup linsrv1.manila.com  
nslookup w3.manila.com

## **GPO**

On a domain client:

gpupdate /force  
gpresult /r

## **File Share**

From client:

\\\\winsrv1\\pictures

Test with:

Customer Service \= Read only  
Graphics \= Modify  
IT \= Full Control  
Other users \= No access

## **Firewall Relevance**

Make sure pfSense later allows LAN-to-Servers AD ports and PKI TCP 9389\. The MA2 firewall table requires AD-related ports from LAN to Servers and PKI TCP 9389\.

---

# **Priority Order**

Use this exact order:

1\. Rename server  
2\. Static IP  
3\. Install AD DS \+ DNS \+ GPMC \+ File Server  
4\. Promote to manila.com DC  
5\. Validate DNS  
6\. Create OUs, groups, users  
7\. Create DNS records  
8\. Configure password policy  
9\. Configure fine-grained executive password policy  
10\. Configure login banner  
11\. Configure control GPO  
12\. Configure registry GPO  
13\. Configure google Chrome GPO  
14\. Configure file share  
15\. Configure file auditing  
16\. Create certenroll GPO placeholder  
17\. Validate from client  
18\. Snapshot

Final snapshot name:

WINSRV1\_BASELINE\_COMPLETE

This WinSRV1 becomes the control plane for the rest of the lab: AD, DNS, GPO, file services, domain authentication, and the policy foundation for pfSense VPN, WinSRV3 certificates, and LINSRV1 domain integration.

# Internet bypass(temporary solution)

Type **`ncpa.cpl`** and press **Enter** to open the Network Connections control pane.  
Right-click your network card (labeled **`Ethernet0`**) and choose **Properties**.  
Double-click **Internet Protocol Version 4 (TCP/IPv4)** from the middle list box.  
Keep your working static IP parameters exactly as they are:

* **IP address:** `192.168.99.50`  
* **Subnet mask:** `255.255.255.0`  
* **Default gateway:** `192.168.99.254` *(Your pfSense WAN IP)*

Change the **Preferred DNS server** field to point directly to your firewall's WAN gateway:

* **Preferred DNS server:** **`192.168.99.254`**

Leave the Alternate DNS server field completely **blank**.  
Click **OK**, then click **OK** again to apply the settings.

netsh winsock reset  
netsh int ip reset  
ipconfig /flushdns

# Tab 33

