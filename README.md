#**FOOTPRINTING & NETWORL SCANNING**

---

## Overview

This covers footprinting the networkwalks.com domain using multiple Kali Linux tools  and scanning my own local network with Zenmap . One module covers the footprinting phase and the other covers the scanning phase, so together they show how an attacker moves from gathering public information to mapping live hosts on a network. 
All commands were run in Kali Linux (footprinting) and on a Windows PC with Zenmap installed (scanning). Every step below includes the exact command used, the result I observed, and a short note on why the finding matters from an attacker's point of view.



---

## 🎯 Objectives

- Run WHOIS enumeration on the target domain
- Fingerprint the web stack with WhatWeb
- Confirm DNS resolution with `nslookup`
- Inspect HTTP response headers with `curl`
- Detect the presence of a WAF with `wafw00f`
- Enumerate DNS records in depth with `dnsrecon`
- Save output to file for later reference

---

## 🪜 Tasks & Findings

### 1. WHOIS Lookup

```
whois networkwalks.com
```

Registrar: GoDaddy.com, LLC
Domain created: 2019-11-06
Registry expiry: 2027-11-06
Name servers: NS6135/NS6136.HOSTGATOR.COM, NS29/NS30.DOMAINCONTROL.COM
Registrant info: privacy-protected via Domains By Proxy, LLC (Tempe, AZ) — no personal registrant details exposed

**What this tells me:** the domain uses WHOIS privacy protection, so ownership details aren't directly exposed — pretty standard and sensible setup.

Output saved to `whois.txt`.



### 2. WhatWeb Fingerprinting

```
whatweb networkwalks.com
```

Key findings:
- Server: Apache
- CMS: WordPress 7.1
- Plugin: WordPress Download Manager 3.3.58
- Front-end: Bootstrap 7.1, jQuery 3.7.1, HTML5
- Uses Google Tag Manager
- IP: 192.232.216.135
- HTTP → HTTPS redirect confirmed (301 on port 80)



### 3. DNS Resolution Check

```
nslookup networkwalks.com
```

Resolves via `8.8.8.8` to `192.232.216.135` — matches the IP WhatWeb reported, so it's a single consistent A record rather than something like a CDN doing per-region resolution.



### 4. HTTP Header Inspection

```
curl -I https://networkwalks.com
```

Notable headers:
- `permissions-policy` includes Cloudflare, Google reCAPTCHA, and hCaptcha references — suggests bot/anti-automation protections are in play somewhere in the stack even though the WAF check (below) points to ModSecurity, not Cloudflare directly
- `link` header exposes the WordPress REST API base (`/wp-json/`) and a specific page ID (`/wp-json/wp/v2/pages/53`)
- `set-cookie: __wpdm_client=...` — flagged `secure; HttpOnly`, tied to the WordPress Download Manager plugin seen in WhatWeb
- `x-nginx-cache: WordPress` — despite Apache being the reported server, there's an Nginx caching layer in front of it
- `x-endurance-cache-level: 0` — Endurance is a hosting-group caching header (HostGator's parent company), consistent with the HostGator nameservers seen in WHOIS



### 5. WAF Detection

```
wafw00f networkwalks.com
```

Result: site is behind **ModSecurity (SpiderLabs) WAF**.



### 6. Deep DNS Enumeration

```
dnsrecon -d networkwalks.com
```

Key records:
- SOA: `ns6135.hostgator.com`
- NS: `ns6135` / `ns6136.hostgator.com` (BIND version `9.16.23-RH` disclosed on both)
- MX: `mail.networkwalks.com → 192.232.216.135` — same IP as the main A record, so mail and web appear to sit on the same host
- A: `networkwalks.com → 192.232.216.135`
- TXT: Google site verification string, and an SPF record (`v=spf1 +a +mx +ip4:50.87.144.87 +include:websitewelcome.com ~all`)
- SRV: 8 `_autodiscover._tcp` records pointing to `cpanelemaildiscovery.cpanel.net`, spread across multiple IPs — standard cPanel-hosted email autodiscover setup


---

## 🔎 Summary of What Was Learned

| Category | Finding |
|---|---|
| Hosting | HostGator-managed nameservers, cPanel email autodiscover |
| Registrar | GoDaddy, WHOIS-privacy protected |
| Web stack | Apache + Nginx cache layer + WordPress 7.1 |
| Plugins | WordPress Download Manager 3.3.58 |
| Protection | ModSecurity (SpiderLabs) WAF; Cloudflare/reCAPTCHA/hCaptcha referenced in policy headers |
| DNS hygiene | BIND version disclosed on both nameservers; MX and A records point to the same IP; SPF record present |

---

## 💡 What I Learned

- **Passive recon adds up fast.** No single tool gave the full picture on its own, but combined, whois + whatweb + dnsrecon + curl + wafw00f built a pretty detailed map of the target's hosting, CMS, plugins, and defenses — all without sending a single "abnormal" request.
- **Version disclosure is a real finding, even when passive.** The BIND version leak and the WordPress plugin version are exactly the kind of detail a real engagement report would flag, since they're the first thing someone checks against a CVE database.
- **Layered infrastructure isn't always obvious from one tool.** It took comparing `curl`'s headers against WhatWeb's server report to notice the Nginx cache sitting in front of Apache — good reminder to cross-check tool outputs instead of trusting just one.
- **A WAF changes the rules for anything active.** Confirming ModSecurity was there before doing any further testing matters — active tests without accounting for a WAF risk getting blocked, rate-limited, or generating noisy logs on someone else's infrastructure.

---

## 🔗 Tools Used

- **whois** (built-in)
- **WhatWeb**
- **nslookup** (built-in)
- **curl** (built-in)
- **wafw00f**
- **dnsrecon**

  ## Reference
  Please refer attached project report in pdf format for detailed information about Tools used,Screenshots, risks identified and for detailed analysis.

---

## 👤 Author

**Kavith A L**\
Cybersecurity Engineer

LinkedIN:
https://www.linkedin.com/in/kavithaal/
---

## 📌 Project Information

**Program Name:** Cybersecurity at Networkwalks | **Week:** 02 | **Project:** Footprinting & Network Scanning | **Repository:** GitHub
