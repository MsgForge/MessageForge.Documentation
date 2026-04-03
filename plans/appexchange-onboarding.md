# AppExchange Onboarding Plan

## Classification

Our app is a **Composite App** — it uses both on-platform code (Apex, LWC) and off-platform infrastructure (Go middleware, PostgreSQL). This classification triggers additional security review requirements.

> **Note (ADR-20/21):** External infrastructure is limited to Go middleware, PostgreSQL, and residential proxies. Media storage uses Salesforce ContentVersion (not external). Real-time updates use Platform Events (not external WebSocket).

## 5-Step Onboarding Pipeline

### Step 1: Partner Community Enrollment

Register in the Salesforce Partner Community — centralized hub for licensing, support, and business operations.

### Step 2: Architecture Review & Build

- Declare as "Composite App" (on-platform: Apex + LWC; off-platform: Go middleware)
- Document all data flows and boundary protections
- Prepare external components for penetration testing
- All off-platform services must be ready for rigorous pen testing

### Step 3: Listing Creation & Commercial Alignment

- Populate metadata in Listing Builder
- Negotiate **Partner Application Distribution Agreement (PADA)**
- Commit to commercial model (revenue share or annual fee based on projected volume)

### Step 4: Security Review

The most rigorous hurdle. Submit:
- Complete managed package
- Test environments (scratch org or sandbox with sample data)
- Scan reports for ALL external components

### Step 5: Publication

Upon passing security review, approved for public distribution and installation across customer orgs.

## Security Review Requirements for Go Server

### DAST Scanning

- Perform Dynamic Application Security Testing on all external web services
- Authorized scanners: **OWASP ZAP**, **Burp Suite**, or **Chimera**
- Must demonstrate **zero high-severity vulnerabilities**:
  - Cross-Site Scripting (XSS)
  - SQL injection
  - OS command injection
  - Other OWASP Top 10
- False positives documented in supplementary report

### TLS Enforcement

- All endpoints must enforce **TLS 1.2+**
- Provide SSL Labs PDF/screenshot as proof
- No outdated encryption standards

### Credential Security

- **No CRM credentials or passwords** in external middleware code
- All secrets via Protected Custom Metadata Types or environment variables

### Transient Data Handling

- Prove secure handling without unauthorized logging
- Document what data passes through Go middleware and retention policies

## Status Checklist

- [ ] Partner Community account created
- [ ] Architecture documented as Composite App
- [ ] DAST scan completed (zero high-severity findings)
- [ ] TLS 1.2+ verified (SSL Labs report)
- [ ] No hardcoded credentials in codebase
- [ ] PADA negotiated and signed
- [ ] Listing metadata populated
- [ ] Test environment prepared for reviewers
- [ ] Security review submitted
- [ ] Publication approved
