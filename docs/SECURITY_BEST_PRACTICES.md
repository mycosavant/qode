# Security Best Practices for Kode

This guide provides security best practices for developers and users of Kode.

## Table of Contents

1. [For Developers](#for-developers)
2. [For Users](#for-users)
3. [Secure Deployment](#secure-deployment)
4. [Security Tools](#security-tools)
5. [Common Security Pitfalls](#common-security-pitfalls)

---

## For Developers

### Development Environment Setup

#### 1. Install Security Tools

```bash
# Install project dependencies
npm install

# Install security scanning tools (optional)
npm install -g snyk
npm install -g npm-check-updates
```

#### 2. Enable Git Hooks

The project uses Husky for Git hooks. Ensure they're enabled:

```bash
# Git hooks are automatically installed via npm install
# To manually install:
npx husky install
```

#### 3. Run Security Checks Before Commit

```bash
# Run all security checks
npm run security:check

# Run npm audit only
npm run security:audit

# Fix auto-fixable vulnerabilities
npm run security:fix
```

### Code Security Guidelines

#### 1. Never Hardcode Secrets

❌ **Bad:**
```typescript
const apiKey = "sk-ant-1234567890abcdef"
```

✅ **Good:**
```typescript
const apiKey = process.env.ANTHROPIC_API_KEY
```

#### 2. Validate All Inputs

Always use Zod schemas for input validation:

```typescript
import { z } from 'zod'

const inputSchema = z.strictObject({
  filePath: z.string().describe('File path to read'),
  encoding: z.enum(['utf-8', 'ascii', 'base64']).optional(),
})
```

#### 3. Sanitize File Paths

Use the `SecureFileService` for all file operations:

```typescript
import { SecureFileService } from '@utils/secureFile'

const secureFile = SecureFileService.getInstance()
const validation = secureFile.validateFilePath(userInput)

if (!validation.isValid) {
  throw new Error(validation.error)
}
```

#### 4. Escape Shell Commands

Always use `shell-quote` for shell commands:

```typescript
import shellquote from 'shell-quote'

const escapedCommand = shellquote.quote([command, ...args])
```

#### 5. Handle Errors Securely

Don't expose sensitive information in error messages:

❌ **Bad:**
```typescript
catch (error) {
  throw new Error(`Failed to read file ${apiKeyPath}: ${error.message}`)
}
```

✅ **Good:**
```typescript
catch (error) {
  logError('File read error', { error })
  throw new Error('Failed to read configuration file')
}
```

### Security Testing

#### 1. Run Tests

```bash
# Run all tests
npm test

# Run tests with coverage
npm test -- --coverage
```

#### 2. Type Checking

```bash
# Check TypeScript types
npm run typecheck
```

#### 3. Linting

```bash
# Run linter
npm run lint

# Auto-fix issues
npm run lint:fix
```

### Dependency Management

#### 1. Regular Updates

```bash
# Check for updates
npm outdated

# Update dependencies
npm update

# Update to latest (major versions)
npx npm-check-updates -u
npm install
```

#### 2. Audit Dependencies

```bash
# Run security audit
npm audit

# Fix vulnerabilities automatically
npm audit fix

# Force fix (may include breaking changes)
npm audit fix --force
```

#### 3. Review New Dependencies

Before adding a new dependency:

1. Check npm package page for:
   - Download statistics
   - Last update date
   - Open issues count
   - GitHub stars
   - Security advisories

2. Review the package source code on GitHub

3. Check for install scripts:
```bash
npm info <package-name> scripts
```

4. Use exact versions in package.json:
```json
{
  "dependencies": {
    "trusted-package": "1.2.3"  // No ^ or ~
  }
}
```

---

## For Users

### Installation Security

#### 1. Install from Official Sources

```bash
# Preferred: Install from official npm registry
npm install -g @shareai-lab/kode

# Verify installation
kode --version
```

#### 2. Verify Package Integrity

```bash
# Before installing, check for vulnerabilities
npm audit @shareai-lab/kode

# View package details
npm info @shareai-lab/kode
```

### Configuration Security

#### 1. Protect API Keys

Store API keys in environment variables or config files:

```bash
# Create config file
kode config

# Set API key (stored securely in ~/.kode.json)
# Enter API key when prompted
```

**Never:**
- Commit API keys to Git
- Share API keys in chat/email
- Use the same key across multiple projects
- Store keys in plaintext files in your project

#### 2. Secure Config Files

```bash
# Ensure proper permissions on config files
chmod 600 ~/.kode.json
chmod 600 ./.kode.json
```

#### 3. Project-Level Security

Create a `.kode.json` in your project to restrict tool access:

```json
{
  "allowedTools": ["FileRead", "FileWrite", "Bash"],
  "dontCrawlDirectory": false,
  "mcpServers": {}
}
```

### Usage Security

#### 1. Review Commands Before Approval

Always review bash commands before approving:

```
Kode wants to run:
> rm -rf /important-directory

[Approve] [Deny] [Edit]
```

**Best Practice:** Choose "Edit" if you're unsure, or "Deny" if it looks suspicious.

#### 2. Review File Operations

Check file paths before approving writes:

```
Kode wants to modify:
/home/user/project/config.json

[View Changes] [Approve] [Deny]
```

**Best Practice:** Always view changes for configuration files.

#### 3. Limit File Access

Use project-specific permissions:

```bash
# Only allow Kode to access current project
cd /your/project
kode

# Kode will be restricted to this directory and its children
```

#### 4. MCP Server Security

Only add trusted MCP servers:

```bash
# Review server configuration before adding
cat ~/.kode/mcprc.json

# Remove untrusted servers
kode config
# Navigate to MCP Servers and remove
```

---

## Secure Deployment

### Docker Deployment

#### 1. Use Secure Dockerfile

```bash
# Build with security-hardened Dockerfile
docker build -f Dockerfile.secure -t kode:secure .

# Run as non-root user (already configured in Dockerfile.secure)
docker run --user 1001:1001 kode:secure
```

#### 2. Configure Registry Mirrors

For production, use official registries:

```bash
docker build -f Dockerfile.secure \
  --build-arg ALPINE_MIRROR=dl-cdn.alpinelinux.org \
  --build-arg NPM_REGISTRY=https://registry.npmjs.org/ \
  --build-arg PIP_INDEX_URL=https://pypi.org/simple \
  -t kode:secure .
```

#### 3. Scan Docker Images

```bash
# Using Trivy
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image kode:secure

# Using Snyk
snyk container test kode:secure
```

### Production Environment

#### 1. Environment Variables

```bash
# Set required environment variables
export ANTHROPIC_API_KEY=your-api-key
export NODE_ENV=production
export NPM_CONFIG_AUDIT=true
```

#### 2. File Permissions

```bash
# Restrict access to sensitive files
chmod 600 ~/.kode.json
chmod 600 .env

# Ensure proper ownership
chown $USER:$USER ~/.kode.json
```

#### 3. Network Security

- Use HTTPS for all external connections
- Enable firewall rules
- Use VPN for sensitive operations
- Restrict outbound connections if possible

---

## Security Tools

### Recommended Tools

1. **npm audit** - Built-in vulnerability scanner
   ```bash
   npm audit
   ```

2. **Snyk** - Comprehensive security scanner
   ```bash
   npm install -g snyk
   snyk test
   ```

3. **TruffleHog** - Secret scanner
   ```bash
   docker run --rm -v "$PWD:/pwd" trufflesecurity/trufflehog:latest \
     filesystem /pwd
   ```

4. **Trivy** - Container vulnerability scanner
   ```bash
   trivy image kode:latest
   ```

5. **OWASP Dependency-Check**
   ```bash
   dependency-check --project kode --scan .
   ```

### Automated Scanning

Enable GitHub security features:

1. **Dependabot** - Automated dependency updates
   - Already configured in `.github/dependabot.yml`

2. **Code Scanning** - CodeQL analysis
   - Configured in `.github/workflows/security-audit.yml`

3. **Secret Scanning** - Detect committed secrets
   - Enable in GitHub repository settings

---

## Common Security Pitfalls

### 1. Path Traversal

❌ **Vulnerable:**
```typescript
const filePath = userInput  // Could be "../../etc/passwd"
const content = readFileSync(filePath)
```

✅ **Secure:**
```typescript
import { SecureFileService } from '@utils/secureFile'

const secureFile = SecureFileService.getInstance()
const validation = secureFile.validateFilePath(userInput)

if (validation.isValid) {
  const content = secureFile.readFile(validation.normalizedPath)
}
```

### 2. Command Injection

❌ **Vulnerable:**
```typescript
exec(`git commit -m "${userMessage}"`)  // Vulnerable to injection
```

✅ **Secure:**
```typescript
import { spawn } from 'child_process'
spawn('git', ['commit', '-m', userMessage])  // Safe
```

### 3. Information Disclosure

❌ **Vulnerable:**
```typescript
catch (error) {
  res.status(500).json({ error: error.stack })  // Exposes internals
}
```

✅ **Secure:**
```typescript
catch (error) {
  logError(error)  // Log internally
  res.status(500).json({ error: 'Internal server error' })
}
```

### 4. Insecure Dependencies

❌ **Risky:**
```bash
npm install unknown-package  # From untrusted source
```

✅ **Safe:**
```bash
# 1. Research the package
npm info unknown-package

# 2. Check for vulnerabilities
npm audit unknown-package

# 3. Review source code
# Visit GitHub repository

# 4. Use with caution
npm install unknown-package --save-exact
```

### 5. API Key Exposure

❌ **Dangerous:**
```typescript
const config = {
  apiKey: "sk-ant-1234567890"  // Hardcoded
}
```

✅ **Secure:**
```typescript
const config = {
  apiKey: process.env.ANTHROPIC_API_KEY  // From environment
}

// In .env file (never commit!)
ANTHROPIC_API_KEY=sk-ant-1234567890

// In .gitignore
.env
.env.local
.env.*.local
```

---

## Security Checklist

### Before Every Commit

- [ ] No hardcoded secrets or API keys
- [ ] All user inputs validated
- [ ] File paths sanitized
- [ ] Shell commands properly escaped
- [ ] Tests pass (`npm test`)
- [ ] Linter passes (`npm run lint`)
- [ ] Type checking passes (`npm run typecheck`)
- [ ] Security audit clean (`npm audit`)

### Before Every Release

- [ ] All dependencies updated
- [ ] Security audit clean
- [ ] CodeQL analysis passes
- [ ] Docker image scanned
- [ ] Documentation updated
- [ ] CHANGELOG.md updated
- [ ] Version bumped appropriately

### For Production Deployment

- [ ] Environment variables configured
- [ ] File permissions set correctly
- [ ] Using secure Dockerfile
- [ ] Network security configured
- [ ] Monitoring and logging enabled
- [ ] Incident response plan in place
- [ ] Security contact information available

---

## Getting Help

### Security Issues

For security vulnerabilities, see [SECURITY.md](./SECURITY.md)

### General Security Questions

- GitHub Discussions: https://github.com/shareAI-lab/kode/discussions
- Documentation: https://github.com/shareAI-lab/kode/blob/main/README.md

### Emergency Contacts

For urgent security issues:
- Email: ai-lab@foxmail.com
- Include "SECURITY" in subject line
- Provide detailed information
- Allow 48 hours for initial response

---

**Last Updated:** December 11, 2024
