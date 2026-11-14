# Frontend Security — Concepts & Interview Preparation

**Priority:** P0/P1 · **Level:** SDE-1 to Senior · **Category:** Web security

> **Revision slogan:** Treat every input as untrusted, every browser boundary as a security boundary, and the server as the authority for access control.

## 1. Security mindset and threat model

Frontend security is not just hiding buttons or validating forms. A browser application handles untrusted data, runs code in a powerful environment, stores credentials, communicates with APIs and embeds third-party resources.

For every feature, ask:
1. What data or action needs protection?
2. Who could attack it, and from which entry point?
3. What happens if an attacker controls this input, URL, message or dependency?
4. Which controls belong in the browser, server, infrastructure and build pipeline?
5. How will the team detect and respond to abuse?

**Core principle:** Client-side checks improve UX but never replace server-side authentication, authorization, validation or rate limiting. Users can modify frontend code and requests.

## 2. Cross-Site Scripting (XSS)

XSS happens when an attacker causes malicious script or markup to execute in another user's browser in the context of a trusted site.

### Types

- **Stored XSS:** Malicious content is saved (for example, in a profile or comment) and executes when another user views it.
- **Reflected XSS:** Untrusted input from a request is included in an immediate response and executes.
- **DOM-based XSS:** Client-side JavaScript reads attacker-controlled data (such as a URL fragment) and sends it to an unsafe DOM sink.

### Vulnerable example

~~~js
const params = new URLSearchParams(location.search);
const name = params.get("name");

// Unsafe: parses attacker-controlled string as HTML.
document.querySelector("#welcome").innerHTML = name;
~~~

Safer when rendering plain text:

~~~js
document.querySelector("#welcome").textContent = name ?? "";
~~~

For rich text, use a well-maintained HTML sanitizer with a strict allowlist, then render only its sanitized output. Context-aware output encoding matters: HTML text, attributes, URLs, CSS and JavaScript contexts have different rules.

### Prevention
- Prefer framework text interpolation and safe DOM APIs.
- Avoid \`innerHTML\`, \`outerHTML\`, \`insertAdjacentHTML\`, \`document.write\` with untrusted content.
- Sanitize user-authored rich HTML using a maintained sanitizer (for example, DOMPurify) and a restrictive configuration.
- Validate URL schemes before allowing links; reject dangerous schemes such as \`javascript:\`.
- Deploy a strong Content Security Policy (CSP), ideally with nonces or hashes.
- Use Trusted Types where supported to help restrict dangerous DOM sinks.
- Do not assume input validation alone prevents XSS.

**React note:** React escapes values rendered as text by default. \`dangerouslySetInnerHTML\` bypasses that protection and must only receive trusted, properly sanitized HTML.

## 3. Cross-Site Request Forgery (CSRF)

CSRF tricks a browser that is already authenticated to send an unwanted request to a site. It is especially relevant when authentication credentials (often cookies) are automatically attached by the browser.

Example: a user is logged into a banking site. A malicious page causes their browser to submit a money-transfer request to that site.

### Prevention
- Use anti-CSRF tokens for state-changing requests when appropriate.
- Set session cookies with \`SameSite=Lax\` or \`SameSite=Strict\` where the product flow permits.
- Use \`Secure\` and \`HttpOnly\` on session cookies.
- Validate \`Origin\` (and, where appropriate, \`Referer\`) on sensitive requests.
- Do not use GET requests for state-changing actions.
- Consider Fetch Metadata request headers as an additional defence.
- Re-authenticate or require step-up verification for high-risk actions.

**Important:** CORS is not a general CSRF defence. A cross-origin form can often send a request even when the attacker cannot read the response.

## 4. CORS and Same-Origin Policy

The Same-Origin Policy (SOP) restricts how documents and scripts from one origin can interact with resources from another origin. An origin is the combination of scheme, host and port.

CORS (Cross-Origin Resource Sharing) is a server-controlled mechanism that lets a server declare which origins may read its responses from browser JavaScript.

### Key points
- CORS is enforced by browsers; it is not authentication or authorization.
- The server sends headers such as \`Access-Control-Allow-Origin\`.
- Some cross-origin requests are sent without a preflight; the browser may block JavaScript from reading the response.
- A preflight is an OPTIONS request used for requests that require permission, such as many non-simple methods or custom headers.
- When credentials are included, the server must explicitly allow the requesting origin and send \`Access-Control-Allow-Credentials: true\`; wildcard origin \`*\` cannot be used for credentialed access.
- The server must still authenticate and authorize every request.

Example response:

~~~http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT
Access-Control-Allow-Headers: Content-Type, Authorization
Vary: Origin
~~~

Credentialed response (with a specific allowed origin):

~~~http
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Credentials: true
~~~

Do not reflect arbitrary Origin headers. Use an explicit allowlist.

## 5. Authentication, authorization and session handling

- **Authentication:** Who are you?
- **Authorization:** What are you allowed to do?
- **Session management:** How does the server remember an authenticated user across requests?

The frontend can display UI based on role or permissions, but the backend must enforce access on every protected operation and resource.

### Cookie-based sessions

A common secure session cookie configuration:

~~~http
Set-Cookie: session=opaque-value; HttpOnly; Secure; SameSite=Lax; Path=/
~~~

- \`HttpOnly\`: JavaScript cannot read the cookie.
- \`Secure\`: Cookie is sent only over secure connections.
- \`SameSite\`: Controls when the browser sends the cookie in cross-site contexts.
- \`Path\` and \`Domain\`: Scope where the cookie is sent; avoid broad scope unless required.
- Use appropriate expiry and rotation policies.

### Access tokens and refresh tokens
- Keep access tokens short-lived and scope them narrowly.
- Prefer refresh-token rotation and reuse detection when using refresh tokens.
- Protect refresh tokens as credentials; do not expose them to scripts unnecessarily.
- Consider a Backend-for-Frontend (BFF) pattern: browser holds a secure session cookie and the server keeps OAuth tokens.
- Never put secrets or private API keys in frontend bundles; anything shipped to the browser is public.

### localStorage vs sessionStorage vs cookies

| Storage | JavaScript access | Lifetime / scope | Security notes |
|---|---|---|---|
| localStorage | Yes | Persists for origin | XSS can read it; avoid storing session credentials where possible |
| sessionStorage | Yes | Usually tab-scoped session | XSS can read it; not a secure vault |
| Cookies | Depends on HttpOnly | Expiry and domain/path scope | HttpOnly blocks JS reads; cookies may be sent automatically, so consider CSRF |

No browser storage mechanism makes a token safe from an attacker who can execute arbitrary same-origin JavaScript. Choose storage based on the threat model, not convenience alone.

## 6. Content Security Policy (CSP)

CSP is a response-header policy that tells the browser which sources are allowed for scripts, styles, images, frames and other resource types. It can reduce the impact of XSS and limit resource loading.

Example starter policy (adapt and test for the application):

~~~http
Content-Security-Policy:
  default-src 'self';
  script-src 'self' 'nonce-{RANDOM_PER_RESPONSE}';
  object-src 'none';
  base-uri 'self';
  frame-ancestors 'none';
  upgrade-insecure-requests;
~~~

Important directives:
- \`default-src\`: fallback policy for many resource types.
- \`script-src\`: allowed script sources and nonces/hashes.
- \`style-src\`: allowed style sources.
- \`img-src\`, \`font-src\`, \`connect-src\`: images, fonts and network endpoints.
- \`object-src 'none'\`: disables plugin content.
- \`base-uri\`: restricts base URL changes.
- \`frame-ancestors\`: controls who can embed the page.

Use a unique, unpredictable nonce per response when using nonce-based policies. Avoid relying on \`'unsafe-inline'\` or \`'unsafe-eval'\` as a shortcut. Start with Report-Only mode to discover breakage before enforcing a policy.

## 7. Clickjacking and framing protection

Clickjacking tricks a user into clicking something different from what they believe they are interacting with, often by embedding a target site in a hidden or disguised frame.

Defences:
- Send CSP \`frame-ancestors\` to restrict which sites can embed your page.
- Use \`X-Frame-Options: DENY\` or \`SAMEORIGIN\` for older browser support where needed.
- Design sensitive confirmation flows carefully; do not rely on UI obscurity.

## 8. HTTPS, HSTS and transport security

- HTTPS protects data in transit and authenticates the server endpoint.
- HSTS tells browsers to use HTTPS for future connections to a host.
- Use \`Strict-Transport-Security\` only when the domain and subdomains are ready for HTTPS.
- Avoid mixed content (loading insecure HTTP resources from an HTTPS page).
- Do not treat HTTPS as a replacement for application-level authorization or safe coding.

Example:

~~~http
Strict-Transport-Security: max-age=31536000; includeSubDomains
~~~

Only enable \`includeSubDomains\` when every relevant subdomain supports HTTPS.

## 9. URL, redirect and navigation security

Untrusted URLs can enable phishing, open redirects, script execution through unsafe schemes, or data leakage.

- Validate redirect destinations against an allowlist or use relative paths.
- Do not blindly redirect to a \`next\`, \`returnUrl\` or \`redirect\` query parameter.
- Allow only expected URL schemes, typically \`https:\` and sometimes \`http:\` for explicitly trusted cases.
- Use safe link attributes for links opened in new tabs; modern browsers implicitly apply noopener in many cases, but explicit \`rel="noopener noreferrer"\` is clear and compatible.
- Avoid putting secrets or sensitive personal data in query strings because URLs can appear in history, logs and referrer data.

## 10. postMessage and cross-window communication

\`window.postMessage\` enables communication between windows, iframes and workers. Treat received messages as untrusted input.

~~~js
// Sender: target a known origin, not "*".
iframe.contentWindow.postMessage(
  { type: "RESIZE", height: 500 },
  "https://widget.example.com"
);

// Receiver: validate both origin and message shape.
window.addEventListener("message", (event) => {
  if (event.origin !== "https://widget.example.com") return;
  if (event.source !== iframe.contentWindow) return;

  const data = event.data;
  if (
    !data ||
    data.type !== "RESIZE" ||
    !Number.isInteger(data.height) ||
    data.height < 0 ||
    data.height > 2000
  ) return;

  setWidgetHeight(data.height);
});
~~~

Never trust a message only because it arrived through postMessage. Validate the sender origin, expected source and message schema. Use an exact target origin instead of \`*\` whenever possible.

## 11. Dependency and supply-chain security

Frontend applications depend on npm packages, build tools, plugins, scripts and CI workflows. A compromised dependency can execute code during install, build or runtime.

- Minimize dependencies and remove unused packages.
- Review package ownership, maintenance, release history and transitive dependencies.
- Pin or constrain versions appropriately and commit the lockfile.
- Run dependency and secret scanning in CI.
- Review install scripts and unexpected package changes.
- Use Subresource Integrity (SRI) for eligible third-party scripts/styles loaded from CDNs.
- Restrict third-party scripts and monitor their changes.
- Keep build tools and runtimes patched.

Example SRI:

~~~html
<script
  src="https://cdn.example.com/library.min.js"
  integrity="sha384-BASE64_HASH_FROM_TRUSTED_BUILD"
  crossorigin="anonymous"
></script>
~~~

The hash must be generated from the exact trusted asset. Do not copy a placeholder hash.

## 12. Other frontend security risks

### Sensitive data exposure
Never put private keys, secrets, internal endpoints, credentials or sensitive user data in client bundles, source maps exposed publicly, logs or URLs. Anything delivered to the browser can be inspected.

### Prototype pollution
Attacker-controlled object keys can modify object prototypes when merged unsafely. Use safe merge utilities, validate keys, avoid unsafe recursive merges, and prefer \`Map\` for dictionary-like data where appropriate.

### DOM clobbering
HTML elements with attacker-controlled \`id\` or \`name\` values can interfere with named properties on browser objects and legacy code assumptions. Avoid relying on implicit named globals; use explicit references and safe DOM APIs.

### File upload risks
Client-side file type checks are for user experience only. The server must validate content, size and authorization; store uploads safely; generate server-side names; and consider malware scanning and content-disposition policies. Do not trust a filename or MIME type supplied by the browser.

### Rate limiting and abuse
Client-side throttling improves UX but can be bypassed. Enforce rate limits, quotas and abuse detection on the server or API gateway.

### Privacy and permissions
Request only necessary browser permissions. Explain why they are needed, handle denial gracefully, and avoid collecting or retaining data unnecessarily.

## 13. Security headers quick reference

| Header | Purpose |
|---|---|
| Content-Security-Policy | Restricts resource sources and mitigates some XSS impact |
| Strict-Transport-Security | Enforces HTTPS on future visits |
| X-Content-Type-Options: nosniff | Prevents MIME-type sniffing |
| Referrer-Policy | Controls referrer information sent with requests |
| Permissions-Policy | Restricts use of selected browser features |
| X-Frame-Options | Legacy framing restriction; CSP frame-ancestors is preferred |
| Cross-Origin-Opener-Policy | Helps isolate browsing context groups |
| Cross-Origin-Resource-Policy | Controls cross-origin resource loading |

Headers should be configured and tested at the server/CDN layer, not treated as frontend JavaScript settings.

## 14. Interview questions and answers

### Fundamentals

**Q1. What are the most common security vulnerabilities in frontend applications?**

XSS, CSRF, insecure session/token handling, unsafe redirects, clickjacking, overly permissive CORS, unsafe postMessage handling, sensitive data exposure and vulnerable third-party dependencies. The exact risks depend on the app's data and threat model.

**Q2. What is the difference between authentication and authorization?**

Authentication verifies who a user is. Authorization determines what that authenticated identity can access or do. Authorization must be enforced by the server for every protected operation.

**Q3. Can frontend validation secure an API?**

No. A user can modify JavaScript, bypass UI checks and send requests directly. The backend must validate input, authenticate the caller, authorize the requested action and enforce rate limits.

### XSS

**Q4. Explain XSS and its types.**

XSS is attacker-controlled script execution in a trusted site's browser origin. Stored XSS is persisted and shown later; reflected XSS is returned in an immediate response; DOM-based XSS occurs when client-side code sends untrusted data into an unsafe DOM sink.

**Q5. How do you prevent XSS in React?**

Use normal JSX text interpolation, which escapes text by default. Avoid dangerouslySetInnerHTML with untrusted content. If rich HTML is required, sanitize it with a maintained allowlist-based sanitizer, validate URLs and deploy CSP as defence in depth.

**Q6. Is input validation enough to prevent XSS?**

No. Validation can constrain expected data but does not replace context-aware output encoding or sanitization. Data may be inserted into different contexts, each with different parsing rules.

**Q7. What is the difference between escaping and sanitization?**

Escaping encodes characters so data is treated as text in a specific output context. Sanitization parses content and removes or restricts unsafe markup/attributes according to a policy. Choose based on whether the application needs plain text or a limited subset of HTML.

**Q8. What is Content Security Policy?**

CSP is a browser-enforced policy, usually delivered as an HTTP response header, that restricts where scripts and other resources can load from. It reduces the impact of some injection attacks, but it does not replace safe DOM handling.

### CSRF and CORS

**Q9. What is CSRF?**

CSRF causes a browser with an existing authenticated session to send an unwanted request to a site. It commonly targets cookie-based authentication because browsers attach cookies automatically in eligible contexts.

**Q10. How do you prevent CSRF?**

Use anti-CSRF tokens where appropriate, SameSite cookies, Origin/Referer validation and Fetch Metadata checks. Keep state-changing operations out of GET requests. Use layered defences based on the application's session design.

**Q11. Is CORS a CSRF protection?**

No. CORS controls whether browser JavaScript can read a cross-origin response. It does not generally stop a browser from sending all cross-origin requests, including form submissions that can be used for CSRF.

**Q12. What is a preflight request?**

It is an OPTIONS request the browser sends before certain cross-origin requests to ask whether the server permits the origin, method and headers. The browser sends the actual request only if the preflight checks pass.

**Q13. Why can't Access-Control-Allow-Origin be * with credentials?**

Browsers reject credentialed CORS responses that use wildcard origin. The server must return the specific allowed origin and set Access-Control-Allow-Credentials: true. The server should use an explicit origin allowlist.

**Q14. What is the Same-Origin Policy?**

It is a browser security model that restricts how documents and scripts from one origin can interact with resources from another. CORS provides a controlled way for servers to permit cross-origin response access.

### Tokens, cookies and sessions

**Q15. Where should frontend applications store authentication tokens?**

There is no universal answer. JavaScript-readable storage such as localStorage is exposed if XSS occurs. HttpOnly secure cookies reduce direct token theft by JavaScript but require CSRF protections because cookies are automatically attached. A BFF/session-cookie pattern is often a strong option for browser apps.

**Q16. What does HttpOnly do?**

It prevents JavaScript from reading a cookie through document.cookie. The browser can still attach the cookie to eligible HTTP requests. It reduces credential exfiltration through XSS but does not prevent XSS itself or all actions an injected script can perform.

**Q17. What is SameSite on cookies?**

SameSite controls whether a cookie is sent in cross-site contexts. Strict is most restrictive, Lax permits some top-level safe navigations, and None allows cross-site use but requires Secure. Choose the setting based on application flows.

**Q18. What is the difference between localStorage, sessionStorage and cookies?**

localStorage persists for an origin, sessionStorage is generally scoped to a tab's page session, and cookies are sent automatically with matching requests and can be configured HttpOnly/Secure/SameSite. localStorage and sessionStorage are accessible to JavaScript; HttpOnly cookies are not.

**Q19. Why should secrets not be stored in frontend environment variables?**

Build tools commonly inline client-exposed variables into generated JavaScript. Users can inspect bundles and network traffic. Only non-sensitive configuration belongs in the client; private credentials must remain server-side.

**Q20. What is a refresh-token rotation strategy?**

When a refresh token is used, the authorization server issues a new refresh token and invalidates the previous one. Reuse of an old token can indicate theft and trigger revocation or session invalidation. The exact strategy is implemented server-side.

### Browser security and platform

**Q21. What is clickjacking and how do you prevent it?**

Clickjacking overlays or embeds a trusted page so users unknowingly interact with it. Restrict framing with CSP frame-ancestors and, for legacy support, X-Frame-Options.

**Q22. What is the purpose of HTTPS and HSTS?**

HTTPS encrypts traffic in transit and authenticates the server. HSTS tells browsers to use HTTPS for future connections to the host, reducing downgrade and insecure-transport risks.

**Q23. How should postMessage be secured?**

Use an exact target origin when sending. On receipt, validate event.origin, event.source and the shape and allowed values of event.data. Never trust a message merely because it came from a window or iframe you recognize.

**Q24. What is Subresource Integrity?**

SRI lets the browser verify that a fetched CDN script or stylesheet matches a cryptographic hash declared in the integrity attribute. If the content does not match, the browser refuses to use it.

**Q25. What is Trusted Types?**

Trusted Types is a browser API that can require values passed to certain dangerous DOM injection sinks to be created through approved policies. It helps prevent accidental use of raw strings in sinks such as innerHTML; it complements sanitization and CSP.

**Q26. What is prototype pollution?**

It occurs when unsafe handling of attacker-controlled object properties modifies an object's prototype or inherited properties. Use safe object construction/merge patterns, validate keys and avoid unsafe recursive merges of untrusted data.

**Q27. Why is client-side rate limiting insufficient?**

Users can disable or bypass client code and call the API directly. Enforce quotas and rate limits at the server or gateway; client throttling is only a usability and traffic optimization.

**Q28. How do you secure file uploads?**

Treat client checks as UX only. Validate authorization, size and actual file content server-side; generate safe names; store files outside executable paths; restrict downloads and content types; and scan where appropriate.

**Q29. What is the purpose of X-Content-Type-Options: nosniff?**

It tells browsers not to MIME-sniff certain responses and to respect the declared content type, reducing risks from content being interpreted as an executable type unexpectedly.

**Q30. What security checks would you add to a frontend CI pipeline?**

Dependency and license review, vulnerability scanning, secret scanning, static analysis, linting for unsafe DOM sinks, lockfile checks, build artifact review and tests for authentication/authorization flows. CI checks complement server-side controls and security testing.

### Scenario-based / senior-level

**Q31. A user can see an Admin button only when their role is admin. Is the application secure?**

Not by itself. Hiding the button is only a presentation decision. The API must verify the user's identity and permission on every admin endpoint and resource. Test direct API calls with a non-admin account.

**Q32. Your React app renders comments containing rich HTML. How do you make it safe?**

Define the smallest required HTML allowlist, sanitize on a trusted boundary with a maintained sanitizer, avoid unsafe attributes and URL schemes, render sanitized output only, and use CSP as defence in depth. Keep the original content separate from the sanitized display representation.

**Q33. A security team reports an XSS vulnerability through a query parameter. How do you investigate?**

Trace the parameter from source to sink. Identify whether it reaches innerHTML, an unsafe URL, script or CSS context. Reproduce safely, remove or encode the unsafe sink, validate any required URL, add regression tests and consider CSP/Trusted Types. Review other similar sinks.

**Q34. A third-party analytics script is compromised. What can it access?**

A script loaded into the page generally executes with the page's origin privileges and may access DOM data and JavaScript-readable storage. Reduce third-party scripts, avoid exposing sensitive data to the page, use CSP and SRI where applicable, and isolate untrusted widgets in sandboxed iframes with carefully limited permissions.

**Q35. Your API uses cookie authentication and a partner site embeds a form that submits to it. What do you review?**

Review SameSite cookie behaviour, anti-CSRF token requirements, Origin/Referer validation, allowed content types and state-changing methods. Do not assume CORS alone blocks the request.

**Q36. A frontend engineer proposes storing a private payment API key in a Vite/Next public environment variable. What do you say?**

Anything exposed to browser code is public. Move the secret to a server-side API or BFF and have the frontend call that endpoint. Apply authorization, input validation, rate limiting and secret rotation server-side.

**Q37. How would you secure an iframe integration?**

Use a restrictive sandbox attribute, allow only required capabilities, set a specific target origin for postMessage, validate origin/source/schema on receipt, and apply CSP frame-src/frame-ancestors policies as appropriate. Avoid granting broad permissions by default.

**Q38. How would you respond to a suspected frontend security incident?**

Triage affected versions and users, preserve evidence, assess data exposure and exploitability, disable or contain the vulnerable path, rotate/revoke exposed credentials if needed, deploy a fix, notify the appropriate security/privacy owners and add regression tests and monitoring. Follow the organization's incident process.

## 15. Practical security review checklist

### Before release
- [ ] No secrets or private keys in client bundles, source maps or public config.
- [ ] Server validates input and enforces authorization on every protected operation.
- [ ] Untrusted text is rendered as text; unsafe HTML sinks are avoided or sanitized.
- [ ] CSP is configured and tested.
- [ ] Cookies use appropriate Secure, HttpOnly and SameSite attributes.
- [ ] CSRF protections match the authentication model.
- [ ] CORS uses an explicit allowlist and does not reflect arbitrary origins.
- [ ] Redirect destinations and URL schemes are validated.
- [ ] postMessage validates origin, source and payload schema.
- [ ] Third-party scripts and dependencies are minimized and reviewed.
- [ ] HTTPS and appropriate security headers are enabled.
- [ ] Uploads are validated and constrained server-side.
- [ ] Sensitive data is not leaked in URLs, logs or analytics.
- [ ] CI runs dependency and secret scanning.

## 16. Last-minute revision sheet

| Concept | One-line answer |
|---|---|
| XSS | Untrusted input executes as script in a trusted origin |
| Stored XSS | Payload is persisted and served to victims |
| Reflected XSS | Payload is reflected in an immediate response |
| DOM XSS | Client-side source reaches unsafe DOM sink |
| CSRF | Browser is tricked into sending an authenticated unwanted request |
| SOP | Restricts cross-origin interactions |
| CORS | Server grants browser JavaScript cross-origin response access |
| CSP | Browser policy restricts resource sources |
| HttpOnly | JavaScript cannot read the cookie |
| Secure | Cookie sent only over secure transport |
| SameSite | Controls cookie sending in cross-site contexts |
| Clickjacking | User is tricked into interacting with an embedded page |
| SRI | Browser verifies CDN resource integrity using a hash |
| postMessage | Validate origin, source and payload |
| Authorization | Must be enforced server-side |
| Secret in bundle | Not secret; browser users can inspect it |

### Explain in 30 seconds

"Frontend security starts with treating all external data as untrusted. I prevent XSS by using safe rendering and sanitization for rich HTML, add CSP as defence in depth, protect cookie-based sessions against CSRF, configure CORS with explicit origins, secure cookies with HttpOnly/Secure/SameSite, validate postMessage origins, avoid exposing secrets in bundles, and rely on the backend for authorization and rate limiting."

## 17. Self-check before marking complete

- [ ] Explain stored, reflected and DOM-based XSS.
- [ ] Explain CSRF and why CORS is not a CSRF defence.
- [ ] Explain SOP, CORS and preflight.
- [ ] Compare cookie, localStorage and sessionStorage trade-offs.
- [ ] Explain HttpOnly, Secure and SameSite.
- [ ] Explain CSP, nonce and frame-ancestors.
- [ ] Secure a postMessage integration.
- [ ] Explain SRI and dependency supply-chain risk.
- [ ] Identify why frontend-only authorization is unsafe.
- [ ] Walk through a practical XSS or CSRF incident.

## 18. Further reading

- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [OWASP Cross Site Scripting Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [OWASP Cross-Site Request Forgery Prevention](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [MDN: Content Security Policy](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [MDN: Cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies)
- [OWASP HTML5 Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html)
- [web.dev: Trusted Types](https://web.dev/articles/trusted-types)

---
**Next recommended chapters:** Authentication & OAuth/OIDC → Secure API integration → Browser storage and privacy.
