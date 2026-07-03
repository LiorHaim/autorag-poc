# Engineering Standards & Practices

## Code Review Process

All code changes must be reviewed before merging:
- Minimum 2 approvals required for production code
- At least 1 reviewer must be a senior engineer or above
- Reviews must be completed within 24 hours of submission
- Authors must respond to review comments within 12 hours
- No self-merging allowed

## Branching Strategy

NovaTech uses trunk-based development:
- `main` branch is always deployable
- Feature branches are short-lived (maximum 3 days)
- All merges go through pull requests
- Branch names follow: `type/JIRA-ID-brief-description` (e.g., `feat/ENG-1234-add-auth`)
- Squash merge is the default merge strategy

## Testing Requirements

- **Unit tests**: Minimum 80% code coverage for new code
- **Integration tests**: Required for all API endpoints and database interactions
- **End-to-end tests**: Required for critical user flows
- **Performance tests**: Required for changes that affect latency-sensitive paths
- All tests must pass in CI before merge is allowed

## Deployment Process

- **Staging**: Auto-deployed on merge to main
- **Production**: Deployed Tuesday and Thursday at 10 AM CT
- **Hotfixes**: Can be deployed anytime with on-call engineer + manager approval
- Feature flags are used for all new user-facing features
- Rollback plan must be documented before production deploy

## Incident Response (Engineering)

Severity levels:
- **SEV1**: Complete service outage affecting all customers. Target resolution: 1 hour.
- **SEV2**: Partial outage or degraded performance affecting >10% of customers. Target: 4 hours.
- **SEV3**: Minor issue affecting small subset of customers. Target: 24 hours.
- **SEV4**: Cosmetic or low-impact issues. Target: next sprint.

On-call rotation: 1-week shifts, rotating among senior engineers. Compensation: $500/week on-call + $200 per incident engaged.

## Architecture Decision Records (ADRs)

All significant technical decisions must be documented as ADRs:
- Stored in the `docs/adr/` directory of the relevant repository
- Must include: context, decision, consequences, and alternatives considered
- Reviewed and approved by the architecture team
- Numbered sequentially (ADR-001, ADR-002, etc.)
