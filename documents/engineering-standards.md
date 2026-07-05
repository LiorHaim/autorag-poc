# Engineering Standards & Practices

## Code Review Process

All code changes must be reviewed before merging:
- Minimum 2 approvals required for production code
- At least 1 reviewer must be a senior engineer or above
- Reviews must be completed within 24 hours of submission
- Authors must respond to review comments within 12 hours
- No self-merging allowed
- Security-sensitive changes (authentication, authorization, cryptography, PII handling) require an additional review from the Security Engineering team
- Database migration PRs require approval from the Database SRE team

### Review Quality Standards

Reviewers are expected to evaluate:
- Correctness and logic errors
- Test coverage for new and changed code paths
- Performance implications (N+1 queries, unnecessary allocations, missing indexes)
- API contract compatibility (breaking vs non-breaking changes)
- Adherence to the NovaTech style guide (enforced automatically by linters, but reviewers should catch semantic issues)
- Documentation updates if public-facing behavior changes

Reviewers should use the "Request Changes" option sparingly — only for issues that would cause bugs, security vulnerabilities, or data loss. Style preferences and minor suggestions should use the "Comment" option.

## Branching Strategy

NovaTech uses trunk-based development:
- `main` branch is always deployable
- Feature branches are short-lived (maximum 3 days)
- All merges go through pull requests
- Branch names follow: `type/JIRA-ID-brief-description` (e.g., `feat/ENG-1234-add-auth`)
- Squash merge is the default merge strategy

Long-running branches (e.g., major refactors spanning more than 3 days) require Tech Lead approval and must be synced with main at least once per day via rebase.

## Testing Requirements

- **Unit tests**: Minimum 80% code coverage for new code
- **Integration tests**: Required for all API endpoints and database interactions
- **End-to-end tests**: Required for critical user flows
- **Performance tests**: Required for changes that affect latency-sensitive paths
- **Contract tests**: Required for all service-to-service API boundaries
- All tests must pass in CI before merge is allowed

### Test Environment Standards

NovaTech maintains the following test environments:

- **Local**: Docker Compose-based environment that mirrors production topology. Engineers must be able to run the full service locally.
- **CI**: Ephemeral environments spun up per pull request. Each PR gets its own isolated namespace with seeded test data.
- **Staging**: Persistent environment with production-like data (anonymized). Deployed automatically on merge to main.
- **Canary**: Receives 5% of production traffic for 1 hour before full rollout. Automatic rollback if error rate exceeds baseline by 2x.

### Performance Benchmarks

All services must meet the following latency targets measured at the 99th percentile:

- API endpoints serving user-facing pages: < 200ms
- Background job processing: < 5 seconds per job
- Search queries: < 500ms
- Webhook delivery: < 10 seconds from trigger event
- Database queries: < 50ms (alert threshold), < 100ms (hard limit, must optimize)

Performance budgets are enforced in CI. Any PR that regresses p99 latency by more than 10% is automatically flagged.

## API Standards

### Versioning

All public APIs use URL-based versioning (e.g., `/api/v1/`, `/api/v2/`). Internal service-to-service APIs use header-based versioning (`X-API-Version`). A minimum of 12 months deprecation notice is required before removing an API version. Deprecated API versions must return a `Sunset` header with the planned removal date.

### Error Handling

All APIs return errors in the following standard format:
- HTTP status codes must follow RFC 7231 semantics
- Error response body must include: `error_code` (machine-readable), `message` (human-readable), `request_id` (for tracing), and optionally `details` (array of specific field errors)
- Rate limiting returns 429 with `Retry-After` header
- Internal server errors (5xx) must never expose stack traces or internal details to clients

### Authentication

All API endpoints require authentication via OAuth 2.0 bearer tokens unless explicitly marked as public. Service-to-service calls use mTLS with certificates rotated every 90 days. API keys are supported for legacy integrations but are being phased out by Q4 2025.

## Deployment Process

- **Staging**: Auto-deployed on merge to main
- **Production**: Deployed Tuesday and Thursday at 10 AM CT
- **Hotfixes**: Can be deployed anytime with on-call engineer + manager approval
- Feature flags are used for all new user-facing features
- Rollback plan must be documented before production deploy

### Deployment Freeze Windows

No production deployments are allowed during:
- Company-wide all-hands meetings
- 48 hours before and after major product launches
- December 20 through January 3 (holiday freeze)
- During active SEV1 incidents (unless the deployment is the fix)

### Feature Flag Lifecycle

Feature flags must follow this lifecycle:
1. **Created**: Flag defined in LaunchDarkly with owner and expiry date
2. **Development**: Flag off by default, enabled per-engineer in dev/staging
3. **Rollout**: Gradual percentage rollout (10% → 25% → 50% → 100%) with monitoring at each stage
4. **Stable**: Flag fully rolled out and stable for at least 2 sprints
5. **Cleanup**: Flag removed from code, configuration archived. Maximum flag lifetime: 90 days.

Flags older than 90 days generate automated Jira tickets for cleanup.

## Incident Response (Engineering)

Severity levels:
- **SEV1**: Complete service outage affecting all customers. Target resolution: 1 hour.
- **SEV2**: Partial outage or degraded performance affecting >10% of customers. Target: 4 hours.
- **SEV3**: Minor issue affecting small subset of customers. Target: 24 hours.
- **SEV4**: Cosmetic or low-impact issues. Target: next sprint.

On-call rotation: 1-week shifts, rotating among senior engineers. Compensation: $500/week on-call + $200 per incident engaged.

### Post-Incident Process

All SEV1 and SEV2 incidents require a blameless post-mortem:
- Post-mortem document due within 3 business days of incident resolution
- Must include: timeline, root cause analysis, impact assessment, and action items
- Post-mortem review meeting held within 5 business days with all stakeholders
- Action items tracked in Jira with assigned owners and due dates
- Trends reviewed monthly by the engineering leadership team

## Architecture Decision Records (ADRs)

All significant technical decisions must be documented as ADRs:
- Stored in the `docs/adr/` directory of the relevant repository
- Must include: context, decision, consequences, and alternatives considered
- Reviewed and approved by the architecture team
- Numbered sequentially (ADR-001, ADR-002, etc.)

## Technical Debt Management

NovaTech allocates 20% of each sprint's capacity to technical debt reduction. Tech debt items are tracked in Jira under the "TECHDEBT" epic. Each quarter, the engineering leadership team reviews the tech debt backlog and prioritizes the top items based on risk and impact. Teams are expected to leave code better than they found it — small improvements in surrounding code are encouraged during feature work.
