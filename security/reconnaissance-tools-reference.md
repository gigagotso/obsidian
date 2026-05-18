# Reconnaissance Tools Reference

A practical, categorized guide to modern external reconnaissance and attack surface management tooling. Focused on tools that are actively maintained as of 2026 and the pipelines security engineers actually use.

> **Why this document exists:** Older tools like Sublist3r are effectively obsolete because their scraping-based architecture doesn't survive modern bot protection, and they miss the highest-signal modern source (Certificate Transparency logs). This guide covers what replaced them.

---

## Table of Contents

1. [The Recon Funnel — Mental Model](#1-the-recon-funnel--mental-model)
2. [Passive Subdomain Discovery](#2-passive-subdomain-discovery)
3. [Active Subdomain Discovery](#3-active-subdomain-discovery)
4. [DNS Resolution & Validation](#4-dns-resolution--validation)
5. [HTTP/HTTPS Probing](#5-httphttps-probing)
6. [Port Scanning](#6-port-scanning)
7. [Internet-Wide Search Engines](#7-internet-wide-search-engines)
8. [Web Crawling & Content Discovery](#8-web-crawling--content-discovery)
9. [JavaScript Analysis](#9-javascript-analysis)
10. [Vulnerability Scanning](#10-vulnerability-scanning)
11. [Cloud Asset Discovery](#11-cloud-asset-discovery)
12. [OSINT & Metadata](#12-osint--metadata)
13. [Email & People Enumeration](#13-email--people-enumeration)
14. [WHOIS, ASN, IP Intelligence](#14-whois-asn-ip-intelligence)
15. [All-in-One Frameworks](#15-all-in-one-frameworks)
16. [Continuous Monitoring](#16-continuous-monitoring)
17. [Recommended Starter Pipelines](#17-recommended-starter-pipelines)
18. [ISO 27001 / Compliance Context](#18-iso-27001--compliance-context)
19. [Installation Cheatsheet](#19-installation-cheatsheet)

---

## 1. The Recon Funnel — Mental Model

External recon follows a funnel from broadest to most specific:

```
Organization / Brand
        │
        ▼
     Domains
        │
        ▼
   Subdomains
        │
        ▼
  Resolved IPs
        │
        ▼
   Open Ports
        │
        ▼
  HTTP Services
        │
        ▼
URLs / Parameters
        │
        ▼
 Vulnerabilities
```

Each category below maps to one or more stages of this funnel. A real pipeline picks one or two tools per stage and chains them.

---

## 2. Passive Subdomain Discovery

Finding subdomains **without sending traffic to the target**. Pulls from Certificate Transparency logs, search engines, threat intel feeds, and scan databases.

| Tool | Strength | Notes |
|------|----------|-------|
| **subfinder** | Daily driver, ~30 sources | ProjectDiscovery, actively maintained |
| **amass** (passive) | Most thorough aggregator | OWASP project, slower |
| **assetfinder** | Minimalist, ~10 sources | tomnomnom, no config needed |
| **chaos** | Curated bug bounty dataset | ProjectDiscovery, free with API key |
| **crt.sh** | Direct CT log queries | Free, no auth, JSON output |
| **Censys** | TLS/cert hunting | Free tier, better UI |

### Examples

**subfinder — basic usage:**
```bash
subfinder -d example.com -silent
```

**subfinder — all sources, with API keys configured:**
```bash
subfinder -d example.com -all -silent -o subs.txt
```

API keys go in `~/.config/subfinder/provider-config.yaml`:
```yaml
censys:
  - "API_ID:API_SECRET"
securitytrails:
  - "YOUR_KEY"
shodan:
  - "YOUR_KEY"
virustotal:
  - "YOUR_KEY"
```

**crt.sh — direct CT log query:**
```bash
curl -s "https://crt.sh/?q=%25.example.com&output=json" \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' \
  | sort -u
```

**amass — passive aggregation:**
```bash
amass enum -passive -d example.com -src
```

**chaos — pull from ProjectDiscovery's dataset:**
```bash
chaos -d example.com -silent
```

**Combining sources (recommended pattern):**
```bash
{
  subfinder -d example.com -all -silent
  curl -s "https://crt.sh/?q=%25.example.com&output=json" | jq -r '.[].name_value'
  chaos -d example.com -silent
} | sort -u > all-subs.txt
```

---

## 3. Active Subdomain Discovery

Actually hitting DNS to find what passive sources miss. Brute-force, permutation, zone walking.

| Tool | Strength | Notes |
|------|----------|-------|
| **puredns** | Best wildcard handling | Wraps massdns with proper filtering |
| **shuffledns** | ProjectDiscovery equivalent | Integrates with the rest of their pipeline |
| **gobuster dns** | Simple, well-known | Single-binary Go tool |
| **massdns** | Underlying engine | Raw speed, less polish |
| **altdns / gotator / dnsgen / ripgen** | Permutation generators | Feed seeds in → get candidates out |

### Examples

**puredns brute-force:**
```bash
puredns bruteforce all.txt example.com \
  -r resolvers.txt \
  --write valid-subs.txt
```

**Get a good resolver list:**
```bash
curl -s https://raw.githubusercontent.com/trickest/resolvers/main/resolvers.txt > resolvers.txt
```

**Recommended wordlists:**
- [n0kovo's subdomain wordlists](https://github.com/n0kovo/n0kovo_subdomains)
- [assetnote commonspeak2](https://github.com/assetnote/commonspeak2-wordlists)
- jhaddix's `all.txt`

**dnsgen — permutations from known subs:**
```bash
cat known-subs.txt | dnsgen - | puredns resolve - -r resolvers.txt
```

**gotator — more permutation control:**
```bash
gotator -sub known-subs.txt -perm permutations.txt -depth 1 -numbers 3 -mindup -adv > candidates.txt
```

---

## 4. DNS Resolution & Validation

Once you have a candidate list, resolve it efficiently and filter out wildcards.

| Tool | Strength |
|------|----------|
| **dnsx** | Fast, scriptable, the standard (ProjectDiscovery) |
| **massdns** | Raw speed, less polish |
| **zdns** | Academic-grade, very fast for huge lists |

### Examples

**dnsx — resolve and show records:**
```bash
dnsx -l subs.txt -silent -a -resp
```

**dnsx — filter to only resolvable hosts:**
```bash
cat subs.txt | dnsx -silent > resolved.txt
```

**dnsx — find CNAME records (useful for subdomain takeover hunting):**
```bash
dnsx -l subs.txt -silent -cname -resp
```

---

## 5. HTTP/HTTPS Probing

Which resolved hosts are actually serving web content, on what ports, with what tech stack.

| Tool | Strength |
|------|----------|
| **httpx** | Title, status, tech detection, TLS info, screenshots |
| **httprobe** | Older, simpler (tomnomnom) |
| **aquatone / gowitness / eyewitness** | Screenshot grids |

### Examples

**httpx — basic probe:**
```bash
httpx -l resolved.txt -silent
```

**httpx — full enrichment:**
```bash
httpx -l resolved.txt \
  -silent \
  -title \
  -status-code \
  -tech-detect \
  -tls-grab \
  -web-server \
  -content-length \
  -o http-results.txt
```

**httpx — screenshot every live host:**
```bash
httpx -l resolved.txt -silent -screenshot -srd screenshots/
```

**gowitness — alternative screenshot tool:**
```bash
gowitness file -f resolved.txt
gowitness report serve
```

---

## 6. Port Scanning

| Tool | Strength |
|------|----------|
| **naabu** | Fast TCP SYN, plays nice with the rest of the pipeline |
| **masscan** | Fastest, internet-scale |
| **rustscan** | masscan-fast with nmap-like UX |
| **nmap** | Slowest, but unmatched for service/version detection and NSE |

### Examples

**naabu — top 1000 ports:**
```bash
naabu -l hosts.txt -silent -top-ports 1000
```

**naabu — specific ports + nmap follow-up:**
```bash
naabu -l hosts.txt -silent -p 80,443,8080,8443 -nmap-cli 'nmap -sV -sC'
```

**masscan — fast wide scan:**
```bash
sudo masscan -p1-65535 192.0.2.0/24 --rate 10000 -oX scan.xml
```

**nmap — deep dive on known-open ports:**
```bash
nmap -sV -sC -p 80,443,8080 -iL hosts.txt -oA nmap-results
```

> **Standard pattern:** masscan/naabu finds open ports fast → nmap `-sV -sC` deep-scans only those ports.

---

## 7. Internet-Wide Search Engines

Pre-scanned data — query instead of scan. Especially powerful when you already know the fingerprint you're hunting.

| Service | Notes |
|---------|-------|
| **Shodan** | The original, broad coverage |
| **Censys** | Cleaner data, better for TLS/cert hunting |
| **FOFA** | Chinese, sometimes finds different things |
| **ZoomEye** | Chinese, Shodan-like |
| **Netlas** | Newer, generous free tier |
| **Hunter.how** | Newer, growing |
| **BinaryEdge** | Solid, less crowded |

### Examples

**Shodan CLI:**
```bash
shodan init YOUR_API_KEY
shodan search 'hostname:example.com'
shodan search 'ssl.cert.subject.cn:example.com'
shodan host 192.0.2.1
```

**Common Shodan dorks:**
```
hostname:example.com
ssl:"Example Inc"
org:"Example Inc"
http.title:"Login" ssl:example.com
product:nginx country:GE
```

**Censys CLI:**
```bash
censys search 'services.tls.certificates.leaf_data.subject.common_name: example.com'
```

> Use two or three of these in parallel — they each scan slightly different things.

---

## 8. Web Crawling & Content Discovery

Finding URLs, parameters, hidden endpoints, JS-referenced paths.

| Tool | Strength |
|------|----------|
| **katana** | Modern crawler, headless mode (ProjectDiscovery) |
| **gospider / hakrawler** | Older alternatives |
| **gau / waybackurls** | Historical URLs from Wayback, Common Crawl, AlienVault |
| **ffuf** | Fastest content/parameter fuzzer |
| **feroxbuster** | Rust-based recursive directory brute-forcer |
| **arjun / x8** | HTTP parameter discovery |

### Examples

**katana — crawl with headless browser:**
```bash
katana -u https://example.com -d 3 -jc -kf all -o urls.txt
```

**gau — pull historical URLs:**
```bash
echo "example.com" | gau --threads 5 > historical-urls.txt
```

**waybackurls — Wayback Machine specifically:**
```bash
echo "example.com" | waybackurls > wayback.txt
```

**ffuf — directory fuzzing:**
```bash
ffuf -u https://example.com/FUZZ \
  -w /usr/share/wordlists/dirb/common.txt \
  -mc 200,301,302,403 \
  -o ffuf-results.json
```

**ffuf — parameter fuzzing:**
```bash
ffuf -u 'https://example.com/api?FUZZ=test' \
  -w params.txt \
  -fs 1234   # filter out 1234-byte responses
```

**feroxbuster — recursive directory discovery:**
```bash
feroxbuster -u https://example.com -w wordlist.txt -t 50 -d 3
```

**arjun — find hidden parameters:**
```bash
arjun -u https://example.com/api/endpoint -m GET
```

---

## 9. JavaScript Analysis

JS files leak endpoints, API keys, internal hostnames constantly.

| Tool | Strength |
|------|----------|
| **subjs** | Extract JS URLs from a target |
| **LinkFinder / SecretFinder** | Pull endpoints and secrets out of JS |
| **JSluice** | Newer, better parsing |
| **nuclei** | With secret-detection templates |

### Examples

**subjs — find all JS files:**
```bash
echo "https://example.com" | subjs > js-urls.txt
```

**LinkFinder — extract endpoints from a JS file:**
```bash
python3 linkfinder.py -i https://example.com/app.js -o cli
```

**JSluice — modern alternative:**
```bash
cat app.js | jsluice urls
cat app.js | jsluice secrets
```

**nuclei with exposure templates:**
```bash
nuclei -l js-urls.txt -t exposures/ -severity medium,high,critical
```

---

## 10. Vulnerability Scanning

| Tool | Strength |
|------|----------|
| **nuclei** | Template-based, the de facto standard; thousands of community templates |
| **nikto** | Old but still finds things |
| **nmap NSE** | Service-specific checks |
| **wpscan** | WordPress-specific |
| **trivy / grype** | Container/dependency scanning |

### Examples

**nuclei — basic scan:**
```bash
nuclei -l hosts.txt -severity medium,high,critical
```

**nuclei — specific template categories:**
```bash
nuclei -l hosts.txt -t cves/ -t exposures/ -t misconfiguration/
```

**nuclei — update templates:**
```bash
nuclei -update-templates
```

**nuclei — scan with rate limiting (be polite):**
```bash
nuclei -l hosts.txt -rate-limit 50 -c 25
```

**wpscan — WordPress audit:**
```bash
wpscan --url https://example.com --enumerate u,p,t
```

**trivy — container scan:**
```bash
trivy image nginx:latest
```

---

## 11. Cloud Asset Discovery

Finding S3 buckets, Azure blobs, GCS buckets, exposed cloud assets.

| Tool | Strength |
|------|----------|
| **cloud_enum** | Multi-cloud (AWS, Azure, GCP) |
| **s3scanner / s3enum** | S3-specific |
| **CloudBrute** | Permutation-based |
| **trufflehog** | Cloud creds in repos/files |

### Examples

**cloud_enum — multi-cloud:**
```bash
python3 cloud_enum.py -k example -k example-corp -k examplecorp
```

**s3scanner — bucket enumeration:**
```bash
s3scanner scan -f bucket-names.txt
```

**trufflehog — scan a GitHub org:**
```bash
trufflehog github --org=your-org-name
```

---

## 12. OSINT & Metadata

| Tool | Strength |
|------|----------|
| **theHarvester** | Emails, employees, hosts from public sources |
| **Maltego** | Link-analysis GUI |
| **SpiderFoot** | Automation framework |
| **recon-ng** | Modular framework |
| **GitDorker / trufflehog / gitleaks** | Secrets in public GitHub repos |

### Examples

**theHarvester:**
```bash
theHarvester -d example.com -b all -l 500
```

**gitleaks — scan a repo for secrets:**
```bash
gitleaks detect --source . -v
```

**SpiderFoot — automated multi-source recon:**
```bash
sf.py -s example.com -t DOMAIN_NAME -o json > spiderfoot.json
```

---

## 13. Email & People Enumeration

| Tool | Strength |
|------|----------|
| **theHarvester** | Broad |
| **Hunter.io** | Email pattern discovery (paid) |
| **emailfinder / h8mail** | Credential/breach checks |
| **holehe** | Check if email is registered on various sites |

### Examples

**holehe — registered accounts for an email:**
```bash
holehe target@example.com
```

**h8mail — breach lookup:**
```bash
h8mail -t target@example.com
```

---

## 14. WHOIS, ASN, IP Intelligence

| Tool | Strength |
|------|----------|
| **whois** | The basic command |
| **amass intel** | Discovers domains owned by an org via ASN/whois pivots |
| **asnmap** | ASN → IP range lookups (ProjectDiscovery) |
| **bgp.he.net / bgp.tools** | Web UIs for ASN exploration |
| **mapcidr** | CIDR math utility |

### Examples

**amass intel — find domains for an organization:**
```bash
amass intel -org "Example Inc"
amass intel -asn 12345
```

**asnmap — ASN to CIDR:**
```bash
asnmap -a AS12345 -silent
asnmap -d example.com -silent
```

**mapcidr — CIDR manipulation:**
```bash
echo "192.0.2.0/24" | mapcidr -silent
mapcidr -cl cidrs.txt -count
```

---

## 15. All-in-One Frameworks

Wire multiple tools together, run on a schedule, store results.

| Framework | Notes |
|-----------|-------|
| **reconftw** | Opinionated bash megapipeline, very thorough |
| **reNgine** | Web UI, scan management |
| **Osmedeus** | Workflow-based |
| **Axiom** | Distributed scanning across cloud VMs |
| **bbot** | Python-based, modular, growing fast |

### Examples

**reconftw — full recon:**
```bash
./reconftw.sh -d example.com -r
```

**bbot — modular scan:**
```bash
bbot -t example.com -f subdomain-enum,cloud-enum,email-enum
```

> **Trade-off:** Frameworks are convenient but opaque. For compliance/audit work, running individual tools is often easier to document and justify.

---

## 16. Continuous Monitoring

| Approach | Notes |
|----------|-------|
| **Chaos** | Diff their dataset over time |
| **subfinder + cron + diff** | Homegrown but simple |
| **reconftw scheduled** | Built-in scheduling |
| **Detectify / Intruder.io / Assetnote** | Commercial ASM |
| **Pipeline → SIEM** | Pipe results into Splunk/ELK/Loki, alert on changes |

### Example: simple monitoring script

```bash
#!/bin/bash
# /usr/local/bin/recon-monitor.sh
DOMAIN="example.com"
DATE=$(date +%Y%m%d)
DIR="/var/recon/$DOMAIN"
mkdir -p "$DIR"

subfinder -d "$DOMAIN" -all -silent | sort -u > "$DIR/subs-$DATE.txt"

if [ -f "$DIR/subs-latest.txt" ]; then
  diff "$DIR/subs-latest.txt" "$DIR/subs-$DATE.txt" > "$DIR/diff-$DATE.txt"
  if [ -s "$DIR/diff-$DATE.txt" ]; then
    mail -s "Recon diff for $DOMAIN" you@example.com < "$DIR/diff-$DATE.txt"
  fi
fi

cp "$DIR/subs-$DATE.txt" "$DIR/subs-latest.txt"
```

Cron entry:
```
0 3 * * * /usr/local/bin/recon-monitor.sh
```

---

## 17. Recommended Starter Pipelines

### Minimal pipeline (six tools, all free)

```bash
subfinder -d example.com -all -silent \
  | dnsx -silent \
  | naabu -silent -top-ports 1000 \
  | httpx -silent -title -tech-detect \
  | nuclei -severity medium,high,critical
```

### Comprehensive pipeline

```bash
# 1. Discovery — combine multiple passive sources
{
  subfinder -d example.com -all -silent
  curl -s "https://crt.sh/?q=%25.example.com&output=json" | jq -r '.[].name_value'
  chaos -d example.com -silent
  assetfinder --subs-only example.com
} | sort -u > subs-raw.txt

# 2. Active brute-force (optional, more aggressive)
puredns bruteforce all.txt example.com -r resolvers.txt >> subs-raw.txt
sort -u subs-raw.txt > subs-all.txt

# 3. Resolve and dedupe
dnsx -l subs-all.txt -silent -a -resp > resolved.txt
awk '{print $1}' resolved.txt | sort -u > live-subs.txt

# 4. Port scan
naabu -l live-subs.txt -silent -top-ports 1000 -o ports.txt

# 5. HTTP probe with enrichment
httpx -l live-subs.txt \
  -silent \
  -title -status-code -tech-detect -tls-grab \
  -screenshot -srd screenshots/ \
  -o http-results.txt

# 6. Crawl + historical URLs
katana -list live-subs.txt -d 2 -jc -o crawled.txt
cat live-subs.txt | gau >> crawled.txt
sort -u crawled.txt > all-urls.txt

# 7. Vulnerability scan
nuclei -l live-subs.txt \
  -severity medium,high,critical \
  -rate-limit 50 \
  -o nuclei-findings.txt
```

### Org-level discovery pipeline

```bash
# Find domains owned by the org
amass intel -org "Example Inc" > org-domains.txt

# Find IP ranges via ASN
asnmap -d example.com -silent > org-cidrs.txt

# Run subdomain discovery on every domain
while read domain; do
  subfinder -d "$domain" -all -silent
done < org-domains.txt | sort -u > all-org-subs.txt
```

---

## 18. ISO 27001 / Compliance Context

Categories most relevant to ISMS work:

| Category | Relevant Controls | Why It Matters |
|----------|-------------------|----------------|
| Passive subdomain discovery | A.5.9 (asset inventory), A.8.8 (vuln management) | Establishes externally visible attack surface |
| Cloud asset discovery | A.5.9, A.8.1 | Catches shadow IT and forgotten cloud assets |
| Vulnerability scanning (nuclei) | A.8.8, A.8.29 (security testing) | Auditor-friendly evidence |
| Continuous monitoring | A.5.7 (threat intel), A.8.16 (monitoring) | Demonstrates "regular review" |
| OSINT / GitHub secret scanning | A.5.10 (acceptable use), A.8.3 | Cheap, high-value control |

### Audit-friendly practices

- **Run individual tools, not frameworks** — easier to point at a specific tool's output and say "this is the evidence."
- **Version-pin tools** in your runbook so results are reproducible across audits.
- **Retain raw output + diffs** — auditors care about *trends*, not just current state.
- **Document scope** — what domains, what frequency, what severity threshold triggers a ticket.
- **Cross-reference recon output against authoritative DNS** (Route 53, Cloudflare zone exports). Anything externally discovered that *isn't* in your zone is a finding (stale records, takeover candidates, or third-party use of your domain).

---

## 19. Installation Cheatsheet

### ProjectDiscovery tools (Go)

```bash
# Requires Go 1.21+
go install -v github.com/projectdiscovery/subfinder/v2/cmd/subfinder@latest
go install -v github.com/projectdiscovery/dnsx/cmd/dnsx@latest
go install -v github.com/projectdiscovery/httpx/cmd/httpx@latest
go install -v github.com/projectdiscovery/naabu/v2/cmd/naabu@latest
go install -v github.com/projectdiscovery/nuclei/v3/cmd/nuclei@latest
go install -v github.com/projectdiscovery/katana/cmd/katana@latest
go install -v github.com/projectdiscovery/chaos-client/cmd/chaos@latest
go install -v github.com/projectdiscovery/asnmap/cmd/asnmap@latest
go install -v github.com/projectdiscovery/mapcidr/cmd/mapcidr@latest
go install -v github.com/projectdiscovery/shuffledns/cmd/shuffledns@latest
```

### tomnomnom tools

```bash
go install github.com/tomnomnom/assetfinder@latest
go install github.com/tomnomnom/waybackurls@latest
go install github.com/tomnomnom/httprobe@latest
go install github.com/lc/gau/v2/cmd/gau@latest
go install github.com/lc/subjs@latest
```

### Other essentials

```bash
# Amass
go install -v github.com/owasp-amass/amass/v4/...@master

# puredns + massdns
go install github.com/d3mondev/puredns/v2@latest
git clone https://github.com/blechschmidt/massdns.git && cd massdns && make

# ffuf, feroxbuster
go install github.com/ffuf/ffuf/v2@latest
cargo install feroxbuster

# Vuln/secret scanners
go install github.com/trufflesecurity/trufflehog/v3@latest
go install github.com/zricethezav/gitleaks/v8@latest

# nmap, masscan (system packages)
sudo apt install nmap masscan      # Debian/Ubuntu
brew install nmap masscan          # macOS

# rustscan
cargo install rustscan
```

### Configuration files

Most ProjectDiscovery tools read config from `~/.config/<tool>/`:

```bash
# subfinder providers
~/.config/subfinder/provider-config.yaml

# nuclei templates
~/.config/nuclei/.templates-config.yaml

# chaos
~/.config/chaos/config.yaml
```

### One-liner installer for the starter kit

```bash
# Assumes Go is installed
for tool in \
  github.com/projectdiscovery/subfinder/v2/cmd/subfinder \
  github.com/projectdiscovery/dnsx/cmd/dnsx \
  github.com/projectdiscovery/httpx/cmd/httpx \
  github.com/projectdiscovery/naabu/v2/cmd/naabu \
  github.com/projectdiscovery/nuclei/v3/cmd/nuclei \
  github.com/zricethezav/gitleaks/v8; do
  go install -v "$tool@latest"
done
```

---

## License & Attribution

This reference compiles publicly available information about open-source tools. Each tool listed has its own license and authors — credit goes to the respective maintainers (notably ProjectDiscovery, OWASP Amass team, tomnomnom, and the broader infosec community).

Use these tools only against assets you own or are explicitly authorized to test. Unauthorized scanning may violate computer-misuse laws in your jurisdiction.
