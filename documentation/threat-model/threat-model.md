Threat Modelling & Risk Assessment
OWASP Juice Shop: DevSecOps Security Pipeline Project

Module: IE3142 DevOps Security Prepared by: Amarapathi A.M.P.D. (IT24103045), Member 2, Threat Modelling & Risk Assessment Application: OWASP Juice Shop (Node.js/Express REST API + Angular SPA served by the same Node process, SQLite via Sequelize, MarsDB for reviews/orders) Inputs: Architecture diagram by Member 1 (diagrams/, documentation/)

1. Methodology

STRIDE was applied to the architecture diagram prepared by Member 1. Each of the six STRIDE categories was considered at each trust boundary (Section 2). From the resulting candidates, five threats were selected. T1–T4 were chosen because they are (a) present in the unmodified Juice Shop code, (b) intended to be demonstrated as working exploits by Member 3, and (c) fixable with a standard secure-coding control. T5 is a supply-chain threat: it is evidenced by the project's own npm audit and Trivy output rather than by an exploit, and it links the model to Member 4's dependency and container gates. This gives a direct handoff to Member 3 (exploit-and-fix) and Member 4 (CI/CD gates).

Juice Shop runs as a single Node.js container: Express serves both the REST API and the compiled Angular bundle, and the Angular code executes in the user's browser. The trust boundaries below follow that reality.

Boundary	Description
TB1: Internet / Browser → Node container	Untrusted users and their browsers reach the Express server (port 3000) over HTTP. All client-side code and all request data are untrusted.
TB2: Express application → Data layer	Backend code queries SQLite (users, baskets, products via Sequelize) and MarsDB (reviews, orders).
TB3: Source repository / CI pipeline → Container image	Dependencies (npm), the base image and the Docker build flow from GitHub into the running image (supply-chain boundary).

If Member 1's final diagram labels the boundaries differently, keep the names from the diagram and adjust the labels above; the threat analysis itself does not change.

2. STRIDE Analysis per Trust Boundary
STRIDE	TB1: Internet → Node container	TB2: Express → Data layer	TB3: Repo/CI → Image
Spoofing	Token forgery using the exposed hardcoded signing key; login bypass by injection (T1, T4)	n/a (same process)	Malicious/typosquatted package impersonating a legitimate one
Tampering	Modifying another user's basket via ID manipulation (T3)	SQL injection altering query logic (T1)	Vulnerable/tampered dependency in the image (T5)
Repudiation	No audit log of logins or orders, so actions cannot be attributed (identified, not selected)	No record of data changes (identified, not selected)	Unsigned images, no provenance (identified, not selected)
Information Disclosure	Script injection stealing tokens (T2); other users' baskets (T3)	Injection returning unauthorised rows (T1)	Hardcoded signing key committed to source (T4)
Denial of Service	Unlimited login/search requests exhaust the server (identified, not selected; partly addressed by T4 rate limiting)	Expensive queries / NoSQL operator abuse (identified, not selected)	n/a
Elevation of Privilege	Brute-forced admin account; forged token if the exposed key is abused (T4)	Injection logging in as admin (T1)	Vulnerable dependency giving code execution (T5)

Repudiation and DoS threats were identified but not selected for exploit-and-fix, because Member 3 must show an exploit working, being fixed and being blocked on re-test; the four selected exploit-and-fix threats (T1–T4) fit that requirement cleanly, while T5 is validated through dependency and container scan evidence.

3. Selected Threats: Summary
#	Threat	STRIDE	Boundary	Likelihood	Impact	Score	Rating
T1	Authentication bypass via SQL injection	Spoofing, Elevation of Privilege	TB1 → TB2	5	5	25	Critical
T2	DOM-based XSS in product search	Information Disclosure	TB1	4	4	16	Critical
T3	Broken access control (IDOR on baskets)	Information Disclosure, Tampering	TB1	3	4	12	High
T4	Broken authentication (no login rate limiting, hardcoded JWT signing key)	Spoofing, Elevation of Privilege	TB1, TB3	4	4	16	Critical
T5	Vulnerable dependencies / container image	Tampering, Elevation of Privilege	TB3	4	3	12	High

Rating bands: 1–4 Low · 5–9 Medium · 10–14 High · 15–25 Critical Score = Likelihood × Impact (5×5 scale).

4. Detailed Threat Analysis
T1: SQL Injection → Authentication Bypass
Field	Detail
STRIDE	Spoofing, Elevation of Privilege
Attack scenario	The login handler builds its SQL by string interpolation of the email field. An attacker submits ' OR 1=1-- as the email with any password; the condition becomes always true and the first user returned (normally the administrator) is logged in without valid credentials.
Asset at risk	All user accounts, admin privileges, application data
Likelihood: 5	Login form is public, unauthenticated, and the payload is trivial and widely known.
Impact: 5	Full account takeover including admin, with complete loss of confidentiality and integrity.
Risk	25, Critical
Mitigating control	Parameterised queries (Sequelize replacements/bind or findOne({ where })) plus server-side validation of email format.
Control location	application/juice-shop/routes/login.ts
Pipeline control	Semgrep SAST gate (SQL injection rules) in .github/workflows/
Verification	Re-send the same payload after the fix; expect an "Invalid email or password" response (HTTP 401). Semgrep should report the finding before and none after.
T2: DOM-based Cross-Site Scripting (Search)
Field	Detail
STRIDE	Information Disclosure
Attack scenario	The search page in the Angular frontend renders the search query as trusted HTML (bypassSecurityTrustHtml), which disables Angular's built-in sanitiser. An attacker crafts a link such as /#/search?q=<iframe src="javascript:alert(1)">; when a victim opens it, the script runs in the victim's browser and can read the JWT/session data held in the browser.
Asset at risk	Session tokens (JWT), the victim's browser context and account
Likelihood: 4	Exploitation needs only a crafted URL delivered to a victim, with no authentication and no special tooling.
Impact: 4	Session hijacking and actions performed as the victim.
Risk	16, Critical
Mitigating control	Remove bypassSecurityTrustHtml and bind the value as text so Angular sanitises it; add a Content-Security-Policy header.
Control location	application/juice-shop/frontend/src/app/search-result/search-result.component.ts (and its template); CSP set in application/juice-shop/server.ts
Pipeline control	Semgrep SAST gate (Angular/JS XSS rules)
Verification	Re-open the same crafted URL; the payload must appear as inert text and no script executes.

Note for Member 3: a plain <script> tag injected through innerHTML is not executed by browsers, so use the <iframe src="javascript:alert(1)"> payload for the demonstration.

T3: Broken Access Control (IDOR on Basket)
Field	Detail
STRIDE	Information Disclosure, Tampering
Attack scenario	An authenticated attacker changes the basket ID in GET /rest/basket/:id from their own to another user's. The backend returns the other user's basket because it does not enforce that the basket belongs to the requesting user.
Asset at risk	Other users' basket/order data and personal information
Likelihood: 3	Requires a registered account and ID guessing, but IDs are sequential and no special tooling is needed.
Impact: 4	Direct exposure and modification of another user's private data, breaking isolation between accounts.
Risk	12, High
Mitigating control	Ownership check comparing the authenticated user's basket ID with the requested ID; return 403 Forbidden on mismatch.
Control location	application/juice-shop/routes/basket.ts
Pipeline control	None (access-control logic flaws are not reliably found by SAST). Verified by the manual before/after re-test; optional OWASP ZAP scan.
Verification	Log in as user A, request user B's basket ID; before fix returns B's data (200), after fix returns 403.
T4: Broken Authentication (No Login Rate Limiting + Hardcoded JWT Signing Key)
Field	Detail
STRIDE	Spoofing, Elevation of Privilege
Attack scenario	(a) /rest/user/login has no rate limiting or lockout, so an attacker can automate unlimited password guesses against known accounts (for example the admin email). (b) The JWT signing key is hardcoded in source (lib/insecurity.ts), creating a secret-management risk because anyone with access to the repository can obtain the signing key.
Asset at risk	User credentials, admin session tokens
Likelihood: 4	Brute-forcing is trivially automatable, and the key is readable by anyone with repository access.
Impact: 4	Successful brute force, or misuse of the exposed signing key, can give full account/admin access.
Risk	16, Critical
Mitigating control	express-rate-limit on /rest/user/login; load the JWT key from an environment variable/secret (GitHub Actions secret for the pipeline, runtime environment or Docker secret for the container) instead of source; explicitly pin algorithms: ['RS256'] on verification as a hardening measure.
Control location	application/juice-shop/server.ts (middleware chain, rate limiter), application/juice-shop/lib/insecurity.ts (key handling and JWT verification), GitHub Actions secrets store
Pipeline control	Gitleaks gate detects the hardcoded private key in lib/insecurity.ts; this is a natural failing-pipeline demonstration for Member 4.
Verification	Send a burst of login attempts after the fix and expect HTTP 429 after N attempts; Gitleaks scan of the fixed tree reports no key.

Dependency: .gitleaks.toml currently has an allowlist for Juice Shop test fixtures (commit "allow remaining juice shop test fixtures"). Member 4 must confirm this allowlist does not hide lib/insecurity.ts before the failing-pipeline demonstration, and that the final submission history presented as current practice contains no hardcoded key.

T5: Vulnerable Dependencies / Container Image
Field	Detail
STRIDE	Tampering, Elevation of Privilege
Attack scenario	Juice Shop depends on many npm packages and a base image that may contain publicly known CVEs (to be confirmed with the project's own npm audit and Trivy output, which must be cited as evidence in the final report). An attacker who identifies a reachable vulnerable component can exploit it to alter application behaviour or execute code in the container.
Asset at risk	Container integrity, host resources, application data
Likelihood: 4	Known CVEs may be identified by dependency and container scanners; exploitability depends on whether the vulnerable component and code path are reachable.
Impact: 3	Impact varies by component, but a reachable flaw can compromise the whole container.
Risk	12, High
Mitigating control	Dependency scanning and container image scanning as blocking gates; update or pin fixed versions; use a minimal, non-root runtime image.
Control location	.github/workflows/ (npm audit and Trivy steps), Dockerfile
Pipeline control	npm audit gate and Trivy gate
Verification	Pipeline fails on High/Critical findings; the count drops after dependency/base image updates.
5. Risk Matrix (5 × 5)
Impact ↓ / Likelihood →	1 Rare	2 Unlikely	3 Possible	4 Likely	5 Almost certain
5 Severe	5	10	15	20	T1 = 25
4 Major	4	8	T3 = 12	T2, T4 = 16	20
3 Moderate	3	6	9	T5 = 12	15
2 Minor	2	4	6	8	10
1 Negligible	1	2	3	4	5

Bands: 1–4 Low · 5–9 Medium · 10–14 High · 15–25 Critical.

6. Threat → Control → Location Mapping
Threat	Control type	Control	Location
T1 SQL injection	Secure coding	Parameterised queries, input validation	routes/login.ts
T1 SQL injection	Automated gate	Semgrep (SAST)	.github/workflows/
T2 DOM XSS	Secure coding	Remove bypassSecurityTrustHtml; CSP header	frontend/src/app/search-result/search-result.component.ts, server.ts
T2 DOM XSS	Automated gate	Semgrep (SAST)	.github/workflows/
T3 IDOR	Secure coding	Basket ownership check, 403 on mismatch	routes/basket.ts
T4 Broken auth	Secure coding	Login rate limiting; pin JWT algorithm (hardening)	server.ts, lib/insecurity.ts
T4 Broken auth	Secrets management	Signing key from environment/secret store	GitHub Actions secrets, runtime environment
T4 Broken auth	Automated gate	Gitleaks (secrets scan)	.github/workflows/, .gitleaks.toml
T5 Vulnerable components	Automated gate	npm audit (dependency scan)	.github/workflows/
T5 Vulnerable components	Automated gate	Trivy (container scan)	.github/workflows/, Dockerfile

All paths are relative to application/juice-shop/ unless they start with .github/.

7. Handoff Notes

For Member 3 (Exploit & Fix):

T1: Demonstrate the ' OR 1=1-- login bypass on the unmodified app, apply the parameterised query fix in routes/login.ts, repeat the identical payload, then compare Semgrep results before and after.
T2: Demonstrate <iframe src="javascript:alert(1)"> via the search box, fix search-result.component.ts, repeat the identical input, then compare Semgrep results.
T3: Demonstrate cross-account basket access by changing the ID in /rest/basket/:id, add the ownership check, repeat and capture the 403 response.
T4: Demonstrate unlimited login attempts (token forgery only if Member 3 can show it working), add rate limiting and move the signing key out of source, then capture the HTTP 429 / blocked result.

For Member 4 (CI/CD):

T1, T2 → Semgrep gate. T4 → Gitleaks gate. T5 → npm audit and Trivy gates. T3 has no SAST detection and is covered by the manual re-test evidence.
Gitleaks on the original lib/insecurity.ts gives a genuine failing-pipeline demonstration; check that .gitleaks.toml does not allowlist it.
Use the threat IDs T1–T5 in pipeline evidence for consistency across the report.
8. Report-Ready Summary (about 450 words, for the Threat Modelling & Risk section)

We applied STRIDE to Member 1's architecture of OWASP Juice Shop, in which a single Node.js container serves both the Angular frontend and the Express REST API, backed by SQLite and MarsDB. Three trust boundaries were analysed: the internet-to-container boundary (TB1), the application-to-data-layer boundary (TB2) and the repository/CI-to-image supply-chain boundary (TB3). Every STRIDE category was considered at each boundary, and five realistic, application-specific threats were selected. Four (T1–T4) are present in the unmodified code and are demonstrated as working exploits before being fixed; the fifth (T5) is a supply-chain threat evidenced by dependency and container scan results. Each maps to a concrete control.

T1 is SQL injection in the login handler, where string-interpolated SQL lets ' OR 1=1-- log an attacker in as the administrator. T2 is DOM-based XSS: the search page disables Angular's sanitiser, so a crafted link executes script in a victim's browser and exposes their session token. T3 is an insecure direct object reference in the basket route, which returns another user's basket when the ID is changed. T4 is broken authentication: the login endpoint has no rate limiting and the JWT signing key is hardcoded in source, enabling brute force and creating a secret-management risk. T5 covers known-vulnerable npm packages and base-image components that could be exploited to compromise the container.

Risk was scored on a 5×5 likelihood × impact matrix (1–4 Low, 5–9 Medium, 10–14 High, 15–25 Critical). T1 scores 25 (Critical) because the login form is public and the payload is trivial, while impact is total account takeover. T2 scores 16 (Critical) because it can be triggered against an unauthenticated victim through a crafted URL. T4 scores 16 (Critical) because login attempts can be automated without authentication, while exposure of the hardcoded signing key creates an additional account-compromise risk for anyone with repository access. T3 scores 12 (High): it requires a registered account, but breaks isolation between users. T5 scores 12 (High): known CVEs may be identified by dependency and container scanners, but exploitability depends on whether the vulnerable component and code path are reachable.

Threat	Control	Location
T1	Parameterised queries; Semgrep gate	routes/login.ts; CI workflow
T2	Remove bypassSecurityTrustHtml; CSP; Semgrep gate	search-result.component.ts, server.ts; CI workflow
T3	Basket ownership check (403)	routes/basket.ts
T4	Login rate limiting; key from secret store; Gitleaks gate	server.ts, lib/insecurity.ts; GitHub secrets; CI workflow
T5	npm audit and Trivy gates; dependency updates	CI workflow, Dockerfile

Repudiation and denial-of-service threats were also identified but not selected, since they do not lend themselves to a clean exploit-and-fix demonstration; rate limiting in T4 partly reduces the denial-of-service risk. The threat IDs are reused in the exploit-and-fix and pipeline sections so that every finding, fix and gate result can be traced back to this model.

9. Deliverables Checklist (Member 2)
 STRIDE table per trust boundary (Section 2)
 Five application-specific threats with attack scenarios (Section 4)
 Likelihood, impact and one-line justification for each (Section 4)
 5×5 risk matrix with defined bands (Section 5)
 Threat-to-control mapping with code and pipeline locations (Section 6)
 Handoff notes for Members 3 and 4 (Section 7)
 Report-ready condensed summary (Section 8)
