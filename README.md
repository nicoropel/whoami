# nicoropel.com

Personal portfolio site for Nicolás Oropel, Senior Application Security Engineer.

Live at [nicoropel.com](https://nicoropel.com)

## Philosophy

The site itself is a security artifact. Every decision — from the choice to use no frameworks to the structure of the `_headers` file — was made with a threat model in mind. A personal portfolio has no legitimate reason to be complex, and complexity is attack surface.

## Explicit Non-decisions

The following were considered and deliberately excluded:

- **JavaScript framework** — no React, Vue, or similar. No virtual DOM, no node_modules, no dependency graph to audit.
- **Analytics** — no tracking scripts, no third party beacons, no `connect-src` requirements beyond `'none'`.
- **Contact form** — no backend, no form handler, no CSRF surface. Email links only.
- **Cookies** — none set, none needed. No cookie banner, no session management.
- **CMS** — no admin panel, no authentication surface, no database. Content lives in version controlled HTML.
- **Analytics** — Cloudflare Web Analytics was disabled to maintain a strict `connect-src 'none'` policy and eliminate the only remaining third party script dependency.

## Stack

- **HTML, CSS, vanilla JS** — no frameworks, no npm, no build step, no supply chain
- **Cloudflare Pages** — static hosting, global CDN, free tier
- **Cloudflare Email Routing** — inbound-only custom domain email, no mail server
- **Cabinet Grotesk via Fontshare** — only external dependency (see known limitations)
- **GitHub** — source of truth, deploy on push to `main`

## Threat Model

This is a static personal site with no backend, no authentication, no user data, and no dynamic content. The realistic threat surface is:

- **CDN-layer tampering** — mitigated by SRI on all local assets
- **Malicious script injection via compromised dependency** — mitigated by strict CSP and zero third party scripts
- **Clickjacking** — mitigated by `frame-ancestors 'none'` and `X-Frame-Options: DENY`
- **Data exfiltration via injected script** — mitigated by `connect-src 'none'`
- **SSL stripping on first visit** — mitigated by HSTS preload list inclusion
- **MIME confusion attacks** — mitigated by `X-Content-Type-Options: nosniff`
- **Email harvesting** — accepted risk, contact address is intentionally public
- **Font supply chain** — accepted risk, documented below



## Security Architecture

### Content Security Policy

The CSP is defined in `_headers` and applied at the edge by Cloudflare Pages before any response reaches the client. It is version controlled alongside the code — not configured through a UI, not dependent on a dashboard setting surviving a team change.

```
default-src 'none'
script-src 'self'
style-src 'self' https://api.fontshare.com
font-src https://cdn.fontshare.com
img-src 'self' data:
connect-src 'none'
frame-ancestors 'none'
base-uri 'self'
form-action 'none'
```

Key decisions:

- `default-src 'none'` — deny everything not explicitly allowed. Allowlist, not blocklist.
- `connect-src 'none'` — no JavaScript on this page makes network requests. If injected script attempts exfiltration, the browser blocks it at the network layer.
- `frame-ancestors 'none'` — clickjacking mitigation, complements `X-Frame-Options: DENY`
- `form-action 'none'` — no forms exist, no form submissions are permitted
- No `unsafe-inline`, no `unsafe-eval` — CSS lives in an external stylesheet, JS has no eval usage

### Subresource Integrity

Both `style.css` and `script.js` are served with SRI hashes. The browser verifies the cryptographic hash of each file before parsing or executing it. If either file is tampered with at the CDN layer or in transit, the browser refuses to load it.

```html
<link rel="stylesheet" href="/style.css" integrity="sha384-..." crossorigin="anonymous">
<script src="/script.js" integrity="sha384-..." crossorigin="anonymous"></script>
```

SRI hashes must be regenerated on every change to either file:

```bash
openssl dgst -sha384 -binary style.css | openssl base64 -A
openssl dgst -sha384 -binary script.js | openssl base64 -A
```

### HSTS

```
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
```

Two year max-age, subdomains included, submitted to the HSTS preload list. The domain is hardcoded into browsers — HTTPS is enforced before a connection is made, eliminating the first-request SSL stripping window that the header alone cannot protect.

### Additional Headers

| Header | Value | Purpose |
|---|---|---|
| `X-Frame-Options` | `DENY` | Clickjacking — legacy browser support alongside CSP |
| `X-Content-Type-Options` | `nosniff` | Prevents MIME type sniffing attacks |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Limits referrer leakage on navigation |
| `Permissions-Policy` | all denied | Blocks camera, mic, geolocation, payment, USB, bluetooth, serial |
| `X-XSS-Protection` | `1; mode=block` | Legacy XSS filter for older browsers |

### security.txt

A `security.txt` file is served at `/.well-known/security.txt` per RFC 9116. It defines a responsible disclosure contact, expiry date, canonical URL, and preferred languages. The file is rotated before the `Expires` field lapses.



## Infrastructure

- **DNS:** Cloudflare — DNSSEC signing enabled, DS record pending 
  publication at registrar (Donweb). Full chain of trust pending 
  domain transfer to Cloudflare Registrar at next renewal.
- **Secrets:** none exist in this repository or deployment pipeline
- **Origin server:** none — Cloudflare Pages serves directly from the edge, eliminating an entire class of server-side vulnerabilities
- **Email:** Cloudflare Email Routing — inbound only, no mail server, no SMTP credentials
- **Cloudflare Rocket Loader:** disabled — prevents server-side inline script injection that would require `unsafe-inline` in CSP



## Known Limitations & Tradeoffs

### Fontshare as external dependency

Cabinet Grotesk is loaded from `api.fontshare.com` / `cdn.fontshare.com`. This is the only external dependency and the only reason `style-src` and `font-src` are not `'self'`-only. The tradeoff is accepted — self-hosting the font would tighten the CSP completely but adds a maintenance burden for a personal site.

SRI on the Fontshare stylesheet is not applied because Fontshare controls the file and may rotate it, which would break the page. This is a documented and accepted risk.

### SRI maintenance overhead

Every deployment that touches `style.css` or `script.js` requires hash regeneration. This is a manual step included in the deploy checklist below.

### Cloudflare-injected scripts

Cloudflare Pages can inject third party scripts server-side (analytics beacon, Rocket Loader) that bypass version control and violate a strict CSP. Both have been explicitly disabled. 
Any future Cloudflare feature that injects scripts would require a CSP update and should be evaluated before enabling.



## Monitoring & Verification

Headers and CSP are verified after each deployment using:

- [securityheaders.com](https://securityheaders.com)
- [csp-evaluator.withgoogle.com](https://csp-evaluator.withgoogle.com)

**Recovery:** the site is fully reproducible from the GitHub repository. A full redeployment takes under 60 seconds via Cloudflare Pages. There is no stateful infrastructure to restore.



## Deploy Checklist

```
[ ] Make changes
[ ] If style.css or script.js has changed  → regenerate SRI hash → update integrity attribute in index.html
[ ] If security.txt is within 30 days of Expires → rotate the date and redeploy
[ ] Commit all changed files together
[ ] Push to main → Cloudflare Pages deploys automatically
[ ] Verify headers at securityheaders.com
[ ] Verify CSP at csp-evaluator.withgoogle.com
```


## Contact

- **Security issues:** security@nicoropel.com
- **General:** ping@nicoropel.com
- **En español:** hola@nicoropel.com