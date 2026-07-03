# IT Security Policy

## Password Requirements

All NovaTech systems require passwords that meet the following criteria:
- Minimum 14 characters
- Must include uppercase, lowercase, numbers, and special characters
- Cannot reuse the last 12 passwords
- Must be changed every 90 days
- Multi-factor authentication (MFA) is mandatory for all accounts

## Device Management

### Company Devices
- All company laptops are managed through Microsoft Intune
- Automatic security patches are applied weekly
- Full-disk encryption (BitLocker/FileVault) is required
- USB storage devices are disabled by default
- Only IT-approved software may be installed

### Personal Devices (BYOD)
- Personal devices may access company email and calendar only
- Must have a screen lock with 6+ digit PIN or biometric
- Company reserves the right to remote-wipe company data
- Personal devices must run supported OS versions (within 2 major versions)

## Data Classification

- **Public**: Marketing materials, published blog posts
- **Internal**: Company announcements, meeting notes, project documents
- **Confidential**: Customer data, financial reports, employee records, source code
- **Restricted**: Encryption keys, authentication secrets, board materials

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

## Acceptable Use

Employees must not:
- Share credentials with others (including IT staff)
- Access systems they are not authorized to use
- Install unauthorized software or browser extensions
- Store company data on personal cloud storage (Google Drive, Dropbox, etc.)
- Send confidential data via unencrypted email
- Connect unauthorized devices to the corporate network
