# Security Configuration Guide

This document outlines the security configuration for the Cloudflare Workers CI/CD platform, including OIDC authentication, API token scoping, and security best practices.

## OIDC Authentication Setup

OpenID Connect (OIDC) eliminates the need for long-lived secrets by using short-lived tokens issued by GitHub.

### GitHub OIDC Provider Configuration

1. **Configure GitHub as Identity Provider**:
```yaml
# In your GitHub Actions workflow
permissions:
  id-token: write  # Required for OIDC
  contents: read
  deployments: write
```

2. **Cloudflare Trust Relationship**:
```bash
# Create Cloudflare API token with OIDC support
# This requires Cloudflare Enterprise plan with OIDC integration
```

### OIDC Workflow Example

```yaml
name: Secure Deploy with OIDC

on:
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure OIDC Token
        uses: actions/github-script@v7
        id: oidc-token
        with:
          script: |
            const token = await core.getIDToken('cloudflare');
            core.setSecret(token);
            return token;

      - name: Deploy with OIDC
        run: |
          # Use OIDC token for authentication
          wrangler deploy --oidc-token ${{ steps.oidc-token.outputs.result }}
```

## API Token Scoping

### Cloudflare API Token Best Practices

1. **Minimal Permissions**:
```json
{
  "token_name": "cf-pipeline-ci",
  "permissions": [
    {
      "effect": "allow",
      "resources": {
        "com.cloudflare.api.account.zone.*": "*",
        "com.cloudflare.api.account.workers.script": "*"
      },
      "permission_groups": [
        {
          "id": "c8fed203ed3043cba015a93ad1616b1f",
          "name": "Zone:Read"
        },
        {
          "id": "e086da7e2179491d91ee5f35b3ca210a", 
          "name": "Workers Scripts:Edit"
        }
      ]
    }
  ],
  "condition": {
    "request.ip": {
      "in": ["192.30.252.0/22", "185.199.108.0/22"]  # GitHub IP ranges
    }
  },
  "ttl": 3600  # 1 hour expiration
}
```

2. **Environment-Specific Tokens**:
```bash
# Production token (restricted)
wrangler auth token create \
  --name "production-deploy" \
  --permissions "Zone:Read,Workers Scripts:Edit" \
  --resources "zone:example.com" \
  --ttl 3600

# Staging token (less restricted)
wrangler auth token create \
  --name "staging-deploy" \
  --permissions "Zone:Read,Workers Scripts:Edit" \
  --resources "zone:staging.example.com" \
  --ttl 7200
```

### Token Rotation Strategy

```yaml
# .github/workflows/rotate-tokens.yml
name: Rotate API Tokens

on:
  schedule:
    - cron: '0 2 * * 1'  # Every Monday at 2 AM

jobs:
  rotate:
    runs-on: ubuntu-latest
    steps:
      - name: Generate New Token
        run: |
          # Create new token with 30-day expiration
          NEW_TOKEN=$(wrangler auth token create --ttl 2592000 --json | jq -r '.result.value')
          
          # Update GitHub secret
          gh secret set CLOUDFLARE_API_TOKEN --body "$NEW_TOKEN"
          
          # Schedule old token deletion
          echo "OLD_TOKEN_ID=$(wrangler auth token list --json | jq -r '.result[0].id')" >> $GITHUB_ENV

      - name: Test New Token
        run: |
          # Verify new token works
          wrangler whoami
          
      - name: Revoke Old Token
        run: |
          # Delete old token after verification
          wrangler auth token delete $OLD_TOKEN_ID
```

## Secret Management

### Repository Secrets Configuration

```bash
# Required secrets for all repositories
CLOUDFLARE_API_TOKEN=<scoped-api-token>
DATADOG_API_KEY=<datadog-api-key>      # Optional
DATADOG_APP_KEY=<datadog-app-key>      # Optional

# For custom GitHub App (recommended)
APP_ID=<github-app-id>
APP_PRIVATE_KEY=<github-app-private-key>
```

### Environment Variables vs Secrets

```toml
# wrangler.toml - Non-sensitive configuration
[vars]
ENVIRONMENT = "production"
API_VERSION = "v1"
LOG_LEVEL = "info"

# Secrets (set via wrangler secret put)
# DATABASE_URL = "secret"
# THIRD_PARTY_API_KEY = "secret"
# ENCRYPTION_KEY = "secret"
```

### Secret Scanning Prevention

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on: [push, pull_request]

jobs:
  secrets:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for secret scanning
          
      - name: Run secret scan
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: main
          head: HEAD
```

## Security Scanning Integration

### SAST (Static Application Security Testing)

```yaml
# Integrated into ci.yml workflow
security-scan:
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    
    - name: Initialize CodeQL
      uses: github/codeql-action/init@v3
      with:
        languages: javascript
        
    - name: Autobuild
      uses: github/codeql-action/autobuild@v3
      
    - name: Perform CodeQL Analysis
      uses: github/codeql-action/analyze@v3
```

### Dependency Scanning

```yaml
    - name: Run Snyk security scan
      uses: snyk/actions/node@master
      env:
        SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      with:
        args: --severity-threshold=high
        
    - name: Upload Snyk results
      uses: github/codeql-action/upload-sarif@v3
      if: always()
      with:
        sarif_file: snyk.sarif
```

### Container Scanning

```yaml
    - name: Build container
      run: docker build -t worker:${{ github.sha }} .
      
    - name: Scan container
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: 'worker:${{ github.sha }}'
        format: 'sarif'
        output: 'trivy-results.sarif'
        
    - name: Upload Trivy scan results
      uses: github/codeql-action/upload-sarif@v3
      with:
        sarif_file: 'trivy-results.sarif'
```

## Compliance and Audit

### Audit Logging

```javascript
// In Worker code - audit logging
export default {
  async fetch(request, env, ctx) {
    const startTime = Date.now();
    const requestId = crypto.randomUUID();
    
    // Log request start
    console.log(JSON.stringify({
      type: 'request_start',
      requestId,
      method: request.method,
      url: request.url,
      userAgent: request.headers.get('user-agent'),
      timestamp: new Date().toISOString()
    }));
    
    try {
      const response = await handleRequest(request, env, ctx);
      
      // Log successful response
      console.log(JSON.stringify({
        type: 'request_complete',
        requestId,
        status: response.status,
        duration: Date.now() - startTime,
        timestamp: new Date().toISOString()
      }));
      
      return response;
    } catch (error) {
      // Log errors (without sensitive data)
      console.log(JSON.stringify({
        type: 'request_error',
        requestId,
        error: error.message,
        duration: Date.now() - startTime,
        timestamp: new Date().toISOString()
      }));
      
      throw error;
    }
  }
};
```

### Compliance Reporting

```yaml
# .github/workflows/compliance.yml
name: Compliance Check

on:
  schedule:
    - cron: '0 6 * * *'  # Daily at 6 AM

jobs:
  compliance:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Generate compliance report
        run: |
          # Check for required security files
          echo "## Security Compliance Report" > compliance-report.md
          echo "Date: $(date)" >> compliance-report.md
          echo "" >> compliance-report.md
          
          # Check for security files
          if [ -f "SECURITY.md" ]; then
            echo "✅ SECURITY.md present" >> compliance-report.md
          else
            echo "❌ SECURITY.md missing" >> compliance-report.md
          fi
          
          # Check for dependency scanning
          if grep -q "npm audit" .github/workflows/*.yml; then
            echo "✅ Dependency scanning enabled" >> compliance-report.md
          else
            echo "❌ Dependency scanning missing" >> compliance-report.md
          fi
          
          # Check for secret scanning
          if [ -f ".github/workflows/security-scan.yml" ]; then
            echo "✅ Secret scanning configured" >> compliance-report.md
          else
            echo "❌ Secret scanning missing" >> compliance-report.md
          fi
          
      - name: Upload compliance report
        uses: actions/upload-artifact@v4
        with:
          name: compliance-report
          path: compliance-report.md
```

## Security Policies

### Required Security Headers

```javascript
// Security headers for all Worker responses
const securityHeaders = {
  'X-Content-Type-Options': 'nosniff',
  'X-Frame-Options': 'DENY',
  'X-XSS-Protection': '1; mode=block',
  'Strict-Transport-Security': 'max-age=31536000; includeSubDomains',
  'Content-Security-Policy': "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'",
  'Referrer-Policy': 'strict-origin-when-cross-origin'
};

// Apply to all responses
function addSecurityHeaders(response) {
  const newResponse = new Response(response.body, response);
  
  Object.entries(securityHeaders).forEach(([key, value]) => {
    newResponse.headers.set(key, value);
  });
  
  return newResponse;
}
```

### Input Validation

```javascript
// Input validation and sanitization
function validateInput(input, type) {
  switch (type) {
    case 'email':
      const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
      return emailRegex.test(input);
      
    case 'uuid':
      const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[1-5][0-9a-f]{3}-[89ab][0-9a-f]{3}-[0-9a-f]{12}$/i;
      return uuidRegex.test(input);
      
    case 'alphanumeric':
      const alphanumericRegex = /^[a-zA-Z0-9]+$/;
      return alphanumericRegex.test(input);
      
    default:
      return false;
  }
}

// Rate limiting
const rateLimiter = {
  requests: new Map(),
  
  isAllowed(ip, limit = 100, window = 60000) {
    const now = Date.now();
    const userRequests = this.requests.get(ip) || [];
    
    // Remove old requests outside the window
    const recentRequests = userRequests.filter(time => now - time < window);
    
    if (recentRequests.length >= limit) {
      return false;
    }
    
    recentRequests.push(now);
    this.requests.set(ip, recentRequests);
    return true;
  }
};
```

## Incident Response

### Security Incident Workflow

```yaml
# .github/workflows/security-incident.yml
name: Security Incident Response

on:
  issues:
    types: [opened]
    
jobs:
  security-response:
    if: contains(github.event.issue.labels.*.name, 'security')
    runs-on: ubuntu-latest
    steps:
      - name: Acknowledge incident
        uses: actions/github-script@v7
        with:
          script: |
            await github.rest.issues.createComment({
              issue_number: ${{ github.event.issue.number }},
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: '🚨 Security incident acknowledged. Response team notified.'
            });
            
      - name: Notify security team
        run: |
          # Send to Datadog, Slack, or other notification system
          curl -X POST ${{ secrets.SECURITY_WEBHOOK_URL }} \
            -H "Content-Type: application/json" \
            -d '{"text": "Security incident: ${{ github.event.issue.html_url }}"}'
            
      - name: Create emergency deployment
        if: contains(github.event.issue.title, 'CRITICAL')
        run: |
          # Trigger emergency rollback or patch deployment
          gh workflow run emergency-deploy.yml
```

### Emergency Procedures

```bash
# Emergency rollback script
#!/bin/bash
set -e

echo "🚨 EMERGENCY ROLLBACK INITIATED"

# Get previous deployment
PREVIOUS_VERSION=$(cat .cf-scripts/deployments.json | jq -r '.[1].version')
PREVIOUS_SHA=$(cat .cf-scripts/deployments.json | jq -r '.[1].commit_sha')

echo "Rolling back to version: $PREVIOUS_VERSION"
echo "Rolling back to commit: $PREVIOUS_SHA"

# Checkout previous version
git checkout $PREVIOUS_SHA

# Deploy immediately
wrangler deploy --env production

# Notify teams
echo "✅ Emergency rollback completed"
echo "Previous version $PREVIOUS_VERSION is now live"
```

## Security Checklist

### Pre-Deployment Security Review

- [ ] No hardcoded secrets in code
- [ ] All secrets properly scoped
- [ ] API tokens have minimal permissions
- [ ] Security headers implemented
- [ ] Input validation in place
- [ ] Error handling doesn't leak information
- [ ] Dependency scan passed
- [ ] SAST scan passed
- [ ] Container scan passed (if applicable)
- [ ] Compliance requirements met

### Post-Deployment Security Verification

- [ ] Security headers present in responses
- [ ] Rate limiting working correctly
- [ ] Audit logging enabled
- [ ] Monitoring alerts configured
- [ ] Incident response procedures tested
- [ ] Documentation updated

## Resources

- [Cloudflare Security Documentation](https://developers.cloudflare.com/workers/platform/security/)
- [GitHub OIDC Documentation](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- [OWASP API Security Top 10](https://owasp.org/www-project-api-security/)
- [Cloudflare API Token Documentation](https://developers.cloudflare.com/api/tokens/)

---

**Remember**: Security is an ongoing process, not a one-time setup. Regularly review and update your security configurations.