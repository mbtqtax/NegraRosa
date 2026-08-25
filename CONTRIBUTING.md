# Contributing to NegraRosa

Welcome! We're building the Inclusive Security Framework together. This guide ensures all contributions meet our security, quality, and accessibility standards.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Workflow](#development-workflow)
- [Coding Standards](#coding-standards)
- [Security Requirements](#security-requirements)
- [Testing & CI](#testing--ci)
- [Pull Request Process](#pull-request-process)
- [Naming Conventions](#naming-conventions)

---

## Code of Conduct

We are committed to providing a welcoming and inclusive environment for all contributors. Be respectful, collaborative, and focused on the security and accessibility of our work.

**Security-first mindset**: Every line of code impacts user privacy and security. Treat vulnerabilities with urgency.

---

## Getting Started

### Prerequisites

- Node.js 20+
- PostgreSQL (local or Docker)
- Git
- Familiarity with TypeScript and React

### Local Setup

```bash
# Clone the repository
git clone https://github.com/mbtqtax/NegraRosa.git
cd NegraRosa

# Install dependencies
npm install

# Set up environment
cp .env.example .env
# Edit .env with your settings (never commit real secrets)

# Verify everything works
npm run check
npm test
```

### Development Server

```bash
npm run dev
# Server at http://localhost:5000
```

---

## Development Workflow

### 1. Create a Feature Branch

Always branch from `main`:

```bash
git checkout main
git pull origin main
git checkout -b feature/your-feature-name
```

**Branch naming**: `feature/`, `fix/`, `docs/`, `chore/`, `security/` prefixes.

### 2. Make Changes

- Write code following [Coding Standards](#coding-standards)
- Add tests for new functionality
- Update documentation
- Run security checks (see [Testing & CI](#testing--ci))

### 3. Commit Regularly

Use clear, descriptive commit messages:

```bash
git commit -m "feat: add user identity verification endpoint

- Implements NFT-based verification
- Adds Zod schema validation
- Includes audit logging
- No PII persisted by default (PERSISTENCE=false)

Refs: #123"
```

**Commit message format**:
- `feat:` - New feature
- `fix:` - Bug fix
- `docs:` - Documentation
- `refactor:` - Code restructuring (no functional change)
- `test:` - Tests only
- `security:` - Security hardening or fix
- `chore:` - Build/dependency/tooling changes

### 4. Push and Open PR

```bash
git push origin feature/your-feature-name
```

Go to GitHub and open a Pull Request. Use the [PR template](./.github/PULL_REQUEST_TEMPLATE/pull_request_template.md).

### 5. Respond to Reviews

- Address feedback promptly
- Re-request review when ready
- Don't force-push after review starts (makes history hard to follow)

---

## Coding Standards

### TypeScript

- **Strict mode**: Always enable (in `tsconfig.json`)
- **No `any`**: Type everything. Use `unknown` if truly untyped, then narrow.
- **Naming**: `camelCase` for variables/functions, `PascalCase` for classes/types

```typescript
// ✅ Good
interface UserAttestationRequest {
  userNftId: string;
  verificationLevel: number;
}

function submitAttestation(req: UserAttestationRequest): Promise<AttestationResponse> {
  // ...
}

// ❌ Avoid
interface userAttestationRequest {
  user_nft_id: string;
  verification_level: any;
}

function submitAttestation(req: any): any {
  // ...
}
```

### File Organization

```
server/
├── api/v1/
│   ├── attestations.ts      # Endpoint handlers
│   ├── webhooks.ts
│   └── agents.ts
├── services/
│   ├── verification/
│   │   ├── index.ts         # Main export
│   │   ├── civic.ts         # Civic verification service
│   │   └── civic.test.ts    # Tests co-located
│   └── audit/
│       ├── index.ts
│       └── audit.test.ts
├── middleware/
│   ├── auth.ts
│   ├── redaction.ts
│   └── rate-limit.ts
└── utils/
    ├── crypto.ts
    └── validation.ts
```

### Import Organization

1. External packages
2. Internal absolute imports
3. Relative imports
4. Side effects (last)

```typescript
// ✅ Good
import express from 'express';
import { z } from 'zod';

import { AuditService } from '@/services/audit';
import { verifyToken } from '@/middleware/auth';

import { formatResponse } from './utils';

import '@/styles/globals.css';
```

### Error Handling

- Never swallow errors silently
- Always log security-relevant errors
- Return meaningful error responses (avoid exposing internals)

```typescript
// ✅ Good
try {
  const result = await verifyWithCivic(request);
} catch (error) {
  logger.error('Civic verification failed', {
    userId: req.user.id,
    error: error.message,
    timestamp: new Date().toISOString(),
  });
  return res.status(500).json({
    error: 'Verification service unavailable',
  });
}

// ❌ Avoid
try {
  const result = await verifyWithCivic(request);
} catch (error) {
  console.error(error); // Never use console in production
  return res.status(500).json({ error });
}
```

---

## Security Requirements

**All PRs must pass security checks**. These are non-negotiable.

### Before Submitting

1. **No secrets in code**:
   ```bash
   # Check for accidental secrets
   npm audit
   git log -p | grep -i "api_key\|password\|secret" || echo "✅ No secrets found"
   ```

2. **Input validation**:
   - All API endpoints use Zod schemas
   - User inputs sanitized before use
   - No SQL injection vectors

3. **PII handling**:
   - No PII logged without redaction
   - `PERSISTENCE=false` in development/testing
   - Audit logs encrypted at rest

4. **No banned imports in `/api`**:
   ```bash
   # Fail the build if DB client imported directly in API layer
   ! grep -r "import.*drizzle" ./api/ || exit 1
   ! grep -r "import.*pg\>" ./api/ || exit 1
   ```

5. **Dependency audit**:
   ```bash
   npm audit
   # High/critical CVEs must be resolved or explicitly suppressed with justification
   ```

6. **Token/credential rotation**:
   - Session tokens: 15-minute TTL (Paseto v4)
   - Webhook secrets: rotated weekly
   - Partner keys: rotated quarterly

### Security Checklist for PRs

Add this to your PR description:

```markdown
## Security Checklist
- [ ] No hardcoded secrets (API keys, passwords, tokens)
- [ ] Input validation on all endpoints (Zod schemas)
- [ ] PII not logged or persisted without encryption
- [ ] No direct DB imports in `/api/*`
- [ ] Audit logging added for sensitive operations
- [ ] Rate limiting appropriate for endpoint
- [ ] Authentication required where needed
- [ ] Error messages don't expose internals
- [ ] Dependencies have no high/critical CVEs
```

---

## Testing & CI

### Local Testing

```bash
# Type checking
npm run check

# Unit tests
npm test

# Integration tests
npm run test:integration

# All at once
npm run test:all
```

### Test Coverage

- **Services**: Aim for 80%+ coverage
- **API endpoints**: Every happy path + error cases
- **Middleware**: Authentication, rate limiting, redaction

### CI Workflows (Automated)

These run automatically on push and PR:

1. **Type Check** - TypeScript strict mode
2. **Lint** - Code style consistency
3. **Test** - Jest unit + integration tests
4. **Security Scan** - Semgrep SAST + Trivy secrets
5. **Dependency Audit** - npm audit for CVEs
6. **Import Ban** - Fail if DB imported in `/api`

All must pass before merging. See [CI workflow files](./.github/workflows/).

---

## Pull Request Process

### 1. PR Title & Description

**Title format**:
```
[TYPE] Brief description

Examples:
- feat: add user verification endpoint
- fix: prevent PII leakage in error responses
- docs: update security guidelines
```

**Description template** (auto-filled):

```markdown
## Changes
Brief summary of what this PR changes.

## Motivation
Why this change? What problem does it solve?

## Testing
How did you test this? Include steps to reproduce.

## Security Impact
- [ ] No new secrets introduced
- [ ] Input validation added
- [ ] PII handling reviewed
- [ ] Rate limiting appropriate
- [ ] Audit logging added

## Checklist
- [ ] Tests added/updated
- [ ] Documentation updated
- [ ] No high/critical CVEs
- [ ] Ready for review
```

### 2. Automatic Reviews

Our CI automatically checks:
- ✅ Build passes
- ✅ Tests pass
- ✅ No secrets detected
- ✅ Linting passes
- ✅ Type checking passes
- ✅ No import bans violated

### 3. Human Review

At least one `@security-council` member must approve. They review:
- Code quality and correctness
- Security implications
- Documentation accuracy
- Test coverage

### 4. Auto-Merge

Once approved and CI passes, a GitHub Action automatically merges when:
- ✅ All CI checks passing
- ✅ At least one approval
- ✅ Not marked as draft
- ✅ No `do-not-merge` label

---

## Naming Conventions

Consistent naming prevents confusion and improves security visibility.

### Variables & Functions

```typescript
// Tokens
const sessionToken = generatePasetoToken();
const webhookHmacSecret = rotateSecret();
const apiKeyHash = hashKey(apiKey);

// Authentication
const isAuthenticated = verifyToken(token);
const hasPermission = checkCapability(user, 'attest:submit');

// Data
const userNftId = 'IAM#12345';
const verificationLevel = 3;
const attestationData = { name_verified: true };

// Audit
const auditLogId = 'audit_abc123';
const requestId = generateRequestId();

// Error handling
const error = new VerificationError('Civic service unavailable');
const isRetryable = error instanceof TemporaryError;
```

### Files & Directories

```
server/
├── api/v1/
│   ├── agents.ts              # Agent endpoints
│   ├── attestations.ts        # Attestation endpoints
│   ├── webhooks/
│   │   ├── civic.ts           # Civic webhook handler
│   │   └── stripe.ts          # Stripe webhook handler
│   └── health.ts              # Health check
├── services/
│   ├── verification/          # Verification logic
│   ├── audit/                 # Audit logging
│   ├── auth/                  # Authentication
│   └── redaction/             # PII redaction
├── middleware/
│   ├── auth.ts                # Authentication middleware
│   ├── rate-limit.ts          # Rate limiting
│   ├── redaction.ts           # PII redaction middleware
│   └── error-handler.ts       # Error handling
├── utils/
│   ├── crypto.ts              # Cryptographic utilities
│   ├── validation.ts          # Input validation helpers
│   └── logger.ts              # Structured logging
└── storage.ts                 # Data access layer (no DB imports in /api)
```

### Environment Variables

**Prefix patterns**:
```bash
# Secrets (never commit)
DATABASE_URL=postgresql://...
SESSION_SECRET=<32-byte-base64>
HMAC_SIGNING_KEY=<32-byte-hex>

# API Configuration
VITE_API_BASE_URL=https://api.negrarosa.com
VITE_ENVIRONMENT=production

# Feature Flags
ENABLE_CIVIC_VERIFICATION=true
ENABLE_STRIPE_INTEGRATION=true

# Operational
LOG_LEVEL=info
NODE_ENV=production
PORT=3000
```

### Classes & Types

```typescript
// Services
class AuditService { }
class VerificationService { }
class WebhookService { }

// Middleware
class AuthMiddleware { }
class RateLimitMiddleware { }
class RedactionMiddleware { }

// Errors
class VerificationError extends Error { }
class AuthenticationError extends Error { }
class ValidationError extends Error { }

// Request/Response types
interface AttestationRequest { }
interface AttestationResponse { }
interface WebhookPayload { }

// Enums
enum VerificationLevel {
  UNVERIFIED = 0,
  BASIC = 1,
  STANDARD = 2,
  ENHANCED = 3,
}

enum TokenType {
  SESSION = 'session',
  WEBHOOK = 'webhook',
  API_KEY = 'api_key',
}
```

### Git Branches & Tags

```bash
# Branches
feature/user-verification
fix/pii-leakage-in-logs
docs/update-security-guide
security/upgrade-paseto-library

# Tags (semantic versioning)
v1.0.0
v1.1.0-beta
v1.2.0-rc1
```

---

## Review & Approval

### Code Review Checklist

Reviewers will check:

- ✅ **Security**: No vulnerabilities, secrets, or PII exposure
- ✅ **Quality**: Code is readable, maintainable, well-tested
- ✅ **Performance**: No N+1 queries, efficient algorithms
- ✅ **Documentation**: Comments clear, docs updated
- ✅ **Tests**: Adequate coverage, edge cases handled
- ✅ **Compatibility**: No breaking changes without discussion

### Requesting Review

1. Assign at least one `@security-council` member
2. Link related issues with "Fixes #123" or "Refs #123"
3. Be responsive to feedback
4. Thank reviewers for their time!

---

## Questions?

- **General**: Open a discussion on GitHub
- **Security**: Email security@mbtq.dev
- **Process**: Ask in PR comments or open an issue

We're here to help and excited to review your work!

---

**Last Updated**: 2026-01-02  
**Maintained by**: Security Council (@security-council)
