# Security Policy

## Supported Versions

We release patches for security vulnerabilities. Currently supported versions:

| Version | Supported          |
| ------- | ------------------ |
| 1.1.x   | :white_check_mark: |
| < 1.1   | :x:                |

## Reporting a Vulnerability

We take the security of Kode seriously. If you believe you have found a security vulnerability, please report it to us as described below.

### Please Do Not

- **Do not** open public GitHub issues for security vulnerabilities
- **Do not** disclose the vulnerability publicly until it has been addressed
- **Do not** test the vulnerability on production systems you don't own

### Please Do

1. **Email** security concerns to: ai-lab@foxmail.com
2. **Include** detailed information:
   - Type of vulnerability
   - Full paths of source file(s) related to the vulnerability
   - Location of the affected source code (tag/branch/commit or direct URL)
   - Step-by-step instructions to reproduce the issue
   - Proof-of-concept or exploit code (if possible)
   - Impact of the vulnerability and how it could be exploited

3. **Encrypt** sensitive information using our PGP key (if available)

## Response Timeline

- **Initial Response:** Within 48 hours of report
- **Status Update:** Within 7 days with preliminary analysis
- **Fix Timeline:** 
  - Critical vulnerabilities: Within 7 days
  - High severity: Within 30 days
  - Medium/Low severity: Next regular release

## Security Update Process

1. **Acknowledgment:** We'll acknowledge receipt of your report
2. **Validation:** We'll confirm the vulnerability and determine severity
3. **Development:** We'll develop and test a fix
4. **Disclosure:** We'll coordinate disclosure with you
5. **Release:** We'll release the security update
6. **Credit:** We'll credit you in the security advisory (if desired)

## Security Best Practices for Users

### Installation

- Always install from the official npm registry
- Verify package integrity: `npm audit`
- Use exact versions in production: `npm ci`

### Configuration

- **Never** commit `.env` files or API keys to version control
- Store sensitive data in environment variables
- Use strong, unique API keys
- Regularly rotate credentials

### Deployment

- Use the multi-stage Dockerfile provided
- Run containers as non-root user (see Dockerfile recommendations)
- Keep dependencies updated: `npm outdated && npm update`
- Enable security scanning in your CI/CD pipeline

### File System Access

- Kode requests explicit permission before file modifications
- Review all file operation requests carefully
- Be cautious with commands that modify system files
- Use project-specific `.kode.json` to restrict tool access

### Command Execution

- Review bash commands before approval
- Kode restricts directory traversal to project directory
- Banned commands are automatically blocked
- Commands run with your user privileges - be careful

## Known Security Considerations

### Bash Tool

The Bash tool executes shell commands with your user privileges. It includes:
- Directory traversal protection
- Banned command list
- Command validation before execution
- Sandboxing to project directory

**Best Practice:** Review all commands before approving execution.

### File Operations

File read/write operations include:
- Path traversal protection via `SecureFileService`
- Permission system for granular control
- File size limits (10MB default)
- Extension whitelisting
- Suspicious pattern detection

**Best Practice:** Use project-specific permissions to limit file access.

### AI Model Integration

When using AI models:
- API keys are stored locally in user config
- Network requests go directly to model providers
- No telemetry or data collection by default
- Conversations are stored locally

**Best Practice:** Avoid sharing sensitive data in prompts.

### MCP (Model Context Protocol) Servers

MCP servers can extend Kode functionality:
- Servers run as separate processes
- Approval required before adding new servers
- Server configurations in `.kode/mcprc.json`
- Servers have limited permissions

**Best Practice:** Only use trusted MCP servers from known sources.

## Security Features

### Built-in Protections

1. **Input Validation**
   - Zod schemas for all tool inputs
   - Path normalization and validation
   - Command syntax validation

2. **Permission System**
   - Explicit approval for file modifications
   - Tool access control per project
   - Directory-based access restrictions

3. **Secure File Service**
   - Path traversal prevention
   - Allowed directory enforcement
   - Suspicious pattern detection
   - File size limits

4. **Command Protection**
   - Banned command list
   - Directory restriction enforcement
   - Shell quote escaping
   - Command validation

5. **Dependency Security**
   - Minimal install scripts (only esbuild)
   - Regular security audits
   - Zero known vulnerabilities (as of audit date)

### What We Don't Do

- We do **not** collect telemetry or usage data
- We do **not** send your code to third parties (except chosen AI providers)
- We do **not** store conversations on remote servers
- We do **not** access files without permission

## Security Audits

Regular security audits are performed:
- Automated: npm audit on every install
- Code review: Security patterns checked in PRs
- Dependency updates: Monthly review cycle
- Comprehensive audit: Quarterly

Latest audit: See [SECURITY_AUDIT.md](./SECURITY_AUDIT.md)

## Compliance

Kode follows security best practices:
- OWASP Top 10 compliance
- NIST secure development principles
- CWE/SANS Top 25 mitigation
- npm security best practices

## Bug Bounty

We currently do not offer a bug bounty program, but we deeply appreciate responsible disclosure and will publicly credit researchers who report valid vulnerabilities.

## Security Tools

Recommended tools for security-conscious users:

```bash
# Check for vulnerabilities
npm audit

# Check for outdated dependencies
npm outdated

# Verify package integrity
npm ci

# Check for secrets in git history
git secrets --scan

# Scan dependencies
snyk test
```

## Contact

- **Security Email:** ai-lab@foxmail.com
- **General Issues:** https://github.com/shareAI-lab/kode/issues
- **GitHub Security:** Use GitHub Security Advisories for coordinated disclosure

## Attribution

We believe in giving credit where credit is due. If you report a valid security vulnerability, we will:
- Credit you in the security advisory (if you wish)
- Include you in our CONTRIBUTORS.md
- Thank you publicly (with your permission)

## Changes to This Policy

We may update this security policy from time to time. We will notify users of any material changes by:
- Updating the date at the top of this file
- Posting a notice in release notes
- Announcing in the project README

---

**Last Updated:** December 11, 2024  
**Version:** 1.0
