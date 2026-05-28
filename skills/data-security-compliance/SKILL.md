---
name: data-security-compliance
description: Enforce PCI DSS v4.0.1, GDPR (EU), and PDP (Indonesia UU No. 27/2022) compliance — scope, technical controls, data subject rights, breach notification, overlapping requirements, and converged audit strategy.
version: 1.0.0
author: User
---

# Data Security Compliance (PCI DSS + GDPR + PDP Indonesia)

## Purpose

Enforce data security compliance across three regulatory frameworks: PCI DSS v4.0.1 (payment card data), GDPR (EU personal data), and PDP Indonesia UU No. 27 Tahun 2022 (Indonesian personal data). Every piece of data handling must satisfy the strictest applicable control. The goal: one converged compliance program, not three separate audits.

---

# Core Principle

> Protect all personal and cardholder data with the same control baseline. PCI DSS provides the technical floor. GDPR and PDP layer privacy governance on top. If a control exists in any framework, apply it everywhere — don't wait for the auditor to ask.

Every system touching personal data must be demonstrably compliant. Every engineer must know which rules apply to the data they handle. If you can't prove compliance from logs, policies, and configurations, you're not compliant.

---

# Framework Overview

```
┌────────────────────────────────────────────────────┐
│  DATA SECURITY COMPLIANCE LANDSCAPE                 │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │  PCI DSS      │  │  GDPR (EU)   │  │  PDP (ID) │ │
│  │  v4.0.1       │  │              │  │  UU 27/22 │ │
│  ├──────────────┤  ├──────────────┤  ├───────────┤ │
│  │ Scope: CHD    │  │ Scope: EU    │  │ Scope: ID │ │
│  │ + SAD         │  │ personal     │  │ personal  │ │
│  │               │  │ data         │  │ data       │ │
│  ├──────────────┤  ├──────────────┤  ├───────────┤ │
│  │ Fine: card    │  │ Fine: €20M   │  │ Fine: 2%  │ │
│  │ brand         │  │ / 4% revenue │  │ revenue +  │ │
│  │ penalties     │  │              │  │ criminal   │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
│         │                 │                 │        │
│         └─────────┬───────┴─────────────────┘        │
│                   ▼                                   │
│  ┌──────────────────────────────────────┐            │
│  │  CONVERGED CONTROL BASELINE           │            │
│  │  • Encryption (AES-256, TLS 1.2+)    │            │
│  │  • Access control (RBAC, MFA, PoLP)  │            │
│  │  • Logging & monitoring (SIEM)       │            │
│  │  • Incident response (72h SLA)       │            │
│  │  • Vendor management (DPA + AoC)     │            │
│  │  • Data retention & disposal         │            │
│  │  • Security testing (pentest, ASV)   │            │
│  │  • Privacy governance (DSR, DPIA)    │            │
│  └──────────────────────────────────────┘            │
└────────────────────────────────────────────────────┘
```

---

# 1. PCI DSS v4.0.1

## Scope of Application

Applies to any entity that **stores, processes, or transmits** cardholder data (CHD) or sensitive authentication data (SAD). Scope includes people, processes, and technology that touch CHD or could affect its security.

**Cardholder Data (CHD)**: Primary Account Number (PAN), cardholder name, expiration date, service code.
**Sensitive Authentication Data (SAD)**: Full track data, CAV2/CVC2/CVV2/CID, PIN, PIN block.

Annual scoping required: identify all CHD flows, map systems, determine in-scope vs. out-of-scope components. Network segmentation is the primary tool for reducing scope.

## 12 Core Requirements (6 Groups)

### Group 1 — Build and Maintain Secure Network

**Req 1**: Install and maintain network security controls (firewalls, router configs, network diagrams, configuration standards, review firewall rules every 6 months).

**Req 2**: Apply secure configurations to all system components (hardening, no vendor defaults, change default passwords, enable only necessary services/ports/protocols, document security configurations).

### Group 2 — Protect Account Data

**Req 3**: Protect stored account data.
- No full PAN storage post-authorization unless justified
- Render PAN unreadable: tokenization, truncation (first 6 / last 4), one-way hash (salted), or strong cryptography
- No SAD storage post-authorization (even if encrypted)
- Key management lifecycle per NIST SP 800-57: generation, distribution, storage, rotation, retirement
- Key custodian separation of duties

**Req 4**: Protect CHD with strong cryptography during transmission.
- TLS 1.2+ mandatory. No SSL or early TLS. No TLS 1.0/1.1.
- Use trusted keys/certificates. Validate certificates properly.
- Encrypt CHD over open/public networks only. (Internal trusted networks: risk-based.)
- Never send unprotected PAN via end-user messaging (email, SMS, chat).

### Group 3 — Maintain Vulnerability Management

**Req 5**: Protect systems from malicious software.
- Anti-malware on all commonly-affected systems
- Keep signatures current, periodic scans, logs retained
- Phishing protection for personnel

**Req 6**: Develop and maintain secure systems and software.
- Patch all system components promptly (critical patches within 30 days)
- Secure SDLC: threat modeling, code review, security testing pre-production
- Web application firewall (WAF) or code review for all internet-facing web apps
- Change control process for all production changes
- Bespoke/custom software: security assessments required (v4.0 addition)

### Group 4 — Implement Strong Access Control

**Req 7**: Restrict access by business need-to-know.
- Least privilege at application and database layers
- Default deny-all, then grant per role
- Document and review access rights every 6 months

**Req 8**: Identify users and authenticate access.
- Unique user IDs for all accounts
- MFA for all non-console CDE access (v4.0 expanded from remote-only to all non-console)
- Password minimum 12 characters (or 8 with complexity in v4.0, transitioning to 12 by v4.0.1)
- Password change every 90 days; session idle timeout 15 minutes
- No shared accounts; no generic/group accounts

**Req 9**: Restrict physical access.
- Badge/access control systems, visitor logs, CCTV
- Secure media disposal (shred, degauss, secure wipe)

### Group 5 — Regularly Monitor and Test

**Req 10**: Log and monitor all access.
- Centralized logging: user, event type, date/time, success/failure, origin, affected data/component
- Time synchronization (NTP) across all systems
- Retain logs 12 months minimum, with 3 months immediately available for analysis
- Daily log review for security events and anomalies
- File integrity monitoring (FIM) on critical system files
- Audit trails must be tamper-proof, write once

**Req 11**: Test security regularly.
- Quarterly ASV (Approved Scanning Vendor) external vulnerability scans
- Quarterly internal vulnerability scans (authenticated, all in-scope IPs)
- Annual external and internal penetration tests (NIST SP 800-115 methodology)
- Segmentation testing every 6 months (penetration test of segmentation controls)
- Web application vulnerability assessments (WAF or SAST/DAST)
- Intrusion detection/prevention systems (IDS/IPS) at perimeter and critical points

### Group 6 — Maintain Security Policy

**Req 12**: Organizational policies and programs.
- Information security policy published and communicated
- Annual risk assessment (people, processes, technology)
- Acceptable use policy for all personnel
- Incident response plan: test annually, update after incidents
- Security awareness training at hire and annually
- Third-party service provider management: inventory, written agreements with security responsibilities, annual review
- Annual policy review and update

## SAQ Types

| SAQ | Who | Questions |
|-----|-----|-----------|
| SAQ A | Card-not-present, fully outsourced e-commerce | ~25 |
| SAQ A-EP | E-commerce with partial outsourcing (iframe/redirect, website impact risk) | ~190 |
| SAQ B | Imprint-only terminals, no electronic CHD | ~40 |
| SAQ B-IP | Standalone PTS-approved IP terminals | ~80 |
| SAQ C | Payment app connected to internet | ~160 |
| SAQ C-VT | Virtual terminal via browser, one page at a time | ~80 |
| SAQ P2PE | Validated P2PE-listed terminals only | ~35 |
| SAQ D (Merchant) | All others + SAQ for Level 1-3 requiring full RoC | ~330 |
| SAQ D (Service Provider) | Service providers assessed by SAQ | ~330 |

## Merchant Compliance Levels

| Level | Transaction Volume | Validation |
|-------|-------------------|------------|
| Level 1 | >6M Visa/MC transactions/year | Annual on-site QSA assessment (RoC) + quarterly ASV scans |
| Level 2 | 1M–6M transactions/year | Annual SAQ + quarterly ASV scans |
| Level 3 | 20K–1M e-commerce transactions/year | Annual SAQ + quarterly ASV scans |
| Level 4 | <20K e-commerce or <1M non-e-commerce | Annual SAQ (recommended) + quarterly ASV scans |

Card brands (Visa, Mastercard, etc.) set final validation requirements. Levels above are typical.

## Key Dates

- v4.0.1 effective: June 2024
- v3.2.1 retired: March 31, 2024
- All v4.0 future-dated requirements fully effective: March 31, 2025 (now in effect)

---

# 2. GDPR (EU General Data Protection Regulation)

## Scope of Application

Applies to **any controller or processor** in the EU/EEA (regardless of where processing occurs), OR outside the EU/EEA if **offering goods/services to EU data subjects** or **monitoring their behavior** within the EU.

**Personal data**: Any information relating to an identified or identifiable natural person. Broad — includes name, email, IP address, cookie identifiers, location data, device IDs.

**Special categories** (Art. 9, processing prohibited unless exception applies): racial/ethnic origin, political opinions, religious beliefs, trade union membership, genetic data, biometric data (for identification), health data, sex life/orientation.

**Criminal conviction data** (Art. 10): processing only under official authority control or EU/member state law authorization.

Exemptions: purely personal/household activities, law enforcement (covered by LED), national security.

## Key Principles (Art. 5)

1. **Lawfulness, fairness, transparency** — data subject must know what happens to their data
2. **Purpose limitation** — collected for specified, explicit, legitimate purposes; no further incompatible processing
3. **Data minimisation** — adequate, relevant, limited to what is necessary
4. **Accuracy** — kept accurate, up to date; inaccurate data erased or rectified without delay
5. **Storage limitation** — kept no longer than necessary; anonymize or delete after
6. **Integrity and confidentiality** — appropriate security against unauthorized/unlawful processing, accidental loss, destruction, damage
7. **Accountability** — controller is responsible for and must demonstrate compliance

## Lawful Bases (Art. 6)

Processing requires at least one lawful basis:

| Basis | Notes |
|-------|-------|
| **Consent** | Freely given, specific, informed, unambiguous; clear affirmative action; easily withdrawable; no power imbalance; children under 16 need parental consent (MS can lower to 13) |
| **Contract** | Necessary for performance of contract or pre-contractual steps at data subject's request |
| **Legal obligation** | Controller subject to a legal requirement |
| **Vital interests** | Protect someone's life |
| **Public interest** | Task in public interest or exercise of official authority |
| **Legitimate interests** | Controller or third party, unless overridden by data subject's rights; requires balancing test (used for fraud prevention, direct marketing, intra-group admin) |

## Data Subject Rights

| Right | Art. | Timeline | Notes |
|-------|------|----------|-------|
| Right to be informed | 12–14 | At collection | Transparent info in concise, clear language; privacy notice |
| Right of access | 15 | 1 month (+2 months for complex) | Obtain copy of personal data, confirm processing details |
| Right to rectification | 16 | 1 month | Correct inaccurate data, complete incomplete data |
| Right to erasure ("right to be forgotten") | 17 | 1 month | Delete when no longer necessary, consent withdrawn, objection upheld, unlawful processing. Exceptions: public interest, legal claims, free expression |
| Right to restriction | 18 | 1 month | Limit processing while accuracy contested or objection assessed |
| Right to data portability | 20 | 1 month | Receive data in structured, machine-readable format; transmit to another controller; only for consent/contract + automated means |
| Right to object | 21 | At request | Object to legitimate interests/public task processing; **absolute right** to object to direct marketing |
| Rights re: automated decision-making | 22 | At request | Not subject to solely automated decisions producing legal/significant effects. Exceptions: contract necessity, law, explicit consent — but safeguards required |

## DPO Requirement (Art. 37–39)

Mandatory appointment when: (1) processing by public authority (except courts); (2) core activities consist of **regular and systematic large-scale monitoring**; (3) core activities involve **large-scale processing of special categories** or criminal conviction data.

DPO may be employee or external contractor. Must have expert knowledge of data protection law. Cannot receive instructions on DPO tasks. Cannot be dismissed/penalized for performing DPO role. Reports to highest management. Contact details must be published and communicated to supervisory authority.

Non-EU controllers/processors must also appoint an **EU representative** (Art. 27) — distinct from DPO.

## Breach Notification (Art. 33–34)

**To supervisory authority**: within **72 hours** of becoming aware of breach. Must describe: nature of breach, categories/approximate number of data subjects and records affected, likely consequences, measures taken/proposed. If delayed beyond 72 hours, provide reasons.

**To data subjects**: without undue delay if breach likely to result in **high risk** to rights/freedoms. Exceptions: data encrypted/unintelligible, subsequent measures eliminate high risk, individual notification disproportionate effort (then public communication instead).

**Processor → Controller**: without undue delay after becoming aware.

## DPIA (Art. 35–36)

Mandatory for processing likely to result in **high risk** to rights/freedoms, particularly:
- Systematic and extensive automated profiling with legal/significant effects
- Large-scale processing of special categories or criminal data
- Systematic large-scale monitoring of publicly accessible areas

Must include: systematic description of processing, necessity/proportionality assessment, risk assessment, measures to address risks. Must **consult supervisory authority** before processing if DPIA indicates high risk and insufficient mitigation.

## Cross-Border Transfers (Art. 44–49)

Three pathways:
1. **Adequacy decision**: European Commission determines country provides adequate protection (Japan, UK, South Korea, Israel, and others as of 2025)
2. **Appropriate safeguards**: Standard Contractual Clauses (SCCs), Binding Corporate Rules (BCRs), approved codes of conduct, approved certification
3. **Derogations**: explicit consent (informed of risks), contract necessity, important reasons of public interest, legal claims, vital interests

US transfers governed by **EU-US Data Privacy Framework (DPF)** as of July 2023 adequacy decision.

## Fines (Art. 83)

| Tier | Max Fine | Violations |
|------|----------|------------|
| Tier 1 | €10M **or** 2% annual worldwide turnover (whichever higher) | Controller/processor obligations, security, breach notification, DPIA, DPO, certification body obligations |
| Tier 2 | €20M **or** 4% annual worldwide turnover (whichever higher) | Basic principles (Art. 5–7), data subject rights (Art. 12–22), special category rules (Art. 9), cross-border transfer rules (Art. 44–49), non-compliance with DPA orders |

Data subjects also have right to compensation for damages (Art. 82), **including non-material damages**.

Supervisory authority corrective powers: warnings, reprimands, compliance orders, temporary/permanent processing bans, administrative fines.

---

# 3. PDP Indonesia (UU No. 27 Tahun 2022)

## Scope of Application

Enacted **October 17, 2022**. Full compliance deadline: **October 17, 2024** (2-year transition period — now passed).

Applies to: any person, public body, or international organization performing legal actions in Indonesia or having legal consequences within Indonesian jurisdiction, OR located outside Indonesia but having **legal consequences within Indonesian jurisdiction** or **concerning Indonesian data subjects**. Extraterritorial reach similar to GDPR.

**Personal data**: Any data about an identified or identifiable individual (broad definition).

**Specific personal data (sensitive)**: health data, biometric data, genetic data, criminal records, children's data, **personal financial data**, and other data per government regulation. Note: personal financial data is explicitly sensitive under PDP — GDPR does not classify financial data as a special category.

## Key Principles

Personal data must be:
1. Collected in a limited and specific, legally valid manner
2. Processed in accordance with its purpose
3. Processed guaranteeing the rights of data subjects
4. Accurate, complete, and not misleading
5. Protected from loss, misuse, unauthorized access, and unauthorized disclosure
6. Retained only as long as necessary
7. Accountable throughout its lifecycle

## Lawful Bases for Processing

| Basis | Notes |
|-------|-------|
| **Explicit valid consent** (primary) | Written or recorded, in Bahasa Indonesia, clear, informed, specific, free of duress, withdrawable |
| Contractual obligation | Necessary for contract performance |
| Legal obligation | Controller's legal requirement |
| Vital interests | Protect life/health |
| Public interest | Public service or exercise of authority per legislation |
| Legitimate interests | Balancing controller interests against data subject rights |

PDP places heavier emphasis on **consent** as the primary basis compared to GDPR's six equal bases. Written/recorded consent in Bahasa Indonesia is the default expectation.

## Roles and Obligations

**Data Controller (Pengendali Data Pribadi)**: determines purposes and means. Must:
- Register with the Data Protection Authority (DPA)
- Have lawful basis for processing
- Implement security measures
- Maintain processing records
- Carry out DPIA for high-risk processing
- Appoint DPO if: processing for public services, core activities involving large-scale/regular systematic monitoring, or large-scale specific personal data or criminal data processing
- Respond to data subject requests
- Notify breaches

**Data Processor (Prosesor Data Pribadi)**: processes on behalf of controller. Must:
- Process only per controller instructions
- Implement security measures
- Maintain processing records
- Notify controller of breaches
- Assist controller with DPIAs and data subject requests

## Data Subject Rights

- **Right to information**: clarity of identity and purpose, lawfulness of processing
- **Right to access**: obtain copy of personal data
- **Right to rectification**: correct inaccurate/incomplete data
- **Right to restriction**: temporarily stop processing
- **Right to erasure**: delete data when no longer needed, consent withdrawn, or processing unlawful
- **Right to withdraw consent**: at any time
- **Right to object**: to automated decision-making with significant consequences
- **Right to data portability**: receive in commonly used, machine-readable format
- **Right to compensation**: sue for damages from unlawful processing
- **Right to delay/limit processing**: suspend or restrict under certain conditions

## Breach Notification

| Recipient | Timeline | Requirement |
|-----------|----------|-------------|
| Data subjects | **3 × 24 hours** (72 hours) | Reasons, affected data, remedial actions |
| Data Protection Authority (DPA) | **3 × 24 hours** (72 hours) | Reasons, affected data, remedial actions |
| Public announcement (mass media) | For serious harm | Additional requirement beyond GDPR |

## Cross-Border Transfers

Permitted only to countries with **protection level equal to or higher** than PDP Law, as determined by the DPA/Minister. If receiving country lacks equivalent protection, controller must ensure adequate safeguards (contractual agreements, BCRs) and obtain **DPA approval**.

## Sanctions

### Administrative
- Written warning
- Temporary suspension of data processing
- Deletion or destruction of personal data
- Administrative fine: **maximum 2% of annual revenue** or operating income

### Criminal (individuals)
| Violation | Imprisonment | Fine |
|-----------|-------------|------|
| Unlawful obtaining/disclosure | Up to 5 years | Up to IDR 5 billion (~USD 320K) |
| Unlawful use for self-enrichment | Up to 6 years | Up to IDR 6 billion (~USD 385K) |
| Falsification of personal data | Up to 6 years | Up to IDR 6 billion (~USD 385K) |

### Criminal (corporations)
- Fines up to **10× the individual fine**
- Additional penalties: freezing of business, revocation of business license, dissolution

Note: GDPR does not include criminal sanctions — it leaves criminal enforcement to member state law. PDP's criminal liability is a significant additional risk.

---

# 4. Converged Compliance Strategy

## Controls Satisfying All Three Frameworks

The following technical and procedural controls satisfy **PCI DSS, GDPR, and PDP simultaneously** when applied to ALL personal data — not just cardholder data:

### Encryption (at rest and in transit)

**PCI DSS**: Req 3.4 (render PAN unreadable), Req 4.1 (TLS 1.2+ for transmission)
**GDPR**: Art. 32 (encryption as appropriate technical measure; also factor in breach notification waiver — encrypted data doesn't require data subject notification)
**PDP**: Art. 35–36 (security obligations including encryption)

**Control baseline**:
- AES-256 or equivalent for stored data at rest
- TLS 1.2+ for all data in transit (enforce minimum TLS version at load balancer/gateway)
- Proper key management per NIST SP 800-57 (rotation, separation of duties, HSM for production keys)
- Never use: SSL 3.0, TLS 1.0, TLS 1.1, default credentials, hardcoded keys
- Panic procedure: keys compromised → immediate rotation, breach assessment, notification

### Access Control and Identity Management

**PCI DSS**: Req 7 (need-to-know), Req 8 (MFA for CDE, unique IDs, password policy)
**GDPR**: Art. 25 (data protection by design), Art. 32 (access controls as security measure)
**PDP**: Art. 36 (obligation to protect personal data)

**Control baseline**:
- RBAC with least privilege (deny-all, grant per business justification)
- MFA for all non-console access to systems containing personal/cardholder data
- Unique user IDs — no shared/generic accounts, no shared credentials
- Password policy: minimum 12 characters, 90-day rotation, prevent last 4 reuse
- Session timeout: 15 minutes idle, 8 hours absolute maximum
- Access review every 6 months: revoke stale access, validate business need
- Break-glass emergency access with full audit trail

### Logging and Monitoring

**PCI DSS**: Req 10 (log all CHD/CDE access, retain 12 months, daily review, FIM)
**GDPR**: Art. 30 (records of processing), Art. 33 (breach detection)
**PDP**: Art. 50–52 (processing records, breach detection)

**Control baseline**:
- Centralized SIEM with structured log format: timestamp (NTP-synced), user ID, event type, success/failure, source IP, target resource, data elements accessed
- Tamper-proof logs: write-once, append-only, cryptographically signed
- Retention: 12 months full, 3 months hot/queryable
- File integrity monitoring (FIM) on critical configs, binaries, and log files
- Automated alerting: anomalous access patterns, privilege escalation, mass data export, failed auth spikes
- Daily security review of alerts (automated where possible; human review for critical/high)

### Security Testing

**PCI DSS**: Req 11 (quarterly ASV, quarterly internal, annual pentest, segmentation test 6mo)
**GDPR**: Art. 32 (regularly test and evaluate security measures)
**PDP**: Art. 36 (security measures including regular testing)

**Control baseline**:
- Quarterly external vulnerability scans (ASV for PCI scope; equivalent for non-PCI scope)
- Quarterly authenticated internal vulnerability scans (all internal IPs)
- Annual external + internal penetration tests (NIST SP 800-115 methodology)
- Segmentation testing every 6 months (penetration test of network segmentation controls)
- Web application assessments for all internet-facing apps (SAST in CI/CD + DAST pre-release)
- WAF or equivalent for all public-facing web applications

### Incident Response and Breach Notification

**PCI DSS**: Req 12.10 (IR plan, test annually, update after incidents)
**GDPR**: Art. 33–34 (72-hour authority notification, data subject notification for high risk)
**PDP**: Art. 46 (72-hour DPA + data subject notification, mass media for serious harm)

**Control baseline** (single IR plan covering all three):
- **72-hour notification SLA** to all authorities and affected data subjects (strictest: PDP and GDPR)
- Incident response plan tested annually (tabletop + live drill)
- Pre-approved notification templates for each regulator
- Defined escalation tiers: L1 (minor, no notification) → L2 (notifiable within 72h) → L3 (critical, immediate notification + public announcement)
- Breach register documenting all incidents, root causes, and corrective actions
- Forensics capability: preserve evidence, maintain chain of custody, engage external forensics firm for L3
- Post-incident: lessons learned, plan update, retest

### Third-Party / Vendor Management

**PCI DSS**: Req 12.8 (manage service providers, maintain inventory, written agreements with security responsibilities, annual review)
**GDPR**: Art. 28 (data processing agreements, processor obligations, sub-processor chain)
**PDP**: Art. 50–51 (processor agreements, processor obligations)

**Control baseline**:
- Maintain inventory of all third parties handling personal or cardholder data
- Data Processing Agreement (DPA) with every processor: includes security obligations, breach notification SLAs, audit rights, sub-processor approval, data return/deletion at termination
- Annual vendor security assessment: questionnaire + evidence review + risk rating
- Right to audit (or third-party audit report e.g., SOC 2 Type II, ISO 27001)
- PCI-specific: written acknowledgment of security responsibilities + Attestation of Compliance (AoC) for PCI service providers
- Vendor offboarding checklist: revoke access, return data, confirm deletion, document

### Data Retention and Disposal

**PCI DSS**: Req 3.1 (no SAD post-auth), Req 3.2 (limit CHD storage to business/legal need), Req 9.8 (secure media disposal)
**GDPR**: Art. 5(1)(c)–(e) (data minimisation and storage limitation)
**PDP**: Art. 30–33 (retention limited to purpose)

**Control baseline**:
- Documented data retention schedule per data category
- Automated purging: records deleted when retention period expires
- No SAD storage post-authorization (PCI — non-negotiable)
- Secure disposal: NIST SP 800-88 media sanitization (crypto-erase, physical destruction for HDD/SSD)
- Data classification labels: public → internal → confidential → restricted (cardholder/sensitive personal)
- Quarterly review: identify data past retention, trigger disposal

### Privacy Governance Layer

The following are **privacy-specific** controls required by GDPR and PDP but not PCI DSS:

**Lawful basis documentation**: For every processing activity, document the lawful basis, purpose, data categories, and retention period. Maintain an Article 30 (GDPR) / equivalent PDP processing register.

**Privacy notices**: Publish clear, layered privacy notices at collection points. Include: controller identity/DPL contact, purposes, lawful basis, recipients, retention period, data subject rights, right to complain to DPA. For PDP: notice must be in **Bahasa Indonesia**.

**Data Subject Request (DSR) handling**: Implement DSR portal/ticketing system for access, rectification, erasure, portability, objection, and restriction requests. SLA: respond within **1 month** (extendable 2 months for complex requests). Verify identity before fulfilling requests. Track all requests and outcomes.

**Data Protection Impact Assessment (DPIA)**: Conduct DPIA before launching new processing likely to present high risk. Trigger conditions: automated profiling, special category data, large-scale monitoring, new technology, processing of children's data or financial data. Publish summary of DPIA (confidential details may be redacted).

**DPO appointment**: Required under both GDPR and PDP for similar thresholds. Single DPO can cover both if qualified in both regulatory frameworks. DPO must be independent, resourced, and report to highest management.

**Cross-border transfer assessment**: Maintain Transfer Impact Assessment (TIA) for each third country. Document transfer mechanism (adequacy decision, SCCs, BCRs, or derogation). For Indonesia: verify recipient country has equal/higher protection per DPA determination.

---

# 5. Key Differences Between Frameworks

| Dimension | PCI DSS | GDPR | PDP Indonesia |
|-----------|---------|------|---------------|
| **Scope** | Cardholder data only | All personal data + special categories | All personal data + specific data (incl. financial) |
| **Focus** | Technical security controls | Privacy rights + security | Privacy rights + security + criminal enforcement |
| **Consent** | Not a primary concept | One of six equal lawful bases | **Primary** basis, written/recorded in Bahasa Indonesia |
| **Criminal liability** | No (card brand penalties) | No (left to member state law) | **Yes** — imprisonment up to 6 years, fines up to 10× for corporations |
| **Max fine** | Card brand discretion (fines + loss of processing capability) | €20M or 4% global turnover | 2% annual revenue (admin) + criminal fines + corporate sanctions |
| **Breach notification** | To acquirer/payment brands per their rules | 72h to DPA; data subjects if high risk | 72h to DPA + data subjects; mass media for serious harm |
| **Technology mandate** | Prescriptive (specific controls, ASV, WAF, IDS/IPS) | Technology-neutral ("appropriate measures") | Technology-neutral |
| **DPO** | Not required (Req 12.1: assign security responsibilities) | Required for public authorities, large-scale monitoring/sensitive data | Required for public services, large-scale/regular monitoring, specific data |
| **Cross-border data** | No restriction (but CHD must be protected regardless of location) | Adequacy decision, SCCs, BCRs, or derogation | Only to countries with equal/higher protection + DPA approval |
| **Data subject rights** | None | 8 rights (access, erasure, portability, etc.) | 10 rights (similar to GDPR + right to compensation, delay/limit) |
| **DPIA** | Not required (risk assessment per Req 12.2) | Art. 35 — high-risk processing | Art. 41 — similar triggers + specific data on large scale |
| **Financial data** | Cardholder data = PAN + name + expiry + service code | Not a special category (but still personal data) | **Explicitly sensitive** |

---

# 6. Implementation Patterns

## Data Classification

```
┌──────────────────────────────────────────────────┐
│  DATA CLASSIFICATION HIERARCHY                    │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │  RESTRICTED                                  │ │
│  │  • Cardholder data (PAN, expiry, CVV)        │ │
│  │  • Sensitive auth data (PIN, track data)     │ │
│  │  • Special/specific personal data            │ │
│  │    - Health, biometric, genetic              │ │
│  │    - Criminal records, children's data       │ │
│  │    - Financial data (PDP Indonesia)          │ │
│  │  Controls: encrypt, MFA, audit all access,   │ │
│  │  auto-purge, tokenize where possible         │ │
│  ├─────────────────────────────────────────────┤ │
│  │  CONFIDENTIAL                                 │ │
│  │  • Personal data (name, email, phone, IP)    │ │
│  │  • Internal business data                    │ │
│  │  Controls: RBAC, encrypt at rest, log access, │ │
│  │  retention schedule, DSR-addressable         │ │
│  ├─────────────────────────────────────────────┤ │
│  │  INTERNAL                                     │ │
│  │  • Non-sensitive operational data            │ │
│  │  • Anonymized/aggregated analytics           │ │
│  │  Controls: access control, no public exposure│ │
│  ├─────────────────────────────────────────────┤ │
│  │  PUBLIC                                       │ │
│  │  • Marketing materials, public docs           │ │
│  │  Controls: no restrictions (review before     │ │
│  │  publishing)                                  │ │
│  └─────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

## Handling PAN and Sensitive Data

```
CLIENT SIDE                      SERVER SIDE
┌──────────┐                    ┌──────────────────────┐
│  PAN     │                    │                      │
│  4111... │──── TLS 1.2+ ─────▶│  Tokenize immediately │
│          │                    │  │                    │
│          │                    │  ▼                    │
│          │                    │  Token: tok_ABC123    │
│          │     NEVER:         │  PAN:  [encrypted]   │
│          │  • Log PAN         │                      │
│          │  • Store PAN in    │  Store token, not PAN │
│          │    localStorage    │  Encrypt PAN at rest  │
│          │  • Send PAN via    │  Mask in logs:        │
│          │    email/SMS       │  411111******1111     │
│          │  • Cache PAN in    │                      │
│          │    browser memory  │  No SAD stored POST-  │
│          │                    │  AUTH, period.        │
└──────────┘                    └──────────────────────┘
```

## DSR Workflow (Data Subject Request)

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Data Subject │     │  DSR Portal   │     │  Internal    │
│  submits      │────▶│  (ticketing)  │────▶│  Team reviews│
│  request      │     │               │     │  & fulfills  │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                 │
                    ┌──────────────────────────────┘
                    ▼
  ┌─────────────────────────────────────────────────┐
  │  VERIFICATION CHECKLIST                          │
  │  □ Identity verified (government ID or 2FA)      │
  │  □ Request type classified (access/erase/...)    │
  │  □ Data sources identified (DB, logs, backups)   │
  │  □ Exemptions checked (legal hold, public        │
  │    interest, legal claims)                        │
  │  □ Response prepared within 1-month SLA          │
  │  □ Action logged for audit trail                 │
  └─────────────────────────────────────────────────┘
```

## Breach Response Timeline

```
T+0h ─── Breach discovered
 │
 ├─ T+0-1h: Containment (isolate affected systems, revoke compromised keys/creds)
 ├─ T+1-2h: Forensics begin (preserve logs, disk images, memory dumps)
 ├─ T+2-24h: Impact assessment (what data, how many subjects, root cause)
 ├─ T+24-48h: Draft notifications (DPA, data subjects, acquirer if PCI)
 ├─ T+48-72h: Notify DPA + data subjects (meet 72h SLA)
 │
 └─ T+72h+: Remediation (fix root cause, rotate keys, update IR plan, post-mortem)

For serious harm (PDP): also publish announcement in mass media.
For encrypted data (GDPR): may waive data subject notification.
For PCI: notify acquirer/payment brands per their defined timelines.
```

---

# 7. Enforcement Rules

## When Handling Any Personal or Cardholder Data

1. **Classify data at creation time.** Every data element must have a classification label (restricted, confidential, internal, public). No unclassified data.

2. **Encrypt everywhere.** TLS 1.2+ in transit. AES-256 at rest. Never store raw values that can be tokenized. Never log sensitive data.

3. **Access = business justification.** No access without a documented business need. Review every 6 months. MFA mandatory for all non-console access to restricted/confidential data.

4. **Log everything, review daily.** Every access, every change, every export. Logs are tamper-proof and retained 12 months. Automated alerting for anomalies.

5. **72-hour breach notification window.** From the moment of awareness, the clock is ticking. Pre-approved templates. Pre-defined escalation paths. Test the plan annually.

6. **DPA for every processor.** No personal data leaves your control to a third party without a signed, security-reviewed DPA in place.

7. **DPIA before launch.** New processing likely to be high-risk → DPIA first, launch second. No exceptions.

8. **Honor DSRs within 1 month.** Access, rectification, erasure, portability, objection, restriction — every request gets a response within the SLA. Track all requests.

9. **Retention has an expiration date.** Every data category has a documented retention period. Automated purging when the clock runs out. No data lives forever.

10. **PDP consent in Bahasa Indonesia.** For Indonesian data subjects: consent must be written/recorded in Bahasa Indonesia, explicit, informed, and easily withdrawable. Not buried in a 40-page ToS.

11. **No SAD post-auth. Ever.** Sensitive authentication data (CVV, PIN, full track) must not be stored after authorization. This is a PCI DSS hard rule with zero exceptions.

12. **Test everything regularly.** Quarterly vuln scans. Annual pentests. Segmentation tests every 6 months. IR plan tested annually. If you haven't tested it, assume it's broken.

---

# 8. Quick Rules Reference

| If you... | Then you must... |
|-----------|-----------------|
| Collect PAN | Tokenize immediately; never log raw PAN; TLS 1.2+; encrypt at rest; PCI DSS scope triggered |
| Store personal data | Classify it; define retention period; document lawful basis; register in processing inventory |
| Process EU resident data | Have lawful basis (Art. 6); honor DSRs; privacy notice; DPIA if high risk; breach notify 72h |
| Process Indonesian resident data | Written consent in Bahasa Indonesia; honor DSRs; DPIA if high risk; breach notify 72h; cross-border assessment |
| Use a third-party service | Signed DPA with security obligations; vendor assessment; annual review; offboarding checklist |
| Suffer a data breach | Contain within 1h; assess impact within 24h; notify DPA + data subjects within 72h; post-mortem |
| Delete data | Log the deletion; confirm deletion from backups too; issue data subject confirmation if DSR |
| Add a new microservice | Classification review; DPIA if high risk; update processing inventory; security review; pentest if CDE |
| Transfer data cross-border | Adequacy review; SCCs if no adequacy decision; DPA notification; document TIA |
| Process children's data | Parental consent (under 16 GDPR, under specific age PDP); enhanced DPIA; stricter retention |
| Handle sensitive auth data | Never store post-auth; transmit only over TLS 1.2+; log access (not values); PCI SAQ scope |
| Face a regulatory audit | Processing records ready; DPIA register ready; breach register ready; vendor inventory ready; training records ready |

---

# 9. Anti-Patterns

| Anti-Pattern | Why It Fails |
|-------------|--------------|
| "We'll classify data later" | Unclassified data = unprotected data. Classification must happen at ingestion time. |
| "Encrypt when the auditor asks" | Encryption is a design requirement, not a checkbox. Retrofitting encryption into a live system is high-risk and expensive. |
| "Logs are just for debugging" | Logs are legal evidence. Tamper-proof, structured, retained. If you can't prove who accessed what, you weren't compliant. |
| "Consent in the privacy policy is enough" | Consent must be specific, informed, and unambiguous per action. Blanket consent in a privacy policy fails GDPR and PDP. |
| "Store all data forever — might be useful" | Storage limitation violation under all three frameworks. Every data element must have a documented deletion date. |
| "We handle PCI so we don't need GDPR/PDP" | PCI covers cardholder data. GDPR/PDP cover ALL personal data including employees, prospects, and non-payment customers. Different scope. |
| "Our cloud provider handles compliance" | Shared responsibility model. Provider secures infrastructure; you secure your application, configurations, access controls, and data handling. |
| "The DPO handles all privacy stuff" | Compliance is shared responsibility. Engineers must know what rules apply to the data they handle. DPO advises, engineers implement. |
| "One generic privacy notice for everything" | Different processing purposes need different notices. Different jurisdictions have different requirements (Bahasa Indonesia for PDP). |
| "Self-attest for everything" | PCI Level 1 needs QSA on-site assessment. Certain GDPR processing needs DPA consultation. PDP needs DPA registration. Know when self-attestation is insufficient. |
| "Security through obscurity" | Hiding data instead of protecting it. If it's personal or cardholder data, it needs encryption, access control, and logging — regardless of where it sits. |
