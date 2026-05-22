# ProjectDiscovery Toolchain Skill

## Purpose

Use this skill when the user needs authorized asset discovery, subdomain enumeration, DNS resolution, port scanning, HTTP service probing, web crawling, vulnerability template scanning, external attack surface review, or workflow orchestration with ProjectDiscovery tools.

This skill covers the following commonly used ProjectDiscovery tools:

- `pdtm`: ProjectDiscovery Tool Manager
- `subfinder`: Subdomain discovery
- `dnsx`: DNS resolution and DNS record querying
- `naabu`: Port scanning
- `httpx`: HTTP/HTTPS probing and fingerprinting
- `katana`: Web crawling and URL discovery
- `nuclei`: Template-based vulnerability and misconfiguration scanning
- `uncover`: Asset discovery through internet intelligence/search platforms
- `asnmap`: ASN and network asset mapping
- `tlsx`: TLS/SSL information collection
- `notify`: Result notification
- `interactsh-client`: Out-of-band interaction testing support
- `mapcidr`: CIDR manipulation and expansion
- `alterx`: Subdomain permutation generation
- `cdncheck`: CDN/WAF/cloud provider detection

According to the official ProjectDiscovery documentation, `pdtm` is the ProjectDiscovery Tool Manager. It can install all ProjectDiscovery tools with `-ia` / `-install-all`. The default binary path is `$HOME/.pdtm/go/bin`, and a custom binary path can be specified with `-bp`.

---

## Safety and Authorization Boundaries

Before running any scan, probe, enumeration, crawl, or vulnerability detection, confirm that the target is within the user's explicitly authorized scope.

Allowed use cases:

- Scanning assets owned by the user
- Internal security inspection
- Authorized penetration testing
- Authorized public attack surface assessment
- Low-risk reconnaissance before vulnerability verification
- Passive discovery based on public asset intelligence
- Security testing of user-provided domains, IPs, CIDRs, or URLs

Disallowed use cases:

- Scanning targets without authorization
- Authentication bypass, credential brute forcing, or credential stuffing
- Exploiting vulnerabilities to gain access
- Running destructive payloads
- Large-scale indiscriminate internet scanning
- Offensive exploitation against third-party systems
- Providing step-by-step exploitation instructions that enable intrusion

Use low-risk options by default. Avoid high concurrency, aggressive probing, and destructive templates unless the user clearly confirms authorization and asks for higher scan speed.

---

## Working Directory Convention

Create a dedicated working directory under `/data` for each task:
check disk space, when < 1GB, give tips for user.

```bash
mkdir -p /data/pd-work
cd /data/pd-work
```

Recommended directory structure:

```text
/data/pd-work/
├── input/
│   ├── domains.txt
│   ├── urls.txt
│   ├── ips.txt
│   └── cidrs.txt
├── output/
│   ├── subdomains.txt
│   ├── resolved.txt
│   ├── ports.txt
│   ├── httpx.jsonl
│   ├── urls.txt
│   ├── nuclei.jsonl
│   └── report.md
├── logs/
└── templates/
```

---

## Tool Availability Check

Before starting a task, check whether the required tools are available:

```bash
command -v pdtm
command -v subfinder
command -v dnsx
command -v naabu
command -v httpx
command -v katana
command -v nuclei
```

Check versions:

```bash
pdtm -version || true
subfinder -version || true
dnsx -version || true
naabu -version || true
httpx -version || true
katana -version || true
nuclei -version || true
```

If a tool is unavailable, first check whether `/opt/projectdiscovery/bin` is in `PATH`:

```bash
echo "$PATH"
ls -lah /opt/projectdiscovery/bin
```

---

## PDTM Tool Management

List installed tools:

```bash
pdtm -l
```

Install all ProjectDiscovery tools:

```bash
pdtm -ia
```

Install tools to a custom directory:

```bash
pdtm -bp /opt/projectdiscovery/bin -ia -duc -nc
```

Update all tools:

```bash
pdtm -ua
```

Update a specific tool:

```bash
pdtm -u nuclei
pdtm -u httpx
pdtm -u subfinder
```

Option notes:

- `-ia`: install all tools
- `-ua`: update all tools
- `-bp`: specify binary installation path
- `-duc`: disable update check
- `-nc`: disable colored output

---

## Standard Asset Discovery Workflow

### 1. Prepare Domain Input

Write authorized domains into:

```bash
mkdir -p input output logs
cat > input/domains.txt <<'EOF2'
example.com
example.org
EOF2
```

### 2. Subdomain Enumeration

```bash
subfinder -dL input/domains.txt -all -silent -o output/subdomains.txt
```

Deduplicate:

```bash
sort -u output/subdomains.txt -o output/subdomains.txt
```

### 3. DNS Resolution

```bash
dnsx -l output/subdomains.txt -silent -a -resp -o output/resolved.txt
```

Keep only resolvable hostnames:

```bash
dnsx -l output/subdomains.txt -silent -o output/resolved_hosts.txt
```

### 4. Port Scanning

Common ports:

```bash
naabu -list output/resolved_hosts.txt -top-ports 1000 -rate 1000 -silent -o output/ports.txt
```

Full port scan, use with caution:

```bash
naabu -list output/resolved_hosts.txt -p - -rate 1000 -silent -o output/ports_full.txt
```

### 5. HTTP Service Probing

```bash
httpx -l output/ports.txt \
  -silent \
  -json \
  -title \
  -tech-detect \
  -status-code \
  -content-length \
  -follow-redirects \
  -o output/httpx.jsonl
```

Export live URLs:

```bash
httpx -l output/ports.txt -silent -o output/live_urls.txt
```

### 6. Web Crawling

```bash
katana -list output/live_urls.txt \
  -silent \
  -d 2 \
  -jc \
  -kf all \
  -o output/katana_urls.txt
```

Deduplicate:

```bash
sort -u output/katana_urls.txt -o output/katana_urls.txt
```

### 7. Vulnerability and Misconfiguration Scanning

Low-risk scan:

```bash
nuclei -l output/live_urls.txt \
  -severity low,medium,high,critical \
  -jsonl \
  -rate-limit 20 \
  -bulk-size 10 \
  -retries 1 \
  -timeout 8 \
  -o output/nuclei.jsonl
```

High and critical findings only:

```bash
nuclei -l output/live_urls.txt \
  -severity high,critical \
  -jsonl \
  -rate-limit 10 \
  -bulk-size 5 \
  -retries 1 \
  -timeout 8 \
  -o output/nuclei_high_critical.jsonl
```

---

## One-Pipeline Example

Use this for a normal authorized domain-based attack surface assessment:

```bash
mkdir -p input output logs

subfinder -dL input/domains.txt -all -silent \
  | sort -u \
  | tee output/subdomains.txt \
  | dnsx -silent \
  | tee output/resolved_hosts.txt \
  | naabu -top-ports 1000 -rate 1000 -silent \
  | tee output/ports.txt \
  | httpx -silent -json -title -tech-detect -status-code -content-length -follow-redirects \
  | tee output/httpx.jsonl
```

Extract URLs from HTTP results and scan them:

```bash
jq -r '.url' output/httpx.jsonl | sort -u > output/live_urls.txt

nuclei -l output/live_urls.txt \
  -severity medium,high,critical \
  -jsonl \
  -rate-limit 20 \
  -bulk-size 10 \
  -o output/nuclei.jsonl
```

---

## Tool Reference

### subfinder

Purpose: subdomain discovery.

Single domain:

```bash
subfinder -d example.com -all -silent -o output/subdomains.txt
```

Batch mode:

```bash
subfinder -dL input/domains.txt -all -silent -o output/subdomains.txt
```

Recommendations:

- Use `-silent` for pipeline-friendly output
- Use `-dL` for batch scans
- Configure API keys where possible to improve coverage

---

### dnsx

Purpose: DNS resolution and DNS record querying.

Resolve hostnames:

```bash
dnsx -l output/subdomains.txt -silent -o output/resolved_hosts.txt
```

A records with responses:

```bash
dnsx -l output/subdomains.txt -a -resp -silent -o output/dns_a_records.txt
```

CNAME records:

```bash
dnsx -l output/subdomains.txt -cname -resp -silent -o output/cname.txt
```

---

### naabu

Purpose: fast port scanning.

Common ports on a single host:

```bash
naabu -host example.com -top-ports 1000 -silent
```

Batch scan:

```bash
naabu -list output/resolved_hosts.txt -top-ports 1000 -rate 1000 -silent -o output/ports.txt
```

Specific ports:

```bash
naabu -list output/resolved_hosts.txt -p 80,443,8080,8443,8000,9000 -silent -o output/ports_custom.txt
```

Safety recommendations:

- Control speed with `-rate`
- Do not run full-port scans against unauthorized public ranges
- Increase rate only in internal or explicitly authorized environments

---

### httpx

Purpose: HTTP/HTTPS liveness probing, status code collection, title extraction, technology detection, and fingerprinting.

Basic probe:

```bash
httpx -l output/ports.txt -silent -o output/live_urls.txt
```

JSON output:

```bash
httpx -l output/ports.txt \
  -json \
  -title \
  -tech-detect \
  -status-code \
  -content-length \
  -follow-redirects \
  -silent \
  -o output/httpx.jsonl
```

Show title and status code:

```bash
httpx -l output/live_urls.txt -title -status-code -silent
```

---

### katana

Purpose: crawl web pages, APIs, JavaScript files, and paths.

Basic crawl:

```bash
katana -u https://example.com -silent -o output/katana_urls.txt
```

Batch crawl:

```bash
katana -list output/live_urls.txt -d 2 -silent -o output/katana_urls.txt
```

Enable JavaScript parsing:

```bash
katana -list output/live_urls.txt -d 2 -jc -kf all -silent -o output/katana_urls.txt
```

Recommendations:

- Use `-d 2` by default
- Avoid unnecessarily deep crawling
- Rate-limit carefully on production systems

---

### nuclei

Purpose: template-based vulnerability, exposure, and misconfiguration scanning.

Update templates:

```bash
nuclei -update-templates
```

Basic scan:

```bash
nuclei -l output/live_urls.txt -severity medium,high,critical -o output/nuclei.txt
```

JSONL output:

```bash
nuclei -l output/live_urls.txt \
  -severity medium,high,critical \
  -jsonl \
  -rate-limit 20 \
  -bulk-size 10 \
  -o output/nuclei.jsonl
```

Specify template directory:

```bash
nuclei -l output/live_urls.txt -t templates/ -jsonl -o output/nuclei_custom.jsonl
```

Run only selected tags:

```bash
nuclei -l output/live_urls.txt -tags exposure,misconfig,cve -jsonl -o output/nuclei_tagged.jsonl
```

Exclude high-risk templates:

```bash
nuclei -l output/live_urls.txt \
  -exclude-tags intrusive,dos,fuzz \
  -severity low,medium,high,critical \
  -jsonl \
  -o output/nuclei_safe.jsonl
```

Safety recommendations:

- Exclude `intrusive`, `dos`, and `fuzz` templates by default
- Use conservative scan rates for production systems
- Never run vulnerability scans against unauthorized targets
- Report evidence, impact, and remediation guidance; do not expand into exploitation chains

---

### uncover

Purpose: discover exposed assets through internet intelligence/search platforms.

Example:

```bash
uncover -q 'ssl:"example.com"' -silent -o output/uncover.txt
```

Notes:

- Usually requires API keys for supported platforms
- Best used for passive asset discovery
- Validate results with `httpx` and `nuclei`

---

### tlsx

Purpose: TLS/SSL certificate and configuration collection.

```bash
tlsx -l output/live_urls.txt -json -silent -o output/tlsx.jsonl
```

Common uses:

- Certificate-based domain discovery
- Certificate expiration checks
- TLS configuration review
- Identifying assets that reuse certificates

---

### asnmap

Purpose: map ASN information to CIDR ranges or assets.

```bash
asnmap -a AS13335 -silent -o output/asn_cidrs.txt
```

---

### mapcidr

Purpose: manipulate, expand, and split CIDR ranges.

Expand a CIDR:

```bash
mapcidr -cidr 192.168.1.0/24 -silent -o output/ips.txt
```

Split a large range:

```bash
mapcidr -cidr 10.0.0.0/8 -sbc 24 -silent -o output/subnets_24.txt
```

---

### alterx

Purpose: generate subdomain permutations based on known subdomains.

```bash
alterx -l output/subdomains.txt -silent -o output/alterx_candidates.txt
```

Validate generated candidates with `dnsx`:

```bash
dnsx -l output/alterx_candidates.txt -silent -o output/alterx_resolved.txt
```

---

### cdncheck

Purpose: identify whether an IP belongs to a CDN, WAF, or cloud provider.

Single IP:

```bash
cdncheck -i 1.1.1.1
```

Batch mode:

```bash
cat output/ips.txt | cdncheck -silent > output/cdncheck.txt
```

---

## Common Task Templates

### Task: Discover the Public Attack Surface of a Domain

```bash
mkdir -p input output logs

echo "example.com" > input/domains.txt

subfinder -dL input/domains.txt -all -silent -o output/subdomains.txt

dnsx -l output/subdomains.txt -silent -o output/resolved_hosts.txt

naabu -list output/resolved_hosts.txt \
  -top-ports 1000 \
  -rate 1000 \
  -silent \
  -o output/ports.txt

httpx -l output/ports.txt \
  -json \
  -title \
  -tech-detect \
  -status-code \
  -content-length \
  -follow-redirects \
  -silent \
  -o output/httpx.jsonl

jq -r '.url' output/httpx.jsonl | sort -u > output/live_urls.txt
```

---

### Task: Run a Low-Risk Vulnerability Scan Against Live Web Assets

```bash
nuclei -l output/live_urls.txt \
  -severity low,medium,high,critical \
  -exclude-tags intrusive,dos,fuzz \
  -jsonl \
  -rate-limit 20 \
  -bulk-size 10 \
  -retries 1 \
  -timeout 8 \
  -o output/nuclei_safe.jsonl
```

---

### Task: Check Only High and Critical Issues

```bash
nuclei -l output/live_urls.txt \
  -severity high,critical \
  -exclude-tags intrusive,dos,fuzz \
  -jsonl \
  -rate-limit 10 \
  -bulk-size 5 \
  -retries 1 \
  -timeout 8 \
  -o output/nuclei_high_critical.jsonl
```

---

### Task: Crawl URLs First, Then Scan

```bash
katana -list output/live_urls.txt \
  -d 2 \
  -jc \
  -kf all \
  -silent \
  -o output/katana_urls.txt

sort -u output/katana_urls.txt -o output/katana_urls.txt

nuclei -l output/katana_urls.txt \
  -severity medium,high,critical \
  -exclude-tags intrusive,dos,fuzz \
  -jsonl \
  -rate-limit 20 \
  -o output/nuclei_from_katana.jsonl
```

---

### Task: Start from IPs or CIDR Ranges

```bash
mkdir -p input output logs

cat > input/cidrs.txt <<'EOF2'
192.168.1.0/24
EOF2

mapcidr -l input/cidrs.txt -silent -o output/ips.txt

naabu -list output/ips.txt \
  -p 80,443,8080,8443,8000,9000,9443 \
  -rate 1000 \
  -silent \
  -o output/ip_ports.txt

httpx -l output/ip_ports.txt \
  -json \
  -title \
  -tech-detect \
  -status-code \
  -silent \
  -o output/ip_httpx.jsonl
```

---

## JSONL Result Processing

Extract URLs from `httpx`:

```bash
jq -r '.url' output/httpx.jsonl | sort -u > output/live_urls.txt
```

Extract titles:

```bash
jq -r '[.url, .status_code, .title] | @tsv' output/httpx.jsonl > output/httpx_titles.tsv
```

Extract a `nuclei` finding summary:

```bash
jq -r '[.host, .info.severity, .["template-id"], .info.name] | @tsv' output/nuclei.jsonl \
  > output/nuclei_summary.tsv
```

Count findings by severity:

```bash
jq -r '.info.severity' output/nuclei.jsonl | sort | uniq -c | sort -nr
```

---

## Report Output Standard

After the scan is complete, generate `output/report.md` using the following structure:

```md
# ProjectDiscovery Security Assessment Report

## 1. Task Information

- Task name:
- Authorized scope:
- Scan time:
- Operator:
- Toolchain:
- Output directory:

## 2. Asset Discovery Results

| Type | Count |
|---|---:|
| Input domains |  |
| Subdomains |  |
| Resolvable hosts |  |
| Open ports |  |
| Live web services |  |
| Crawled URLs |  |

## 3. Web Service Overview

| URL | Status Code | Title | Technologies |
|---|---:|---|---|

## 4. Vulnerability and Risk Statistics

| Severity | Count |
|---|---:|
| critical |  |
| high |  |
| medium |  |
| low |  |
| info |  |

## 5. Key Findings

### 1. Finding Name

- Target:
- Severity:
- Template ID:
- Discovery time:
- Evidence:
- Impact:
- Remediation:

## 6. Remediation Recommendations

1. Prioritize critical and high-risk findings.
2. Restrict access to exposed admin panels, test systems, and debug interfaces.
3. Close unnecessary internet-facing ports.
4. Fix expired certificates, weak TLS configuration, and insecure CORS settings.
5. Add discovered assets to continuous monitoring.

## 7. Attachment Files

- `output/subdomains.txt`
- `output/resolved_hosts.txt`
- `output/ports.txt`
- `output/httpx.jsonl`
- `output/katana_urls.txt`
- `output/nuclei.jsonl`
```

---

## Automatic Report Generation Script

```bash
cat > output/report.md <<'EOF_REPORT'
# ProjectDiscovery Security Assessment Report

## 1. Task Information

- Scan time: $(date '+%Y-%m-%d %H:%M:%S')
- Working directory: $(pwd)
- Toolchain: subfinder / dnsx / naabu / httpx / katana / nuclei

## 2. Asset Statistics

| Type | Count |
|---|---:|
| Subdomains | $(test -f output/subdomains.txt && wc -l < output/subdomains.txt || echo 0) |
| Resolvable hosts | $(test -f output/resolved_hosts.txt && wc -l < output/resolved_hosts.txt || echo 0) |
| Open ports | $(test -f output/ports.txt && wc -l < output/ports.txt || echo 0) |
| Live web services | $(test -f output/live_urls.txt && wc -l < output/live_urls.txt || echo 0) |
| Crawled URLs | $(test -f output/katana_urls.txt && wc -l < output/katana_urls.txt || echo 0) |
| Nuclei findings | $(test -f output/nuclei.jsonl && wc -l < output/nuclei.jsonl || echo 0) |

## 3. Finding Severity Statistics

```text
$(test -f output/nuclei.jsonl && jq -r '.info.severity' output/nuclei.jsonl | sort | uniq -c | sort -nr || echo "No nuclei results")
```

## 4. Key Findings

```text
$(test -f output/nuclei.jsonl && jq -r '[.host, .info.severity, .["template-id"], .info.name] | @tsv' output/nuclei.jsonl | head -50 || echo "None")
```

## 5. Output Files

- output/subdomains.txt
- output/resolved_hosts.txt
- output/ports.txt
- output/httpx.jsonl
- output/live_urls.txt
- output/katana_urls.txt
- output/nuclei.jsonl
EOF_REPORT
```

---

## Recommended Default Parameters

### Low-Risk Scan Defaults

```bash
-rate-limit 20
-bulk-size 10
-retries 1
-timeout 8
-exclude-tags intrusive,dos,fuzz
```

### Large-Scale Asset Discovery Defaults

```bash
naabu -rate 1000
katana -d 2
httpx -retries 1 -timeout 8
```

### Conservative Production Defaults

```bash
naabu -rate 200
nuclei -rate-limit 5 -bulk-size 3
katana -d 1
```

---

## Troubleshooting

### 1. Tools Installed by PDTM Cannot Be Found

Check the path:

```bash
ls -lah /opt/projectdiscovery/bin
echo "$PATH"
```

Temporary fix:

```bash
export PATH=/opt/projectdiscovery/bin:$PATH
```

### 2. Nuclei Templates Are Missing

Update templates:

```bash
nuclei -update-templates
```

List templates:

```bash
nuclei -tl
```

### 3. httpx Produces No Results

Check input format:

```bash
head output/ports.txt
```

If the input is in `host:port` format, pass it directly to `httpx`.
If the input is a bare domain or hostname, `httpx` will automatically try HTTP and HTTPS.

### 4. naabu Is Too Fast

Lower the scan rate:

```bash
naabu -list output/resolved_hosts.txt -top-ports 1000 -rate 200 -silent
```

### 5. nuclei Has Too Many False Positives

Use filters:

```bash
nuclei -l output/live_urls.txt \
  -severity medium,high,critical \
  -exclude-tags intrusive,dos,fuzz \
  -jsonl \
  -o output/nuclei_filtered.jsonl
```

Manually review findings:

```bash
jq -r '[.host, .info.severity, .["template-id"], .info.name, .matched-at] | @tsv' output/nuclei_filtered.jsonl
```

---

## Response Standard When Reporting Back to the User

When presenting results to the user, prioritize:

1. Actions completed
2. Input scope
3. Number of discovered assets
4. Number of high/critical findings
5. Key evidence file paths
6. Remediation recommendations
7. Follow-up commands

Example:

```text
Completed a ProjectDiscovery toolchain scan for the authorized domain example.com.

Asset discovery results:
- Subdomains: 128
- Resolvable hosts: 96
- Open ports: 34
- Live web services: 21
- Crawled URLs: 642

Risk results:
- critical: 0
- high: 2
- medium: 5
- low: 11

Main output files:
- output/httpx.jsonl
- output/nuclei.jsonl
- output/report.md

Recommended next step: manually verify high-risk findings first, then restrict exposed admin panels, test environments, and debug interfaces.
```

---

## Notes

- All scans must stay within the authorized scope.
- Do not use destructive, DoS, or fuzzing templates by default.
- Use conservative rates for production systems.
- `nuclei` findings must be manually verified and should not be treated as confirmed vulnerabilities without review.
- If sensitive information, secrets, or credentials are discovered, report only their existence and location. Do not spread or expose the sensitive values.
- For high-risk vulnerabilities, provide remediation guidance rather than extended exploitation steps.
- Before large-scale scanning, confirm network scope, maintenance window, rate limits, and authorization boundaries.

---

## Quick Command Index

```bash
# Subdomain discovery
subfinder -dL input/domains.txt -all -silent -o output/subdomains.txt

# DNS resolution
dnsx -l output/subdomains.txt -silent -o output/resolved_hosts.txt

# Port scanning
naabu -list output/resolved_hosts.txt -top-ports 1000 -rate 1000 -silent -o output/ports.txt

# Web probing
httpx -l output/ports.txt -json -title -tech-detect -status-code -silent -o output/httpx.jsonl

# Extract URLs
jq -r '.url' output/httpx.jsonl | sort -u > output/live_urls.txt

# Crawling
katana -list output/live_urls.txt -d 2 -jc -kf all -silent -o output/katana_urls.txt

# Vulnerability scanning
nuclei -l output/live_urls.txt -severity medium,high,critical -exclude-tags intrusive,dos,fuzz -jsonl -o output/nuclei.jsonl

# Count findings by severity
jq -r '.info.severity' output/nuclei.jsonl | sort | uniq -c | sort -nr

# Generate finding summary
jq -r '[.host, .info.severity, .["template-id"], .info.name] | @tsv' output/nuclei.jsonl
```
