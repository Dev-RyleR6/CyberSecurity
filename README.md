# Ryle's A1 Network and Web Security Guide

Guide for WorldSkills Philippines CyberSecurity, Module A1.

Ryle's role is **Network Infrastructure / Routing / Firewall**. Earl handles most Linux and Windows operating-system administration. Ryle's main A1 responsibilities are external reconnaissance, HTTP/HTTPS behavior, TLS, exposed services, firewall validation, and proving whether website access restrictions actually work.

## Mission

> From the network side, can I reach anything I should not, is traffic protected properly, and does the website enforce the client's access restrictions?

A1 is an investigation, not a pentest. Use only provided credentials, observe configurations, and validate access. Do not exploit, modify, patch, or intentionally break the environment.

Use this workflow:

**Baseline -> Scan -> HTTP/HTTPS -> TLS -> Access Control -> Firewall -> Evidence -> Rank -> Write**

## 1. Prepare Evidence

```bash
mkdir -p ~/A1/evidence
cd ~/A1
date | tee evidence/00-start-time.txt
```

Record findings in this format:

```text
Candidate:
Command:
Result:
Security impact:
Severity guess:
Evidence file:
Keep for final report? YES/NO
```

Do not choose the final two vulnerabilities before collecting and comparing the evidence.

## 2. Baseline Network Verification

Replace placeholders with the supplied competition values.

```bash
ip addr
ip route
ping -c 4 <LINUXWEB-IP>
getent hosts <LINUXWEB-HOSTNAME>
```

Confirm that the client IP, gateway, LinuxWeb reachability, and hostname resolution match the documentation. Keep this phase short.

## 3. Full Service Reconnaissance

```bash
nmap -sV -p- <LINUXWEB-IP> \
	-oN evidence/01-nmap-full.txt
```

Consider each exposed service in context:

* `22/tcp`: likely expected administration.
* `80/tcp`: expected only if it redirects safely to HTTPS.
* `443/tcp`: expected for the protected website.
* Database or other unexpected ports: investigate why they are exposed, who can reach them, and whether another firewall is a compensating control.

An open port is not automatically a critical vulnerability. Establish its security impact before ranking it.

## 4. HTTP and HTTPS Behavior

```bash
curl -I http://<LINUXWEB-IP> \
	| tee evidence/02-http-headers.txt

curl -Ik https://<LINUXWEB-IP> \
	| tee evidence/03-https-headers.txt
```

Check whether:

* HTTP redirects to HTTPS.
* HTTPS works normally.
* Restricted paths exist on both protocols.
* Authentication requirements are identical.
* The `Server` header discloses unnecessary version or operating-system details.

A version banner is usually low priority. If HTTP serves the real site or restricted content instead of redirecting, investigate whether credentials or sensitive data can travel without encryption.

Walk through the site normally in a browser. Check documented or obvious paths such as `/public`, `/private`, `/restricted`, `/admin`, `/manual`, `/server-status`, and `/server-info`. Do not go beyond what is necessary to establish a finding.

## 5. TLS Validation

```bash
openssl s_client -connect <LINUXWEB-IP>:443 \
	</dev/null 2>&1 | tee evidence/04-tls-default.txt

openssl s_client -connect <LINUXWEB-IP>:443 -tls1 \
	</dev/null 2>&1 | tee evidence/05-tls1.txt

openssl s_client -connect <LINUXWEB-IP>:443 -tls1_2 \
	</dev/null 2>&1 | tee evidence/06-tls12.txt
```

TLS 1.0 or another deprecated protocol negotiating successfully is a candidate finding. TLS 1.2 should normally succeed.

Inspect the certificate:

```bash
openssl s_client -connect <LINUXWEB-IP>:443 \
	-servername <LINUXWEB-HOSTNAME> \
	</dev/null 2>&1
```

Check the subject, issuer, validity dates, hostname, trust result, and whether it is expired, self-signed, or issued by an untrusted authority. Certificate issues are often lower priority than a proven access-control failure or protected content over HTTP.

## 6. Access-Control Validation

This is the most important test. Coordinate with Earl: Earl reviews Apache and AD/LDAP configuration while Ryle validates the actual behavior from the client side.

Assume the challenge provides one authorized account and one non-member account.

### Anonymous request

```bash
curl -I https://<HOST>/restricted/ \
	| tee evidence/07-anonymous-user.txt
```

Expected results are `401 Unauthorized` or `403 Forbidden`, depending on the design. A `200 OK` response is suspicious.

### Authorized account

Do not put passwords directly into commands. Let `curl` prompt for the password.

```bash
curl -u <AUTHORIZED-USER> -I https://<HOST>/restricted/ \
	2>&1 | tee evidence/08-authorized-user.txt
```

The authorized account should receive `200 OK`.

### Non-member account

```bash
curl -u <NONMEMBER-USER> -I https://<HOST>/restricted/ \
	2>&1 | tee evidence/09-unauthorized-user.txt
```

The non-member should receive `403 Forbidden` or another denial. A successful `200 OK` response is strong evidence of broken access control and may be the highest-priority finding because it directly defeats the client's required restriction.

If permitted, confirm membership with Earl on WinSRV1:

```powershell
Get-ADGroupMember -Identity "AuthorizedWebUsers"
```

The strongest evidence chain is:

```text
AD says the authorized account is in the approved group.
AD says the non-member is not in the approved group.
The authorized account can access the page.
The non-member can also access the page.
Therefore the website authorization control has failed.
```

## 7. Check for an HTTP Access Bypass

Test both anonymous and authorized requests over HTTP:

```bash
curl -I http://<HOST>/restricted/ \
	| tee evidence/10-restricted-http-anonymous.txt

curl -u <AUTHORIZED-USER> -I http://<HOST>/restricted/ \
	2>&1 | tee evidence/11-restricted-http-authorized.txt
```

Protected content should redirect to HTTPS. A successful response over HTTP, especially when Basic Authentication is used, can expose credentials and restricted data to interception or manipulation. Record the exact status and `Location` header.

## 8. Firewall Validation

If direct inspection is in scope:

```bash
sudo firewall-cmd --list-all \
	| tee evidence/12-firewalld.txt
```

For every unexpected service, ask:

* Is this service required?
* Who can reach it?
* Is another firewall blocking it?
* Is there a documented compensating control?

The marking scheme rewards recognizing compensating controls and decoys. Do not label an exposed port critical without proving its impact.

## 9. Secondary Checks

These checks support Earl's host-based review when the primary network work is complete:

```bash
ps aux | grep httpd
ls -la /var/www/html
sestatus
ls -Z /var/www/html
```

Look for Apache workers running as root and world-writable web content. An SELinux context mismatch by itself is generally informational, not a final vulnerability.

Check for accidentally published files without downloading or abusing sensitive content:

```bash
curl -I http://<HOST>/.git/config
curl -I http://<HOST>/backup.zip
curl -I http://<HOST>/config.php.bak
```

`404 Not Found` is good. Investigate a `200 OK` response and save only the evidence needed to establish the exposure.

## 10. Coordinate with Earl

While Ryle runs `nmap`, `curl`, `openssl`, browser validation, and firewall checks, Earl should investigate:

```bash
apachectl -S
httpd -M
grep -RniE 'AuthType|AuthLDAPURL|Require ldap-group|AllowOverride' /etc/httpd
```

Earl should also review the Apache configuration and logs. Meet at the access-control test: configuration evidence should confirm or contradict the observed behavior, not replace it.

## 11. Rank and Select Exactly Two Findings

Do not submit every candidate. Submit exactly two findings, selected by evidence and real-world impact.

Typical priority order in this scenario:

1. Non-authorized user can access restricted content: Critical when directly proven.
2. Restricted content or authentication remains available over HTTP: High.
3. Unexpected service exposure: severity depends on necessity, reachability, and compensating controls.
4. Deprecated TLS protocol: usually Medium.
5. Apache version disclosure or certificate warnings: often Low to Medium.

Avoid duplicates. For every selected finding, identify the failing control, business impact, exact command output, evidence file, severity, and recommended fix.

## 12. Evidence Checklist

```text
evidence/
├── 00-start-time.txt
├── 01-nmap-full.txt
├── 02-http-headers.txt
├── 03-https-headers.txt
├── 04-tls-default.txt
├── 05-tls1.txt
├── 06-tls12.txt
├── 07-anonymous-user.txt
├── 08-authorized-user.txt
├── 09-unauthorized-user.txt
├── 10-restricted-http-anonymous.txt
├── 11-restricted-http-authorized.txt
└── 12-firewalld.txt
```

For each final finding, be able to answer:

* What command proved it?
* What exact output proved it?
* Where is that evidence stored?
* When was it captured?
* What is the actual impact?
* Why is it more important than the findings rejected?

## 13. Report Structure

The executive summary is for a non-technical client. Lead with the business impact, not configuration details. Table 1 should contain exactly two findings, each with:

* Vulnerability
* Description
* Severity and risk rating
* Why it is a problem
* Evidence
* Recommendation
* Implementation sequence or validation steps

For a broken access-control finding, explain that a user outside the approved group can view protected content and recommend correcting the Apache authorization configuration, then retesting with both an approved member and a non-member.

For protected content over HTTP, explain that authentication information or restricted data may be intercepted and recommend forcing HTTP to HTTPS and validating that protected content is available only through TLS.

## Mental Decision Tree

```text
Is LinuxWeb reachable?
	-> What ports and services are exposed?
	-> Does HTTP redirect to HTTPS?
	-> Does HTTPS work?
	-> Are deprecated TLS protocols accepted?
	-> Can anonymous users access restricted content?
	-> Can authorized users access it?
	-> Can non-authorized users access it?
			 YES -> high-priority access-control candidate
	-> Is protected content accessible over HTTP?
	-> Is anything unexpected allowed through the firewall?
	-> Compare evidence, remove decoys, pick exactly two
```

The goal is not to find the strangest configuration. The goal is to turn command output into a defensible decision: identify the failing control, prove the impact, check for compensating controls, preserve the evidence, and choose the two findings that matter most.
