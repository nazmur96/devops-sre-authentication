# DevOps / SRE Authentication — Reference, Drill Book, and Troubleshooting Playbook

**Version 2** — reorganised, deduplicated, and expanded with commands, real configuration, failure modes, labs, and a self-test bank.

---

## How to use this document

This is three documents in one, and they are meant to be used differently:

| Layer | Parts | How to use it |
|---|---|---|
| **Reference** | I–XII | Read once end to end. Return to single sections when you hit the topic at work. |
| **Playbook** | XIII | Do *not* read cover to cover. Jump to the symptom when something is broken. |
| **Drill** | XIV–XVI | Labs, question bank, and the single master checklist. This is where the learning actually happens. |

### The drill loop

Reading this file will not make the knowledge automatic. For each topic, run the loop:

1. **Explain it out loud in 30 seconds** without looking. If you stall, you do not know it.
2. **Draw the flow** — who proves identity, to whom, with what, and what comes back.
3. **Run the command.** Every core topic in this document has at least one command you can run against a real system.
4. **Break it deliberately.** Delete the CA bundle, skew the clock, mismatch the audience. Read the error. That error is what you will see in production at 03:00, and recognising it instantly is worth more than any definition.

Steps 3 and 4 are the ones people skip, and they are the ones that separate "I have read about mTLS" from "I can fix mTLS".

### Priority markers

Every topic is tagged. Budget your time accordingly.

| Tag | Meaning |
|---|---|
| 🔴 | **Core.** You will meet this in the first month of any DevOps/SRE job. Must be instant recall. |
| 🟠 | **Conceptual.** Know what it is, how it fits, and how to recognise it. Depth on demand. |
| 🟢 | **On demand.** Learn only when a specific environment forces you to. |

### Conventions

- `$` prefixed lines are shell commands you can actually run.
- Configuration snippets are realistic and close to copy-pasteable, but always check them against current upstream docs (Appendix D) before shipping.
- Where a claim is version-sensitive (Kubernetes defaults, AWS behaviour), the version or date is stated. These move.

---

## Table of contents

**Layer 1 — Foundations**
- Part I — The shape of every authentication system

**Layer 2 — Shared secrets**
- Part II — Passwords, Basic, Digest
- Part III — API keys, sessions, cookies

**Layer 3 — Cryptographic identity**
- Part IV — Asymmetric cryptography and SSH
- Part V — PKI, X.509, and TLS
- Part VI — mTLS and certificate identity

**Layer 4 — Tokens and federation**
- Part VII — Tokens and JWT
- Part VIII — OAuth 2.0
- Part IX — OIDC, SSO, SAML, and legacy enterprise auth
- Part X — MFA for humans

**Layer 5 — Platform identity**
- Part XI — AWS IAM, STS, and federation
- Part XII — Kubernetes identity
- Part XIII — Workload identity as a pattern
- Part XIV — Vault and secrets management
- Part XV — The delivery pipeline (git, CI, GitOps, registries)
- Part XVI — Operating credentials

**Practice**
- Part XVII — Troubleshooting playbook
- Part XVIII — Hands-on labs
- Part XIX — Self-test question bank
- Part XX — Master drill checklist

**Appendices**
- A — Command reference
- B — Cheat sheet
- C — Confusable pairs
- D — Primary sources

---
# Layer 1 — Foundations

# Part I — The shape of every authentication system

## I.1 Authentication vs authorization 🔴

Two questions, always in this order, always separable:

| | Question | Produces | Fails with |
|---|---|---|---|
| **Authentication (AuthN)** | Who are you? | An identity / principal | `401 Unauthorized` |
| **Authorization (AuthZ)** | What may you do? | An allow / deny decision | `403 Forbidden` |

```text
Client ──credential──► AuthN ──identity──► AuthZ ──decision──► Resource
                         │                    │
                    "you are alice"   "alice may GET /users"
```

There is a third A worth knowing: **accounting/audit** — the record of what that identity actually did. Every system in this document produces audit events, and Part XVI covers where to find them.

**Why this distinction is the single most useful thing here:** almost every production auth incident is diagnosable by first deciding which half failed. A `401` means the system does not believe your credential. A `403` means it believes you and is refusing anyway. Those have completely disjoint fix paths, and people waste hours rotating credentials to fix authorization problems.

> **Caveat that will bite you:** many real APIs return `403` when the *token is bad*, and Kubernetes returns `403` with a message naming the user (`User "system:anonymous" cannot list ...`) that reveals it authenticated you as anonymous. Read the message body, not just the status code.

## I.2 The vocabulary 🔴

These words are used loosely in blog posts and precisely in specs. Use them precisely.

| Term | Precise meaning |
|---|---|
| **Identity** | Who or what something is. `alice@example.com`, `arn:aws:iam::123:role/deployer`, `system:serviceaccount:prod:api`. |
| **Principal** | The identity as the authorizing system names it. AWS and Kubernetes both use this word. |
| **Credential** | The thing presented to prove an identity. A password, key, certificate, or token. |
| **Secret** | A credential whose security depends entirely on it staying confidential. |
| **Key** | A cryptographic value. Symmetric (one shared) or asymmetric (public + private). |
| **Certificate** | A *signed statement* binding an identity to a public key. Public, not secret. |
| **Token** | A credential issued by an authority, usually time-limited, usually presented verbatim. |
| **Claim** | One assertion inside a token. `sub=alice`, `groups=[sre]`. |
| **Assertion** | A signed bundle of claims. SAML's word; a JWT is the JSON equivalent. |
| **Scope** | What a token is *permitted to request* — narrows the client's power (OAuth). |
| **Audience (`aud`)** | Who a token is *intended for*. A verifier must reject tokens addressed elsewhere. |
| **Policy** | The rule set consulted at authorization time. IAM policy, RBAC, Vault policy, OPA. |
| **Trust** | A configured decision to accept identities asserted by another system. |
| **Federation** | Trusting an external identity provider instead of holding credentials yourself. |

**Distinctions people get wrong constantly:**

- A **certificate is not a secret.** The *private key* is the secret. You can post your certificate publicly; that is the point of it.
- **Scope ≠ audience.** Scope limits the action; audience limits the recipient. A token with `scope=admin` and `aud=billing-api` is useless against the shipping API.
- **A key is not a certificate.** A certificate wraps a public key with a signed identity statement and validity period.

## I.3 The eight questions 🔴

This is the spine of the whole document. When you meet any authentication system — one in this file or one invented next year — answer these eight, in order. If you can, you understand the system.

**1. Who is proving identity?**
A human, an application, a Pod, a CI job, a physical machine, a network device.

**2. Who is verifying it?**
The resource itself, an identity provider, a sidecar/proxy, AWS STS, the Kubernetes API server, Vault.

**3. What proof is presented?**
Password, API key, X.509 certificate, SSH key signature, JWT, opaque token, Kerberos ticket, cloud instance identity document.

**4. Where did that proof come from, and what is the root of trust?**
Traced back far enough, every chain ends at something configured out-of-band: a CA in a trust store, a shared secret, a hardware root, an OIDC issuer URL you told the verifier to trust. **Find that root.** Most auth failures are a broken link in the chain, and most auth compromises are an over-broad root.

**5. Is this authentication or authorization?**
See I.1.

**6. Is the credential long-lived or short-lived, and is it bound to anything?**
Bearer credentials work for anyone holding them. Sender-constrained ones do not. See I.5.

**7. Is the channel protected?**
A bearer token over plaintext HTTP is a bearer token you have given away.

**8. Human or workload?**
This determines the entire correct design:

| | Human | Workload |
|---|---|---|
| Wants | SSO, one login, MFA | No login, no human, no secret at rest |
| Uses | OIDC / SAML / WebAuthn | IAM roles, ServiceAccounts, mTLS, SPIFFE |
| Session | Hours to days | Minutes to an hour |
| Recovery | Helpdesk, recovery codes | Redeploy |

Using a human mechanism for a workload gives you a service account with a password in a vault that nobody rotates. Using a workload mechanism for a human gives you a shared credential with no audit trail. Both are extremely common and both are wrong.

## I.4 The HTTP mechanics 🔴

Nearly all of this rides on a handful of HTTP primitives (RFC 7235).

```http
GET /api/v1/users HTTP/1.1
Host: api.example.com
Authorization: <scheme> <credentials>
```

Registered schemes you will see: `Basic`, `Bearer`, `Digest`, `Negotiate` (Kerberos/SPNEGO), `AWS4-HMAC-SHA256` (SigV4), `DPoP`.

The server advertises what it wants when it rejects you:

```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer realm="api", error="invalid_token",
                  error_description="token is expired"
```

**Read `WWW-Authenticate`.** It usually tells you exactly what went wrong, and almost nobody looks at it.

```bash
$ curl -si https://api.example.com/v1/users | grep -i 'www-authenticate\|^HTTP'
```

| Status | Meaning | First thing to check |
|---|---|---|
| `401` | Not authenticated, or credential rejected | Is a credential being sent at all? Is it expired? |
| `403` | Authenticated, not permitted | Which identity did it resolve to? What policy denied it? |
| `407` | Proxy demands authentication | `Proxy-Authorization`, corporate egress proxy |
| `421` | Misdirected request | TLS/SNI routing mismatch |

Non-header credential carriers you will also meet: cookies (browsers), `X-API-Key` and similar custom headers, query parameters (avoid — they land in access logs, referrers, and shell history), and client certificates (TLS layer, not HTTP at all).

## I.5 The credential lifetime spectrum 🔴

The strongest single design principle in modern infrastructure security:

> **Prefer a short-lived credential derived from a verifiable identity over a long-lived secret that must be stored.**

| Credential | Typical lifetime | Revocable? | Notes |
|---|---|---|---|
| Password | Months–years | Yes (change it) | Human only. Never for workloads. |
| API key | Until rotated (often never) | Yes, at the issuer | Static secret. Highest leak risk. |
| Long-lived cloud access key | Until deleted | Yes | Avoid. The classic breach vector. |
| SSH key | Years | Only by editing `authorized_keys` everywhere | Distribution and revocation are the hard parts. |
| TLS server certificate | 47–398 days (shrinking) | CRL/OCSP, poorly | Automate renewal or you will page yourself. |
| SSH certificate | Minutes–hours | Expiry | Massively better than raw keys at scale. |
| OAuth access token | 5 min–1 hour | Sometimes | Short by design. |
| Refresh token | Days–months | Yes | Rotate on use; detect reuse. |
| AWS STS credentials | 15 min–12 h | Session policy / role deletion | The default for everything in AWS. |
| K8s bound SA token | 1 hour default, auto-rotated | Delete the SA/Pod | Since 1.21+ projected tokens. |
| Vault token | TTL, renewable to max TTL | Yes, immediately | Best revocation story of anything here. |
| CI OIDC token | Single job, minutes | N/A | Cannot leak usefully after the job ends. |

**Bearer vs sender-constrained.** A bearer credential is usable by whoever holds it — steal it, use it. A sender-constrained credential is cryptographically tied to its holder, so theft alone is not enough:

- **mTLS-bound tokens** (RFC 8705) — token contains a hash of the client certificate.
- **DPoP** (RFC 9449) — client proves possession of a key on each request.
- **SSH keys, client certificates, WebAuthn** — inherently sender-constrained; the private key never leaves.

Most of what you handle daily (JWTs, API keys, Vault tokens, `kubeconfig` tokens) is bearer. Treat every bearer credential as equivalent to the identity itself.

---
# Layer 2 — Shared secrets

These are the mechanisms where both sides know the same thing. They are the oldest, simplest, and weakest family — but they are everywhere, and a lot of internal tooling still runs on them.

# Part II — Passwords, Basic, Digest

## II.1 Password storage 🔴

Servers store a **hash**, never the password. What matters is *which* hash.

```text
password + salt ──► slow KDF ──► stored: algorithm, params, salt, hash
```

| Algorithm | Verdict |
|---|---|
| **Argon2id** | Current first choice. Memory-hard, tunable memory/time/parallelism. |
| **scrypt** | Good. Memory-hard. |
| **bcrypt** | Acceptable and everywhere. Cost factor 12+. Note the 72-byte input truncation. |
| **PBKDF2** | Acceptable when FIPS compliance requires it. High iteration count. |
| SHA-256, SHA-512, MD5, plain | **Wrong.** Fast hashes are designed for speed; that is exactly what an attacker wants. |

- **Salt** — unique random value per password, stored alongside the hash. Defeats rainbow tables and makes identical passwords hash differently. Not secret.
- **Pepper** — an additional secret held outside the database (in an HSM or app config). Optional; protects against a database-only compromise.
- **Hashing is not encryption.** A hash is one-way and has no key that can reverse it. If a system can email you your existing password, it is not hashing.

As an SRE you rarely implement this, but you will be asked to review it, and you will inherit databases where you need to know whether a leak was catastrophic (MD5, unsalted) or merely bad (bcrypt cost 12).

## II.2 Basic authentication 🔴

RFC 7617. The simplest HTTP scheme.

```text
"alice:secret123" ──base64──► YWxpY2U6c2VjcmV0MTIz
```

```http
Authorization: Basic YWxpY2U6c2VjcmV0MTIz
```

**Base64 is an encoding, not encryption.** Prove it to yourself — this is worth typing once so you never forget:

```bash
$ printf 'alice:secret123' | base64
YWxpY2U6c2VjcmV0MTIz
$ echo 'YWxpY2U6c2VjcmV0MTIz' | base64 -d
alice:secret123
```

Consequences: Basic Auth over HTTP transmits the password in effectively cleartext, on **every single request** (there is no session — the credential is replayed constantly). Over HTTPS it is acceptable for machine-to-machine internal use.

```bash
# curl builds the header for you
$ curl -u admin:password https://registry.example.com/v2/

# better: never put a secret in your shell history
$ curl -u admin https://registry.example.com/v2/   # prompts for password
$ curl -H "Authorization: Basic $(printf '%s' "$USER:$PASS" | base64)" https://...
```

**Where you still meet it in DevOps:** container registries, Prometheus scrape configs, Alertmanager receivers, nginx `auth_basic`, Elasticsearch, RabbitMQ management, internal admin UIs, `.netrc` files, and countless appliance web interfaces.

Generating credentials for nginx or a registry:

```bash
$ htpasswd -Bbn admin 'strong-password' > htpasswd   # -B = bcrypt
```

```nginx
location /metrics {
    auth_basic           "metrics";
    auth_basic_user_file /etc/nginx/htpasswd;
}
```

## II.3 Digest authentication 🟠

RFC 7616. A challenge-response scheme designed so the password is not sent over the wire.

```text
Client  → Server:  GET /resource
Server  → Client:  401, WWW-Authenticate: Digest realm=..., nonce=..., qop=auth
Client  → Server:  Authorization: Digest response=H(H(user:realm:pass):nonce:...:H(method:uri))
Server:            computes the same hash and compares
```

The server still needs the password (or `H(user:realm:pass)`) stored in recoverable form, which is a significant weakness. MD5 is the classic algorithm; SHA-256 variants exist and are rarely deployed.

**Verdict:** obsolete for new work — HTTPS plus a token beats it on every axis. Know it exists because you will meet it in IPMI/BMC interfaces, old routers, some SIP/VoIP gear, and legacy intranet apps. Do not invest time.

---

# Part III — API keys, sessions, cookies

## III.1 API keys 🔴

An API key is a static shared secret that identifies a **calling application**, not usually a human.

```http
X-API-Key: sk_live_a1b2c3...
Authorization: ApiKey a1b2c3...
Authorization: Bearer a1b2c3...
```

There is no standard. Every vendor picks a header and a format; read their docs.

**API key vs access token — the distinction that matters:**

| | API key | OAuth access token |
|---|---|---|
| Issued by | The service, once, to a client | An authorization server, per session/job |
| Lifetime | Until manually rotated | Minutes to an hour |
| Represents | The application | Delegated permission, often on behalf of a user |
| Scoping | Coarse, sometimes none | Explicit scopes and audience |
| Leak impact | Persistent until noticed and rotated | Bounded by expiry |

**Operational rules — these are the exam questions and the real-life failures:**

1. **Never commit keys.** Enable push protection and run secret scanning (`gitleaks`, `trufflehog`, GitHub secret scanning). Assume that once a key touches a git history it is compromised, even in a private repo, even after a force-push.
2. **Never bake keys into container images.** They persist in layers forever. `docker history` and `dive` will show them.
3. **Store in a secrets manager** (Vault, AWS Secrets Manager, SOPS-encrypted files), inject at runtime.
4. **Scope down.** Read-only where possible. Per-environment keys, never one key across dev and prod.
5. **Restrict by source** where the provider supports it — IP allowlist, referrer, or bound to a specific resource.
6. **Rotate on a schedule and on every departure/incident**, using the two-key overlap pattern in XVI.1.
7. **Log the key ID, never the key.** Keys with a non-secret prefix (`sk_live_`, `AKIA...`) are designed for this.

## III.2 Sessions 🔴

The server remembers that you logged in.

```text
POST /login (user+pass)
   → server verifies, creates session record, session_id = abc789
   ← Set-Cookie: session=abc789; Secure; HttpOnly; SameSite=Lax
Subsequent requests carry the cookie; server looks up abc789 → alice
```

The defining property: **state lives on the server side.** That gives you instant, reliable revocation — delete the session row and the user is logged out everywhere, immediately. That is the thing stateless JWTs give up.

Where the session store lives is an SRE concern:

| Store | Consequence |
|---|---|
| In process memory | Breaks the moment you scale to two replicas or restart a Pod |
| Sticky sessions on the LB | Works, but wrecks rolling deploys and load distribution |
| Shared Redis/Memcached | The normal answer. Now Redis is a hard dependency of login. |
| Database | Simple, slower, survives restarts |
| Signed cookie (client-side state) | No server store — but now you have the JWT revocation problem |

You will be paged for this: "nobody can log in" is very often "the session Redis failed over".

## III.3 Cookies 🔴

A cookie is a transport and storage mechanism, not an authentication protocol. It commonly carries a session ID, and sometimes a token.

```http
Set-Cookie: session=abc789; Secure; HttpOnly; SameSite=Lax; Path=/; Max-Age=3600
```

| Attribute | What it does | Why you care |
|---|---|---|
| `Secure` | HTTPS only | Without it the cookie leaks over any plaintext request |
| `HttpOnly` | Not readable by JavaScript | Blunts XSS session theft |
| `SameSite=Lax\|Strict\|None` | Controls cross-site sending | CSRF defence; `None` **requires** `Secure` |
| `Domain` | Scope across subdomains | Over-broad `Domain` shares sessions with every subdomain — including that forgotten one |
| `Path` | Path scope | Weak isolation, rarely load-bearing |
| `Max-Age` / `Expires` | Client-side lifetime | Absent = session cookie, dies with the browser |
| `__Host-` prefix | Enforces `Secure`, `Path=/`, no `Domain` | Strongest binding; use for session cookies |

**Session fixation** — always issue a *new* session ID on privilege change (login, sudo-to-admin). Reusing the pre-login ID lets an attacker who planted it ride the authenticated session.

**Infrastructure gotchas that land on your desk:**
- A reverse proxy that rewrites or drops `Set-Cookie` on redirect.
- TLS terminated at the LB, so the app thinks it is on HTTP and never sets `Secure`. Fix with `X-Forwarded-Proto` handling, not by dropping `Secure`.
- Caching layers (CDN, Varnish) caching a response that carries `Set-Cookie` — this serves one user's session to everyone. Vary and no-store on authenticated responses.
- Cookie size limits (~4 KB) breaking when SAML/OIDC claims get stuffed into the session cookie.

---
# Layer 3 — Cryptographic identity

Everything from here on rests on one idea: prove you hold a private key, without ever revealing it.

# Part IV — Asymmetric cryptography and SSH

## IV.1 Asymmetric crypto in one page 🔴

A **key pair**: a private key you keep, and a public key you distribute freely.

| Operation | Who uses which key | Gives you |
|---|---|---|
| **Sign** | Sign with private, verify with public | Authenticity + integrity |
| **Encrypt** | Encrypt with public, decrypt with private | Confidentiality (rarely used directly — slow) |
| **Key agreement** (ECDHE) | Both sides contribute, derive a shared secret | The session key for TLS, with forward secrecy |

Authentication almost always uses **signing**, not encryption:

```text
Verifier → "sign this random challenge"
Holder   → signature (made with the private key)
Verifier → verifies with the public key it already trusts
```

The private key never moves. That is why key-based auth beats password auth: there is nothing on the wire to steal, replay, or phish.

| Algorithm | Use | Notes |
|---|---|---|
| **Ed25519** | SSH, modern signing | Default choice. Small, fast, no parameter footguns. |
| **ECDSA P-256** | TLS certificates | Widely supported. |
| **RSA 3072/4096** | Legacy compatibility | Big and slow; 2048 is the floor, avoid new 1024. |

The whole trust problem reduces to one question: **how does the verifier come to trust that public key?** Three answers, and they define the rest of this layer:

1. **You put it there manually** — `authorized_keys`, `known_hosts`, a pinned key.
2. **A CA vouched for it** — X.509 certificates, SSH certificates.
3. **An issuer publishes it** — JWKS endpoints for token signing keys.

## IV.2 SSH key authentication 🔴

```bash
$ ssh-keygen -t ed25519 -C "nazmur@laptop-2026"
# → ~/.ssh/id_ed25519      (private — mode 600, never leaves this machine)
# → ~/.ssh/id_ed25519.pub  (public — copy anywhere)
```

The public key goes into `~/.ssh/authorized_keys` on the server:

```bash
$ ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server
```

Authentication flow: the client offers a public key, the server checks it appears in `authorized_keys`, the server sends a challenge, the client signs it with the private key, the server verifies. **The private key is never transmitted.**

Always protect the private key with a passphrase, and use an agent so you type it once:

```bash
$ eval "$(ssh-agent -s)"
$ ssh-add ~/.ssh/id_ed25519
$ ssh-add -l            # what identities are loaded
$ ssh-add -D            # drop everything
$ ssh-add -t 3600 ~/.ssh/id_ed25519   # auto-expire after an hour
```

### Host key verification 🔴

The other half, and the half everyone ignores.

```text
SSH user authentication  → proves who the client is       (authorized_keys)
SSH host key verification → proves which server you reached (known_hosts)
```

```text
The authenticity of host 'server (10.0.0.5)' can't be established.
ED25519 key fingerprint is SHA256:abc123...
```

That prompt is a genuine security decision, not a nuisance. Typing `yes` blindly is trust-on-first-use with no verification, and `StrictHostKeyChecking=no` in a script disables the only defence against a man-in-the-middle.

Do it properly:

```bash
# Get the real fingerprint out of band (console, cloud metadata, config management)
$ ssh-keyscan -t ed25519 server.example.com >> ~/.ssh/known_hosts
$ ssh-keygen -lf ~/.ssh/known_hosts          # list fingerprints
$ ssh-keygen -R server.example.com           # remove after a legitimate rebuild
```

At scale, distribute `known_hosts` (or an `@cert-authority` line) via configuration management. In ephemeral CI, use `ssh-keyscan` against a pinned, verified value rather than disabling checking.

### `authorized_keys` restrictions 🟠

The file supports per-key options, which turn a general shell key into a narrow capability:

```text
command="/usr/local/bin/backup-only",from="10.0.0.0/8",no-port-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAAC3... backup@ci
```

### SSH certificates 🟠 — the scaling answer

Raw keys do not scale: adding a person means editing `authorized_keys` on every host, and removing a person means the same — which is why leavers' keys survive for years.

With an SSH CA, the server trusts one CA public key, and users present **short-lived certificates**:

```bash
# server side, /etc/ssh/sshd_config
TrustedUserCAKeys /etc/ssh/ca_user_key.pub

# sign a user key, valid 8 hours, with principals
$ ssh-keygen -s ca_user_key -I "nazmur@2026-08-30" -n ubuntu,deploy -V +8h id_ed25519.pub
$ ssh-keygen -Lf id_ed25519-cert.pub    # inspect a certificate
```

Revocation becomes "stop issuing", and offboarding becomes instant. Real implementations: HashiCorp Vault's SSH secrets engine, Teleport, smallstep `step-ssh`, Netflix BLESS. Also sign **host** keys (`HostCertificate`) to kill the TOFU prompt entirely.

### Hardening checklist 🔴

```text
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
KbdInteractiveAuthentication no
AllowGroups ssh-users
```

- Reach private hosts with `ProxyJump` (`ssh -J bastion target`), **not** agent forwarding — a compromised bastion with your forwarded agent can authenticate as you everywhere.
- Audit `authorized_keys` files as config, in git, not by hand.
- `ssh -vvv` is the debugging tool; it shows exactly which key was offered and why it was rejected.

---

# Part V — PKI, X.509, and TLS

## V.1 What PKI is 🔴

PKI (Public Key Infrastructure) is the system that answers "why should I trust this public key?" with "because someone I already trust signed a statement about it".

```text
Root CA          (self-signed; its public key is in your trust store)
   │ signs
Intermediate CA  (the one doing daily issuance; the root stays offline)
   │ signs
Leaf certificate (api.example.com, or a client identity)
```

The **trust store** is the root of trust. On Linux:

```bash
/etc/ssl/certs/ca-certificates.crt      # Debian/Ubuntu bundle
/etc/pki/tls/certs/ca-bundle.crt        # RHEL/Fedora
$ update-ca-certificates                # Debian: after dropping a CA in /usr/local/share/ca-certificates
$ update-ca-trust extract               # RHEL: after /etc/pki/ca-trust/source/anchors/
```

Language runtimes have their own stores (Java `cacerts`, Python `certifi`, Node's bundled list). "It works with curl but not with the app" is nearly always this.

## V.2 X.509 certificate contents 🔴

| Field | Meaning |
|---|---|
| **Subject** | Who this certificate is about (`CN=api.example.com`) |
| **Issuer** | Which CA signed it |
| **SAN** | Subject Alternative Name — **the field actually used for hostname matching** |
| **Validity** | `notBefore` / `notAfter` |
| **Public key** | The key being bound |
| **Key Usage / EKU** | What it may be used for — `serverAuth`, `clientAuth`, `codeSigning` |
| **Serial** | Unique per issuer; how revocation identifies it |
| **AKI / SKI** | Authority/Subject Key Identifier — used to build the chain |
| **Signature** | The CA's signature over all of the above |

> **CN is dead for hostname matching.** Browsers and modern clients have required SAN for years. A certificate with only a CN will fail with `x509: certificate relies on legacy Common Name field`. If you generate certs by hand, you must set SAN.

Reading certificates — memorise these three:

```bash
# a live endpoint
$ openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -text

# just the important bits
$ openssl s_client -connect api.example.com:443 -servername api.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName

# a file on disk
$ openssl x509 -in cert.pem -noout -text
$ openssl req  -in req.csr  -noout -text        # a CSR
$ openssl rsa  -in key.pem  -noout -check       # a private key
```

**Do the key and certificate match?** (Answers the classic "key values mismatch" error.)

```bash
$ openssl x509 -noout -pubkey -in cert.pem | openssl sha256
$ openssl pkey -pubout   -in  key.pem | openssl sha256
# the two hashes must be identical
```

## V.3 Issuance and revocation 🟠

```text
generate key pair → build CSR (subject + SAN + public key, self-signed proof of possession)
   → CA validates the request → CA signs → leaf certificate → deploy with the chain
```

```bash
$ openssl req -new -newkey rsa:2048 -nodes -keyout key.pem -out req.csr \
    -subj "/CN=api.example.com" \
    -addext "subjectAltName=DNS:api.example.com,DNS:www.api.example.com"
```

**Serve the full chain.** A leaf without its intermediate works in browsers (which cache intermediates or use AIA fetching) and fails in Go, Java, and curl. This is one of the most common "works on my machine" TLS bugs. `openssl s_client` prints the chain the server actually sent — compare it against what you intended.

Revocation, honestly:

| Mechanism | Reality |
|---|---|
| **CRL** | Lists of revoked serials. Large, cached, often stale. |
| **OCSP** | Live query per certificate. Privacy and availability problems; the public web is moving away from it (Let's Encrypt has been retiring OCSP responder support in favour of CRLs). |
| **OCSP stapling** | Server attaches a fresh signed status. Better, still optional. |
| **Short lifetimes** | The actual answer. A 24-hour certificate barely needs revocation. |

The industry direction is unambiguous: **shorter certificates, automated issuance**. Public TLS maximum lifetimes are on a stepped path down from 398 days toward roughly 47 days by 2029 under the CA/Browser Forum ballot. If your renewal process involves a human and a calendar reminder, it is already broken.

## V.4 TLS 🔴

```text
HTTPS = HTTP over TLS
```

What TLS gives you:
- **Confidentiality** — encrypted in transit
- **Integrity** — tampering is detected
- **Server authentication** — the client verifies the server's certificate

What TLS does **not** give you: any statement about *who the client is* (unless you add mTLS), and no application-level authorization at all. TLS protects the pipe; the token in the header identifies the caller.

### The handshake 🔴

TLS 1.3 (RFC 8446), simplified:

```text
Client → ClientHello    (versions, cipher suites, key share, SNI, ALPN)
Server → ServerHello    (chosen suite, key share)
       → {Certificate, CertificateVerify, Finished}     ← encrypted from here
Client → {Finished}
       → application data                                (1 round trip)
```

TLS 1.2 needs an extra round trip and permits non-forward-secret and legacy suites. **Use 1.3 where you can, 1.2 as the floor, disable 1.0/1.1.**

Extensions that cause real outages:
- **SNI** — the hostname sent in the clear so a server hosting many sites picks the right certificate. Omit it and you get the default certificate and a name mismatch. This is why `curl --resolve` and `-servername` exist.
- **ALPN** — negotiates HTTP/2 or HTTP/1.1. gRPC failures behind old load balancers are frequently ALPN.
- **Session resumption / 0-RTT** — faster reconnects; 0-RTT data is replayable, so never use it for non-idempotent requests.

### Termination patterns 🔴

```text
Passthrough      client ══TLS══════════════════════► pod        (mTLS end to end; LB sees nothing)
Termination      client ══TLS══► LB ──plaintext──►   pod        (easy; plaintext inside the network)
Re-encryption    client ══TLS══► LB ══TLS══════════► pod        (usual production answer)
```

Whoever terminates holds the private key and sees the plaintext. That decision determines where your certificates live, who can read traffic, and what your service mesh can enforce.

### Debugging TLS 🔴

```bash
$ curl -v https://api.example.com                     # handshake + cert summary
$ curl -vI --tlsv1.3 https://api.example.com
$ openssl s_client -connect host:443 -servername host -showcerts   # full chain
$ openssl s_client -connect host:443 -tls1_2                       # force a version
$ echo | openssl s_client -connect host:443 2>/dev/null | openssl x509 -noout -dates
$ nmap --script ssl-enum-ciphers -p 443 host          # what is actually offered
```

Certificate expiry monitoring is not optional. A blackbox exporter probe on `probe_ssl_earliest_cert_expiry` with an alert at 21 days will save you more incidents than most of what you build.

---

# Part VI — mTLS and certificate identity

## VI.1 What mTLS adds 🔴

Ordinary TLS authenticates one direction:

```text
Client ────────► Server        client verifies the server's certificate
```

Mutual TLS authenticates both:

```text
Client ◄───────► Server        each verifies the other's certificate
  │                  │
  └ client cert      └ server cert
```

Mechanically, the server sends a `CertificateRequest` during the handshake, the client responds with its certificate plus a `CertificateVerify` signature proving it holds the matching private key.

**The client's identity comes out of the certificate** — from the SAN URI (SPIFFE ID), a DNS SAN, or the Subject CN. The server then authorizes on that. This is why mTLS is authentication *and* the input to authorization, unlike plain TLS.

```bash
$ curl --cert client.crt --key client.key --cacert ca.crt https://api.internal:8443/
$ openssl s_client -connect api.internal:8443 -cert client.crt -key client.key -CAfile ca.crt
```

nginx side:

```nginx
ssl_client_certificate /etc/nginx/ca.crt;
ssl_verify_client on;
ssl_verify_depth 2;
# pass the identity to the app
proxy_set_header X-Client-DN $ssl_client_s_dn;
```

> If your app trusts a header like `X-Client-DN`, the proxy **must** strip that header from inbound requests. Otherwise anyone can set it themselves. This is a real and recurring vulnerability class.

## VI.2 Where you meet mTLS 🔴

| System | Use |
|---|---|
| **Kubernetes internals** | kubelet ↔ API server, etcd peers and clients — all mTLS by default |
| **Service meshes** | Istio, Linkerd, Consul Connect — automatic mTLS between sidecars |
| **Envoy** | The data plane doing the handshake in most meshes |
| **Kafka, etcd, CockroachDB** | Client certificate authentication |
| **Bank/partner APIs** | Regulated B2B integration |
| **Zero-trust networking** | Identity per workload instead of trust per network location |

## VI.3 The hard part: rotation 🔴

mTLS is conceptually simple and operationally hard, because now *every workload* needs a certificate, and certificates expire. Doing this by hand does not survive contact with a real cluster.

The answers:

- **cert-manager** (Kubernetes) — `Issuer`/`ClusterIssuer` + `Certificate` CRs, renews into a Secret automatically. Backends: ACME (Let's Encrypt), Vault, a private CA, self-signed.
- **SPIFFE/SPIRE** — a standard identity format plus an agent that attests workloads and issues very short-lived SVIDs.
- **Service mesh** — Istio's istiod issues workload certificates (default 24 h, rotated automatically); you mostly do not touch them.
- **Vault PKI secrets engine** — dynamic certificate issuance with TTLs.

Minimal cert-manager example:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-tls
  namespace: prod
spec:
  secretName: api-tls          # cert-manager writes tls.crt / tls.key here
  duration: 2160h              # 90d
  renewBefore: 360h            # 15d
  dnsNames:
    - api.example.com
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

ACME challenge choice matters: **HTTP-01** needs port 80 reachable from the internet and cannot do wildcards; **DNS-01** needs API credentials for your DNS provider and can do wildcards and internal-only hosts.

## VI.4 SPIFFE and SVIDs 🟠

SPIFFE standardises the question "what is a workload's name?".

```text
spiffe://trust-domain/ns/prod/sa/payments-api
        └ trust domain ┘└─── path identifying the workload ───┘
```

- **SVID** — the credential carrying that ID, either an X.509 certificate (SAN URI = the SPIFFE ID) or a JWT.
- **Workload API** — a local Unix socket the workload calls to get its SVID. No secret is provisioned; the agent *attests* the workload from kernel/platform facts (which Pod, which node, which SA).
- **SPIRE** — the reference implementation. Istio uses SPIFFE IDs natively.

This is the cleanest existing answer to the "secret zero" problem (XIII.2) and it is worth understanding conceptually even if you never run SPIRE.

---
# Layer 4 — Tokens and federation

# Part VII — Tokens and JWT

## VII.1 Opaque vs self-contained 🔴

Every token is one of two kinds, and the trade-off between them explains most token design arguments.

| | **Opaque (reference)** | **Self-contained (JWT)** |
|---|---|---|
| Content | A random string; meaningless by itself | Claims, readable by anyone holding it |
| Validation | Call the issuer (introspection, RFC 7662) | Verify the signature locally |
| Revocation | **Immediate** — delete it at the issuer | **Hard** — valid until `exp` unless you build a denylist |
| Scale | Every request hits the issuer | No network call; scales flat |
| Size | Tiny | Hundreds of bytes to a few KB |
| Leaks data? | No | Yes — the payload is *encoded*, not encrypted |

Vault tokens, GitHub PATs, and classic session IDs are opaque. OIDC ID tokens, Kubernetes ServiceAccount tokens, and most cloud OIDC tokens are JWTs.

**The revocation problem is the thing to internalise**: once a JWT is issued, it is valid until it expires. There is no "log this token out" unless you add a denylist — which reintroduces the central lookup you used JWTs to avoid. This is why access tokens are short: expiry *is* the revocation mechanism.

## VII.2 JWT anatomy 🔴

RFC 7519. Three base64url segments separated by dots:

```text
eyJhbGciOiJSUzI1NiIsImtpZCI6ImFiYzEifQ . eyJzdWIiOiJhbGljZSIsImV4cCI6MTc4MH0 . MEUCIQD...
└──────────── header ────────────────┘   └────────── payload ──────────────┘   └ signature ┘
```

Decode one right now — this is a required muscle memory:

```bash
# payload only, with padding fixed up
$ echo "$TOKEN" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq .

# header (tells you the algorithm and key id)
$ echo "$TOKEN" | cut -d. -f1 | tr '_-' '/+' | base64 -d 2>/dev/null | jq .

# or, if available
$ jwt decode "$TOKEN"
```

**Header:**
```json
{ "alg": "RS256", "typ": "JWT", "kid": "abc1" }
```
`kid` tells the verifier which public key from the JWKS to use — essential for key rotation.

**Payload — registered claims:**

| Claim | Meaning | Verification rule |
|---|---|---|
| `iss` | Issuer | Must exactly equal the expected issuer |
| `sub` | Subject — the identity | The thing you authorize on |
| `aud` | Audience | Must contain *your* identifier. Reject otherwise. |
| `exp` | Expires at (epoch seconds) | Reject if past, allow small clock leeway |
| `nbf` | Not before | Reject if future |
| `iat` | Issued at | Useful for max-age policies |
| `jti` | JWT ID | Unique — enables replay detection and denylists |

Plus whatever custom claims the issuer adds: `email`, `groups`, `roles`, `repository`, `namespace`.

## VII.3 Signing and verification 🔴

| `alg` | Type | Where used |
|---|---|---|
| `HS256` | Symmetric HMAC | Same secret signs and verifies. Only when one party does both. |
| `RS256` | RSA signature | The default for OIDC providers. Verifiers need only the public key. |
| `ES256` | ECDSA P-256 | Smaller, faster. |
| `EdDSA` | Ed25519 | Modern, increasingly supported. |
| `none` | **Nothing** | Never accept. See below. |

Asymmetric signing is what makes federation work: the issuer publishes public keys, and any number of independent verifiers validate tokens without holding a shared secret.

**A verifier must check all of this, in order:**

1. The signature is valid.
2. The algorithm is one you **explicitly allow** (never take `alg` from the token itself).
3. `iss` matches exactly.
4. `aud` contains your identifier.
5. `exp` has not passed; `nbf` has.
6. `kid` resolved to a key from the issuer's current JWKS.
7. Any application claims (`groups`, `scope`) are as expected.

**Attacks this prevents** (RFC 8725, JWT Best Current Practices):
- **`alg: none`** — a library that honours it accepts an unsigned token. Allowlist algorithms.
- **Algorithm confusion** — an attacker changes `RS256` to `HS256` and signs with the *public* key as the HMAC secret. A naive verifier that "uses the configured key" will accept it. Allowlist algorithms.
- **`kid` injection** — path traversal or SQL in `kid`. Treat it as untrusted input.
- **Skipping `aud`** — a token issued for service A replayed against service B. Extremely common in internal microservice setups.

**JWKS** — the issuer's public key set, published at a well-known URL:

```bash
$ curl -s https://accounts.google.com/.well-known/openid-configuration | jq -r .jwks_uri
$ curl -s "$(curl -s https://token.actions.githubusercontent.com/.well-known/openid-configuration | jq -r .jwks_uri)" | jq '.keys[].kid'
```

Verifiers cache JWKS. During a key rotation the issuer publishes both keys for an overlap period; a verifier with an over-long cache and no refresh-on-unknown-`kid` will fail hard the moment the issuer rotates. That is a classic outage.

**JWS vs JWE:** what you handle daily is JWS — *signed*, readable by anyone. JWE is *encrypted*. **Never put a secret in a JWT payload**; base64 is not a wall.

## VII.4 Bearer tokens 🔴

```http
Authorization: Bearer eyJhbGciOi...
```

"Bearer" (RFC 6750) is a statement about *semantics*, not format: whoever bears the token may use it. Two orthogonal axes people conflate:

```text
Bearer  = how the token is presented and what possession implies
JWT     = what format the token happens to be in
```

Both of these are bearer tokens; only the second is a JWT:

```http
Authorization: Bearer hvs.CAESIJ...        # opaque Vault token
Authorization: Bearer eyJhbGciOiJSUzI1...  # JWT
```

Rules: HTTPS always; never in a URL query string; never in logs (redact `Authorization` in every proxy and log pipeline); short lifetimes; and for high-value APIs, consider sender-constrained tokens (DPoP, mTLS-bound) so theft alone is insufficient.

## VII.5 Access, refresh, and ID tokens 🔴

Three tokens, three jobs. Confusing them is the single most common OAuth/OIDC mistake.

| | **Access token** | **Refresh token** | **ID token** |
|---|---|---|---|
| Answers | "May this client do X?" | "Give me a new access token" | "Who logged in?" |
| Sent to | The **resource server** (API) | The **authorization server** only | Nobody — consumed by the client |
| Lifetime | Minutes–1 hour | Days–months | Minutes (a login receipt) |
| Format | Opaque or JWT | Usually opaque | **Always** a JWT |
| Audience | The API | The auth server | The client application |

```text
access token expired
      ↓
client → POST /token  grant_type=refresh_token
      ↓
authorization server → new access token (+ usually a new refresh token)
```

**Never use an ID token as an API access token.** Its audience is the client, not the API, and a correctly written API will reject it. Systems that accept it are misconfigured, and you will eventually be the one who has to explain why.

**Refresh token rotation:** issue a new refresh token on every use and invalidate the old one. If an old one is presented again, that means it was stolen and replayed — revoke the entire family and force re-authentication. This is now standard practice (RFC 9700).

---

# Part VIII — OAuth 2.0

## VIII.1 What OAuth actually solves 🔴

OAuth 2.0 (RFC 6749) is a **delegated authorization** framework. The problem it solves:

> "I want this application to read my Google Drive files — without giving it my Google password."

```text
You ──"yes, allow it"──► Google ──access token (scope=drive.readonly)──► Application
                                                                             │
                                                             App calls Drive API with the token
```

Properties that matter: the app never sees your password; the grant is **scoped** (only Drive, only read); it is **revocable** without changing your password; and it **expires**.

**OAuth is not an authentication protocol.** It tells an API what a client is allowed to do. It does not, by itself, tell an application who the user is — that is exactly the gap OIDC fills. "Log in with OAuth" implemented without OIDC is a well-known class of security bug.

## VIII.2 Roles and endpoints 🔴

| Role | Who |
|---|---|
| **Resource owner** | The user who owns the data |
| **Client** | The application requesting access |
| **Authorization server** | Authenticates the user and issues tokens (Okta, Entra ID, Keycloak, Auth0, Google) |
| **Resource server** | The API that accepts the token |

| Endpoint | Purpose |
|---|---|
| `/authorize` | Browser-facing; user consents here |
| `/token` | Back-channel; exchanges a grant for tokens |
| `/introspect` | Resource server asks "is this opaque token valid?" (RFC 7662) |
| `/revoke` | Kill a token (RFC 7009) |
| `/.well-known/openid-configuration` | Discovery document listing all of the above |
| `/jwks` | Public keys for verification |

**Client types:** *confidential* clients can keep a secret (a backend service); *public* clients cannot (SPAs, mobile apps, CLIs). Public clients must use PKCE and never hold a client secret.

## VIII.3 The flows 🔴

### Authorization Code + PKCE — the default for anything with a user

PKCE (RFC 7636) is now required for **all** clients, confidential ones included (RFC 9700).

```text
1. client generates:  code_verifier (random)
                      code_challenge = BASE64URL(SHA256(code_verifier))

2. browser → /authorize?response_type=code
                       &client_id=...
                       &redirect_uri=https://app/callback
                       &scope=openid profile api.read
                       &state=<csrf>
                       &code_challenge=<challenge>&code_challenge_method=S256

3. user authenticates at the IdP and consents

4. browser ← 302 https://app/callback?code=<auth-code>&state=<csrf>

5. client → POST /token  grant_type=authorization_code
                         code=<auth-code>
                         code_verifier=<verifier>
                         redirect_uri=...

6. client ← { access_token, refresh_token, id_token, expires_in }
```

Why each guard exists: `state` blocks CSRF on the callback; the `code` is single-use and short-lived; PKCE ensures that an intercepted code is useless without the verifier; exact `redirect_uri` matching blocks open-redirect token theft.

### Client Credentials — the one you will use most in DevOps

Machine to machine. No user, no browser.

```bash
$ curl -s -X POST https://idp.example.com/oauth2/token \
    -d grant_type=client_credentials \
    -d client_id="$CLIENT_ID" \
    -d client_secret="$CLIENT_SECRET" \
    -d scope="metrics.write" | jq .
```

This is the flow behind service-to-service API calls, Terraform providers, and monitoring integrations. Note the weakness: it is a **static client secret**, which is just an API key with extra steps. Prefer certificate-based client authentication (`private_key_jwt`, mTLS) or a workload identity federation where the platform supports it.

### Device Authorization Grant — for CLIs and devices without a browser

RFC 8628. The flow behind `aws sso login`, `gh auth login`, and smart TV sign-ins.

```text
CLI → /device_authorization  → { device_code, user_code: "WDJB-MJHT",
                                 verification_uri: https://idp/activate }
CLI displays the code; you open the URL on a phone/laptop and approve
CLI polls /token with device_code until it gets tokens
```

### Deprecated — recognise and remove

| Flow | Why it is gone |
|---|---|
| **Implicit** (`response_type=token`) | Token in the URL fragment: browser history, referrers, logs. |
| **Resource Owner Password Credentials** | The client handles the user's password — defeats the point of OAuth, blocks MFA. |

Both are formally deprecated by RFC 9700 and excluded from OAuth 2.1.

## VIII.4 Scope, audience, consent 🟠

- **Scope** — what the client may do: `read:repo`, `s3:read`. Requested by the client, approved by the user or by policy.
- **Audience / resource indicator** (RFC 8707) — which API the token is for. Without it, a token from your IdP is potentially valid at every API that trusts that IdP.
- **Consent** — the screen listing scopes. In enterprise setups admins pre-consent on users' behalf.

Recurring failure: one IdP, many internal APIs, no audience restriction. A low-value service can then replay its token against a high-value one. Always set and verify `aud`.

---

# Part IX — OIDC, SSO, SAML, and legacy enterprise auth

## IX.1 OIDC 🔴

OpenID Connect is a thin, precisely specified **authentication** layer on top of OAuth 2.0.

```text
OAuth 2.0 → authorization  → "this client may call the API"
OIDC      → authentication → "this is Alice, verified by this IdP, at this time"
```

What OIDC adds to OAuth: the `openid` scope, the **ID token** (a JWT with standard identity claims), the **UserInfo** endpoint, a **discovery** document, standard claims, and the `nonce` parameter binding an ID token to a specific authentication request.

**Discovery** — everything a client needs, from one URL:

```bash
$ curl -s https://accounts.google.com/.well-known/openid-configuration | jq '{
    issuer, authorization_endpoint, token_endpoint, jwks_uri,
    id_token_signing_alg_values_supported, scopes_supported }'
```

That endpoint is your first debugging stop for any OIDC integration: it tells you the exact `issuer` string (which must match `iss` in tokens **character for character**, trailing slash included) and where the keys live.

**Why OIDC dominates infrastructure:** it is how a Kubernetes cluster, a Grafana, an Argo CD, a Vault, and an AWS account can all trust one identity provider without any of them storing user credentials — and how a GitHub Actions job proves what repository and branch it is running from without any secret at all (XV.3).

**Claims to RBAC** is the practical integration work: map an IdP `groups` claim onto Kubernetes group names, Grafana roles, or Argo CD RBAC. Get the claim name right (`groups`, `roles`, `cognito:groups`, `http://schemas...`) — it is provider-specific and the top cause of "SSO works but I have no permissions".

## IX.2 SSO 🔴

SSO is an **architecture and a user experience**, not a protocol:

```text
                Identity Provider (one login, one session)
                          │
        ┌─────────┬───────┼────────┬──────────┐
      AWS      Grafana  Argo CD  Jira      Vault
```

```text
SSO ≠ OIDC        SSO ≠ OAuth        SSO ≠ SAML
SSO is implemented *using* OIDC or SAML.
```

Operationally, SSO means:
- **One IdP session** — the applications delegate; the IdP remembers.
- **Central offboarding** — disable the account once, lose access everywhere. This is the main business justification.
- **Central MFA policy** — enforce once, applies everywhere.
- **A single point of failure.** When the IdP is down, nobody logs into anything. This is why you need a **break-glass** path: a local admin credential, stored offline, MFA-protected, alerted on use, tested quarterly.
- **SP-initiated vs IdP-initiated** — starting at the app vs starting at the IdP dashboard. IdP-initiated SAML is weaker (no request to correlate against) and some apps refuse it.
- **Single Logout (SLO)** is notoriously unreliable across products. Assume that logging out of one app does not end the IdP session.

## IX.3 SAML 🟠

XML-based federation, published 2005, still the backbone of enterprise SSO.

```text
User → App (SP) → redirect with AuthnRequest → IdP → user authenticates
     → signed SAML Assertion POSTed to the SP's ACS URL → session created
```

Vocabulary: **IdP** (Okta, Entra ID, Ping), **SP** (the app), **Assertion** (signed XML with the identity and attributes), **ACS URL** (Assertion Consumer Service — where the assertion is POSTed), **Entity ID** (the unique name of each side), **Metadata** (an XML document describing endpoints and signing certificates).

| | SAML | OIDC |
|---|---|---|
| Format | XML | JSON / JWT |
| Transport | Browser POST/redirect | HTTP + JSON |
| Native fit | Web apps, enterprise | APIs, mobile, CLIs, workloads |
| Complexity | High (XML signatures, canonicalisation) | Moderate |

**The four things that actually break SAML**, in order of frequency:
1. **Clock skew** — assertions have tight validity windows. Run NTP.
2. **IdP signing certificate expiry/rollover** — nobody notices until every login fails at once. Monitor it like any other certificate.
3. **ACS URL or Entity ID mismatch** — a copy-paste error, or an app moved behind a new hostname.
4. **Attribute/claim name mismatch** — the app expects `email`, the IdP sends `EmailAddress`.

Debug by capturing the SAML response in the browser (SAML-tracer extension), base64-decoding it, and reading the XML. Do not try to become a SAML protocol expert unless your job requires it.

## IX.4 LDAP and Active Directory 🟠

LDAP is a **directory protocol** — a hierarchical database of users, groups, and machines — that also supports authentication via **bind**.

```text
dn: cn=alice,ou=engineering,dc=example,dc=com
```

- **Simple bind** — send the DN and password. Requires LDAPS (`:636`) or StartTLS; over plain 389 it is cleartext.
- **Search-then-bind** — the app binds as a service account, searches for the user's DN by `uid`/`mail`, then rebinds as the user to verify the password.
- **Group membership** drives authorization: `memberOf`, or a search of `groupOfNames` entries.

```bash
$ ldapsearch -x -H ldaps://dc.example.com -D "cn=svc,dc=example,dc=com" -W \
    -b "dc=example,dc=com" "(uid=alice)" memberOf
```

Still very common as a backend for Vault, Grafana, Jenkins, Gitea, VPNs, and anything predating OIDC. In AD environments the modern path is Entra ID + OIDC/SAML, with LDAP surviving for legacy apps.

## IX.5 Kerberos 🟢

Ticket-based authentication, the core of Active Directory domain logins.

```text
Login → AS-REQ to the KDC → TGT (ticket-granting ticket, ~10 h)
Need a service? → TGS-REQ with the TGT → service ticket for that SPN
Present the service ticket → access, no password sent
```

Terms: **KDC** (Key Distribution Center), **TGT**, **service ticket**, **SPN** (Service Principal Name, e.g. `HTTP/web.example.com`), **keytab** (a file holding a principal's long-term key so a service can authenticate unattended), **realm** (the uppercase domain).

```bash
$ kinit alice@EXAMPLE.COM     # get a TGT
$ klist                        # show the ticket cache
$ kdestroy                     # discard
```

Kerberos is **exquisitely sensitive to clock skew** — the default tolerance is five minutes, and "Kerberos broke" is very often "NTP broke". Over HTTP it appears as `Authorization: Negotiate` (SPNEGO/GSSAPI). You will meet it with Hadoop, older enterprise Linux, Windows file shares, and some databases. Learn it when forced.

---

# Part X — MFA for humans

## X.1 Factors 🔴

```text
Something you KNOW   → password, PIN
Something you HAVE   → phone, hardware key, certificate on a device
Something you ARE    → fingerprint, face
```

MFA requires factors from **different categories**. A password plus a security question is one factor twice.

## X.2 The methods, ranked 🔴

| Method | Phishing-resistant? | Verdict |
|---|---|---|
| **WebAuthn / FIDO2 / passkeys** | **Yes** — bound to the origin | Best available. Use for all privileged access. |
| **Hardware key (YubiKey) in FIDO2 mode** | Yes | Same. Buy two: one primary, one in a safe. |
| **TOTP app** | No — a phished code works for 30 s | Good baseline. |
| **Push notification** | No — MFA-fatigue attacks | Acceptable *with* number matching. |
| **SMS / voice** | No — SIM swap, SS7 | Weakest. Better than nothing, avoid for admins. |

## X.3 TOTP 🔴

RFC 6238. Server and authenticator share a secret at enrolment (the QR code) and both compute `HMAC(secret, current_30s_window)`, truncated to six digits.

Operational facts: it works **offline** (no network needed to generate a code); it is **time-sensitive**, so server clock drift breaks it and most servers accept ±1 window; the enrolment secret is the recovery material — losing it and the device means account recovery; and **recovery codes** must be generated, stored securely, and treated as equivalent to the password.

## X.4 WebAuthn and passkeys 🟠

A key pair is created on the device, per site. The private key never leaves the authenticator (often hardware-backed). Authentication is a signed challenge.

The crucial property: **the signature is bound to the origin**. A phishing site at `github-login.com` cannot obtain a signature valid for `github.com`, because the browser will not produce one. This is a structural defence, not a user-education one, and it is why passkeys are the direction of travel.

**Passkeys** = discoverable WebAuthn credentials, often synced across a user's devices via a platform keychain. Synced passkeys trade a little assurance for a lot of usability; device-bound hardware keys remain the strongest option for production access.

## X.5 MFA in DevOps practice 🔴

- Enforce MFA on: cloud consoles, the IdP itself, source control, the secrets manager, VPN, and the CI/CD control plane.
- **Root/break-glass accounts**: hardware key, physically secured, alert on any use.
- **MFA does not apply to workloads.** A CI pipeline cannot tap a phone. The workload equivalent of MFA is workload identity plus short-lived credentials (Part XIII).
- **Programmatic access is the gap.** A user with MFA on the console but a long-lived static API key has MFA in name only. In AWS, enforce `aws:MultiFactorAuthPresent` in policies, or eliminate static keys entirely with Identity Center.

---
# Layer 5 — Platform identity

# Part XI — AWS IAM, STS, and federation

## XI.1 The model 🔴

| Concept | What it is |
|---|---|
| **Principal** | The entity making a request — an IAM user, an assumed role session, a service |
| **IAM user** | A long-lived identity with optional access keys. **Minimise these.** |
| **IAM role** | An identity with **no credentials of its own**, assumed temporarily |
| **Identity policy** | Attached to a user/group/role — what it may do |
| **Resource policy** | Attached to a resource (S3 bucket, KMS key, SQS queue) — who may use it |
| **Trust policy** | Attached to a role — **who is allowed to assume it** |
| **Permission boundary** | A ceiling on what an identity can be granted |
| **SCP** | Organisation-wide ceiling across accounts |

A role has **two** policies and confusing them wastes hours:

```text
Trust policy      → WHO can assume this role     (AssumeRole fails → check here)
Permissions policy → WHAT the role can then do   (API call denied → check here)
```

## XI.2 Policy evaluation order 🔴

For any request, AWS evaluates in this order. Memorise it; it explains every "but I gave it Allow" ticket:

```text
1. Explicit DENY anywhere            → DENY. Full stop, nothing overrides this.
2. SCP (organisations)               → must allow
3. Resource policy                   → may grant across accounts
4. Identity policy                   → must allow (unless the resource policy suffices, same account rules apply)
5. Permission boundary               → must allow
6. Session policy                    → must allow
7. Otherwise                         → implicit DENY
```

Default is deny. An explicit `Deny` always wins. Cross-account access needs an allow on **both** sides.

```bash
$ aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::123456789012:role/deployer \
    --action-names s3:PutObject --resource-arns arn:aws:s3:::my-bucket/*
```

## XI.3 Credentials 🔴

**Long-lived access keys** — `AKIA...` plus a secret. These are the classic breach vector: found in git, in Slack, in a Docker image, in a Stack Overflow answer. Treat every one you find as an incident.

```bash
$ aws sts get-caller-identity          # who am I, actually?
$ aws configure list                    # where did the creds come from?
```

`get-caller-identity` is the single most useful AWS auth command. Run it before debugging anything else.

**Temporary credentials** — three values, always: `AccessKeyId` (starts `ASIA`), `SecretAccessKey`, and a **`SessionToken`** that must be sent too. Forgetting the session token produces `InvalidClientTokenId`.

**The SDK credential chain**, in order (know this — it explains "it picked the wrong account"):

```text
1. Explicit code / CLI parameters
2. Environment variables (AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_SESSION_TOKEN)
3. Web identity token file (AWS_WEB_IDENTITY_TOKEN_FILE)  ← IRSA lands here
4. Shared config/credentials (~/.aws/credentials, ~/.aws/config, AWS_PROFILE)
5. Container credentials (ECS/EKS Pod Identity endpoint)
6. EC2 instance metadata (IMDS)
```

## XI.4 STS 🔴

Security Token Service issues temporary credentials.

| API | Used by |
|---|---|
| `AssumeRole` | An IAM principal assuming a role, cross-account access |
| `AssumeRoleWithWebIdentity` | **OIDC federation** — GitHub Actions, IRSA, mobile apps |
| `AssumeRoleWithSAML` | Enterprise SAML federation |
| `GetSessionToken` | MFA-protected temporary credentials for an IAM user |

```bash
$ aws sts assume-role \
    --role-arn arn:aws:iam::123456789012:role/deployer \
    --role-session-name nazmur-2026-08-30 \
    --duration-seconds 3600
```

Durations: `AssumeRole` defaults to 1 hour, up to the role's `MaxSessionDuration` (max 12 h). Role chaining caps at 1 hour. `GetSessionToken` runs 15 minutes to 36 hours.

**Session names matter.** They appear in CloudTrail as `arn:aws:sts::123:assumed-role/deployer/<session-name>`. Put something identifying in there (username, run ID, commit SHA) or your audit trail says nothing.

A trust policy allowing a specific role to assume another, with an external ID for third parties:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "AWS": "arn:aws:iam::111122223333:role/ci-runner" },
    "Action": "sts:AssumeRole",
    "Condition": { "StringEquals": { "sts:ExternalId": "unique-per-customer-value" } }
  }]
}
```

## XI.5 OIDC federation into AWS 🔴

This is the pattern that removes static keys from CI, and it is worth understanding in full because it is the template for all workload federation.

```text
1. Register the OIDC provider's issuer in IAM (once per issuer per account).
   AWS fetches its JWKS and can now verify tokens it signs.
2. Write a role trust policy that accepts tokens from that issuer,
   with conditions on `aud` and `sub`.
3. The workload obtains a signed OIDC JWT from its platform.
4. It calls sts:AssumeRoleWithWebIdentity with that JWT.
5. STS verifies the signature and the conditions, and returns temporary credentials.
```

GitHub Actions trust policy — note the `StringLike` on `sub`, which is where the security actually lives:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
      }
    }
  }]
}
```

> **The `sub` condition is the whole security boundary.** `repo:my-org/*` lets *any* repository in the org — including a new one an attacker can create — assume the role. `repo:my-org/my-repo:*` lets any branch or pull request in that repo do it, which means a fork PR can potentially reach production. Pin to the exact ref or environment (`repo:org/repo:environment:production`). Misconfigured `sub` conditions are a well-documented real-world compromise path.

## XI.6 IAM Identity Center 🟠

The modern answer for **human** AWS access: no IAM users, no static keys.

```text
Developer → IdP (Entra/Okta/built-in) → Identity Center → permission set
          → assumes a role in the target account → temporary credentials (default 1 h)
```

```bash
$ aws configure sso
$ aws sso login --profile prod-admin
$ aws --profile prod-admin sts get-caller-identity
```

`~/.aws/config`:

```ini
[profile prod-admin]
sso_session = corp
sso_account_id = 123456789012
sso_role_name = AdministratorAccess
region = eu-central-1

[sso-session corp]
sso_start_url = https://d-1234567890.awsapps.com/start
sso_region = eu-central-1
sso_registration_scopes = sso:account:access
```

If you still have IAM users with access keys for humans, migrating to Identity Center is one of the highest-value security changes you can make in an AWS estate.

## XI.7 IMDS 🔴

EC2 instances (and by extension nodes) obtain role credentials from the instance metadata service at `169.254.169.254`.

```bash
# IMDSv2 — session-oriented, required for safety
$ TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
$ curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

**IMDSv1 is a simple GET, which means any SSRF vulnerability in an application on that instance can read the node's credentials.** This is the mechanism behind several very large breaches. Enforce IMDSv2 (`HttpTokens: required`) and set `HttpPutResponseHopLimit: 1` so containers on the host cannot reach it. In Kubernetes, block Pod access to the link-local address entirely and use IRSA or Pod Identity instead — otherwise every Pod inherits the whole node role.

## XI.8 SigV4 and clock skew 🟠

AWS API requests are signed with Signature Version 4: a canonical request is hashed and HMAC'd with a key derived from the secret, date, region, and service. Two consequences you will actually hit:

- **Clock skew breaks everything.** More than ~5 minutes off and you get `SignatureDoesNotMatch` or `RequestTimeTooSkewed`. Check NTP first.
- **Presigned URLs** embed the signature and an expiry in the query string — that URL *is* a bearer credential for its lifetime.

---

# Part XII — Kubernetes identity

## XII.1 The authentication chain 🔴

The API server tries authenticators in sequence until one succeeds. **Kubernetes has no user objects** — there is no `User` resource, no password database. Human identity always comes from outside.

```text
Request → API server
   ├─ 1. Client certificate      (CN → username, O → groups)
   ├─ 2. Bearer token — ServiceAccount JWT
   ├─ 3. Bearer token — OIDC (--oidc-issuer-url)
   ├─ 4. Webhook token authentication (this is how EKS/aws-iam-authenticator works)
   ├─ 5. Authenticating proxy headers
   └─ none matched → system:anonymous
        ↓
      Authorization: RBAC (usually), plus Node, ABAC, Webhook
        ↓
      Admission controllers
```

Two commands that answer "who does the cluster think I am?":

```bash
$ kubectl auth whoami                       # 1.26+ ; prints username, groups, extra
$ kubectl auth can-i list secrets -n prod
$ kubectl auth can-i --list -n prod         # everything you can do here
$ kubectl auth can-i get pods --as=system:serviceaccount:prod:api -n prod   # impersonate
```

If `kubectl auth whoami` is unavailable, `kubectl get --raw /api` with `-v=8` or a deliberately failing request will show the identity in the error message.

## XII.2 kubeconfig 🔴

```yaml
apiVersion: v1
kind: Config
current-context: prod
clusters:
- name: prod
  cluster:
    server: https://ABC123.gr7.eu-central-1.eks.amazonaws.com
    certificate-authority-data: LS0tLS1CRUdJTi...      # trust for the API server's cert
contexts:
- name: prod
  context: { cluster: prod, user: prod-user, namespace: default }
users:
- name: prod-user
  user:
    exec:                                              # credential plugin
      apiVersion: client.authentication.k8s.io/v1beta1
      command: aws
      args: ["eks", "get-token", "--cluster-name", "prod"]
```

Three distinct ways the `user` section can authenticate:
- **`client-certificate-data` / `client-key-data`** — mTLS. `CN` becomes the username, each `O` becomes a group. **Client certificates cannot be revoked in Kubernetes** — there is no CRL check — so the only remedy for a leaked one is to re-issue the cluster CA. Avoid them for humans.
- **`token`** — a static bearer token. Fine for a ServiceAccount, poor for a person.
- **`exec`** — a credential plugin that mints a fresh short-lived token per invocation. `aws eks get-token`, `gke-gcloud-auth-plugin`, `kubelogin` for OIDC. **This is the right answer for humans.**

```bash
$ kubectl config get-contexts
$ kubectl config use-context prod
$ kubectl config view --minify --raw        # the effective config, secrets included
```

> A kubeconfig with an embedded long-lived token or client key is a full cluster credential sitting in a dotfile. Treat it exactly like an SSH private key: never in a repo, never in a Slack message, never in a container image.

## XII.3 ServiceAccounts and tokens 🔴

A ServiceAccount is the identity of a **workload** inside the cluster. Its username is:

```text
system:serviceaccount:<namespace>:<name>
```

and it is automatically in the groups `system:serviceaccounts` and `system:serviceaccounts:<namespace>`.

**The token model changed, and knowing the difference matters:**

| | Legacy (pre-1.24) | Bound / projected (current) |
|---|---|---|
| Storage | A `Secret` object per SA, created automatically | No Secret; projected into the Pod at runtime |
| Lifetime | **Never expired** | ~1 hour, **auto-rotated by the kubelet** |
| Audience | None — valid anywhere | Bound to a specific `audience` |
| Bound to | Nothing | The Pod and ServiceAccount; invalid once the Pod is gone |
| Status | Auto-creation removed in 1.24; cleanup of unused ones in 1.29+ | Default since 1.21 |

Inside a Pod:

```text
/var/run/secrets/kubernetes.io/serviceaccount/token      # the JWT, rotated
/var/run/secrets/kubernetes.io/serviceaccount/ca.crt     # to verify the API server
/var/run/secrets/kubernetes.io/serviceaccount/namespace
```

Mint one on demand with the TokenRequest API:

```bash
$ kubectl create token my-sa -n prod --duration=10m
$ kubectl create token my-sa -n prod --audience=vault --duration=10m
$ kubectl create token my-sa -n prod | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq .
```

Turn off the automount for anything that does not call the API server — this is cheap, and it removes a credential from most of your Pods:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: worker
  namespace: prod
automountServiceAccountToken: false
---
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: worker
  automountServiceAccountToken: false     # Pod-level override
```

Projecting a token with an explicit audience (the mechanism behind Vault and IRSA):

```yaml
volumes:
- name: vault-token
  projected:
    sources:
    - serviceAccountToken:
        path: token
        audience: vault
        expirationSeconds: 600
```

## XII.4 RBAC 🔴

| Object | Scope |
|---|---|
| `Role` | Permissions **within one namespace** |
| `ClusterRole` | Cluster-wide permissions, or reusable across namespaces |
| `RoleBinding` | Grants a Role **or a ClusterRole** to subjects, in one namespace |
| `ClusterRoleBinding` | Grants a ClusterRole cluster-wide |

RBAC is **purely additive** — there are no deny rules. Access is the union of every binding that applies.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata: { name: pod-reader, namespace: prod }
rules:
- apiGroups: [""]
  resources: ["pods", "pods/log"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata: { name: api-can-read-pods, namespace: prod }
subjects:
- kind: ServiceAccount
  name: api
  namespace: prod
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

The single most useful trick: **a RoleBinding referencing a ClusterRole** grants those permissions only inside that namespace. That is how you reuse the built-in `view`, `edit`, and `admin` ClusterRoles per team namespace instead of writing Roles by hand.

Privilege escalation paths to watch in reviews: `secrets: get/list` in a namespace with useful secrets; `create pods` (mount any SA token, or a hostPath); `escalate`/`bind` verbs; `impersonate`; and `*` on `*`.

```bash
$ kubectl get clusterrolebindings -o json | jq -r '
    .items[] | select(.roleRef.name=="cluster-admin") | .metadata.name + " -> " +
    ([.subjects[]?|.kind+"/"+.name]|join(","))'
```

## XII.5 EKS: IRSA vs Pod Identity 🔴

Both give a Pod its own AWS credentials with no static keys. They differ in how trust is established, and you will meet both.

### IRSA (IAM Roles for Service Accounts, 2019)

Pure OIDC federation — the cluster is an OIDC issuer, AWS is the relying party.

```text
Pod (SA annotated with a role ARN)
  → kubelet projects an SA token with audience sts.amazonaws.com
  → AWS SDK reads AWS_WEB_IDENTITY_TOKEN_FILE + AWS_ROLE_ARN (injected by a webhook)
  → sts:AssumeRoleWithWebIdentity
  → temporary credentials
```

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  namespace: prod
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-reader
```

Trust policy — cluster-specific, which is the main drawback:

```json
{
  "Effect": "Allow",
  "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.eu-central-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E" },
  "Action": "sts:AssumeRoleWithWebIdentity",
  "Condition": { "StringEquals": {
    "oidc.eks.eu-central-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:sub": "system:serviceaccount:prod:s3-reader",
    "oidc.eks.eu-central-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B716D3041E:aud": "sts.amazonaws.com"
  }}
}
```

### EKS Pod Identity (re:Invent 2023)

An EKS-native association plus a node agent. No OIDC provider to create, and the role's trust policy names a single service principal instead of a per-cluster issuer.

```bash
$ aws eks create-pod-identity-association \
    --cluster-name prod --namespace prod \
    --service-account s3-reader \
    --role-arn arn:aws:iam::123456789012:role/s3-reader
```

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "pods.eks.amazonaws.com" },
  "Action": ["sts:AssumeRole", "sts:TagSession"]
}
```

| | IRSA | Pod Identity |
|---|---|---|
| Setup | OIDC provider per cluster | Install the Pod Identity Agent add-on |
| Trust policy | Cluster-specific issuer URL | One service principal, **reusable across clusters** |
| Where it works | EKS, EKS Anywhere, self-managed K8s, ROSA — anywhere OIDC federation works | EKS (and hybrid nodes via `eks-auth:AssumeRoleForPodIdentity`) |
| Session tags | No | **Yes** — enables ABAC |
| Config location | Annotation on the SA | An association held in the EKS API |
| Debugging | `sts:AssumeRoleWithWebIdentity` failures, sub/aud conditions | Agent health, association existence |

**Choosing:** for a new EKS-only estate, Pod Identity is simpler and scales better across clusters. If you run non-EKS clusters, need one mechanism everywhere, or already have IRSA working at scale, IRSA remains fully supported. They coexist; Pod Identity takes precedence if both are configured for the same SA.

### Cluster access on EKS 🔴

Who may talk to the API server at all is separate from RBAC. Historically this was the `aws-auth` ConfigMap; the current mechanism is **EKS access entries**:

```bash
$ aws eks list-access-entries --cluster-name prod
$ aws eks create-access-entry --cluster-name prod \
    --principal-arn arn:aws:iam::123456789012:role/deployer --type STANDARD
$ aws eks associate-access-policy --cluster-name prod \
    --principal-arn arn:aws:iam::123456789012:role/deployer \
    --access-scope type=namespace,namespaces=prod \
    --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSEditPolicy
```

`error: You must be logged in to the server (Unauthorized)` on EKS almost always means the IAM principal has no access entry (or no `aws-auth` mapping), **not** that your AWS credentials are wrong. Verify with `aws sts get-caller-identity` first.

## XII.6 Registry authentication 🔴

Pulling images is authentication too, and `ImagePullBackOff` is on your pager.

```bash
$ docker login registry.example.com          # writes to ~/.docker/config.json
$ aws ecr get-login-password --region eu-central-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.eu-central-1.amazonaws.com
```

ECR tokens are valid for **12 hours** — which is why a long-lived node that cached one keeps working and a new one fails, and why you should let the kubelet's ECR credential provider (or IRSA/Pod Identity on the node role) handle it rather than baking a Secret.

```bash
$ kubectl create secret docker-registry regcred \
    --docker-server=registry.example.com \
    --docker-username=robot --docker-password="$TOKEN" -n prod
```

```yaml
spec:
  imagePullSecrets:
  - name: regcred
```

You can also attach the pull secret to the ServiceAccount so every Pod using it inherits the secret, which is usually tidier than repeating it in every manifest.

## XII.7 Internal cluster TLS 🟠

Worth knowing exists, because expiry causes total cluster failure:

- **kubelet ↔ API server**, **API server ↔ etcd**, **etcd peers** — all mTLS with certificates from the cluster CA.
- **kubeadm clusters** — control-plane certificates default to **one year**. `kubeadm certs check-expiration` and `kubeadm certs renew all`. Clusters that are never upgraded die on their first birthday, and this is a real and regular outage.
- **Admission webhooks** need a serving certificate the API server trusts, via `caBundle` — usually managed by cert-manager's CA injector. An expired webhook certificate can block all writes to the cluster.
- **Node bootstrap** uses bootstrap tokens plus the CSR API with kubelet certificate rotation.

---
# Part XIII — Workload identity as a pattern

## XIII.1 The idea 🔴

> Give a workload a **verifiable identity** derived from where and what it is, and exchange that identity for short-lived credentials on demand. Do not give it a secret.

```text
OLD                                  NEW
app ─── static secret ──► API        app ─── platform-attested identity ──► token service
      (stored, copied,                                                   ──► short-lived credential ──► API
       leaked, never rotated)
```

Everything you get for free: nothing to rotate, nothing to leak usefully, credentials that expire on their own, an audit trail tied to a real workload, and revocation by deleting the workload.

## XIII.2 The secret-zero problem 🔴

Every credential system eventually asks: how does the workload authenticate the *first* time? If the answer is "with a secret", you have only moved the problem.

Workload identity solves it by having the **platform attest** the workload, using facts the platform already knows and the workload cannot forge:

| Platform | The attestation |
|---|---|
| AWS EC2 | Signed instance identity document from IMDS |
| Kubernetes | The API server signs a token stating which SA and Pod this is |
| EKS | Cluster OIDC issuer signature (IRSA) or the EKS Auth API (Pod Identity) |
| GitHub Actions | GitHub signs a JWT naming repo, ref, workflow, environment |
| GCP | Metadata server identity token |
| Azure | Managed Identity endpoint on the instance |
| SPIRE | Node + workload attestation from kernel and platform facts |

The root of trust is the platform itself, which you were already trusting to run your code.

## XIII.3 Cloud mapping 🔴

| Platform | Mechanism | Workload gets |
|---|---|---|
| AWS EC2 | Instance profile | Role credentials via IMDS |
| AWS ECS | Task role | Role credentials via the container credentials endpoint |
| AWS EKS | IRSA / Pod Identity | Role credentials, per ServiceAccount |
| AWS Lambda | Execution role | Role credentials in the environment |
| GCP | Workload Identity Federation | Service account impersonation |
| Azure | Managed Identity / Workload Identity | Entra token |
| Kubernetes (any) | ServiceAccount + projected token | A JWT it can exchange elsewhere |
| Any → Vault | Kubernetes/JWT/AWS auth methods | A Vault token |
| Any → any | SPIFFE/SPIRE | X.509 or JWT SVID |

Notice the shape is identical everywhere: **platform-signed assertion → exchange → short-lived credential**. Learn it once and every platform's version is a detail.

## XIII.4 Token exchange chains 🟠

Real systems chain these, and being able to trace a chain end to end is a genuinely valuable skill:

```text
K8s ServiceAccount token (aud=vault)
   → Vault Kubernetes auth
   → Vault token
   → Vault AWS secrets engine
   → dynamic, short-lived AWS credentials
   → S3
```

```text
GitHub Actions OIDC JWT (aud=sts.amazonaws.com, sub=repo:org/repo:environment:prod)
   → sts:AssumeRoleWithWebIdentity
   → AWS temporary credentials
   → aws eks get-token
   → Kubernetes API bearer token
   → RBAC decision
```

When one of these breaks, debug it **one hop at a time**, from the start. Print the token at each stage (decode the payload, never log the whole token in CI) and confirm `iss`, `sub`, `aud`, and `exp` are what the next hop expects.

---

# Part XIV — Vault and secrets management

## XIV.1 The Vault model 🔴

Vault does not know who you are. Everything follows one shape:

```text
authenticate (auth method) → Vault token → policies attached → read/write secrets
```

```bash
$ vault login -method=userpass username=alice
$ vault token lookup                 # what am I, what policies, how long left
$ vault token capabilities secret/data/prod/db    # what can I do with this path
$ vault kv get -mount=secret prod/db
```

## XIV.2 Auth methods 🔴

| Method | Who it is for |
|---|---|
| **Kubernetes** | Pods, using their ServiceAccount token |
| **JWT / OIDC** | CI systems (GitHub Actions), and human SSO login |
| **AWS** | EC2 instances and IAM roles |
| **AppRole** | Applications with no platform identity available |
| **TLS certificates** | Workloads with an existing PKI |
| **LDAP / Userpass / GitHub** | Humans, mostly legacy |
| **Token** | Bootstrapping and the root token |

### Kubernetes auth 🔴

```bash
$ vault auth enable kubernetes
$ vault write auth/kubernetes/config kubernetes_host="https://kubernetes.default.svc:443"

$ vault write auth/kubernetes/role/api \
    bound_service_account_names=api \
    bound_service_account_namespaces=prod \
    policies=api-read \
    audience=vault \
    ttl=1h
```

The Pod presents its projected SA token; Vault validates it against the cluster (via the TokenReview API or the cluster's public keys) and checks the SA name, namespace, and audience against the role. **The Pod never holds a Vault secret** — this is the secret-zero problem solved.

### JWT auth for CI 🔴

```bash
$ vault write auth/jwt/role/github-deploy \
    role_type=jwt \
    bound_audiences="https://vault.example.com" \
    bound_claims='{"repository":"my-org/my-repo","ref":"refs/heads/main"}' \
    user_claim="sub" \
    policies=deploy \
    ttl=15m
```

Same principle as the AWS trust policy: **`bound_claims` is the security boundary.** Bind on the tightest set of claims that still works.

## XIV.3 AppRole 🟠

For applications with no platform identity to attest them.

```text
role_id    → not secret; ship with the app config
secret_id  → secret; delivered separately, short-lived, ideally response-wrapped
   ↓ both presented to Vault
Vault token
```

```bash
$ vault read auth/approle/role/my-app/role-id
$ vault write -f -wrap-ttl=60s auth/approle/role/my-app/secret-id   # response wrapping
$ vault write auth/approle/login role_id="$ROLE_ID" secret_id="$SECRET_ID"
```

**Response wrapping** is the interesting part: Vault returns a single-use wrapping token instead of the secret. The delivery agent can pass it on but cannot read it, and if anyone unwraps it in transit, the legitimate recipient's unwrap fails — so tampering is *detectable*, not merely prevented.

AppRole is a fallback. If a platform identity (Kubernetes, AWS, JWT) is available, use that instead.

## XIV.4 Tokens and policies 🔴

```hcl
# policy: api-read
path "secret/data/prod/api/*" {
  capabilities = ["read"]
}
path "database/creds/api-readonly" {
  capabilities = ["read"]
}
path "secret/data/prod/billing/*" {
  capabilities = ["deny"]
}
```

Capabilities: `create`, `read`, `update`, `delete`, `list`, `sudo`, `deny`. **`deny` always wins**, as in AWS.

```bash
$ vault policy write api-read api-read.hcl
$ vault policy read api-read
$ vault token renew
$ vault token revoke <accessor>
$ vault lease revoke -prefix database/creds/     # kill every lease under a path
```

Token properties: **TTL** and **max TTL** (renewal ceiling); **service tokens** (stored, renewable, revocable) vs **batch tokens** (lightweight, not renewable, cheap at high volume); **orphan tokens** survive their parent's revocation; **accessors** let you inspect and revoke a token without holding it.

Vault's revocation story is the best of anything in this document: revoke a token and every dynamic secret leased under it dies with it.

## XIV.5 Dynamic secrets 🟠

The strongest Vault feature and the reason to run it at all: Vault **creates** the credential on demand and destroys it at lease expiry.

```bash
$ vault read database/creds/api-readonly
# → a database user that exists for 1 hour, then is dropped
```

Available for databases, AWS/GCP/Azure IAM, PKI certificates, SSH, RabbitMQ, Consul, and more. There is no long-lived database password to rotate because there is no long-lived database password.

## XIV.6 Getting secrets into workloads 🔴

| Approach | How | Trade-off |
|---|---|---|
| **App calls Vault directly** | SDK, Kubernetes auth | Cleanest; requires app changes |
| **Vault Agent Injector** | Sidecar templates secrets to a file | No app change; a sidecar per Pod |
| **Vault CSI provider** | Secrets mounted as a volume | Native-feeling; needs the CSI driver |
| **External Secrets Operator** | Syncs from Vault/ASM/GSM into K8s Secrets | Very popular; secret now also lives in etcd |
| **Sealed Secrets** | Encrypted secrets committed to git, decrypted in-cluster | GitOps-friendly; no central rotation |
| **SOPS + age/KMS** | Encrypted files in git | Simple, good for config repos |

Whatever you choose, remember that **a Kubernetes Secret is base64-encoded, not encrypted** — enable encryption at rest for etcd (or a KMS provider) and restrict `get`/`list` on secrets through RBAC.

---

# Part XV — The delivery pipeline

Every step from a developer's laptop to production is an authentication event. This part maps them all.

```text
developer ──► git host ──► CI ──► registry ──► GitOps controller ──► cluster ──► cloud
    SSH/PAT      OIDC        OIDC       token         SSO/SA           RBAC       IAM
```

## XV.1 Git authentication 🔴

| Mechanism | Scope | Use for |
|---|---|---|
| **SSH key (personal)** | Everything the user can reach | Individual developers |
| **Deploy key** | **One repository**, read or read/write | A single service's CI/CD access |
| **PAT (classic)** | Broad scopes, all repos | Legacy. Migrate away. |
| **Fine-grained PAT** | Per-repository, per-permission, expiring | Better; still a user-owned static secret |
| **GitHub App installation token** | Per-installation, ~1 hour, scoped | **Best for automation.** Not tied to a person. |
| **`GITHUB_TOKEN`** | Auto-issued per workflow run, expires with the job | Default inside Actions |
| **OIDC** | No credential at all | CI → cloud/Vault, not git itself |

```bash
$ ssh -T git@github.com                       # test SSH auth
$ git config --global credential.helper store # avoid; plaintext in ~/.git-credentials
$ gh auth login                               # device flow, stores in the OS keychain
```

GitHub removed password authentication for git over HTTPS in August 2021 — `remote: Support for password authentication was removed` means "use a token or SSH", not "your password is wrong".

**The offboarding trap:** automation that runs on a person's PAT or personal SSH key dies the day they leave. Use a GitHub App or a deploy key for anything a machine does.

**Commit signing** 🟠 — `git commit -S` with GPG or SSH signing keys, or `gitsign` for keyless signing via OIDC. Enforce with branch protection when you need provenance.

## XV.2 CI secrets 🔴

```text
Repository secrets      → all workflows in the repo
Environment secrets     → only jobs targeting that environment (can require approval)
Organization secrets    → shared, scoped to selected repositories
```

Rules that matter:
- Secrets are **not passed to workflows triggered by `pull_request` from forks**. This is deliberate; do not "fix" it.
- `pull_request_target` runs with secrets **and** the base repo's permissions. Combining it with checking out untrusted PR code is a well-known full-compromise pattern. Do not do it.
- Masking is best-effort. A secret that is transformed (base64'd, JSON-embedded, split) will appear in logs unmasked.
- Pin third-party actions to a **commit SHA**, not a tag. Tags are mutable, and the compromise of a widely used action is a supply-chain event that has actually happened.
- Set `permissions:` explicitly at the top of every workflow; the default `GITHUB_TOKEN` scope is broader than you need.

## XV.3 GitHub Actions OIDC → AWS 🔴

The reference pattern for keyless CI. Learn this one properly; it generalises.

```yaml
name: deploy
on:
  push:
    branches: [main]

permissions:
  id-token: write        # REQUIRED — without it there is no OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-deployer
          role-session-name: gha-${{ github.run_id }}
          aws-region: eu-central-1

      - run: aws sts get-caller-identity
      - run: aws s3 sync ./dist s3://my-bucket/
```

What happens under the hood:

```text
1. The runner requests a JWT from GitHub's OIDC provider
   (ACTIONS_ID_TOKEN_REQUEST_URL + ...TOKEN, present only with id-token: write)
2. The JWT's claims describe the run: sub, repository, ref, workflow, environment, actor
3. The action calls sts:AssumeRoleWithWebIdentity with it
4. AWS verifies the signature against GitHub's JWKS and checks the trust policy conditions
5. Temporary credentials are exported into the job's environment
```

Inspect the claims you are actually getting — do this once and the whole mechanism stops being magic:

```yaml
- name: show OIDC claims
  run: |
    TOKEN=$(curl -sH "Authorization: bearer $ACTIONS_ID_TOKEN_REQUEST_TOKEN" \
      "$ACTIONS_ID_TOKEN_REQUEST_URL&audience=sts.amazonaws.com" | jq -r .value)
    echo "$TOKEN" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq '{sub,aud,repository,ref,environment,job_workflow_ref}'
```

Then pin the trust policy's `sub` condition to exactly what you saw (XI.5).

The same pattern serves GitLab CI, CircleCI, Buildkite, and Terraform Cloud — different issuer, identical shape.

## XV.4 Argo CD 🔴

Argo CD has **three completely separate authentication relationships**. Conflating them is the classic Argo CD confusion, and each fails differently.

```text
[1] User ──────────────► Argo CD          (who may use Argo CD)
[2] Argo CD ───────────► Git repository   (how it reads manifests)
[3] Argo CD ───────────► Kubernetes API   (how it applies them)
```

**[1] User → Argo CD**
- Local `admin` account — disable it after SSO is working (`admin.enabled: "false"`).
- **SSO via OIDC** — either through the bundled Dex or directly to your IdP.
- **Project/account tokens** — JWTs for CI to call the Argo CD API.
- Argo CD's own RBAC maps IdP groups to roles:

```yaml
# argocd-rbac-cm
policy.csv: |
  p, role:dev, applications, sync, prod/*, allow
  p, role:dev, applications, get,  prod/*, allow
  g, my-org:platform-team, role:dev
policy.default: role:readonly
```

```bash
$ argocd login argocd.example.com --sso
$ argocd account get-user-info
```

**[2] Argo CD → Git**
- HTTPS + token, SSH deploy key, or a GitHub App (best for many repos).
- Stored as Secrets labelled `argocd.argoproj.io/secret-type: repository`.
- Failure shows up as `ComparisonError` / `repository not accessible`, **not** as a login problem.

**[3] Argo CD → Kubernetes**
- Same cluster: the `argocd-application-controller` ServiceAccount and its RBAC.
- Remote clusters: a stored cluster credential — a ServiceAccount bearer token, client certificate, or an exec plugin (IRSA for EKS).

```bash
$ argocd cluster list
$ argocd cluster add my-context
```

When something fails, **first decide which of the three relationships broke.** They have nothing to do with each other.

## XV.5 Terraform and IaC 🟠

- **Provider auth** — never hardcode credentials in `.tf`. Use the environment, a profile, or OIDC from CI.
- **Backend auth** — S3+DynamoDB, Terraform Cloud, GCS. Separate credentials from the provider's.
- **State contains secrets in plaintext.** Any resource with a password or key writes it into state. Encrypt the backend, restrict access to it as tightly as production, and never commit state.
- **OIDC from CI** is the correct pattern (XV.3) — no static cloud keys in the pipeline at all.
- Give plan and apply **different roles**: plan gets read-only, apply gets write, and only apply requires an approval gate.

## XV.6 Artifact and supply-chain identity 🟠

- **Registry push from CI** — use OIDC to the cloud registry, or a short-lived robot token; never a personal login.
- **Image signing** — `cosign` with **keyless** signing: your CI's OIDC identity is exchanged with Fulcio for a short-lived certificate, and the signature is logged in a transparency log. No signing key to store.

```bash
$ cosign sign --yes ghcr.io/org/app@sha256:abc...
$ cosign verify ghcr.io/org/app@sha256:abc... \
    --certificate-identity-regexp 'https://github.com/org/repo/.github/workflows/.*' \
    --certificate-oidc-issuer https://token.actions.githubusercontent.com
```

Note what that verification asserts: not "someone with a key signed this" but "**this specific workflow in this specific repository built this image**". That is workload identity applied to artifacts.

---
# Part XVI — Operating credentials

Design is the easy half. This part is the work that actually keeps a system secure over years.

## XVI.1 Rotation 🔴

The universal pattern is **two-key overlap**, because a rotation that requires downtime will not happen:

```text
1. Create the new credential alongside the old one
2. Deploy consumers with the new one
3. Verify — check usage metrics/logs show traffic on the new credential and none on the old
4. Disable the old one (do not delete yet)
5. Wait out the blast radius window; if nothing breaks, delete
```

Step 3 is the one people skip and the reason rotations cause outages.

| Credential | Rotate |
|---|---|
| Cloud access keys | Every 90 days, or eliminate them entirely |
| API keys | 90 days, and immediately on any suspicion |
| TLS certificates | Automatically, at 2/3 of lifetime |
| SSH keys | Annually; better, move to short-lived SSH certificates |
| Signing keys (JWT) | Overlap two keys, rotate the `kid`, keep the old key published until all tokens expire |
| Service account passwords | Replace with workload identity instead of rotating |
| **Short-lived credentials** | **Never — that is the entire point** |

The real goal is not faster rotation. It is **having fewer things that need rotating.** Every static secret you delete is a rotation task you never have to do again.

## XVI.2 Revocation 🔴

Know the revocation path for every credential type *before* you need it at 3 a.m.:

| Credential | How to revoke | Speed |
|---|---|---|
| Session | Delete the server-side session | Immediate |
| Vault token | `vault token revoke` | Immediate, cascades to leases |
| OAuth refresh token | `/revoke`, or kill the session at the IdP | Immediate |
| **JWT access token** | **You largely cannot** — wait for `exp`, or run a denylist | Up to the token lifetime |
| API key | Delete at the provider | Immediate |
| AWS access key | Deactivate, then delete | Immediate |
| AWS role session | Delete the role, attach a deny policy, or use `aws iam put-role-policy` with a `DateLessThan` deny on `aws:TokenIssueTime` | Fast, blunt |
| SSH key | Remove from every `authorized_keys` | As slow as your config management |
| TLS certificate | CRL/OCSP (weak) — or re-issue the CA | Slow and unreliable |
| K8s SA token (bound) | Delete the SA or the Pod | Immediate for new requests |
| K8s client certificate | **No revocation exists.** Rotate the cluster CA. | Painful |

The two entries in bold are why short lifetimes matter more than revocation machinery.

## XVI.3 Detecting leaks 🔴

- **Pre-commit and CI scanning**: `gitleaks`, `trufflehog`, `detect-secrets`. Run in pre-commit *and* CI — pre-commit alone is optional for the developer.
- **Platform push protection** (GitHub secret scanning) blocks known credential formats at push time. Turn it on.
- **Provider-side scanning** — GitHub notifies AWS, Stripe, and others when their key formats appear in public repos, and they auto-quarantine. Do not rely on it, but it has saved people.
- **Scan images and CI logs**, not just source: `dive`, `trivy`, `docker history`.
- **Alert on anomalous credential use** — a key used from a new country, an unused role suddenly assumed, an access key with no activity for 90 days.

## XVI.4 The leaked-credential runbook 🔴

Order matters. Rotate before you investigate.

```text
1. REVOKE FIRST. Do not wait for the investigation. Disable the credential now.
2. Issue a replacement and confirm services recover.
3. Determine the blast radius: what could this credential reach? Assume it was used.
4. Search the audit log for every use, especially from unexpected sources/times/IPs.
5. Check for persistence: new IAM users, new access keys, new SSH keys in authorized_keys,
   new OAuth app grants, new webhooks, modified trust policies, new K8s RBAC bindings.
6. Purge the exposure: git history rewrite is often futile (forks, clones, caches) —
   assume permanent exposure and rely on step 1.
7. Write it up blamelessly. Fix the class of problem, not just the instance.
```

Step 5 is the one people forget. An attacker's first move after using a stolen credential is to create a second, quieter one.

## XVI.5 Auditing 🔴

Know where the record lives for each system:

| System | Audit source |
|---|---|
| AWS | CloudTrail — look at `userIdentity`, `sourceIPAddress`, `sessionContext` |
| Kubernetes | API server audit log (must be explicitly configured — check that it is) |
| Vault | Audit devices (enable at least one; Vault refuses to operate if all enabled devices fail) |
| GitHub | Organisation audit log, plus per-repo events |
| IdP | Sign-in logs — the best single source for human access |
| SSH | `/var/log/auth.log`, or session recording via a bastion (Teleport, SSM Session Manager) |
| Argo CD | Application events and the API server log |

Two questions your logging must be able to answer, and it is worth testing that it can:
1. **Who did X, when, from where?**
2. **Everything identity Y did in window Z.**

If a shared credential makes question 1 unanswerable, that alone is enough reason to eliminate it.

## XVI.6 Least privilege in practice 🔴

- Start from deny. Add permissions in response to observed failures, not in anticipation.
- Use the tooling that reads your actual usage: AWS IAM Access Analyzer (generates policies from CloudTrail), `kubectl auth can-i --list`, Vault `token capabilities`.
- Separate identities per environment. One credential that works in dev and prod is a prod credential.
- Separate read from write, and plan from apply.
- Time-bound elevation over standing admin: JIT access, approval workflows, session recording.
- **Audit for the aggregate**: nobody grants `cluster-admin` on purpose, but three reasonable-looking bindings often add up to it.

## XVI.7 Monitoring that prevents pages 🔴

Set these up before you need them:

- **Certificate expiry** — blackbox exporter `probe_ssl_earliest_cert_expiry`, alert at 21 days. Include internal certs, the IdP's SAML signing cert, and Kubernetes control-plane certs.
- **Token/credential age** — alert on access keys older than 90 days, unused for 90 days, and any newly created key.
- **Auth failure rate** — a spike in `401`s means a rotation went wrong or an attack is running. Both are worth waking up for.
- **OIDC discovery and JWKS reachability** — if `/.well-known/...` is unreachable, every verifier fails as its cache ages out.
- **Break-glass credential use** — any use, page immediately.
- **Clock drift** — NTP offset above a threshold. This breaks TLS validity checks, JWT `exp`, Kerberos, TOTP, and SigV4, and it presents as five unrelated outages at once.

---
# Practice

# Part XVII — Troubleshooting playbook

Do not read this section end to end. Come here when something is broken.

## XVII.0 The triage order 🔴

Before diving into any specific error, four questions in this order:

```text
1. Which identity does the system think I am?
   aws sts get-caller-identity | kubectl auth whoami | vault token lookup | ssh -vvv
2. Is this AuthN (401) or AuthZ (403)?
   Read the response body, not just the code.
3. Is anything expired?
   Certificate dates, token exp, session, cached credentials. And: check the clock.
4. What changed?
   A rotation, a deploy, a certificate renewal, an IdP change, a version upgrade.
```

Question 1 solves more problems than anything else in this document. Most "permission" bugs are identity bugs — you are authenticated as someone other than who you assumed.

## XVII.1 TLS and certificates

| Symptom | Likely cause | Diagnose |
|---|---|---|
| `x509: certificate signed by unknown authority` | The CA is not in the client's trust store, **or the server did not send the intermediate** | `openssl s_client -connect host:443 -servername host -showcerts` — count the certificates returned |
| `SSL certificate problem: unable to get local issuer certificate` (curl 60) | Same as above | `curl -v --cacert /path/ca.pem https://host` to confirm |
| `x509: certificate has expired or is not yet valid` | Expired cert — **or the client's clock is wrong** | `openssl x509 -noout -dates -in cert.pem`; `timedatectl` / `chronyc tracking` |
| `x509: certificate is valid for X, not Y` | Hostname not in the SAN, or wrong SNI sent | `openssl x509 -noout -ext subjectAltName -in cert.pem` |
| `x509: certificate relies on legacy Common Name field` | No SAN at all | Reissue with SAN. Go removed CN fallback in 1.15. |
| `tls: handshake failure` / `no shared cipher` | Protocol or cipher mismatch; often TLS 1.0/1.1 disabled server-side | `openssl s_client -connect host:443 -tls1_2`; `nmap --script ssl-enum-ciphers -p443 host` |
| Works in browser, fails in app | Browser has the intermediate cached or fetches via AIA; the app does not | Serve the full chain |
| Works with curl, fails in Java/Python | Different trust store | Java: `keytool -list -cacerts`. Python: `python -c "import certifi;print(certifi.where())"` |
| `key values mismatch` on load | Certificate and private key are not a pair | Compare public key hashes (V.2) |
| Intermittent failures behind a load balancer | One backend has an old or wrong certificate | Test each backend IP with `curl --resolve host:443:<ip>` |
| Client cert rejected with no useful error | Missing `clientAuth` EKU, or the server does not trust the client's CA | `openssl x509 -noout -text \| grep -A1 'Extended Key Usage'` |

```bash
# the single most useful TLS command
$ openssl s_client -connect api.example.com:443 -servername api.example.com -showcerts </dev/null
# verify a chain manually
$ openssl verify -CAfile ca.pem -untrusted intermediate.pem leaf.pem
```

## XVII.2 SSH

| Symptom | Likely cause | Diagnose |
|---|---|---|
| `Permission denied (publickey)` | Key not offered, not in `authorized_keys`, or wrong user | `ssh -vvv user@host 2>&1 \| grep -i 'offering\|authentications that can continue'` |
| `Permission denied (publickey)` and the key *is* offered | Server-side file permissions | `~` must not be group-writable; `~/.ssh` 700; `authorized_keys` 600. Check `/var/log/auth.log` for `Authentication refused: bad ownership` |
| `Host key verification failed` | Host key changed — rebuild, or an actual MITM | Verify out of band, then `ssh-keygen -R host` |
| `Too many authentication failures` | The agent offers many keys and hits `MaxAuthTries` | `ssh -o IdentitiesOnly=yes -i ~/.ssh/specific_key user@host` |
| `Agent admitted failure to sign` | Key not loaded or agent not reachable | `ssh-add -l`; check `SSH_AUTH_SOCK` |
| Works interactively, fails in cron/CI | No agent, no TTY, different `HOME` | Use an explicit `-i` key path and `BatchMode=yes` |
| `sign_and_send_pubkey: no mutual signature algorithm` | New client, old server: RSA-SHA1 disabled | `-o PubkeyAcceptedKeyTypes=+ssh-rsa` as a stopgap; fix the server |

## XVII.3 HTTP APIs, tokens, JWT

| Symptom | Likely cause | Diagnose |
|---|---|---|
| `401` with no `WWW-Authenticate` | No credential sent at all | `curl -v` and look at the request headers actually sent |
| `401 invalid_token` | Expired, wrong issuer, bad signature | Decode the payload: check `exp`, `iss`, `aud` |
| `401` right after a deploy | The client cached a token across a signing-key rotation | Check the issuer's JWKS `kid` set against the token's `kid` |
| `403` with a valid token | Missing scope/claim/role | Decode and inspect `scope` and `groups` |
| `invalid audience` | Token minted for a different API | Request the right `audience`/`resource` |
| `unable to find key with kid X` | JWKS cache is stale, or the wrong issuer | `curl <jwks_uri> \| jq '.keys[].kid'` |
| Works for an hour, then fails | Token expiry with no refresh path | Check `exp`; implement refresh or re-auth |
| Works locally, fails in the cluster | Egress proxy stripping `Authorization`, or a different trust store | `kubectl exec` into the Pod and `curl -v` from there |
| Intermittent `401` under load | Multiple IdP replicas with unsynced keys, or clock drift between nodes | Compare `date -u` across nodes |

```bash
# decode a JWT payload
$ echo "$TOKEN" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq .
# check expiry against now
$ echo "$TOKEN" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null \
    | jq '{exp, now: now|floor, expired: (.exp < (now|floor))}'
```

## XVII.4 AWS

| Symptom | Likely cause | Diagnose |
|---|---|---|
| `InvalidClientTokenId` | Key deleted/deactivated, or `AWS_SESSION_TOKEN` missing with temporary creds | `aws sts get-caller-identity`; `aws configure list` |
| `SignatureDoesNotMatch` / `RequestTimeTooSkewed` | Clock drift, or a mangled secret key | `timedatectl`; re-copy the secret (watch for trailing whitespace) |
| `AccessDenied` on an API call | Permissions policy, SCP, resource policy, or boundary | Read the message — it names the principal and action. Then `simulate-principal-policy` |
| `is not authorized to perform: sts:AssumeRole` | **Trust** policy, not permissions policy | Read the role's trust policy |
| `Not authorized to perform sts:AssumeRoleWithWebIdentity` | `sub`/`aud` condition does not match the token's claims | Decode the OIDC token and diff its `sub` against the condition, character by character |
| `InvalidIdentityToken` | The OIDC provider is not registered in IAM, or the thumbprint is stale | `aws iam list-open-id-connect-providers` |
| Wrong account entirely | The credential chain picked something unexpected | `aws configure list` shows the source of each value |
| Console works, CLI does not | Console uses federated SSO, CLI uses stale static keys | `aws sso login`; remove old `~/.aws/credentials` entries |
| Credentials expire mid-job | Job exceeds the session duration | Raise `MaxSessionDuration`, or refresh mid-run |

## XVII.5 Kubernetes

| Symptom | Likely cause | Diagnose |
|---|---|---|
| `error: You must be logged in to the server (Unauthorized)` | AuthN failed: expired token, bad kubeconfig, or (EKS) **no access entry / aws-auth mapping** | `aws sts get-caller-identity` first, then `aws eks list-access-entries` |
| `User "X" cannot list resource "pods"` | AuthZ: RBAC. Note it tells you the identity it resolved | `kubectl auth can-i list pods --as=X -n ns` |
| Identity is `system:anonymous` | No credential reached the API server at all | `kubectl config view --minify` |
| Pod gets `403` from the API | The SA has no binding, or the token was not mounted | `kubectl auth can-i --list --as=system:serviceaccount:ns:sa` |
| `ImagePullBackOff` / `unauthorized` | Missing or wrong `imagePullSecret`; ECR token expired | `kubectl describe pod`; `kubectl get secret regcred -o jsonpath='{.data.\.dockerconfigjson}' \| base64 -d` |
| IRSA not working — SDK uses the node role | Missing SA annotation, missing `automount`, pod not restarted, or webhook not running | Check for `AWS_WEB_IDENTITY_TOKEN_FILE` in the Pod's env |
| IRSA — `AccessDenied` on AssumeRoleWithWebIdentity | Trust policy `sub` does not match `system:serviceaccount:<ns>:<sa>` | Decode `/var/run/secrets/eks.amazonaws.com/serviceaccount/token` inside the Pod |
| Webhook calls failing cluster-wide | Expired admission webhook serving certificate | `kubectl get validatingwebhookconfigurations -o yaml \| grep caBundle`; check cert-manager |
| Everything breaks after ~1 year on a kubeadm cluster | Control-plane certificates expired | `kubeadm certs check-expiration` |
| `kubectl` works, in-cluster client does not | The client is not using the projected token / CA | Check the SA mount and `KUBERNETES_SERVICE_HOST` |

```bash
# what identity, what groups
$ kubectl auth whoami
# inside a Pod, what does its token actually say
$ kubectl exec -it POD -- sh -c 'cat /var/run/secrets/kubernetes.io/serviceaccount/token' \
  | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq .
```

## XVII.6 Vault

| Symptom | Likely cause | Diagnose |
|---|---|---|
| `missing client token` | No `VAULT_TOKEN` / no login | `vault token lookup` |
| `permission denied` | Policy does not allow the path, or the token expired | `vault token capabilities <path>`; `vault token lookup` for TTL |
| `Vault is sealed` | Restarted without auto-unseal | `vault status`; unseal or fix the KMS auto-unseal |
| K8s auth fails: `service account name not authorized` | SA name/namespace/audience not in the role's bindings | `vault read auth/kubernetes/role/<role>` |
| K8s auth fails: `invalid audience` | The projected token's audience ≠ the role's audience | Decode the SA token's `aud` |
| JWT auth: `claim not found` / `not in bound claims` | `bound_claims` do not match the CI token | Decode the CI OIDC token and compare |
| Secrets vanish after an hour | The dynamic lease expired and nothing renewed it | `vault lease lookup`; add renewal or shorten the app's cache |

## XVII.7 SSO, OIDC, SAML

| Symptom | Likely cause | Diagnose |
|---|---|---|
| Redirect loop at login | Cookie not persisting (`SameSite`, `Secure` behind a terminating proxy), or clock skew | Browser devtools → Application → Cookies |
| `redirect_uri_mismatch` | Registered URI differs — scheme, port, trailing slash | Compare character by character with the IdP config |
| Login works, no permissions | Group/role claim missing or named differently | Decode the ID token and look at the actual claim names |
| `invalid_client` | Wrong client ID/secret, or wrong auth method (`client_secret_post` vs `basic`) | Check the IdP's client configuration |
| SAML: `invalid signature` | IdP signing certificate rotated | Re-import the IdP metadata |
| SAML: assertion rejected as expired | Clock skew | NTP on both sides |
| SAML: works from the IdP dashboard, not from the app | IdP-initiated only; SP-initiated not configured | Check the ACS URL and Entity ID |
| Everyone logged out at once | IdP session policy change, or signing key rotation | IdP sign-in logs |

## XVII.8 Git and CI

| Symptom | Likely cause | Diagnose |
|---|---|---|
| `Support for password authentication was removed` | Using a password over HTTPS | Use a PAT or SSH |
| `remote: Permission to org/repo.git denied` | Token lacks the scope, or the deploy key is read-only | Check the token's scopes / the key's write flag |
| CI works on `main`, fails on PRs | Secrets are not exposed to fork PRs (by design) | Restructure the workflow; do not use `pull_request_target` with untrusted code |
| GitHub OIDC: `Missing id-token permission` | `permissions: id-token: write` not set | Add it to the job or workflow |
| Argo CD `ComparisonError` on a repo | Repository credentials, not user login | `argocd repo list`; check the repo Secret |
| Argo CD sync fails with `forbidden` | The controller's SA lacks RBAC in the target namespace | Check the controller SA's ClusterRole |
| Action worked yesterday, fails today | A mutable tag on a third-party action moved | Pin actions to a commit SHA |

## XVII.9 When nothing makes sense 🔴

Run through this list. One of these is almost always the answer:

1. **Check the clock.** NTP failure breaks TLS validity, JWT `exp`, Kerberos, TOTP, and SigV4 simultaneously, producing five unrelated-looking outages.
2. **Check what identity you actually have**, not what you think you have.
3. **Check whether something expired** — including things with year-long lifetimes that you have never thought about.
4. **Check for a caching layer** — JWKS caches, DNS, credential caches (`~/.aws/cli/cache`, `~/.kube/cache`), agent state, CDN.
5. **Check for a proxy** stripping or adding headers, or a corporate MITM certificate.
6. **Check the full chain**, not just the endpoint you are testing.
7. **Test from inside**, not just from your laptop — `kubectl exec` and `curl` from the actual Pod.
8. **Read the whole error message.** Auth errors are unusually informative and unusually often skimmed.

---
# Part XVIII — Hands-on labs

Reading does not produce recall. Each lab has a **goal**, **steps**, a **done when** you can verify, and a **now break it** step — because recognising the error is the actual skill.

Everything here runs locally on Linux with Docker plus `kind` or `minikube`, except the AWS labs, which need an account (free tier is enough).

---

### Lab 1 — Basic auth and the base64 myth 🔴
**Goal:** internalise that Basic Auth is plaintext.

1. Run nginx with `auth_basic` and an `htpasswd` file (II.2).
2. `curl -u user:pass http://localhost:8080/` and capture it: `sudo tcpdump -i lo -A port 8080`.
3. Find the `Authorization` header in the capture and decode it with `base64 -d`.
4. Put nginx behind TLS and repeat the capture.

**Done when:** you have read your own password out of a packet capture, and confirmed it is unreadable over TLS.

---

### Lab 2 — Build a CA and issue certificates 🔴
**Goal:** understand chains by building one.

1. Create a root CA key and self-signed certificate.
2. Create an intermediate CA, sign it with the root.
3. Generate a server key + CSR **with a SAN**, sign it with the intermediate.
4. Serve it with `openssl s_server` or nginx; connect with `curl --cacert root.pem`.
5. `openssl verify -CAfile root.pem -untrusted intermediate.pem server.pem`.

**Now break it:** serve the leaf *without* the intermediate and connect. Read the exact error. Then set the system clock forward past `notAfter` and read that error. Then connect using an IP instead of the SAN hostname.

**Done when:** you can produce `unknown authority`, `certificate has expired`, and `valid for X, not Y` on demand and explain each.

---

### Lab 3 — mTLS end to end 🔴
1. Using Lab 2's CA, issue a **client** certificate with `clientAuth` EKU.
2. Configure nginx with `ssl_verify_client on` and `ssl_client_certificate`.
3. `curl --cert client.crt --key client.key --cacert ca.crt https://localhost:8443/`.
4. Have nginx pass `$ssl_client_s_dn` to a backend and print it.

**Now break it:** connect without the client certificate; connect with one signed by a different CA; connect with an expired one.

**Done when:** you can explain who verified what, in which direction, at which step of the handshake.

---

### Lab 4 — SSH keys, then SSH certificates 🔴
1. Container or VM. Key-only auth, `PasswordAuthentication no`.
2. Connect with `ssh -vvv` and find the lines showing which key was offered and accepted.
3. `chmod 777 ~/.ssh` on the server and watch it fail; find the reason in `/var/log/auth.log`.
4. Set up an SSH CA: `TrustedUserCAKeys`, sign a user key with `-V +5m`.
5. Connect, wait six minutes, connect again.

**Done when:** you have watched a certificate expire and understand why this is better than editing `authorized_keys` on 200 hosts.

---

### Lab 5 — Decode and verify a JWT by hand 🔴
1. Get a real JWT: `kubectl create token default`.
2. Decode the header and payload with `cut`/`base64`/`jq`.
3. Identify `iss`, `sub`, `aud`, `exp`, and the Kubernetes-specific claims.
4. Fetch the cluster's JWKS (`kubectl get --raw /openid/v1/jwks`) and match the `kid`.
5. Verify the signature with a library or `step crypto jwt verify`.

**Now break it:** flip one character in the payload and re-verify. Change `alg` to `none`.

**Done when:** you can decode any JWT from memory and list what a verifier must check.

---

### Lab 6 — Run an IdP and do a real OIDC flow 🔴
1. Run Keycloak or Dex in Docker.
2. Create a realm, a user, and a confidential client.
3. `curl` the discovery document and note every endpoint.
4. Complete an Authorization Code + PKCE flow **manually** — generate the verifier and challenge, paste the `/authorize` URL into a browser, grab the `code` from the callback, exchange it at `/token` with curl.
5. Decode the ID token and the access token and compare their claims.
6. Run a Client Credentials flow for a second, machine client.

**Done when:** you can explain the purpose of `state`, `nonce`, and `code_verifier` from having used them.

---

### Lab 7 — Kubernetes ServiceAccount identity 🔴
1. `kind create cluster`.
2. Create a namespace, an SA, a Pod using it.
3. Inside the Pod, `curl` the API server using the projected token and `ca.crt`. Observe the `403`.
4. Add a Role and RoleBinding for `pods: get,list`. Retry.
5. `kubectl auth can-i --list --as=system:serviceaccount:ns:sa`.
6. Set `automountServiceAccountToken: false` and confirm the token is gone.

**Done when:** you can state the Pod's exact username and groups, and produce both a 401 and a 403 deliberately.

---

### Lab 8 — GitHub Actions OIDC into AWS 🔴
1. Register GitHub's OIDC provider in an IAM account.
2. Create a role with a trust policy scoped to **one repo and one branch**.
3. Write the workflow from XV.3, including the step that prints the token's claims.
4. Run it; confirm `aws sts get-caller-identity` shows the assumed role.
5. Compare the printed `sub` with your trust policy condition.

**Now break it:** change the branch; change the `aud`; loosen `sub` to `repo:org/*` and think carefully about who could now assume that role.

**Done when:** you can set this up from scratch without a tutorial. This is the single most job-relevant lab here.

---

### Lab 9 — Vault with Kubernetes auth 🔴
1. Run Vault in dev mode inside `kind`.
2. Enable the KV engine, write a secret, write a policy.
3. Enable Kubernetes auth and create a role bound to one SA, one namespace, one audience.
4. From a Pod, exchange its projected SA token for a Vault token and read the secret.
5. `vault token lookup` on the resulting token.

**Now break it:** use the wrong namespace; use an SA token with the wrong audience.

**Done when:** you can explain why the Pod never held a Vault credential.

---

### Lab 10 — IRSA (or Pod Identity) on EKS 🟠
Needs a real EKS cluster; delete it afterwards.

1. Confirm the cluster's OIDC provider exists and is registered in IAM.
2. Create a role with an IRSA trust policy for one namespace/SA.
3. Annotate the SA, deploy a Pod with the AWS CLI, run `aws sts get-caller-identity`.
4. Inspect the Pod's env for `AWS_WEB_IDENTITY_TOKEN_FILE` and decode that token.
5. Repeat with Pod Identity and compare the setup effort.

**Done when:** you can name every component in the chain from Pod to S3 object.

---

### Lab 11 — Argo CD's three relationships 🟠
1. Install Argo CD in `kind`.
2. Log in as `admin`, then configure OIDC against the Keycloak from Lab 6, then disable `admin`.
3. Add a private git repo using a deploy key.
4. Add a second cluster (or a second namespace with restricted RBAC).
5. Deliberately break each of the three relationships in turn and record the exact error each produces.

**Done when:** given an Argo CD error, you can immediately say which of the three relationships is at fault.

---

### Lab 12 — Rotate a signing key without downtime 🟠
1. Using the Keycloak from Lab 6, note the current `kid` in the JWKS.
2. Issue a token; verify it in a small script that caches JWKS for 24 hours.
3. Rotate the realm's signing key.
4. Issue a new token and watch your script fail with `unknown kid`.
5. Fix the script: refresh JWKS when an unknown `kid` appears.

**Done when:** you understand why "we rotated a key and everything broke" happens, and how the overlap period is meant to work.

---

### Lab 13 — Leaked credential drill 🟠
1. Create a throwaway IAM user with an access key. Commit it to a **private** test repo.
2. Run `gitleaks detect` and confirm it is found.
3. Execute the runbook in XVI.4: deactivate, replace, find every use in CloudTrail, check for persistence.
4. Try to purge git history, then list all the places a copy could still exist.

**Done when:** you have internalised that rewriting history does not un-leak a secret, and that revocation is step one.

---

# Part XIX — Self-test question bank

Cover the answer. Answer out loud. If you hesitate, that topic goes back on the drill list.

## Foundations

**Q1.** A request returns `403`. Was authentication successful?
> Yes — the system identified you and then refused the action. `401` is the authentication failure. Caveat: some APIs return `403` for bad tokens too, so read the body.

**Q2.** Is a certificate a secret?
> No. The certificate is public. The private key is the secret.

**Q3.** What is the difference between scope and audience?
> Scope limits what the client may do. Audience limits which service may accept the token. Both must be checked.

**Q4.** What makes a credential "bearer"?
> Possession alone is sufficient to use it. No proof of being the legitimate holder is required.

**Q5.** Name three sender-constrained credentials.
> Client certificates (mTLS), SSH keys, WebAuthn credentials. Also mTLS-bound tokens (RFC 8705) and DPoP (RFC 9449).

## Passwords, keys, certificates

**Q6.** Why is SHA-256 a bad password hash?
> It is fast by design, so it is fast for an attacker too. Use a slow, memory-hard KDF: Argon2id, scrypt, bcrypt.

**Q7.** What does base64 protect in Basic Auth?
> Nothing. It is an encoding, trivially reversed. Only TLS protects the credential.

**Q8.** In SSH key auth, what crosses the network?
> A signature over a challenge. Never the private key.

**Q9.** What is `known_hosts` for, and how is it different from `authorized_keys`?
> `known_hosts` verifies the **server** to the client. `authorized_keys` verifies the **client** to the server. Opposite directions.

**Q10.** Which certificate field is used for hostname matching?
> SAN (Subject Alternative Name). CN is deprecated for this and modern clients ignore it.

**Q11.** You get `x509: certificate signed by unknown authority` but the CA is definitely in your trust store. What else?
> The server is probably not sending the intermediate certificate. Check with `openssl s_client -showcerts`.

**Q12.** What does TLS authenticate by default?
> The server, to the client. Not the client.

**Q13.** What does mTLS add, and how does the server learn the client's identity?
> Client authentication. The identity comes from the client certificate — SAN URI (SPIFFE ID), DNS SAN, or Subject CN.

**Q14.** Why can a Kubernetes client certificate not be revoked?
> Kubernetes performs no CRL/OCSP checking. The only remedy is rotating the cluster CA. Prefer short-lived tokens for humans.

## Tokens

**Q15.** Is a JWT encrypted?
> No. JWS is signed and readable by anyone holding it. Never put secrets in the payload.

**Q16.** Name the three JWT segments and what the first one is for.
> Header, payload, signature. The header carries `alg` and `kid` — the algorithm and which key to verify with.

**Q17.** Why must a verifier allowlist algorithms?
> To prevent `alg: none` and RS256→HS256 confusion attacks, where the attacker controls the algorithm and the verification key.

**Q18.** Why are access tokens short-lived?
> Because a JWT cannot be reliably revoked. Expiry is the revocation mechanism.

**Q19.** Where does a refresh token get sent?
> To the authorization server's token endpoint only. Never to the resource API.

**Q20.** Can you use an ID token to call an API?
> You should not. Its audience is the client application, not the API, and a correctly implemented API will reject it.

**Q21.** What is refresh token rotation and what does reuse detection buy you?
> A new refresh token on every use, invalidating the old. If an old one is replayed, the token was stolen — revoke the whole family.

**Q22.** Opaque vs JWT: which gives you instant revocation, and what do you pay for it?
> Opaque. You pay a network call to the issuer on every validation.

## OAuth / OIDC / SSO

**Q23.** Is OAuth 2.0 authentication?
> No. It is delegated authorization. OIDC is the authentication layer on top.

**Q24.** Which OAuth flow for a service with no user?
> Client Credentials.

**Q25.** Which flow for a CLI with no browser on the device?
> Device Authorization Grant (RFC 8628).

**Q26.** What does PKCE prevent?
> Use of an intercepted authorization code by an attacker who does not hold the `code_verifier`. Now required for all clients.

**Q27.** Why are Implicit and ROPC deprecated?
> Implicit exposes tokens in URLs (history, logs, referrers). ROPC gives the client the user's password, defeating the point and blocking MFA. Both are excluded by RFC 9700 / OAuth 2.1.

**Q28.** What does OIDC add to OAuth?
> The `openid` scope, the ID token, UserInfo, discovery, standard claims, and `nonce`.

**Q29.** Where do you find an OIDC provider's endpoints and keys?
> `/.well-known/openid-configuration`, which points at `jwks_uri`.

**Q30.** Is SSO a protocol?
> No. It is an architecture, implemented with OIDC or SAML.

**Q31.** What is the biggest operational risk of SSO, and what mitigates it?
> The IdP becomes a single point of failure. Mitigate with a tested, alerted, offline break-glass credential.

**Q32.** What breaks SAML most often?
> Clock skew, IdP signing certificate rollover, ACS/Entity ID mismatch, attribute name mismatch.

## AWS

**Q33.** What is the difference between a role's trust policy and its permissions policy?
> Trust says **who may assume** the role. Permissions say **what the role may do**. `AssumeRole` failures are trust; API denials are permissions.

**Q34.** What beats everything in IAM evaluation?
> An explicit `Deny`, anywhere.

**Q35.** Three values in a temporary credential set?
> Access key ID (`ASIA...`), secret access key, and the session token. Omitting the session token yields `InvalidClientTokenId`.

**Q36.** First command when AWS auth misbehaves?
> `aws sts get-caller-identity`, then `aws configure list`.

**Q37.** How does GitHub Actions authenticate to AWS with no stored keys?
> GitHub signs an OIDC JWT describing the run; the job calls `sts:AssumeRoleWithWebIdentity`; AWS verifies against GitHub's JWKS and the trust policy's `sub`/`aud` conditions.

**Q38.** In that trust policy, where does the security actually live?
> The `sub` condition. `repo:org/*` is far too broad; pin the exact repo plus ref or environment.

**Q39.** Why is IMDSv2 important?
> IMDSv1 is a plain GET, so any SSRF in an app can steal the instance role credentials. IMDSv2 requires a PUT-obtained session token, and the hop limit blocks containers.

**Q40.** What causes `SignatureDoesNotMatch` most often?
> Clock skew, or a corrupted/whitespace-padded secret key.

## Kubernetes

**Q41.** How does Kubernetes store user accounts?
> It does not. There are no user objects; human identity always comes from certificates, OIDC, or a webhook authenticator.

**Q42.** Full username of a ServiceAccount?
> `system:serviceaccount:<namespace>:<name>`.

**Q43.** Difference between legacy and bound ServiceAccount tokens?
> Legacy: a Secret, never expires, no audience. Bound: projected into the Pod, ~1 h, auto-rotated, audience-bound, tied to the Pod's lifetime.

**Q44.** Does RBAC support deny rules?
> No. It is purely additive; access is the union of all applicable bindings.

**Q45.** What does a RoleBinding pointing at a ClusterRole do?
> Grants that ClusterRole's permissions **only within the binding's namespace**. The standard way to reuse `view`/`edit`/`admin` per team.

**Q46.** `You must be logged in to the server (Unauthorized)` on EKS — what is it usually?
> The IAM principal has no EKS access entry (or `aws-auth` mapping). It is authentication, not RBAC, and not usually wrong AWS credentials.

**Q47.** How does IRSA differ from Pod Identity?
> IRSA uses OIDC federation with a per-cluster issuer in each role's trust policy, and works on any Kubernetes. Pod Identity uses an EKS association plus a node agent, trusts one service principal so roles are reusable across clusters, supports session tags, and is EKS-specific.

**Q48.** Why disable `automountServiceAccountToken`?
> Most Pods never call the API server, and the mounted token is a live credential that a compromised container could use.

## Vault, workload identity, pipeline

**Q49.** What is the secret-zero problem, and how does workload identity solve it?
> How does a workload authenticate the first time without a secret? The platform attests it — Kubernetes signs a token stating which SA and Pod it is, and that assertion is exchanged for credentials.

**Q50.** How does a Pod authenticate to Vault with no Vault credential?
> Kubernetes auth: it presents its projected SA token; Vault validates it with the cluster and checks the SA name, namespace, and audience against a role.

**Q51.** What is Vault response wrapping for?
> Delivering a secret via an intermediary that must not read it. Single-use; if it is unwrapped in transit, the legitimate unwrap fails, so interception is detectable.

**Q52.** Best credential for machine access to a git repo?
> A GitHub App installation token (short-lived, scoped, not tied to a person), or a per-repo deploy key. Not a personal PAT.

**Q53.** Why must third-party GitHub Actions be pinned to a SHA?
> Tags are mutable. A compromised or repointed tag executes attacker code with your workflow's permissions.

**Q54.** Name Argo CD's three authentication relationships.
> User→Argo CD, Argo CD→git, Argo CD→Kubernetes API. Independent mechanisms, independent failures.

**Q55.** What does keyless cosign signing actually prove?
> That a specific workflow in a specific repository, identified by its OIDC identity, produced the artifact — recorded in a transparency log. No signing key is stored.

## Operations

**Q56.** Steps of a safe rotation?
> Create new alongside old → deploy consumers → **verify traffic moved** → disable old → wait → delete.

**Q57.** Which credential types cannot be meaningfully revoked?
> JWT access tokens before `exp`, and Kubernetes client certificates. Both are arguments for short lifetimes.

**Q58.** First action on a leaked credential?
> Revoke it. Immediately, before investigating.

**Q59.** What do you check for after a credential compromise that people forget?
> Persistence: new IAM users or keys, new SSH keys, new OAuth grants, new webhooks, altered trust policies, new RBAC bindings.

**Q60.** One clock is wrong. Name five things that break.
> TLS certificate validity, JWT `exp`/`nbf`, Kerberos tickets, TOTP codes, AWS SigV4 signatures. Also SAML assertions.

---
# Part XX — Master drill checklist

One canonical list. Everything the old document spread across four overlapping sections lives here, tagged by priority and ordered so each topic builds on the last.

**How to use it:** work top to bottom. Tick a box only when you can explain the topic out loud *and* run its command. Re-run the 🔴 list monthly until it is boring.

## Stage 1 — Foundations 🔴

- [ ] Authentication vs authorization; `401` vs `403`
- [ ] Identity, principal, credential, secret, key, certificate, token, claim
- [ ] Scope vs audience
- [ ] The eight questions (I.3)
- [ ] `Authorization` header and `WWW-Authenticate`
- [ ] Bearer vs sender-constrained
- [ ] The credential lifetime spectrum; why short-lived wins
- [ ] Human vs workload identity

## Stage 2 — Shared secrets 🔴🟠

- [ ] Password hashing: salt, Argon2id/bcrypt/scrypt, why not SHA-256 🔴
- [ ] Basic Auth; base64 is not encryption; requires HTTPS 🔴
- [ ] `htpasswd`, nginx `auth_basic` 🔴
- [ ] Digest Auth — recognise it, do not invest 🟠
- [ ] API keys: vs passwords, vs access tokens 🔴
- [ ] API key storage, rotation, scoping, never in git or images 🔴
- [ ] Sessions: server-side state, session stores, revocation 🔴
- [ ] Cookies: `Secure`, `HttpOnly`, `SameSite`, `Domain`, `__Host-` 🔴
- [ ] Session fixation; new session ID on privilege change 🟠

## Stage 3 — Keys and SSH 🔴

- [ ] Sign vs encrypt vs key agreement
- [ ] Ed25519 / ECDSA / RSA — when each
- [ ] `ssh-keygen`, `authorized_keys`, key permissions
- [ ] `ssh-agent`, `ssh-add -t`, why not agent forwarding
- [ ] `known_hosts` and host key verification (opposite direction to `authorized_keys`)
- [ ] `ssh -vvv` debugging
- [ ] `authorized_keys` options: `command=`, `from=`, `no-*`
- [ ] SSH certificates and a CA; why they scale 🟠
- [ ] Bastion patterns: `ProxyJump`, session recording 🟠

## Stage 4 — PKI and TLS 🔴

- [ ] Root CA → intermediate → leaf; the trust store
- [ ] X.509 fields; **SAN, not CN**
- [ ] CSR, issuance, serving the full chain
- [ ] `openssl s_client`, `openssl x509`, `openssl verify`
- [ ] Matching a key to a certificate
- [ ] Revocation: CRL, OCSP, stapling — and why short lifetimes replaced them 🟠
- [ ] TLS 1.2 vs 1.3; handshake round trips; forward secrecy
- [ ] SNI, ALPN, session resumption, 0-RTT replay
- [ ] Termination vs passthrough vs re-encryption
- [ ] Certificate expiry monitoring
- [ ] cert-manager: Issuer, Certificate, HTTP-01 vs DNS-01 🟠

## Stage 5 — mTLS 🔴

- [ ] What `CertificateRequest`/`CertificateVerify` do
- [ ] How the server derives client identity from a certificate
- [ ] `curl --cert --key --cacert`
- [ ] nginx `ssl_verify_client`; stripping spoofable identity headers
- [ ] Where mTLS lives: mesh, etcd, kubelet, Kafka
- [ ] Rotation at scale — the actual hard part
- [ ] SPIFFE ID format, SVIDs, the Workload API 🟠

## Stage 6 — Tokens and JWT 🔴

- [ ] Opaque vs self-contained; the revocation trade-off
- [ ] Token introspection (RFC 7662) 🟠
- [ ] JWT structure; decoding one by hand
- [ ] Registered claims: `iss`, `sub`, `aud`, `exp`, `nbf`, `iat`, `jti`
- [ ] `alg` values; symmetric vs asymmetric
- [ ] The full verification checklist
- [ ] `alg: none` and algorithm confusion attacks
- [ ] JWKS, `kid`, key rotation, cache staleness
- [ ] Bearer semantics; never in URLs or logs
- [ ] Access vs refresh vs ID token — audience, lifetime, destination
- [ ] Refresh token rotation and reuse detection
- [ ] DPoP / mTLS-bound tokens 🟠

## Stage 7 — OAuth and OIDC 🔴

- [ ] What OAuth solves; the four roles
- [ ] The endpoints, including discovery
- [ ] Confidential vs public clients
- [ ] Authorization Code + PKCE, parameter by parameter
- [ ] `state` vs `nonce` vs `code_verifier`
- [ ] Client Credentials
- [ ] Device Authorization Grant
- [ ] Implicit and ROPC — why they are dead
- [ ] Scope, audience, resource indicators
- [ ] What OIDC adds; the ID token
- [ ] Discovery document and JWKS in practice
- [ ] Claims → groups → RBAC mapping

## Stage 8 — SSO and enterprise 🟠

- [ ] SSO as architecture, not protocol
- [ ] IdP as single point of failure; break-glass
- [ ] SP-initiated vs IdP-initiated; SLO limitations
- [ ] SAML: assertion, ACS, Entity ID, metadata
- [ ] The four things that break SAML
- [ ] LDAP: bind, DN, search-then-bind, `memberOf`
- [ ] Kerberos: KDC, TGT, service ticket, keytab, SPN, clock skew 🟢

## Stage 9 — MFA 🔴

- [ ] Three factor categories
- [ ] TOTP mechanics; clock sensitivity; recovery codes
- [ ] Push fatigue and number matching
- [ ] Why SMS is weak
- [ ] WebAuthn/passkeys and origin binding = phishing resistance 🟠
- [ ] MFA does not apply to workloads
- [ ] The programmatic-access gap (`aws:MultiFactorAuthPresent`)

## Stage 10 — AWS 🔴

- [ ] Users, roles, identity/resource/trust policies, boundaries, SCPs
- [ ] Policy evaluation order; explicit deny wins
- [ ] Access keys vs temporary credentials; the session token
- [ ] The SDK credential chain, in order
- [ ] `sts:AssumeRole` vs `AssumeRoleWithWebIdentity` vs `AssumeRoleWithSAML`
- [ ] Session names and CloudTrail
- [ ] External ID for third-party access 🟠
- [ ] OIDC federation: register provider → trust policy → conditions
- [ ] `sub` conditions as the real security boundary
- [ ] IAM Identity Center, `aws sso login`, profiles 🟠
- [ ] IMDSv1 vs IMDSv2, hop limit, SSRF
- [ ] `aws sts get-caller-identity`, `aws configure list`, `simulate-principal-policy`

## Stage 11 — Kubernetes 🔴

- [ ] The authenticator chain; there are no user objects
- [ ] `kubectl auth whoami`, `can-i`, `--as`
- [ ] kubeconfig anatomy: certs vs token vs `exec` plugin
- [ ] Client certificates cannot be revoked
- [ ] ServiceAccount username and groups
- [ ] Legacy vs bound/projected tokens; TokenRequest API
- [ ] `kubectl create token --audience --duration`
- [ ] `automountServiceAccountToken: false`
- [ ] Projected token volumes with an audience
- [ ] Role / ClusterRole / RoleBinding / ClusterRoleBinding
- [ ] RBAC is additive; RoleBinding→ClusterRole pattern
- [ ] Escalation paths: secrets, pod create, impersonate, escalate/bind
- [ ] IRSA: OIDC provider, annotation, web identity token file
- [ ] Pod Identity: association, agent, service principal, session tags
- [ ] Choosing between them
- [ ] EKS access entries vs `aws-auth`
- [ ] Registry auth, `imagePullSecrets`, ECR 12-hour tokens
- [ ] Control-plane certificate expiry (`kubeadm certs check-expiration`) 🟠
- [ ] Secrets are base64, not encrypted; encryption at rest

## Stage 12 — Workload identity 🔴

- [ ] The pattern: attested identity → exchange → short-lived credential
- [ ] The secret-zero problem
- [ ] Per-platform attestation mechanisms
- [ ] Cloud mapping table (EC2/ECS/EKS/Lambda/GCP/Azure)
- [ ] Tracing a multi-hop token exchange chain

## Stage 13 — Vault 🔴

- [ ] auth → token → policy → secret
- [ ] Auth methods and when each applies
- [ ] Kubernetes auth: bound SA, namespace, audience
- [ ] JWT auth for CI: `bound_claims`
- [ ] AppRole: `role_id` + `secret_id`, response wrapping 🟠
- [ ] Policy HCL; `deny` wins; `token capabilities`
- [ ] TTL, max TTL, renewal, revocation, accessors
- [ ] Service vs batch tokens 🟠
- [ ] Dynamic secrets and leases
- [ ] Injection options: Agent, CSI, ESO, SOPS, Sealed Secrets

## Stage 14 — Pipeline 🔴

- [ ] Git auth matrix: SSH, deploy key, PAT, fine-grained PAT, GitHub App, `GITHUB_TOKEN`
- [ ] Offboarding risk of person-owned automation credentials
- [ ] Repository vs environment vs organization secrets
- [ ] Fork PRs and `pull_request_target`
- [ ] Pinning actions to a SHA
- [ ] `permissions: id-token: write` and the full OIDC→AWS flow
- [ ] Inspecting your own OIDC claims
- [ ] Argo CD's three relationships and their distinct failures
- [ ] Argo CD RBAC via IdP groups
- [ ] Terraform: provider vs backend auth; secrets in state
- [ ] Keyless signing with cosign 🟠

## Stage 15 — Operations 🔴

- [ ] Two-key overlap rotation, with verification
- [ ] Revocation path per credential type
- [ ] Secret scanning: pre-commit, CI, push protection, images
- [ ] The leaked-credential runbook, including persistence checks
- [ ] Audit sources per system
- [ ] Least privilege tooling and aggregate-permission review
- [ ] Monitoring: cert expiry, key age, auth failure rate, JWKS reachability, clock drift

## Stage 16 — Troubleshooting fluency 🔴

- [ ] Produce and explain each of the main `x509` errors on demand
- [ ] Diagnose `Permission denied (publickey)` in three different ways
- [ ] Decode any JWT and state why a verifier rejected it
- [ ] Distinguish an AWS trust-policy failure from a permissions failure by the error text
- [ ] Distinguish Kubernetes `Unauthorized` from `forbidden` and act accordingly
- [ ] Identify which of Argo CD's three relationships failed from the error alone
- [ ] Run the XVII.9 checklist from memory

---

# Appendix A — Command reference

```bash
### Identity: who am I, right now
aws sts get-caller-identity
aws configure list
kubectl auth whoami
kubectl auth can-i --list -n prod
vault token lookup
gh auth status
ssh -T git@github.com

### JWT
echo "$TOKEN" | cut -d. -f2 | tr '_-' '/+' | base64 -d 2>/dev/null | jq .
echo "$TOKEN" | cut -d. -f1 | tr '_-' '/+' | base64 -d 2>/dev/null | jq .
kubectl create token my-sa -n prod --audience=vault --duration=10m

### TLS / certificates
openssl s_client -connect host:443 -servername host -showcerts </dev/null
openssl s_client -connect host:443 -servername host </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
openssl x509 -in cert.pem -noout -text
openssl req  -in req.csr  -noout -text
openssl verify -CAfile ca.pem -untrusted chain.pem leaf.pem
openssl x509 -noout -pubkey -in cert.pem | openssl sha256   # compare with:
openssl pkey  -pubout    -in key.pem     | openssl sha256
curl -v --cacert ca.pem https://host
curl --cert client.crt --key client.key --cacert ca.crt https://host:8443/
curl --resolve host:443:10.0.0.5 https://host/               # test one backend

### SSH
ssh-keygen -t ed25519 -C "user@host"
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host
ssh -vvv user@host
ssh-add -l ; ssh-add -t 3600 ~/.ssh/id_ed25519
ssh-keyscan -t ed25519 host >> ~/.ssh/known_hosts
ssh-keygen -R host
ssh-keygen -Lf cert.pub

### OIDC / OAuth
curl -s https://ISSUER/.well-known/openid-configuration | jq .
curl -s "$(curl -s https://ISSUER/.well-known/openid-configuration | jq -r .jwks_uri)" | jq '.keys[].kid'
curl -s -X POST https://ISSUER/oauth2/token \
  -d grant_type=client_credentials -d client_id=... -d client_secret=... -d scope=...

### AWS
aws sts assume-role --role-arn ARN --role-session-name NAME
aws iam simulate-principal-policy --policy-source-arn ARN --action-names s3:GetObject --resource-arns ARN
aws iam list-open-id-connect-providers
aws eks list-access-entries --cluster-name prod
aws ecr get-login-password | docker login --username AWS --password-stdin ACCOUNT.dkr.ecr.REGION.amazonaws.com
TOKEN=$(curl -sX PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 300")

### Kubernetes
kubectl config view --minify --raw
kubectl auth can-i get pods --as=system:serviceaccount:prod:api -n prod
kubectl get --raw /openid/v1/jwks | jq '.keys[].kid'
kubectl create secret docker-registry regcred --docker-server=... --docker-username=... --docker-password=...
kubeadm certs check-expiration

### Vault
vault status
vault login -method=oidc
vault token capabilities secret/data/prod/db
vault read auth/kubernetes/role/api
vault lease revoke -prefix database/creds/

### Scanning
gitleaks detect --source .
trufflehog filesystem .
```

---

# Appendix B — Cheat sheet

| Term | One line |
|---|---|
| Authentication | Prove who you are |
| Authorization | What you may do |
| Principal | The identity the authorizer sees |
| Credential | The proof you present |
| Claim | One assertion inside a token |
| Basic Auth | base64(user:pass) in a header; needs HTTPS |
| Digest Auth | Challenge-response password auth; obsolete |
| API key | Static shared secret identifying a client |
| Session | Server-side login state, usually keyed by a cookie |
| Cookie | Browser storage/transport, commonly carrying a session ID |
| TLS | Encrypted, integrity-checked transport with server authentication |
| SNI | Hostname sent in the clear so the server picks a certificate |
| Certificate | Signed binding of an identity to a public key |
| SAN | The certificate field used for hostname matching |
| CA / PKI | The signer, and the trust system around it |
| mTLS | Both sides authenticate with certificates |
| SPIFFE ID | Standard URI naming a workload |
| SSH key | Public/private pair; a signature proves possession |
| SSH certificate | Short-lived, CA-signed SSH identity |
| Opaque token | Random string; validated by asking the issuer |
| JWT | Signed, self-describing token; readable, not secret |
| JWKS | The issuer's public keys, published for verification |
| Bearer | Possession is sufficient to use it |
| Access token | Credential for calling an API |
| Refresh token | Credential for getting a new access token |
| ID token | OIDC identity receipt for the client — not an API credential |
| OAuth 2.0 | Delegated authorization framework |
| PKCE | Binds an authorization code to the client that requested it |
| OIDC | Authentication and identity layer on OAuth 2.0 |
| SSO | One login across many systems; built on OIDC or SAML |
| SAML | XML enterprise federation protocol |
| LDAP | Directory protocol; authentication via bind |
| Kerberos | Ticket-based authentication; clock-sensitive |
| MFA | Factors from different categories |
| TOTP | 30-second time-based code from a shared secret |
| WebAuthn / passkey | Origin-bound public-key credential; phishing-resistant |
| IAM role | AWS identity with no credentials, assumed temporarily |
| Trust policy | Who may assume a role |
| STS | AWS temporary credential service |
| IMDS | Instance metadata; source of node credentials; use v2 |
| ServiceAccount | Kubernetes workload identity |
| Bound SA token | Short-lived, audience-bound, auto-rotated Pod token |
| RBAC | Additive, deny-free authorization in Kubernetes |
| IRSA | Pod → IAM via cluster OIDC federation |
| Pod Identity | Pod → IAM via an EKS association and node agent |
| Workload identity | Platform-attested identity instead of a stored secret |
| Vault auth method | How a principal proves identity to Vault |
| Vault token | Credential issued after Vault authentication, bound to policies |
| Dynamic secret | Credential Vault creates on demand and destroys at lease end |
| Deploy key | SSH key scoped to one repository |
| GitHub App token | Short-lived, scoped, person-independent git automation credential |

---

# Appendix C — Confusable pairs

Being able to state the difference in one sentence is the real test.

| Pair | The difference |
|---|---|
| AuthN / AuthZ | Who you are / what you may do |
| `401` / `403` | Credential rejected / identity refused |
| Hashing / encryption | One-way / reversible with a key |
| Encoding / encryption | Reversible by anyone / requires a key |
| Certificate / private key | Public statement / the actual secret |
| Key / certificate | Raw cryptographic material / a signed identity claim about a public key |
| CN / SAN | Legacy subject name / the field actually matched |
| TLS / mTLS | Server authenticated / both authenticated |
| `authorized_keys` / `known_hosts` | Client → server trust / server → client trust |
| Bearer / JWT | How it is presented / what format it is |
| Opaque / self-contained token | Ask the issuer / verify locally |
| Access / ID token | For the API / for the client |
| Access / refresh token | Use the resource / get a new access token |
| Scope / audience | What you may do / who may accept it |
| OAuth / OIDC | Authorization / authentication |
| OIDC / SSO | A protocol / an architecture built with it |
| SAML / OIDC | XML enterprise federation / JSON-JWT federation |
| Session / JWT | Server-side state, revocable / stateless, expires |
| Trust policy / permissions policy | Who may assume / what it may do (AWS) |
| IAM user / IAM role | Long-lived identity / assumed temporarily |
| Role / policy | An identity / the rules attached to it |
| IRSA / Pod Identity | OIDC federation, portable / EKS association, simpler |
| Role / ClusterRole | Namespaced / cluster-scoped |
| RoleBinding→ClusterRole | Cluster-scoped rules applied in one namespace |
| Legacy / bound SA token | Never expires, no audience / short-lived, audience-bound |
| Vault auth method / policy | How you prove identity / what you may then access |
| Deploy key / PAT | One repository / everything the user can reach |
| API key / access token | Static, long-lived / issued, short-lived, scoped |

---

# Appendix D — Primary sources

Prefer specifications and vendor documentation over blog posts. When a blog and an RFC disagree, the RFC wins.

**Core specifications**

| Topic | Spec |
|---|---|
| HTTP authentication framework | RFC 7235 |
| Basic / Digest | RFC 7617 / RFC 7616 |
| OAuth 2.0 | RFC 6749 |
| Bearer token usage | RFC 6750 |
| PKCE | RFC 7636 |
| Device Authorization Grant | RFC 8628 |
| Token introspection / revocation | RFC 7662 / RFC 7009 |
| Authorization server metadata | RFC 8414 |
| Resource indicators | RFC 8707 |
| **OAuth 2.0 Security BCP** | **RFC 9700** (Jan 2025 — deprecates Implicit and ROPC, mandates PKCE) |
| mTLS-bound tokens / DPoP | RFC 8705 / RFC 9449 |
| JWT / JWS / JWK | RFC 7519 / 7515 / 7517 |
| **JWT Best Current Practices** | **RFC 8725** |
| X.509 | RFC 5280 |
| TLS 1.3 / TLS 1.2 | RFC 8446 / RFC 5246 |
| HOTP / TOTP | RFC 4226 / RFC 6238 |
| Kerberos v5 | RFC 4120 |
| LDAP | RFC 4511 |
| OpenID Connect Core | `openid.net/specs/openid-connect-core-1_0.html` |
| SAML 2.0 | OASIS SAML 2.0 specifications |
| WebAuthn | W3C Web Authentication Level 2/3 |
| SPIFFE | `spiffe.io` specifications |

**Vendor and project documentation**

- Kubernetes — Authenticating; Managing Service Accounts; RBAC Authorization; Certificate Signing Requests (`kubernetes.io/docs/reference/access-authn-authz/`)
- AWS — IAM User Guide (policy evaluation logic, roles, OIDC federation); EKS User Guide (IRSA, Pod Identity, access entries); STS API reference
- HashiCorp Vault — Auth methods, policies, Kubernetes auth, AppRole, response wrapping
- GitHub — Security hardening for GitHub Actions; About security hardening with OpenID Connect
- Argo CD — User Management, Declarative Setup, RBAC
- cert-manager — Concepts and configuration
- Istio / Linkerd — Security and mTLS documentation
- OWASP — Authentication, Session Management, and Secrets Management Cheat Sheets
- Envoy — JWT authentication and RBAC filter docs

**Reference implementations worth reading**

- `aws-actions/configure-aws-credentials` — how the OIDC → STS exchange is actually done
- `hashicorp/vault-action` — JWT auth from CI
- `external-secrets/external-secrets` — secret sync patterns
- `spiffe/spire` — workload attestation in practice
- `sigstore/cosign` — keyless signing via OIDC
- Kubernetes `kubeadm` certificate management code — how cluster PKI is generated and renewed

---

*End of document. If a section here disagrees with the vendor documentation you are working against, trust the vendor and update this file.*
