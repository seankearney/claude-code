# OWASP Top 10 & Security Best Practices

> Shared knowledge module. Referenced by agents that evaluate security.

## OWASP Top 10 (2021)

Evaluate ALL code against these categories. In .NET codebases, pay special attention to the detection signals listed.

### A01: Broken Access Control

**What:** Users acting outside their intended permissions.

| Detection Signal | .NET Example | Severity |
|---|---|---|
| Missing `[Authorize]` on controllers/actions | Public endpoint exposing admin data | Critical |
| IDOR (Insecure Direct Object Reference) | `GET /api/users/{id}` without ownership check | Critical |
| CORS misconfiguration | `AllowAnyOrigin()` with `AllowCredentials()` | High |
| Path traversal | `File.ReadAllText(userInput)` without sanitization | Critical |
| Missing role/policy checks | Action accessible to any authenticated user | High |
| Privilege escalation | User can modify their own role claim | Critical |

### A02: Cryptographic Failures

| Detection Signal | .NET Example | Severity |
|---|---|---|
| Hardcoded secrets | `var key = "MySecretKey123"` in source | Critical |
| Weak algorithms | `MD5.Create()`, `SHA1.Create()` for security purposes | High |
| Missing encryption at rest | Connection strings without `Encrypt=True` | High |
| Insufficient key management | Keys in `appsettings.json` committed to repo | Critical |
| Weak random number generation | `new Random()` for tokens/secrets (use `RandomNumberGenerator`) | High |

### A03: Injection

| Detection Signal | .NET Example | Severity |
|---|---|---|
| SQL string interpolation | `$"SELECT * FROM Users WHERE Id = {id}"` | Critical |
| String concatenation in SQL | `"SELECT * FROM Users WHERE Id = " + id` | Critical |
| Command injection | `Process.Start("cmd", userInput)` | Critical |
| LDAP injection | Unparameterized LDAP queries | High |
| XSS (reflected/stored) | `@Html.Raw(userInput)` in Razor | High |
| Missing parameterized queries | Not using `@param` in ADO.NET / Dapper | Critical |

### A04: Insecure Design

| Detection Signal | .NET Example | Severity |
|---|---|---|
| Missing rate limiting | No `[RateLimiter]` on authentication endpoints | High |
| Business logic flaws | Discount applied multiple times, negative quantity allowed | High |
| Missing input validation at trust boundaries | API accepts any payload without model validation | Medium |
| No threat modeling | No evidence of security design review | Medium |

### A05: Security Misconfiguration

| Detection Signal | .NET Example | Severity |
|---|---|---|
| Debug mode in production | `ASPNETCORE_ENVIRONMENT=Development` in prod config | High |
| Verbose error messages | `app.UseDeveloperExceptionPage()` in production | Medium |
| Default credentials | `sa`/`password` in connection strings | Critical |
| Missing security headers | No `X-Content-Type-Options`, `X-Frame-Options`, CSP | Medium |
| Unnecessary features | Swagger UI enabled in production | Low |
| `ServerCertificateValidationCallback = => true` | Bypasses ALL TLS validation globally | Critical |

### A06: Vulnerable & Outdated Components

| Detection Signal | .NET Example | Severity |
|---|---|---|
| Known CVEs in NuGet packages | `dotnet list package --vulnerable` reports hits | Varies |
| Unsupported framework versions | .NET Core 2.1, .NET 5 (both EOL) | High |
| Outdated packages (2+ years) | `Newtonsoft.Json` 9.x when 13.x is current | Medium |
| No package update process | No evidence of dependency review | Medium |

### A07: Identification & Authentication Failures

| Detection Signal | .NET Example | Severity |
|---|---|---|
| Weak password policies | No minimum length/complexity requirements | Medium |
| Missing MFA | No second factor for privileged operations | Medium |
| Session fixation | Not regenerating session after login | High |
| Credential stuffing vulnerability | No account lockout or rate limiting on login | High |
| Storing passwords in plain text | Not using `PasswordHasher<T>` or bcrypt | Critical |

### A08: Software & Data Integrity Failures

| Detection Signal | .NET Example | Severity |
|---|---|---|
| `BinaryFormatter` deserialization | `BinaryFormatter.Deserialize(stream)` — RCE risk | Critical |
| Unsafe deserialization | `JsonConvert.DeserializeObject` with `TypeNameHandling.All` | Critical |
| Missing CI/CD pipeline security | No code review gates, unsigned packages | Medium |
| Auto-update without verification | Downloading and executing without signature check | High |

### A09: Security Logging & Monitoring Failures

| Detection Signal | .NET Example | Severity |
|---|---|---|
| Missing audit trails | No logging of auth events, data changes | High |
| PII in logs | `_logger.LogInformation($"User {ssn} logged in")` | High |
| No alerting on suspicious activity | Failed logins not monitored | Medium |
| Insufficient log detail | Catching exceptions without logging them | Medium |
| Log injection | User input directly in log messages without structured logging | Medium |

### A10: Server-Side Request Forgery (SSRF)

| Detection Signal | .NET Example | Severity |
|---|---|---|
| Unvalidated URL inputs | `HttpClient.GetAsync(userProvidedUrl)` | High |
| Internal service access | User-controlled URL reaches internal APIs | Critical |
| Cloud metadata exposure | `http://169.254.169.254/` accessible via SSRF | Critical |
| Missing URL allowlisting | No validation of target host/scheme | High |

## .NET-Specific Security Checklist

- [ ] `[ValidateAntiForgeryToken]` on all POST/PUT/DELETE actions (MVC)
- [ ] `[Authorize]` attribute with specific policies, not just authentication
- [ ] `Data Protection API` for encryption, not hand-rolled crypto
- [ ] `IHttpClientFactory` instead of `new HttpClient()` (socket exhaustion + DNS)
- [ ] Connection strings in environment variables or Azure Key Vault, not in code
- [ ] `ModelState.IsValid` checked in all controller actions
- [ ] Response caching disabled for sensitive data (`[ResponseCache(NoStore = true)]`)
- [ ] HTTPS enforced (`UseHttpsRedirection`, HSTS)
