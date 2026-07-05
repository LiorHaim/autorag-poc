# IT Security Policy

## Password Requirements

All NovaTech systems require passwords that meet the following criteria:
- Minimum 14 characters
- Must include uppercase, lowercase, numbers, and special characters
- Cannot reuse the last 12 passwords
- Must be changed every 90 days
- Multi-factor authentication (MFA) is mandatory for all accounts

Password managers are strongly recommended. NovaTech provides enterprise licenses for 1Password to all employees. Passwords must never be stored in plain text, shared via email or Slack, or written down in accessible locations.

## Device Management

### Company Devices
- All company laptops are managed through Microsoft Intune
- Automatic security patches are applied weekly
- Full-disk encryption (BitLocker/FileVault) is required
- USB storage devices are disabled by default
- Only IT-approved software may be installed
- Devices inactive for more than 15 minutes must auto-lock
- Lost or stolen devices must be reported to IT within 1 hour — remote wipe will be initiated immediately

### Personal Devices (BYOD)
- Personal devices may access company email and calendar only
- Must have a screen lock with 6+ digit PIN or biometric
- Company reserves the right to remote-wipe company data
- Personal devices must run supported OS versions (within 2 major versions)
- Jailbroken or rooted devices are prohibited from accessing company systems
- BYOD enrollment requires installing the Intune Company Portal app

## Network Security

### Corporate Network
- All office networks use WPA3-Enterprise with certificate-based authentication
- Network is segmented into zones: Corporate, Guest, IoT, and Production
- Inter-zone traffic passes through next-gen firewalls with deep packet inspection
- Network access control (NAC) enforces device compliance before granting access

### VPN
- All remote access to internal systems must go through the corporate VPN (GlobalProtect)
- Split-tunneling is disabled — all traffic routes through the VPN when connected
- VPN sessions timeout after 12 hours and require re-authentication
- VPN access is revoked automatically when an employee's account is disabled

### Cloud Security
- All cloud resources must be provisioned through Infrastructure as Code (Terraform)
- Direct console changes to production environments are prohibited
- Cloud security posture is monitored by Wiz with daily scans
- All S3 buckets and equivalent storage must have public access blocked by default
- Encryption at rest is required for all data stores (AES-256 minimum)
- Encryption in transit is required for all communications (TLS 1.2 minimum, TLS 1.3 preferred)

## Data Classification

- **Public**: Marketing materials, published blog posts
- **Internal**: Company announcements, meeting notes, project documents
- **Confidential**: Customer data, financial reports, employee records, source code
- **Restricted**: Encryption keys, authentication secrets, board materials

### Data Handling Requirements by Classification

| Classification | Storage | Transmission | Access Control | Disposal |
|---------------|---------|-------------|----------------|----------|
| Public | Any approved system | No restrictions | No restrictions | Standard deletion |
| Internal | Corporate systems only | Internal channels | Team-based access | Standard deletion |
| Confidential | Encrypted storage only | Encrypted channels only | Need-to-know basis | Secure deletion (overwrite) |
| Restricted | Hardware security modules or dedicated vaults | End-to-end encrypted only | Individual approval required | Cryptographic erasure |

## Vendor Security Assessments

All third-party vendors that will access NovaTech data must complete a security assessment before onboarding:

- **Tier 1 vendors** (access to Restricted or Confidential data): Full security audit including SOC 2 Type II report, penetration test results, and on-site assessment. Annual reassessment required.
- **Tier 2 vendors** (access to Internal data): Security questionnaire and SOC 2 Type II report. Reassessed every 2 years.
- **Tier 3 vendors** (no data access): Self-attestation questionnaire. Reassessed every 3 years.

Vendors must sign a Data Processing Agreement (DPA) before any data sharing begins. The Security team maintains the approved vendor list at security.novatech.internal/vendors.

## Security Training

All employees must complete the following security training:

- **New Hire**: Security awareness fundamentals (within first week, 2 hours)
- **Annual Refresher**: Updated threats and policy changes (1 hour, due by March 31)
- **Phishing Simulations**: Monthly simulated phishing emails. Employees who fail 2 consecutive simulations are required to complete additional 30-minute training.
- **Role-Specific**: Engineers complete secure coding training annually (4 hours). Managers complete data handling training annually (2 hours).

Training completion is tracked in Workday. Non-completion results in system access restrictions after the due date.

## Incident Reporting

All security incidents must be reported immediately to:
- Email: security@novatech.internal
- Slack: #security-incidents
- Phone: +1-512-555-0911 (24/7 hotline)

Response SLAs:
- Critical incidents: Response within 15 minutes
- High severity: Response within 1 hour
- Medium severity: Response within 4 hours
- Low severity: Response within 24 hours

### Incident Response Process

1. **Detection**: Incident identified via monitoring, employee report, or third-party notification
2. **Triage**: Security team classifies severity and assigns an incident commander within the response SLA
3. **Containment**: Immediate steps to limit damage (isolate systems, revoke credentials, block IPs)
4. **Investigation**: Root cause analysis, scope assessment, evidence preservation
5. **Remediation**: Fix the vulnerability, restore services, verify remediation effectiveness
6. **Notification**: Affected parties notified per legal requirements (within 72 hours for GDPR, varies by jurisdiction)
7. **Post-Incident Review**: Blameless review within 5 business days, lessons learned documented and shared

## Acceptable Use

Employees must not:
- Share credentials with others (including IT staff)
- Access systems they are not authorized to use
- Install unauthorized software or browser extensions
- Store company data on personal cloud storage (Google Drive, Dropbox, etc.)
- Send confidential data via unencrypted email
- Connect unauthorized devices to the corporate network
- Attempt to bypass security controls, even for testing purposes without written Security team approval
- Use company systems for cryptocurrency mining, torrenting, or other resource-intensive personal activities

## Compliance

NovaTech maintains compliance with the following standards and regulations:
- SOC 2 Type II (audited annually by Deloitte)
- ISO 27001 (certified, recertified every 3 years)
- GDPR (EU operations)
- CCPA (California customer data)
- HIPAA (healthcare customer segment — BAAs available)

Compliance documentation is available at security.novatech.internal/compliance. Audit reports can be shared with customers under NDA upon request.
