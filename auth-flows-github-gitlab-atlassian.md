# Learning Guide: Authentication Flows for GitHub, GitLab, and Atlassian - Browser Agent Reference

**Generated**: 2026-02-22
**Sources**: 28 resources synthesized (1 live fetch + training knowledge through May 2025 + existing codebase)
**Depth**: deep

---

## Prerequisites

- Familiarity with Playwright browser automation
- Understanding of cookies, OAuth, and session management
- Basic knowledge of DOM selectors and iframes
- Read the companion guides: `cli-browser-automation-agents.md` and `web-session-persistence-cli-agents.md`

---

## TL;DR

- GitHub uses Arkose Labs (FunCAPTCHA) on login; the success cookie is `logged_in=yes` on `.github.com`. Login is a single page with username + password, then a separate 2FA page.
- GitLab uses reCAPTCHA on sign-up (not typically on login); the success cookie is `_gitlab_session` on `.gitlab.com`. Login is single-page with optional 2FA redirect.
- Atlassian uses a multi-step login - email first, then password on a separate screen. Success is detected by redirect to `start.atlassian.com`. Atlassian ID is shared across Jira, Confluence, Bitbucket, and Trello.
- All three providers support TOTP, WebAuthn/passkeys, and backup codes for 2FA. GitHub and GitLab also support SMS (GitHub) and email codes (Atlassian).
- Enterprise/self-hosted variants change the domain but generally keep the same DOM structure and auth flow shape.

---

## 1. GitHub (github.com)

### 1.1 Login Page Structure

**URL**: `https://github.com/login`

**Form fields**:
- `input#login_field` - Username or email address (name: `login`)
- `input#password` - Password (name: `password`)
- Hidden field: `authenticity_token` (Rails CSRF token)
- Form action: `POST /session`

**Page structure**:
- Single-page login form (no multi-step email-first flow)
- "Sign in with a passkey" button appears below the form
- "Create an account" link at bottom
- "Forgot password?" link near password field

**Key selectors**:
```javascript
// Login form
'form[action="/session"]'
'input#login_field'
'input#password'
'input[name="commit"]'           // Submit button
'input[name="authenticity_token"]' // CSRF token (hidden)

// Passkey option
'button:has-text("Sign in with a passkey")'

// Error state
'.flash-error'                    // Login failure message
```

**Login URL redirects**: `https://github.com/login` stays on `/login`. After form submit, POST goes to `/session`. On success, redirects to `/` or the `return_to` parameter.

### 1.2 CAPTCHA System

**Provider**: Arkose Labs (FunCAPTCHA)

**When triggered**:
- New/unfamiliar IP address
- Multiple failed login attempts (typically 3-5 failures)
- Suspicious user agent or automation signals
- VPN/proxy/datacenter IP ranges
- After account recovery flows

**Detection selectors**:
```javascript
// Arkose Labs iframe
'iframe[src*="arkose"]'
'iframe[src*="arkoselabs.com"]'
'iframe[src*="funcaptcha"]'
'#FunCaptcha'
'#arkose-enforcement'

// Arkose container div
'div[data-callback*="arkose"]'

// Generic captcha text patterns (body text search)
'suspicious activity'
'verify you are human'
'complete the captcha'
'security check'
```

**Behavior**: Arkose renders inside an iframe. The challenge is typically an image puzzle (rotate image, select matching images). It blocks form submission until solved. The iframe URL pattern is `https://github-api.arkoselabs.com/...` or `https://client-api.arkoselabs.com/...`.

**Important for automation**: Arkose FunCAPTCHA is specifically designed to be unsolvable by automation. When detected, the agent MUST fall back to human-in-the-loop interaction (VNC, headed browser, or similar).

### 1.3 2FA Flows

GitHub supports multiple 2FA methods. After successful username/password, the user is redirected to a 2FA page.

**2FA URL**: `https://github.com/sessions/two-factor` (GET after successful password)

**Method detection**:

| Method | URL/Selector | DOM Indicator |
|--------|-------------|---------------|
| TOTP (authenticator app) | `/sessions/two-factor` | `input#app_totp` or `input[name="app_otp"]` |
| SMS | `/sessions/two-factor/sms` | `input#sms_totp` with SMS prompt text |
| Security key (WebAuthn) | `/sessions/two-factor/webauthn` | `button` with "Use security key" text |
| Passkey (resident credential) | `/sessions/two-factor/webauthn` | WebAuthn API call with `userVerification: "required"` |
| Backup/recovery codes | `/sessions/two-factor/recovery` | `input[name="recovery_code"]` |
| GitHub Mobile push | `/sessions/two-factor/mobile` | Text: "Check your GitHub Mobile app" |

**Key 2FA selectors**:
```javascript
// TOTP input
'input#app_totp'
'input[name="app_otp"]'

// SMS input
'input#sms_totp'

// Recovery code input
'input[name="recovery_code"]'

// WebAuthn prompt
'button:has-text("Use security key")'
'button:has-text("Use passkey")'

// Method switcher links
'a[href*="two-factor/sms"]'
'a[href*="two-factor/recovery"]'
'a[href*="two-factor/webauthn"]'
'a[href*="two-factor/mobile"]'

// 2FA page detection (any of these indicate 2FA is active)
'form[action*="two-factor"]'
'body:has(input#app_totp)'
```

**2FA flow for automation**:
1. Agent detects redirect to `/sessions/two-factor*`
2. Agent pauses and prompts user for TOTP code (or triggers VNC for human interaction)
3. For passkeys/WebAuthn - MUST use headed browser; cannot be automated headlessly

### 1.4 OAuth/SSO Redirects

**SAML SSO (Enterprise)**:
- URL pattern: `https://github.com/orgs/{org}/sso` or redirect to enterprise IdP
- After SSO: redirects back to `https://github.com/orgs/{org}/sso/redirect`
- Detection: URL contains `/sso` or `saml`

**OAuth App authorization**:
- URL pattern: `https://github.com/login/oauth/authorize?client_id=...`
- After auth: redirects to callback URL with `?code=...`

**GitHub Apps installation**:
- URL pattern: `https://github.com/apps/{app}/installations/new`

**SSO vs direct login detection**:
```javascript
// SSO indicators in URL
url.includes('/sso')
url.includes('/saml')
url.includes('/login/oauth')
url.includes('idp.') || url.includes('sso.')  // External IdP redirect

// Direct login indicator
url === 'https://github.com/login' || url === 'https://github.com/session'
```

### 1.5 Success Detection

**Primary method - Cookie** (recommended, used in web-ctl providers.json):
```javascript
// The canonical success cookie
{ domain: ".github.com", name: "logged_in", value: "yes" }

// Additional session cookies present after login:
// _gh_sess - session cookie (HttpOnly, Secure, .github.com)
// dotcom_user - username (not HttpOnly, .github.com)
// user_session - session token (HttpOnly, Secure, .github.com)
```

**Secondary method - DOM selector**:
```javascript
// User avatar in header (indicates logged in)
'meta[name="user-login"]'   // content attribute contains username
'img.avatar'                 // User avatar image
'.Header-link--profile'      // Profile link in header
'[data-login]'               // Any element with data-login attribute
```

**URL-based detection**:
```javascript
// After login, URL changes from /login to / or /dashboard
// Exclude patterns: /login, /session, /sessions/two-factor
!url.includes('/login') && !url.includes('/session')
```

**meta[name="user-login"] edge case**: This meta tag exists on login pages too, but with empty `content`. The auth-check code correctly validates that the content is non-empty (see `auth-check.js` line 34-38).

### 1.6 Anti-Bot Measures

| Measure | Details |
|---------|---------|
| Arkose Labs CAPTCHA | Triggered on suspicious logins (see 1.2) |
| Rate limiting | Failed login: ~10 attempts per IP per hour before lockout escalation |
| Account lockout | After many failures: temporary lockout with email verification required |
| Device fingerprinting | GitHub tracks device cookies; new device triggers verification email |
| `authenticity_token` | Rails CSRF token required on every form submission; changes per page load |
| Suspicious login email | If login from new location/device, sends email notification |
| `navigator.webdriver` detection | GitHub pages include scripts that check automation indicators |

### 1.7 Playwright-Specific Gotchas

1. **CSRF token**: Must load the login page first (GET `/login`) to get `authenticity_token` before POSTing. Cannot skip straight to POST `/session`.
2. **Cookie persistence**: The `logged_in` cookie is HttpOnly and Secure. Must use `context.cookies()` to read it, not JavaScript.
3. **Passkey/WebAuthn**: Cannot be automated in headless mode. Requires `--headless=false` and user physical interaction.
4. **Rate limiting on automation**: Automated page loads from the same IP can trigger rate limits. Add delays between requests.
5. **`_gh_sess` rotation**: The session cookie rotates on navigation. Always save the full cookie jar after auth, not just one cookie.
6. **SameSite cookies**: GitHub uses `SameSite=Lax` on most cookies. Cross-origin iframes won't send them.

### 1.8 GitHub Enterprise Server (GHES) Differences

| Aspect | github.com | GHES |
|--------|-----------|------|
| Login URL | `https://github.com/login` | `https://{hostname}/login` |
| CAPTCHA | Arkose Labs | Usually none (configurable) |
| SSO/SAML | Per-org | Instance-wide, often the only login method |
| 2FA | Optional per-user | Admin can enforce for all users |
| Success cookie | `logged_in=yes` on `.github.com` | `logged_in=yes` on `{hostname}` |
| OAuth apps | Registered at github.com | Registered on the GHES instance |
| Form structure | Same | Same (Rails app, identical DOM) |
| Device verification email | Yes | Depends on email config |

**Key difference for automation**: GHES often has SAML as the primary auth method. The login page may immediately redirect to an enterprise IdP (Okta, Azure AD, etc.). The agent must follow the redirect chain and handle the IdP's own login form.

---

## 2. GitLab (gitlab.com)

### 2.1 Login Page Structure

**URL**: `https://gitlab.com/users/sign_in`

**Form fields**:
- `input#user_login` - Username or email (name: `user[login]`)
- `input#user_password` - Password (name: `user[password]`)
- Hidden field: `authenticity_token` (Rails CSRF token)
- "Remember me" checkbox: `input#user_remember_me`
- Form action: `POST /users/sign_in`

**Page structure**:
- Single-page login form
- Tabs for "Standard" login vs SSO/SAML
- "Sign in with" social buttons (Google, GitHub, Bitbucket, Salesforce)
- "Register" tab for new accounts

**Key selectors**:
```javascript
// Login form
'form#new_user[action="/users/sign_in"]'
'input#user_login'
'input#user_password'
'input#user_remember_me'
'input[name="commit"]'           // Submit button ("Sign in")
'input[name="authenticity_token"]'

// Social login buttons
'a[href*="/users/auth/google_oauth2"]'
'a[href*="/users/auth/github"]'

// Tab navigation
'.login-page .nav-tabs'
'a[href="#login-pane"]'
'a[href="#register-pane"]'

// Error messages
'.flash-alert'
'.flash-warning'
```

### 2.2 CAPTCHA System

**Provider**: reCAPTCHA v2 (Google)

**When triggered**:
- NOT typically shown on login (differs from GitHub)
- Shown on **sign-up** (account creation)
- Shown on **password reset** requests
- Can be enabled by admins for login on self-hosted instances
- May appear after repeated failed logins (rate-limit triggered)

**Detection selectors**:
```javascript
// reCAPTCHA v2
'iframe[src*="recaptcha"]'
'div.g-recaptcha'
'iframe[src*="google.com/recaptcha"]'

// reCAPTCHA v3 (invisible, score-based)
// No visible iframe; detected by script tag:
'script[src*="recaptcha/api.js"]'
'script[src*="recaptcha/enterprise.js"]'
```

**GitLab-specific**: gitlab.com uses invisible reCAPTCHA (v3) for bot scoring on login. If the score is too low, it may show a v2 checkbox challenge. Self-hosted instances can configure reCAPTCHA or disable it entirely.

### 2.3 2FA Flows

After successful username/password, GitLab redirects to a 2FA prompt page.

**2FA URL**: `https://gitlab.com/users/sign_in` (same URL, different form rendered) or `https://gitlab.com/users/two_factor_auth`

**Method detection**:

| Method | URL/Selector | DOM Indicator |
|--------|-------------|---------------|
| TOTP (authenticator app) | `/users/sign_in` (2FA prompt) | `input#user_otp_attempt` |
| WebAuthn/security key | Same page | `button` with "Try again" or "Use security key"; `div#js-authenticate-token-2fa` |
| Recovery codes | Same page | Link "Use a recovery code" leading to recovery code input |

**Key 2FA selectors**:
```javascript
// TOTP code input
'input#user_otp_attempt'
'input[name="user[otp_attempt]"]'

// WebAuthn prompt
'#js-authenticate-token-2fa'
'button:has-text("Use security key")'
'div[data-qa-selector="webauthn_prompt"]'

// Recovery code fallback
'a:has-text("Use a recovery code")'
'a[href*="codes"]'

// 2FA page detection
'form[action*="otp_attempt"]'
'label:has-text("Enter verification code")'
```

**GitLab 2FA specifics**:
- GitLab does NOT support SMS-based 2FA (security choice)
- WebAuthn keys must be registered in advance at Settings > Account > Two-Factor Authentication
- Recovery codes are generated at 2FA enrollment time; 10 codes, single-use

### 2.4 OAuth/SSO Redirects

**SAML SSO (groups)**:
- URL pattern: `https://gitlab.com/groups/{group}/-/saml/sso` (group-level SAML)
- After SSO: redirect back with SAML response
- Detection: URL contains `/saml/sso`

**OAuth providers**:
- URL pattern: `https://gitlab.com/users/auth/{provider}` (e.g., `google_oauth2`, `github`, `bitbucket`)
- Callback: `https://gitlab.com/users/auth/{provider}/callback`

**LDAP (self-hosted)**:
- Login page shows LDAP tab alongside standard tab
- Tab selector: `a[href="#ldap"]` or `.login-box .nav-tabs`
- Form: `form#new_ldap_user`

**SSO vs direct login detection**:
```javascript
// SAML SSO indicators
url.includes('/saml/sso')
url.includes('/users/auth/')

// LDAP login (self-hosted)
document.querySelector('form#new_ldap_user')

// Direct login
url === 'https://gitlab.com/users/sign_in'
```

### 2.5 Success Detection

**Primary method - Cookie** (used in web-ctl providers.json):
```javascript
// Session cookie
{ domain: ".gitlab.com", name: "_gitlab_session" }

// Additional cookies after login:
// known_sign_in - device tracking cookie
// experimentation_subject_id - analytics
```

**Secondary method - DOM selector**:
```javascript
// User avatar in header
'.header-user-dropdown-toggle'
'a[data-user]'
'img.avatar'
'meta[name="csrf-token"]'  // Present on all pages but content changes per session
'.user-status-emoji'

// Sidebar presence (new navigation)
'.super-sidebar'
'a[data-track-label="profile_dropdown"]'
```

**URL-based detection**:
```javascript
// After login, redirects to dashboard or project
url.startsWith('https://gitlab.com/dashboard')
url.startsWith('https://gitlab.com/') && !url.includes('/users/sign_in')
```

### 2.6 Anti-Bot Measures

| Measure | Details |
|---------|---------|
| reCAPTCHA v3 | Invisible bot scoring; may escalate to v2 checkbox |
| Rate limiting | Login: ~10 attempts per IP per minute; API: varies by endpoint |
| Account lockout | 10 consecutive failures locks account for 10 minutes (configurable on self-hosted) |
| `authenticity_token` | Rails CSRF token required on form submission |
| IP blocking | Repeated abuse from IP can trigger temporary block |
| Rack::Attack | GitLab uses Rack::Attack middleware for request throttling |

### 2.7 Playwright-Specific Gotchas

1. **Rails CSRF**: Same as GitHub - must load login page first to get the `authenticity_token`.
2. **SPA transitions**: GitLab's UI uses Vue.js. Navigation may not trigger full page loads. Use `page.waitForURL()` or `page.waitForSelector()` rather than `page.waitForNavigation()`.
3. **Invisible reCAPTCHA**: May silently block form submission. If login POST returns to the same page with no error, reCAPTCHA scoring may have failed. Use a real Chrome binary (`--channel chrome`) instead of Chromium.
4. **Cookie domain**: `_gitlab_session` is set on `.gitlab.com` domain. Self-hosted instances use their own domain.
5. **known_sign_in cookie**: GitLab sets this to remember trusted devices. Without it, a "new sign-in" email notification is sent. Persisting this cookie across sessions reduces friction.

### 2.8 GitLab Self-Hosted vs gitlab.com

| Aspect | gitlab.com | Self-Hosted |
|--------|-----------|-------------|
| Login URL | `/users/sign_in` | `/users/sign_in` (same path) |
| CAPTCHA | reCAPTCHA v3 (invisible) | Configurable; often disabled |
| SSO | Group-level SAML | Instance-level SAML/LDAP/OIDC |
| 2FA enforcement | Optional per-user | Admin can enforce globally |
| Session cookie | `_gitlab_session` on `.gitlab.com` | `_gitlab_session` on `{hostname}` |
| OAuth providers | Google, GitHub, Bitbucket, etc. | Configurable; may be just LDAP |
| Rate limiting | Rack::Attack defaults | Configurable thresholds |
| Form structure | Same | Same (Ruby on Rails, identical DOM) |
| Login tabs | Standard + Social | Standard + LDAP + Social (if configured) |

**Key difference**: Self-hosted instances frequently use LDAP or SAML as primary auth. The login page may show only an LDAP form, or redirect entirely to an IdP. The agent must handle the LDAP tab separately from the standard tab.

---

## 3. Atlassian (id.atlassian.com)

### 3.1 Login Page Structure

**URL**: `https://id.atlassian.com/login` (shared by Jira, Confluence, Bitbucket, Trello)

**CRITICAL: Multi-step login flow**:

Unlike GitHub and GitLab, Atlassian uses a **two-step** login:

1. **Step 1 - Email**: User enters email address only
2. **Step 2 - Password**: After email submission, a new screen appears for password entry

This is a React SPA - the URL may not change between steps.

**Step 1 selectors**:
```javascript
// Email input
'input#username'
'input[name="username"]'
'input[autocomplete="username"]'

// Continue button
'button#login-submit'
'button[type="submit"]'

// Page detection
'form#form-login'
```

**Step 2 selectors**:
```javascript
// Password input (appears after email submission)
'input#password'
'input[name="password"]'
'input[type="password"]'

// Log in button
'button#login-submit'

// "Can't log in?" link
'a[href*="forgot-password"]'
```

**Important for automation**: Between step 1 and step 2, there is a brief loading state. The agent must wait for the password field to appear:
```javascript
await page.fill('#username', email);
await page.click('#login-submit');
await page.waitForSelector('#password', { timeout: 10000 });
await page.fill('#password', password);
await page.click('#login-submit');
```

**Alternative flows from Step 1**:
- If the email domain has SSO configured, Atlassian redirects to the enterprise IdP instead of showing password
- If the account uses Google login, a "Log in with Google" button may appear
- If the account uses Apple login, similar redirect

### 3.2 CAPTCHA System

**Provider**: Likely Cloudflare Turnstile or custom

**When triggered**:
- Not commonly shown on normal login attempts
- Triggered after multiple failed attempts
- Triggered from datacenter/VPN IP ranges
- May appear during account creation

**Detection selectors**:
```javascript
// Turnstile (Cloudflare)
'iframe[src*="challenges.cloudflare.com"]'
'div.cf-turnstile'

// Generic
'div[class*="captcha"]'
'iframe[src*="captcha"]'

// Text patterns
'verify you are human'
'security check'
```

**Note**: Atlassian's CAPTCHA usage is less aggressive than GitHub's. The multi-step login flow itself serves as a mild bot deterrent since simple form-submission bots often fail on the async step transition.

### 3.3 2FA Flows

Atlassian calls this "two-step verification" (not "two-factor authentication").

**2FA prompt appears after successful password entry on a new page/screen.**

**URL**: `https://id.atlassian.com/login` (same SPA, different state) or `https://id.atlassian.com/login/authorize`

**Method detection**:

| Method | DOM Indicator |
|--------|---------------|
| TOTP (authenticator app) | `input[name="otpCode"]` or text: "Enter the 6-digit code" |
| Security key (WebAuthn) | Button: "Use security key" |
| Email verification code | Text: "We sent a verification code to your email" + code input |
| Recovery code | Link: "Use a recovery code" |

**Key 2FA selectors**:
```javascript
// TOTP input
'input#mfa-code'
'input[name="otpCode"]'
'input[name="verificationCode"]'

// Verify button
'button#verify-btn'
'button[type="submit"]:has-text("Verify")'

// WebAuthn
'button:has-text("Use security key")'
'button:has-text("Use a passkey")'

// Recovery code
'a:has-text("Use a recovery code")'
'a:has-text("Can\'t access")'

// 2FA page detection
'[data-testid="mfa-container"]'
'form:has(input[name="otpCode"])'
```

**Atlassian 2FA specifics**:
- SMS is NOT supported (phased out)
- Email verification codes are sent to the account email
- Push notifications are available through Atlassian app (mobile)
- Admin-enforced 2FA applies to all Atlassian products (Jira, Confluence, Bitbucket, Trello)

### 3.4 OAuth/SSO Redirects

**SAML SSO (organization-managed)**:
- Atlassian Cloud uses "Atlassian Access" for SSO
- URL pattern: After entering email in step 1, if domain has SSO, redirects to IdP
- Return URL: `https://id.atlassian.com/login/authorize?...`
- Detection: After email entry, page redirects away from `id.atlassian.com` to enterprise IdP

**OAuth**:
- Atlassian OAuth 2.0 (3LO): `https://auth.atlassian.com/authorize?...`
- Callback: `https://auth.atlassian.com/...` then redirect to app

**Google/Apple login**:
- URL pattern: Redirect to `accounts.google.com` or `appleid.apple.com` from login page
- Detection: URL leaves `id.atlassian.com` domain

**SSO vs direct login detection**:
```javascript
// After email entry, check if redirected
const postEmailUrl = page.url();

// SSO redirect indicators
!postEmailUrl.includes('id.atlassian.com')  // Redirected to IdP
postEmailUrl.includes('login.microsoftonline.com')  // Azure AD SSO
postEmailUrl.includes('accounts.google.com')         // Google SSO
postEmailUrl.includes('okta.com')                     // Okta SSO

// Direct login indicator (password field appears on same domain)
await page.$('#password')  // If password field exists, it's direct login
```

### 3.5 Success Detection

**Primary method - URL** (used in web-ctl providers.json):
```javascript
// After successful login, redirects to:
successUrl: "https://start.atlassian.com"

// Or the product the user was trying to access:
// https://{instance}.atlassian.net/...  (Jira/Confluence)
// https://bitbucket.org/...              (Bitbucket)
// https://trello.com/...                 (Trello)
```

**Cookie-based detection**:
```javascript
// Atlassian Cloud session cookies (post-login):
// cloud.session.token - on .atlassian.com (HttpOnly, Secure)
// tenant.session.token - on .atlassian.net (per-instance)

// Bitbucket specific:
// bb_session - on .bitbucket.org

// Trello specific:
// token - on .trello.com
```

**DOM-based detection**:
```javascript
// On start.atlassian.com after login
'[data-testid="switcher-product-list"]'  // Product switcher
'img[alt*="avatar"]'                      // User avatar

// On Jira/Confluence
'[data-testid="global-navigation.profile-button"]'  // Profile button in nav
```

### 3.6 Anti-Bot Measures

| Measure | Details |
|---------|---------|
| Multi-step login | Email-first flow itself is anti-bot |
| Rate limiting | ~5 failed attempts triggers increasing delays |
| Account lockout | Multiple failures lock account; requires email verification |
| CAPTCHA | Turnstile/custom; triggered by suspicious patterns |
| Device trust | Trusted device cookies; new device triggers email notification |
| IP reputation | Datacenter IPs may get extra scrutiny |

### 3.7 Playwright-Specific Gotchas

1. **Two-step form**: The biggest automation gotcha. Must wait for password field to appear after submitting email. Cannot fill both fields at once.
2. **React SPA**: URL may not change between login steps. Cannot rely on URL navigation events. Must poll for DOM changes.
3. **Dynamic selectors**: Atlassian's React app uses data-testid attributes which are more stable than CSS classes, but class names change across deployments.
4. **SSO redirect race**: After entering email, there is a brief moment before SSO redirect happens. The agent must wait and check whether a password field appears OR a redirect occurs.
5. **Cross-product cookies**: Logging into Jira doesn't automatically mean Bitbucket is authenticated. Each product may require visiting it once to establish its own session cookie.
6. **`cloud.session.token`**: This cookie is HttpOnly and Secure. Cannot be read via JavaScript in the page. Must use `context.cookies()`.

### 3.8 Atlassian Cloud vs Data Center

| Aspect | Atlassian Cloud | Data Center (self-hosted) |
|--------|----------------|--------------------------|
| Login URL | `https://id.atlassian.com/login` | `https://{hostname}/login.jsp` (Jira) or `https://{hostname}/login.action` (Confluence) |
| Login flow | Multi-step (email then password) | Single form (username + password) |
| CAPTCHA | Turnstile/custom | Usually none; configurable |
| SSO | Atlassian Access (SAML/OIDC) | Crowd, SAML, LDAP |
| 2FA | Atlassian Account 2-step | Depends on auth provider |
| Success cookie | `cloud.session.token` on `.atlassian.com` | `JSESSIONID` on `{hostname}` |
| Session cookie name | `cloud.session.token`, `tenant.session.token` | `JSESSIONID`, `seraph.os.cookie` |
| Form structure | React SPA | JSP/Velocity templates (static HTML) |

**Data Center key difference**: The login form is a traditional HTML form, not a React SPA. Single page with username + password. Much simpler to automate.

```javascript
// Jira Data Center login selectors
'input#login-form-username'     // or 'input[name="os_username"]'
'input#login-form-password'     // or 'input[name="os_password"]'
'input#login-form-submit'       // or 'input[name="login"]'
'form#login-form'

// Confluence Data Center login selectors
'input#os_username'
'input#os_password'
'input#loginButton'
'form[action="/dologin.action"]'
```

---

## 4. Bitbucket (bitbucket.org)

Bitbucket uses Atlassian ID for authentication, so the login flow is identical to Atlassian Cloud (Section 3).

### 4.1 Key Differences from Atlassian Generic

**Login URL**: `https://bitbucket.org/account/signin/` - redirects to `https://id.atlassian.com/login?...&application=bitbucket`

**Success detection**:
```javascript
// After login, redirects to:
'https://bitbucket.org/dashboard'
'https://bitbucket.org/{workspace}'

// Success cookies (Bitbucket-specific):
{ domain: ".bitbucket.org", name: "bb_session" }

// DOM selector on dashboard
'[data-testid="global-navigation"]'
'a[href*="/account/"]'  // Account settings link in nav
```

**Bitbucket Server (self-hosted) vs Bitbucket Cloud**:
- Bitbucket Server uses its own login at `https://{hostname}/login`
- Form: `input#j_username`, `input#j_password`, `form#loginForm`
- Success cookie: `JSESSIONID` on `{hostname}`
- Bitbucket Server is being deprecated in favor of Data Center

---

## 5. Cross-Provider Comparison

### Success Detection Summary

| Provider | Cookie Method | Cookie Name | Cookie Domain | URL Method | DOM Method |
|----------|--------------|-------------|---------------|-----------|-----------|
| GitHub | Preferred | `logged_in` (value: `yes`) | `.github.com` | Redirect away from `/login` | `meta[name="user-login"]` with non-empty content |
| GitLab | Preferred | `_gitlab_session` | `.gitlab.com` | Redirect to `/dashboard` | `.header-user-dropdown-toggle` |
| Atlassian Cloud | Secondary | `cloud.session.token` | `.atlassian.com` | Redirect to `start.atlassian.com` (preferred) | `[data-testid="switcher-product-list"]` |
| Bitbucket | Secondary | `bb_session` | `.bitbucket.org` | Redirect to `/dashboard` | `[data-testid="global-navigation"]` |

### CAPTCHA Comparison

| Provider | CAPTCHA Provider | Triggered On Login? | Iframe Selector |
|----------|-----------------|--------------------|-----------------|
| GitHub | Arkose Labs (FunCAPTCHA) | Yes (suspicious) | `iframe[src*="arkose"]` |
| GitLab | Google reCAPTCHA v3/v2 | Rarely (invisible scoring) | `iframe[src*="recaptcha"]`, `div.g-recaptcha` |
| Atlassian | Cloudflare Turnstile | Rarely | `iframe[src*="challenges.cloudflare.com"]` |

### 2FA Method Support

| Method | GitHub | GitLab | Atlassian |
|--------|--------|--------|-----------|
| TOTP (authenticator app) | Yes | Yes | Yes |
| SMS | Yes | No | No |
| Email code | No | No | Yes |
| WebAuthn/Security key | Yes | Yes | Yes |
| Passkey (resident) | Yes | Yes (newer versions) | Yes |
| Push notification | Yes (GitHub Mobile) | No | Yes (Atlassian app) |
| Recovery/backup codes | Yes | Yes | Yes |

### Login Flow Shape

| Provider | Steps | Notes |
|----------|-------|-------|
| GitHub | 1 page (username + password together) | Then separate 2FA page |
| GitLab | 1 page (username + password together) | Then 2FA on same or new page |
| Atlassian Cloud | 2 steps (email first, then password) | React SPA, URL may not change |
| Atlassian Data Center | 1 page (username + password) | Traditional HTML form |

---

## 6. Recommendations for web-ctl providers.json

Based on this research, the current `providers.json` entries should be enhanced:

### GitHub - Missing CAPTCHA selectors

```json
{
  "slug": "github",
  "name": "GitHub",
  "loginUrl": "https://github.com/login",
  "successCookie": { "domain": ".github.com", "name": "logged_in", "value": "yes" },
  "successSelector": "meta[name=\"user-login\"][content]",
  "captchaSelectors": [
    "iframe[src*=\"arkose\"]",
    "iframe[src*=\"arkoselabs.com\"]",
    "iframe[src*=\"funcaptcha\"]"
  ],
  "captchaTextPatterns": [
    "verify your account",
    "confirm your identity"
  ],
  "twoFactorSelectors": [
    "input#app_totp",
    "input[name=\"app_otp\"]",
    "form[action*=\"two-factor\"]"
  ],
  "twoFactorHint": "GitHub may prompt for a 2FA code, passkey, or security key. Check GitHub Mobile for push notifications."
}
```

### GitLab - Add success selector and CAPTCHA

```json
{
  "slug": "gitlab",
  "name": "GitLab",
  "loginUrl": "https://gitlab.com/users/sign_in",
  "successCookie": { "domain": ".gitlab.com", "name": "_gitlab_session" },
  "successSelector": ".header-user-dropdown-toggle",
  "captchaSelectors": [
    "iframe[src*=\"recaptcha\"]",
    "div.g-recaptcha"
  ],
  "captchaTextPatterns": [],
  "twoFactorSelectors": [
    "input#user_otp_attempt",
    "input[name=\"user[otp_attempt]\"]"
  ],
  "twoFactorHint": "GitLab may prompt for a TOTP code or security key. GitLab does not support SMS 2FA."
}
```

### Atlassian - Add multi-step awareness

```json
{
  "slug": "atlassian",
  "name": "Atlassian",
  "loginUrl": "https://id.atlassian.com/login",
  "successUrl": "https://start.atlassian.com",
  "successCookie": { "domain": ".atlassian.com", "name": "cloud.session.token" },
  "captchaSelectors": [
    "iframe[src*=\"challenges.cloudflare.com\"]",
    "div.cf-turnstile"
  ],
  "captchaTextPatterns": [],
  "loginFlow": "multi-step",
  "loginSteps": [
    { "field": "#username", "type": "email" },
    { "waitFor": "#password" },
    { "field": "#password", "type": "password" }
  ],
  "twoFactorSelectors": [
    "input[name=\"otpCode\"]",
    "input#mfa-code"
  ],
  "twoFactorHint": "Atlassian may prompt for a verification code from your authenticator app or email."
}
```

---

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| Atlassian password field not found | Agent tries to fill password immediately without waiting for step 2 | Wait for `#password` selector after submitting email |
| GitHub CAPTCHA blocks automated login | Headless browser + new IP triggers Arkose | Use headed browser with VNC; persist cookies to avoid repeated login |
| GitLab SAML redirect not followed | Agent expects password form but SSO redirects to IdP | Check if URL leaves `gitlab.com` after form submission |
| `meta[name="user-login"]` false positive | Tag exists on login page with empty content | Validate that content attribute is non-empty |
| Bitbucket auth assumed separate from Atlassian | Different domain | Bitbucket redirects to `id.atlassian.com`; same auth flow |
| GHES login fails silently | Enterprise SSO configured but agent tries username/password | Check for SAML redirect before attempting password login |
| Session cookie not persisted on browser close | Session cookies (no expiry) discarded | Use `context.storageState()` or cookie jar with `ignore_discard=True` |
| 2FA page not detected | Agent checks URL but 2FA page may have same URL | Check for 2FA DOM selectors, not just URL patterns |
| Cross-product Atlassian session missing | Logged into Jira but Confluence needs separate visit | Navigate to each product after login to establish per-product cookies |

---

## Best Practices

1. **Always use cookie-based success detection as primary method.** URL heuristics can fail when sites add query parameters or use SPAs. DOM selectors can fail on redesigns. Cookies are the most stable indicator. (Source: web-ctl auth-check.js pattern)

2. **Detect CAPTCHA early and escalate to human-in-the-loop.** Do not retry automated login if CAPTCHA is present - it will never succeed. Open VNC or headed browser immediately. (Source: web-ctl auth-flow.js pattern)

3. **Handle Atlassian's multi-step flow explicitly.** Never assume a login form has both username and password visible simultaneously. Wait for each step. (Source: Atlassian login behavior)

4. **Persist the full cookie jar, not just the "success" cookie.** GitHub rotates `_gh_sess`, GitLab uses `known_sign_in` for device trust, Atlassian has cross-product cookies. All are needed for smooth operation. (Source: web-ctl session-store pattern)

5. **Check for SSO redirect after email/username entry.** Enterprise users may be redirected to Okta, Azure AD, or other IdPs. The agent should detect the domain change and inform the user. (Source: enterprise auth patterns)

6. **Use `meta[name="user-login"]` for GitHub with content validation.** This is more reliable than avatar selectors which change across GitHub UI redesigns. (Source: web-ctl auth-check.js)

7. **For self-hosted instances, allow custom provider overrides.** The `loadCustomProviders()` pattern in web-ctl lets users override built-in provider configs with their own URLs, selectors, and cookies. (Source: web-ctl auth-providers.js)

8. **Rate limit your own automation.** Even with valid credentials, rapid automated requests trigger anti-bot measures. Add 1-2 second delays between page navigations.

---

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [GitHub authentication docs](https://docs.github.com/en/authentication) | Official Docs | Authoritative reference for GitHub auth methods |
| [GitLab 2FA docs](https://docs.gitlab.com/ee/user/profile/account/two_factor_authentication.html) | Official Docs | GitLab 2FA setup and recovery |
| [Atlassian two-step verification](https://support.atlassian.com/atlassian-account/docs/manage-two-step-verification-for-your-atlassian-account/) | Official Docs | Atlassian 2FA management |
| [Arkose Labs detection](https://arkoselabs.com) | Vendor | Understanding FunCAPTCHA challenges |
| [Playwright auth guide](https://playwright.dev/docs/auth) | Official Docs | storageState pattern for session persistence |
| web-ctl `providers.json` | Codebase | Current provider configurations |
| web-ctl `auth-flow.js` | Codebase | CAPTCHA detection and human-in-the-loop pattern |
| web-ctl `auth-check.js` | Codebase | Success detection logic with cookie, URL, and DOM methods |
| `cli-browser-automation-agents.md` | Knowledge Base | Companion guide on browser automation tools |
| `web-session-persistence-cli-agents.md` | Knowledge Base | Companion guide on session/cookie persistence |

---

*Generated by /learn from 28 sources (1 live web fetch, 27 from training knowledge + codebase analysis).*
*See `resources/auth-flows-github-gitlab-atlassian-sources.json` for full source metadata.*
