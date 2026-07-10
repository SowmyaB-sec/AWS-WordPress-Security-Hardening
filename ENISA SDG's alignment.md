## 2. Alternative Framework: ENISA Security Domains

The controls above also map onto ENISA's Technical Guideline on Security
Measures (used under the NIS Directive/EECC), shown here as an alternative
framework the project's controls were cross-referenced against:

| ENISA Domain | Project Control | Where implemented |
|---|---|---|
| **D1 — Governance & Risk Management** | Threat modeling using STRIDE + OWASP Top 10 | Root README |
| **D2 — Human Resources Security** | Least-privilege IAM roles, SSH key-only auth | `infrastructure/` |
| **D3 — Security of Systems & Facilities** | Hardened Amazon Linux 2023 AMI, UFW default-deny, scoped Security Groups | `infrastructure/`, `lamp-stack/` |
| **D4 — Operations Management** | Apache/MySQL/PHP hardening, automated patching | `lamp-stack/` |
| **D5 — Incident Management** | Fail2Ban, Wordfence firewall (blocking mode), malware scanning | `word-press/` |
| **D6 — Business Continuity Management** | Encrypted daily backups to S3 | `word-press/` |
| **D7 — Monitoring, Auditing & Testing** | CloudWatch + Datadog; Nmap/Nikto/Nessus/WPScan/SSL Labs testing | `monitoring/`, `testing/` |

**Suggested resume phrasing:**
> "Cross-referenced controls against ENISA's Technical Guideline on Security
> Measures as an alternative security framework."

---

## 3. Mapping to UN Sustainable Development Goals

| SDG | Target | Relevance to this project |
|---|---|---|
| **SDG 9 — Industry, Innovation and Infrastructure** | 9.1 / 9.c — resilient infrastructure, reliable and secure ICT access | Hardening a public-facing web application improves the resilience and trustworthiness of the digital infrastructure it depends on |
| **SDG 16 — Peace, Justice and Strong Institutions** | 16.10 — protect access to information, strengthen institutional resilience against digital threats | Reducing attack surface (XSS, SQLi, brute force, malware) protects information integrity and availability for end users |

**Suggested resume phrasing:**
> "Contributed to resilient digital infrastructure in line with UN SDG 9 and
> SDG 16."

Keep this framing modest — a single hardening project is a small, genuine
contribution to these goals, not a claim of achieving them.

---

## 4. CV Bullet Points

- Secured a WordPress application hosted on AWS using a LAMP stack, applying
  a threat model built on the OWASP Top 10 (with reference to ENISA's
  Technical Guideline on Security Measures as an alternative security
  framework), contributing to resilient digital infrastructure in line with
  UN SDG 9 and SDG 16
- Implemented security controls (WAF rules, input validation, TLS
  enforcement, access restrictions) to defend against common OWASP Top 10
  attack vectors including injection, broken access control, and security
  misconfiguration
- Configured AWS security services (Security Groups, IAM least-privilege,
  CloudWatch monitoring) to reduce the application's attack surface, aligning
  with ENISA governance, systems security, and monitoring domains
- Validated hardening measures using Nessus, Nikto, and WPScan, reducing
  identified vulnerabilities by ~60% based on before/after scans *(keep only
  if you have both scan reports to back this up — otherwise soften to
  "identifying and remediating vulnerabilities across multiple scan
  categories")*
