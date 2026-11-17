# PENETRATION TESTING REPORT

### Footprinting & Reconnaissance Phase

**W2-PM-FINAL | CYBERSECURITY | NETWORKWALKS**

| | |
|---|---|
| **Pentester Name (Cybersecurity Professional)** | Joseph Wilson |
| **Program/Batch** | B084-Networkwalks |
| **Date** | 14–16 November 2026 |
| **Modules completed** | W2-PM1 (Multiple Kali Tools), W2-PM4 (theHarvester) |
| **Client/Target** | 1. networkwalks.com (written permission secured) 2. microsoft.com (lab-specified target, passive OSINT only) |
| **Permission secured from client?** | Yes |
| **Phases covered** | **Phase 1:** Reconnaissance & Footprinting. **Phase 2–5:** In progress |

---

## 1. Liability Disclaimer

I have performed these activities only on the systems and devices where I had secured written permission, or on the devices and systems that I own myself. All these materials are for education and research purposes only. Do not use anything from here to break the law. The instructor, the authors and Networkwalks are not responsible for what you do with this knowledge. Every action you take is your own responsibility. Misuse can lead to criminal charges, heavy fines, loss of your job and a permanent record. In most countries unauthorised access is a crime even when nothing is damaged.

Hacking is only legal when:
- You test a device or network that you own, or your lab environment
- You have written and documented permission from the owner
- You are working as a security professional under a signed agreement with an agreed scope

---

## 2. Introduction

This report covers footprinting the **networkwalks.com** domain using six Kali Linux tools (W2-PM1) and gathering emails and sub-domains with **theHarvester** (W2-PM4). Both modules belong to the reconnaissance phase, which is where an attacker collects public information about a target before touching it. It is the Week 2 part of my ongoing internship program at Networkwalks.

All commands were run in Kali Linux running in a VirtualBox VM. Every step below includes the exact command used, the result I observed, a screenshot as evidence, and a short note on why the finding matters from an attacker's point of view.

---

## 3. Tools Used

| Tool | Purpose |
|---|---|
| Kali Linux (VirtualBox VM) | Operating system used for all reconnaissance activities |
| WHOIS | Find domain registration details (registrar, dates, name servers) |
| whatweb | Fingerprint web technologies (server, CMS, plugins, IP) |
| nslookup | Resolve the domain name to its IP address using DNS |
| curl -I | Read the HTTP response headers of the website |
| wafw00f | Detect whether a Web Application Firewall protects the site |
| dnsrecon | Enumerate DNS records (SOA, NS, MX, A, TXT/SPF, SRV) |
| theHarvester | Gather emails, sub-domains and hosts from public sources (OSINT) |

---

## 4. Activities Performed

### 4.1 Footprinting & Reconnaissance (W2-PM1)

Six Kali Linux tools were used against **networkwalks.com**. Each one reveals a different piece of the target, and together they build a full profile of it.

#### Task 1: WHOIS (domain registration)

**Command:** `whois networkwalks.com`

**Result:** The domain is registered through **GoDaddy.com, LLC**. It was created on **2019-11-06**, last updated on 2025-11-12, and expires on **2027-11-06**. The name servers are **NS6135.HOSTGATOR.COM** and **NS6136.HOSTGATOR.COM**. DNSSEC is unsigned, and the domain has the client-side lock statuses (delete, renew, transfer and update prohibited).

![WHOIS](screenshots/01-whois.png)

**Why it matters:** The name servers show that the site is hosted with HostGator, so an attacker learns the hosting provider instantly. The registrar, dates and abuse contact also help with planning and social engineering.

#### Task 2: whatweb (technology fingerprinting)

**Command:** `whatweb networkwalks.com`

**Result:** The site runs on **Apache** and redirects HTTP to HTTPS (301). It uses **WordPress 7.1.4** with the **WordPress Download Manager 3.3.76** plugin, **jQuery 3.7.1** and Google Tag Manager. whatweb also exposed the server IP **192.232.216.135**, the contact email **info@networkwalks.com**, the page title "Networkwalks Academy" and a `__wpdm_client` cookie.

![whatweb](screenshots/02-whatweb.png)

**Why it matters:** Exact software and plugin versions can be looked up in vulnerability databases to find known issues. The tool also leaks the server IP and an email address.

#### Task 3: nslookup (DNS resolution)

**Command:** `nslookup networkwalks.com`

**Result:** Using the DNS server 8.8.8.8, the domain resolved to **192.232.216.135** (non-authoritative answer).

![nslookup](screenshots/03-nslookup.png)

**Why it matters:** Knowing the real IP address lets an attacker scan the server directly, check for other sites on the same IP and map the infrastructure.

#### Task 4: curl -I (HTTP headers)

**Command:** `curl -I https://networkwalks.com`

**Result:** The server answered `HTTP/1.1 200 OK` with `Server: Apache`. The headers exposed the WordPress REST API endpoints (`/wp-json/` and `/wp-json/wp/v2/pages/53`), a `X-nginx-cache: WordPress` header, `X-Endurance-Cache-Level: 0`, and a secure, HttpOnly `__wpdm_client` cookie.

![curl headers](screenshots/04-curl-headers.png)

**Why it matters:** Headers leak the web server, the caching stack and hidden endpoints such as the WordPress REST API. An attacker can fingerprint the stack and find entry points without loading the full page.

#### Task 5: wafw00f (Web Application Firewall detection)

**Command:** `wafw00f networkwalks.com`

**Result:** wafw00f v2.4.2 detected that the site is behind **ModSecurity (SpiderLabs) WAF**, using 2 requests.

![wafw00f](screenshots/05-wafw00f.png)

**Why it matters:** Knowing a WAF is present shapes the whole attack, because naive attempts will be blocked or logged and the attacker must adapt.

#### Task 6: dnsrecon (DNS enumeration)

**Command:** `dnsrecon -d networkwalks.com`

**Result:** dnsrecon found **8 records** and completed the enumeration:

- **SOA:** ns6135.hostgator.com (50.87.144.87)
- **NS:** ns6135.hostgator.com (50.87.144.87) and ns6136.hostgator.com (192.232.216.131)
- **DNS software:** Bind version **9.16.23-RH** on both name servers
- **MX:** mail.networkwalks.com (192.232.216.135)
- **A:** networkwalks.com (192.232.216.135)
- **TXT:** a Google site verification record and an **SPF** policy (`v=spf1 +a +mx +ip4:50.87.144.87 +include:websitewelcome.com ~all`)
- **SRV:** `_autodiscover._tcp` records pointing to `cpanelemaildiscovery.cpanel.net`
- A DNSSEC query returned no answer, which matches the unsigned DNSSEC seen in WHOIS.

![dnsrecon start](screenshots/06-dnsrecon-start.png)
![dnsrecon end](screenshots/06-dnsrecon-end.png)

**Why it matters:** dnsrecon maps the whole DNS footprint: mail servers, DNS software version, SPF policy and cPanel service records. Each record helps an attacker understand the email and hosting setup.

---

### 4.2 Footprinting with theHarvester (W2-PM4)

theHarvester (v4.11.1) gathers emails, sub-domains and hosts from public sources. It is passive recon, so it never touches the target directly. The target for this module was **microsoft.com**, as specified in the lab.

#### Task 1: theHarvester with Baidu (limit 1000)

**Command:** `theHarvester -d microsoft.com -l 1000 -b baidu`

**Result:** The Baidu source returned **no IPs, no emails, no people and no hosts** in this run. The lab note explains that results vary because the data sources keep changing, so a source returning nothing is a valid result.

![Baidu command](screenshots/07-harvester-baidu-command.png)
![Baidu result](screenshots/07-harvester-baidu-result.png)

#### Task 2: theHarvester with all sources (limit 50)

**Command:** `theHarvester -d microsoft.com -l 50 -b all`

**Result:** Querying all sources returned a long list of **sub-domains and hosts** under microsoft.com (for example, `zeus.buddy.microsoft.com` resolving to 131.107.86.4). Several sources (bevigil, Bitbucket, bufferoverun, BuiltWith, SecurityScorecard) were skipped because **no API key** was configured, which is expected in a default Kali install.

![theHarvester all sources start](screenshots/08-harvester-all-start.png)
![theHarvester all sources end](screenshots/08-harvester-all-end.png)

**Why it matters:** Each harvested email is a possible phishing or password-attack target, and each sub-domain is another entry point that a defender may have forgotten about. Different sources return different results, so trying more than one is important.

---

## 5. Risk Analysis / Impact

| # | Risk / Finding | Evidence / Observation | Potential Impact | Risk Level |
|---|---|---|---|---|
| 1 | Registration and hosting details public | WHOIS showed GoDaddy as registrar and HostGator name servers | Reveals the hosting provider and domain dates for planning | 🟢 Low |
| 2 | Web technology and version information exposed | whatweb identified WordPress 7.1.4 and WP Download Manager 3.3.76 | Attackers may use exposed version information to find software that needs further review | 🟡 Medium |
| 3 | Server IP address identifiable | nslookup resolved the domain to 192.232.216.135 | Shows the network location of the web service | 🟢 Low |
| 4 | HTTP technical information exposed | curl returned headers and exposed `/wp-json/` | May assist fingerprinting and further enumeration | 🟢 Low |
| 5 | WAF technology identifiable | wafw00f identified ModSecurity (SpiderLabs) | Reveals information about the security architecture | 🟢 Low |
| 6 | DNS infrastructure information exposed | dnsrecon found Bind 9.16.23, SPF, MX and SRV records | Helps build a broader infrastructure profile | 🟡 Medium |
| 7 | Sub-domains and hosts discoverable from public sources | theHarvester listed many hosts for the target domain | Forgotten or exposed sub-domains enlarge the attack surface | 🟡 Medium |

**Risk level key:** 🔴 Critical  🟡 Medium  🟢 Low

The risks above are observations from the footprinting exercises, **not confirmed vulnerabilities**. These modules involved information gathering only. No exploitation or vulnerability validation was performed. The presence of a software version, IP address or DNS record does not by itself mean that a system is vulnerable. Further authorized security testing would be required to confirm any actual vulnerability.

---

## 6. Recommendations

- **Review publicly exposed technology information:** Regularly check what your web technologies, CMS and plugins reveal publicly.
- **Keep software updated:** Update the CMS, plugins and other web technologies, and review them against current security advisories.
- **Review HTTP headers:** Remove unnecessary technical information from response headers and restrict the REST API where it is not needed.
- **Review DNS records regularly:** Make sure only required records and services are publicly exposed, and keep DNS server software (Bind) patched.
- **Consider enabling DNSSEC:** DNSSEC is currently unsigned, so enabling it would protect against forged DNS answers.
- **Properly configure and monitor the WAF:** Keep ModSecurity enabled and tuned.
- **Limit exposed emails and sub-domains:** Use role-based mailboxes, and audit sub-domains regularly so forgotten or test hosts are removed or protected.
- **Run the same tools on yourself:** Defenders should see what an attacker would see, then reduce what they leak.
- **Perform security testing with authorization:** Reconnaissance should only be performed against systems where appropriate authorization has been provided.

---

## 7. Conclusion

During Week 2 of my Cybersecurity & Ethical Hacking internship, I completed practical activities covering footprinting and reconnaissance.

With the six Kali Linux tools, I learned how WHOIS shows registration and hosting details, whatweb identifies web technologies, nslookup resolves domain names, curl inspects HTTP headers, wafw00f identifies a WAF, and dnsrecon maps DNS records. With theHarvester, I learned how emails, sub-domains and hosts can be gathered from public sources without touching the target, and that different sources give different results.

The exercises showed me that information gathering is an important part of cybersecurity. Even before attempting to exploit a system, a security professional can learn a significant amount about an environment from publicly available information. I also learned that technical findings should be documented clearly: what was done, what was found, what it means, what risk it creates and how to reduce it.

Finally, I learned that reconnaissance must always be performed within an authorized scope. These activities were completed as part of the assigned educational cybersecurity lab.

---

## 8. Evidences Collected

| # | Evidence | File |
|---|---|---|
| 1 | WHOIS output | `screenshots/01-whois.png` |
| 2 | whatweb output | `screenshots/02-whatweb.png` |
| 3 | nslookup output | `screenshots/03-nslookup.png` |
| 4 | curl -I output | `screenshots/04-curl-headers.png` |
| 5 | wafw00f output | `screenshots/05-wafw00f.png` |
| 6 | dnsrecon output (2 parts) | `screenshots/06-dnsrecon-start.png`, `06-dnsrecon-end.png` |
| 7 | theHarvester, Baidu (2 parts) | `screenshots/07-harvester-baidu-command.png`, `07-harvester-baidu-result.png` |
| 8 | theHarvester, all sources (2 parts) | `screenshots/08-harvester-all-start.png`, `08-harvester-all-end.png` |
| 9 | Text outputs of each command | `outputs/` |

-End-

---

### 👤 Author

**Joseph Wilson**, Cybersecurity Professional, B084


### 📌 Project Information

**Program Name:** Cybersecurity program at Networkwalks | **Week:** 02 | **Repository:** GitHub
