# Security Report - mcp-gdrive

## Overview

This document outlines security considerations, vulnerabilities, and recommendations for the mcp-gdrive project, an MCP server that integrates with Google Drive and Sheets.

**Last Updated:** 2025-10-16
**Version:** 0.2.0

---

## Executive Summary

The mcp-gdrive project has undergone comprehensive security review. **20 security issues** have been identified across authentication, input validation, concurrency, error handling, and operational security domains.

**Critical Findings:**
- Plaintext credential storage (HIGH)
- Path traversal risk in credential directory (MEDIUM)
- Missing input validation across all tools (MEDIUM)
- Race conditions in token refresh (MEDIUM)
- Unhandled authentication states (MEDIUM)
- Sensitive data logging (MEDIUM)

While the overall architecture is sound for a development tool, **significant hardening is required** before production deployment. The lack of security tests and defensive coding practices increases risk of exploitation.

**Overall Risk Level:** MEDIUM-HIGH (elevated from initial MEDIUM due to depth of findings)

---

## Identified Security Issues

### 1. Credential Storage and Management

**Severity:** HIGH
**Status:** PRESENT

#### Issue Description
OAuth credentials including refresh tokens are stored in plaintext JSON files on the local filesystem (auth.ts:75, auth.ts:124).

**Location:**
- `auth.ts:75` - Initial credential save
- `auth.ts:124` - Token refresh save
- Default storage: `.gdrive-server-credentials.json`

#### Risk
- Credentials stored in plaintext can be accessed by any process with read permissions
- Refresh tokens provide long-lived access to user's Google Drive
- No encryption at rest
- File permissions not explicitly set (relying on umask)

#### Recommendations
1. Implement credential encryption at rest using system keychain/keyring
2. Set explicit restrictive file permissions (0600) when creating credential files
3. Consider using OS-specific secure storage:
   - macOS: Keychain
   - Linux: Secret Service API / gnome-keyring
   - Windows: Credential Manager
4. Add credential file location to .gitignore (already done for `.env`)
5. Document secure credential storage practices in README

#### Workaround
Users should manually set permissions: `chmod 600 .gdrive-server-credentials.json`

---

### 2. Environment Variable Exposure

**Severity:** MEDIUM
**Status:** PRESENT

#### Issue Description
OAuth client secrets are passed via environment variables and configuration files (README.md:68-74, index.ts:2).

**Location:**
- `index.ts:2` - dotenv configuration
- Environment variables: `CLIENT_ID`, `CLIENT_SECRET`, `GDRIVE_CREDS_DIR`

#### Risk
- Environment variables can be logged in process listings
- Parent processes inherit environment variables
- May be exposed in crash dumps or error logs

#### Recommendations
1. Document that `.env` file must not be committed (already in .gitignore:1)
2. Add runtime check to warn if credentials appear in process arguments
3. Consider alternative configuration methods (config files with restricted permissions)
4. Add security documentation about environment variable handling

#### Current Mitigations
- `.env` is properly excluded in .gitignore
- Recent commit (0aa5f65) added dotenv support and ignores compiled files

---

### 3. Input Validation and Injection Risks

**Severity:** MEDIUM
**Status:** PRESENT

#### Issue Description
User input is not thoroughly validated before being used in Google Drive API queries.

**Locations:**
- `tools/gdrive_search.ts:41` - Basic escaping but limited validation
- `tools/gdrive_read_file.ts:43` - fileId not validated
- `tools/gsheets_read.ts:99` - spreadsheetId not validated
- `tools/gsheets_update_cell.ts:33` - range and value not validated
- `index.ts:81` - URI manipulation without validation

#### Specific Vulnerabilities

**a) Query Injection in Search (gdrive_search.ts:41-54)**
```typescript
const escapedQuery = userQuery.replace(/\\/g, "\\\\").replace(/'/g, "\\'");
```
- Only escapes backslashes and single quotes
- May be vulnerable to other special characters in Drive query syntax
- MIME type conditions concatenated without sanitization

**b) Unvalidated File/Spreadsheet IDs**
- No format validation for Google Drive file IDs
- Could cause unexpected API behavior or error disclosure
- No length limits enforced

**c) A1 Notation Injection (gsheets_update_cell.ts)**
- Range parameter accepts arbitrary A1 notation
- No validation of range format
- Could potentially write to unintended cells

**d) URI Manipulation (index.ts:81)**
```typescript
const fileId = request.params.uri.replace("gdrive:///", "");
```
- Simple string replacement without validation
- No check for valid file ID format

#### Recommendations
1. Implement strict input validation:
   - File IDs should match Google's format: `^[a-zA-Z0-9_-]{25,}$`
   - Spreadsheet IDs should match similar format
   - A1 notation should be validated against regex pattern
2. Add input length limits
3. Implement allowlist validation where possible
4. Add comprehensive query sanitization for Drive search
5. Consider using parameterized queries or builder patterns
6. Add input validation tests

---

### 4. Error Information Disclosure

**Severity:** LOW
**Status:** PRESENT

#### Issue Description
Detailed error messages may expose internal system information.

**Locations:**
- `auth.ts:50` - Logs full error objects
- `auth.ts:83` - Logs refresh token errors
- `auth.ts:128` - Logs failed refresh details
- `gsheets_read.ts:154` - Returns raw error messages to clients
- `index.ts:142` - Logs errors to stderr

#### Risk
- Error messages may reveal file paths
- Stack traces could expose code structure
- API error messages might contain sensitive metadata

#### Recommendations
1. Implement error sanitization before logging
2. Separate internal logging from user-facing error messages
3. Remove file paths and stack traces from production error responses
4. Consider structured logging with severity levels
5. Implement error message allowlist for client responses

---

### 5. Dependency Vulnerabilities

**Severity:** LOW
**Status:** PRESENT

#### Issue Description
One low-severity vulnerability detected in dependencies.

**Details:**
- Package: `brace-expansion` (transitive dependency)
- Vulnerability: GHSA-v6h2-p8h4-qcjw
- Severity: LOW (CVSS 3.1)
- Type: Regular Expression Denial of Service (CWE-400)
- Affected versions: 1.0.0 - 1.1.11
- Fix available: Yes

#### Recommendations
1. Run `npm audit fix` to update brace-expansion
2. Implement regular dependency scanning (weekly/monthly)
3. Add npm audit to CI/CD pipeline
4. Consider using tools like Snyk or Dependabot
5. Document dependency update policy

---

### 6. Missing Security Headers and Rate Limiting

**Severity:** LOW
**Status:** PRESENT

#### Issue Description
No rate limiting or request throttling implemented for MCP server operations.

**Location:**
- All tool handlers lack rate limiting
- No request counting or throttling mechanism

#### Risk
- Potential for resource exhaustion
- Could trigger Google API rate limits
- No protection against rapid credential guessing

#### Recommendations
1. Implement rate limiting per tool/operation
2. Add backoff logic for API calls
3. Respect Google API quotas and limits
4. Add request logging for security monitoring
5. Consider implementing request queuing

---

### 7. Authentication Timeout Security

**Severity:** LOW
**Status:** PARTIALLY MITIGATED

#### Issue Description
Authentication flow has a timeout, but timeout handling could be improved.

**Location:**
- `auth.ts:33-53` - authenticateWithTimeout function
- Default timeout: 30 seconds

#### Current State
- Recent commit (d180e73) increased auth timeout
- Timeout prevents indefinite hangs
- Null return on timeout may not clearly indicate timeout vs other failures

#### Recommendations
1. Add distinct error types for timeout vs authentication failure
2. Log timeout events for security monitoring
3. Consider making timeout configurable via environment variable
4. Add retry logic with exponential backoff

---

### 8. Token Refresh Security

**Severity:** MEDIUM
**Status:** PRESENT

#### Issue Description
Automatic token refresh runs in background without security boundaries.

**Location:**
- `auth.ts:153-172` - setupTokenRefresh function
- `index.ts:140` - Background refresh initiated

#### Risk
- Refresh interval is fixed at 45 minutes
- No error rate limiting for failed refreshes
- Background process could continue attempting refresh on compromised credentials
- No notification mechanism for refresh failures

#### Recommendations
1. Implement maximum retry attempts for failed refreshes
2. Add exponential backoff for refresh failures
3. Notify user/admin on repeated refresh failures
4. Add health check endpoint for token status
5. Consider making refresh interval configurable
6. Add circuit breaker pattern for failing refreshes

---

### 9. OAuth Scope Permissions

**Severity:** MEDIUM
**Status:** PRESENT (Design Decision)

#### Issue Description
OAuth scopes grant broad permissions.

**Location:**
- `auth.ts:6-9` - SCOPES definition
  - `https://www.googleapis.com/auth/drive.readonly`
  - `https://www.googleapis.com/auth/spreadsheets` (read/write)

#### Risk
- Drive scope is read-only (good)
- Sheets scope allows full read/write access
- No granular permission control
- If credentials compromised, attacker has full sheet access

#### Recommendations
1. Document why full spreadsheet scope is required
2. Consider implementing per-operation scope requirements
3. Add runtime permission checks before write operations
4. Document the principle of least privilege in security docs
5. Consider offering read-only mode configuration option

---

### 10. Missing Security Documentation

**Severity:** LOW
**Status:** PRESENT

#### Issue Description
No dedicated security documentation for users and developers.

#### Missing Documentation
- Secure deployment guidelines
- Credential rotation procedures
- Incident response plan
- Security best practices
- Threat model

#### Recommendations
1. Create this security.md file (in progress)
2. Add security section to README
3. Document credential lifecycle
4. Provide security checklist for deployment
5. Add SECURITY.md for vulnerability reporting

---

### 11. Path Traversal Risk in Credential Directory

**Severity:** MEDIUM
**Status:** PRESENT

#### Issue Description
The GDRIVE_CREDS_DIR environment variable can be set to arbitrary paths without validation, potentially allowing path traversal attacks.

**Location:**
- `auth.ts:12-14` - CREDS_DIR assignment from environment variable
- `auth.ts:18-29` - ensureCredsDirectory function
- `auth.ts:31` - credentialsPath construction
- `auth.ts:59` - keyfilePath construction

#### Code Analysis
```typescript
const CREDS_DIR =
  process.env.GDRIVE_CREDS_DIR ||
  path.join(path.dirname(new URL(import.meta.url).pathname), "../../../");
```

#### Risk
- No validation of GDRIVE_CREDS_DIR path
- Could contain path traversal sequences (../, ..\)
- Malicious path could write credentials to arbitrary locations
- No check for absolute vs relative paths
- Could potentially overwrite system files if run with elevated privileges

#### Recommendations
1. Validate GDRIVE_CREDS_DIR is an absolute path
2. Check that resolved path doesn't escape intended boundaries
3. Use `path.resolve()` to normalize the path
4. Validate path doesn't contain suspicious patterns
5. Add explicit check that path is within user's home directory or designated config areas
6. Document expected GDRIVE_CREDS_DIR format in README

#### Example Fix
```typescript
const validateCredentialsPath = (dirPath: string): string => {
  const resolved = path.resolve(dirPath);
  const homeDir = os.homedir();

  if (!resolved.startsWith(homeDir)) {
    throw new Error('Credentials directory must be within user home directory');
  }

  return resolved;
};
```

---

### 12. Race Condition in Token Refresh

**Severity:** LOW-MEDIUM
**Status:** PRESENT

#### Issue Description
Multiple concurrent operations could trigger simultaneous token refresh attempts, leading to race conditions.

**Location:**
- `auth.ts:89-138` - loadCredentialsQuietly function
- `auth.ts:118-131` - Token refresh within loadCredentialsQuietly
- `auth.ts:153-172` - setupTokenRefresh background timer
- `index.ts:55` - ensureAuthQuietly in ListResources
- `index.ts:80` - ensureAuthQuietly in ReadResource
- `index.ts:119` - ensureAuth in CallTool

#### Risk
- Multiple request handlers can call ensureAuthQuietly() concurrently
- Two processes could read expired token simultaneously
- Both attempt to refresh, causing duplicate OAuth refresh requests
- Google may reject duplicate refresh attempts
- File writes to credentials could be interleaved, corrupting the file
- No mutex or locking mechanism for credential file access

#### Recommendations
1. Implement mutex/lock for token refresh operations
2. Add in-memory flag to track ongoing refresh
3. Queue refresh requests if one is in progress
4. Use atomic file operations for credential writes
5. Add refresh token deduplication logic
6. Consider using a proper state management library

---

### 13. Unhandled Authentication State in ListResources

**Severity:** MEDIUM
**Status:** PRESENT

#### Issue Description
ListResourcesRequestSchema handler uses ensureAuthQuietly() but doesn't verify authentication succeeded before making API calls.

**Location:**
- `index.ts:54-77` - ListResourcesRequestSchema handler

#### Code Analysis
```typescript
server.setRequestHandler(ListResourcesRequestSchema, async (request) => {
  await ensureAuthQuietly();  // Returns null if auth fails, but not checked
  const pageSize = 10;
  const params: any = {
    pageSize,
    fields: "nextPageToken, files(id, name, mimeType)",
  };
  // ... proceeds to make API call regardless
  const res = await drive.files.list(params);
```

#### Risk
- If ensureAuthQuietly() returns null, drive.files.list() will fail
- Unauthenticated API calls expose internal error messages
- No graceful degradation or retry logic
- Error could leak credential paths or configuration details

#### Recommendations
1. Check return value of ensureAuthQuietly()
2. Return appropriate error if authentication fails
3. Add retry logic with proper authentication
4. Implement consistent auth verification across all handlers
5. Consider falling back to ensureAuth() if quiet auth fails

---

### 14. Hardcoded Tool Array Index

**Severity:** LOW
**STATUS:** PRESENT

#### Issue Description
ReadResourceRequestSchema handler uses hardcoded array index to access tool, which is brittle and could break if tool order changes.

**Location:**
- `index.ts:82` - Hardcoded tool array index

#### Code Analysis
```typescript
const readFileTool = tools[1]; // gdrive_read_file is the second tool
const result = await readFileTool.handler({ fileId });
```

#### Risk
- If tools array order changes, wrong tool will be invoked
- No runtime verification that tools[1] is correct tool
- Could invoke wrong operation with security implications
- Array index access could be out of bounds
- Maintenance hazard

#### Recommendations
1. Use tools.find() with tool name matching
2. Add runtime assertion that correct tool was found
3. Throw explicit error if tool not found
4. Consider using a tool registry/map instead of array
5. Add type safety for tool lookup

#### Example Fix
```typescript
const readFileTool = tools.find(t => t.name === 'gdrive_read_file');
if (!readFileTool) {
  throw new Error('gdrive_read_file tool not found');
}
const result = await readFileTool.handler({ fileId });
```

---

### 15. Unsafe String Splitting in ReadResource

**Severity:** LOW
**STATUS:** PRESENT

#### Issue Description
File contents extraction uses naive string splitting which could fail or produce incorrect results.

**Location:**
- `index.ts:86` - String split to extract file contents

#### Code Analysis
```typescript
const fileContents = result.content[0].text.split("\n\n")[1];
```

#### Risk
- Assumes content array has at least one element (could throw)
- Assumes split produces at least 2 elements (returns undefined if not)
- File content containing "\n\n" at start would be incorrectly parsed
- No validation of expected format
- Could expose internal error messages if format unexpected

#### Recommendations
1. Add bounds checking for array access
2. Validate content structure before parsing
3. Handle edge cases (empty content, different formats)
4. Add explicit error handling for malformed responses
5. Consider more robust parsing approach

---

### 16. Missing Input Sanitization in Tool Handler

**Severity:** MEDIUM
**STATUS:** PRESENT

#### Issue Description
Tool arguments are cast to 'any' type without validation before being passed to handlers.

**Location:**
- `index.ts:118-127` - CallToolRequestSchema handler

#### Code Analysis
```typescript
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  await ensureAuth();
  const tool = tools.find((t) => t.name === request.params.name);
  if (!tool) {
    throw new Error("Tool not found");
  }

  const result = await tool.handler(request.params.arguments as any);
  return convertToolResponse(result);
});
```

#### Risk
- No validation that arguments match tool's inputSchema
- Type cast to 'any' bypasses TypeScript type safety
- Malformed arguments could cause unexpected behavior
- Tool handlers receive untrusted input directly
- Could enable injection attacks or cause crashes

#### Recommendations
1. Validate arguments against tool's inputSchema before invocation
2. Add runtime type checking for tool arguments
3. Consider using JSON schema validator (ajv, zod)
4. Sanitize and normalize inputs before passing to handlers
5. Add input validation to each tool handler as defense-in-depth

---

### 17. No Test Coverage for Security Functions

**Severity:** MEDIUM
**STATUS:** PRESENT

#### Issue Description
No security tests found in repository. Critical security functions lack automated testing.

**Location:**
- Repository root - no `test/` or `__tests__/` directory
- No `*.test.ts` or `*.spec.ts` files in source code

#### Missing Test Coverage
- Authentication flow testing
- Token refresh logic
- Input validation
- Error handling
- Path traversal prevention
- Credential file permissions
- Race condition scenarios
- OAuth error conditions

#### Risk
- Security regressions could be introduced undetected
- No verification of security controls
- Difficult to validate fixes
- No regression testing for vulnerabilities

#### Recommendations
1. Add security test suite
2. Implement tests for authentication edge cases
3. Add fuzzing tests for input validation
4. Test error handling paths
5. Add integration tests for OAuth flow
6. Test file permission setting
7. Add CI/CD security testing gates
8. Consider using security testing frameworks (OWASP ZAP, etc.)

---

### 18. Process Exit Without Cleanup

**Severity:** LOW
**STATUS:** PRESENT

#### Issue Description
Server exits process immediately on error without cleanup or graceful shutdown.

**Location:**
- `index.ts:129-145` - startServer function
- `index.ts:143` - process.exit(1) on error

#### Code Analysis
```typescript
async function startServer() {
  try {
    // ...
  } catch (error) {
    console.error("Error starting server:", error);
    process.exit(1);  // Immediate exit
  }
}
```

#### Risk
- No cleanup of token refresh interval
- Open file handles may not be closed
- In-progress OAuth flows terminated abruptly
- Credentials may not be saved if refresh was in progress
- No graceful disconnection from MCP transport

#### Recommendations
1. Implement graceful shutdown handler
2. Clear token refresh interval on exit
3. Add SIGTERM/SIGINT handlers
4. Ensure credentials saved before exit
5. Close file handles explicitly
6. Add shutdown timeout
7. Log shutdown events for debugging

---

### 19. Sensitive Data in Console Logs

**Severity:** MEDIUM
**STATUS:** PRESENT

#### Issue Description
Sensitive information is logged to console.error, which may be captured in logs or displayed to users.

**Location:**
- `auth.ts:21` - Logs credentials directory path
- `auth.ts:57` - Logs credentials path
- `auth.ts:60` - Logs keyfile path
- `auth.ts:70` - Logs credential scopes
- `auth.ts:90` - Logs credentials path
- `auth.ts:104` - Logs credential scopes
- `auth.ts:112-116` - Logs token expiry details including refresh token presence

#### Risk
- Credential file paths revealed in logs
- Attackers learn where to find credential files
- Scope information reveals permission levels
- Refresh token presence disclosed
- Log aggregation systems may capture sensitive paths
- Could aid reconnaissance for attacks

#### Recommendations
1. Reduce verbosity of credential-related logging
2. Log only non-sensitive summary information
3. Add log level controls (debug, info, warn, error)
4. Don't log file paths in production
5. Sanitize logs before output
6. Consider structured logging with sensitive field redaction
7. Document what information is logged

---

### 20. JSON.parse Without Try-Catch in Critical Path

**Severity:** LOW
**STATUS:** PRESENT

#### Issue Description
JSON.parse of credential file is within try-catch, but parsing errors not specifically handled.

**Location:**
- `auth.ts:102-137` - loadCredentialsQuietly function
- `auth.ts:103` - JSON.parse call

#### Code Analysis
```typescript
try {
  const savedCreds = JSON.parse(fs.readFileSync(credentialsPath, "utf-8"));
  // ... uses savedCreds
} catch (error) {
  console.error("Error loading credentials:", error);
  return null;
}
```

#### Risk
- Malformed JSON in credential file causes authentication failure
- Could be exploited by corrupting credential file
- Generic error handler doesn't distinguish parse errors from other errors
- No attempt to recover or recreate credential file
- User doesn't know if problem is corrupted file vs missing file

#### Recommendations
1. Add specific error handling for JSON parse errors
2. Detect and report corrupted credential files clearly
3. Consider backup/recovery mechanism for credentials
4. Validate credential file structure after parsing
5. Provide clear user guidance when credentials corrupted
6. Add credential file integrity checking

---

## Branch Analysis

### Recent Security-Relevant Commits

**Commit 0aa5f65** (greghunt/main - merged)
- Added `.env` file support
- Updated .gitignore to exclude .env
- **Security Impact:** POSITIVE - Improves credential management hygiene

**Commit d180e73** (main branch)
- Increased authentication timeout
- **Security Impact:** NEUTRAL - Usability improvement, minor security implication

### Branch Status
- Only `main` branch exists in origin
- No other branches contain security-specific fixes
- Recent PRs (#8, #1) have been merged to main

---

## Security Best Practices for Users

### Deployment Security Checklist

1. **Credential Protection**
   - [ ] Set `.gdrive-server-credentials.json` permissions to 600
   - [ ] Store `.env` file with restrictive permissions (600)
   - [ ] Never commit credentials to version control
   - [ ] Rotate OAuth credentials periodically

2. **Environment Setup**
   - [ ] Use dedicated Google Cloud project for this application
   - [ ] Restrict OAuth consent screen to internal users
   - [ ] Enable Google Cloud audit logging
   - [ ] Set GDRIVE_CREDS_DIR to secure location

3. **Operational Security**
   - [ ] Run server with least-privileged user account
   - [ ] Monitor authentication logs
   - [ ] Keep dependencies updated (`npm audit`)
   - [ ] Review Google API access logs regularly

4. **Network Security**
   - [ ] Run server on localhost only (default stdio transport)
   - [ ] Do not expose MCP server to network
   - [ ] Use firewall rules if network access required

---

## Recommended Security Enhancements

### High Priority
1. Implement secure credential storage (keychain/keyring)
2. Add comprehensive input validation
3. Fix dependency vulnerability (npm audit fix)
4. Add explicit file permission setting

### Medium Priority
5. Implement rate limiting
6. Improve error message sanitization
7. Add token refresh circuit breaker
8. Document security procedures

### Low Priority
9. Add security headers to responses
10. Implement request logging
11. Add health check endpoints
12. Create security test suite

---

## Vulnerability Reporting

If you discover a security vulnerability in this project:

1. **Do NOT** open a public GitHub issue
2. Contact the maintainer privately via GitHub Security Advisory
3. Provide detailed information:
   - Vulnerability description
   - Steps to reproduce
   - Potential impact
   - Suggested fix (if available)

**Response Time:** Maintainers aim to respond within 7 days.

---

## Compliance and Standards

### Applicable Standards
- OWASP Top 10 (Web Application Security)
- CWE/SANS Top 25 (Software Errors)
- Google OAuth 2.0 Security Best Practices
- NIST SP 800-63B (Digital Identity Guidelines)

### Compliance Notes
- This tool handles OAuth credentials (authentication secrets)
- May process sensitive user data from Google Drive
- Subject to Google API Terms of Service
- Users responsible for compliance with data protection regulations (GDPR, etc.)

---

## Security Testing

### Recommended Tests
1. **Static Analysis**
   - Run ESLint with security plugins
   - Use SonarQube or similar tools
   - Perform dependency scanning

2. **Dynamic Testing**
   - Fuzz test tool inputs
   - Test with malformed file IDs
   - Test query injection scenarios
   - Verify credential file permissions

3. **Manual Review**
   - Code review focused on auth flows
   - Review error handling paths
   - Check logging for sensitive data

---

## Summary of Findings by Severity

| Severity | Count | Issues |
|----------|-------|--------|
| **HIGH** | 1 | Plaintext credential storage (#1) |
| **MEDIUM** | 8 | Path traversal (#11), Input validation (#3), Token refresh race (#12), Unhandled auth state (#13), Tool argument validation (#16), Sensitive logging (#19), Environment variable exposure (#2), OAuth scope permissions (#9) |
| **LOW-MEDIUM** | 2 | No security tests (#17), Token refresh security (#8) |
| **LOW** | 9 | Dependency vulnerabilities (#5), Rate limiting (#6), Error disclosure (#4), Auth timeout (#7), Documentation (#10), Hardcoded index (#14), String splitting (#15), Process exit (#18), JSON.parse (#20) |
| **TOTAL** | **20** | |

---

## Changelog

| Date | Version | Changes |
|------|---------|---------|
| 2025-10-16 | 1.0 | Initial security assessment |
| 2025-10-16 | 1.1 | Deep review: Added 10 additional findings (#11-#20) covering path traversal, race conditions, authentication handling, logging, and testing gaps |

---

## References

- [Google OAuth 2.0 Security](https://developers.google.com/identity/protocols/oauth2/security)
- [OWASP Input Validation](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [MCP Security Considerations](https://modelcontextprotocol.io/docs/security)

---

**Document Owner:** Security Review Team
**Next Review:** 2026-04-16 (6 months)
