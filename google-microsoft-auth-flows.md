# Learning Guide: Google and Microsoft Authentication Flows for Browser Automation Agents

**Generated**: 2026-02-22
**Sources**: 28 resources synthesized (Microsoft docs, Google support docs, Playwright/automation community knowledge, web-ctl codebase)
**Depth**: deep

---

## Prerequisites

- Familiarity with Playwright or Puppeteer browser automation
- Understanding of cookies, DOM selectors, and HTTP redirects
- Basic knowledge of OAuth 2.0 concepts
- Read `web-session-persistence-cli-agents.md` and `cli-browser-automation-agents.md` first

---

## TL;DR

- Google uses a multi-step login (email page -> password page -> optional 2FA page), each on different URL paths under `accounts.google.com`. The SAPISID cookie on `.google.com` is the most reliable auth success indicator.
- Microsoft uses `login.microsoftonline.com` with tenant-specific paths. The flow is also multi-step (email -> password or passwordless -> optional MFA). ESTSAUTH and ESTSAUTHPERSISTENT cookies confirm auth success.
- Both providers use risk-based analysis that can trigger CAPTCHAs, device verification, or "unusual activity" pages at any step - automation fingerprints (headless flags, missing GPU, fast input) increase risk scores.
- Google has no CAPTCHA-free programmatic path for browser-based login; Microsoft's conditional access policies are tenant-configurable and may block automation entirely.
- The only reliable automated approach is human-in-the-loop: open a headed/VNC browser, let the user complete auth manually, then capture the session.

---

## Google Authentication

### 1. Login Page Structure

Google uses a multi-step sign-in flow. Each step is a distinct page load with different DOM elements.

#### Step 1: Email/Identifier

**URL**: `https://accounts.google.com/v3/signin/identifier?...` (or `/ServiceLogin` for legacy)

Key URL parameters:
- `flowName=GlifWebSignIn` - the standard web sign-in flow name
- `flowEntry=ServiceLogin` - entry point type
- `continue=<encoded-url>` - where to redirect after auth
- `hl=en` - language
- `passive=true` - silent check (no UI if already logged in)

**DOM selectors**:
```
Email input:       input[type="email"], input#identifierId
Next button:       button#identifierNext, div#identifierNext button
                   (or button containing span with "Next" text)
"Try another way": a[data-action="switchTo"]
Google One Tap:    div#credential_picker_container (if embedded on third-party site)
```

**Detection**: Page contains `identifierId` input or URL path includes `/signin/identifier`.

#### Step 2: Password

**URL**: `https://accounts.google.com/v3/signin/challenge/pwd?...`

URL path changes to `/challenge/pwd` after email submission. The `TL` parameter in the URL is a flow token.

**DOM selectors**:
```
Password input:    input[type="password"], input[name="Passwd"]
Next button:       button#passwordNext, div#passwordNext button
Forgot password:   button[jsname="Cuz2Ue"], a containing "Forgot password"
Show password:     div[jsname="YRMnle"] (toggle visibility)
```

**Detection**: URL path includes `/challenge/pwd` or `/signin/challenge/pwd`.

#### Step 3: 2FA Challenge (if enabled)

**URL varies by method**:
```
Google Prompt:     /challenge/ipp     (phone prompt - "tap yes on your phone")
TOTP:              /challenge/totp    (authenticator app code)
SMS:               /challenge/sms     (text message code)
Backup codes:      /challenge/bc      (8-digit backup code)
Security key:      /challenge/sk      (FIDO/U2F hardware key)
Phone call:        /challenge/pc      (voice call)
Recovery email:    /challenge/recov   (code sent to recovery email)
```

**DOM selectors by 2FA type**:

```
--- Google Prompt (ipp) ---
Challenge text:    div[data-challengetype="6"]
                   Content: "Check your phone" or "Tap Yes on your phone"
Phone hint:        div.Srfskc (shows last digits of phone number)
"Try another way": button/a with text "Try another way"

--- TOTP (totp) ---
Code input:        input[name="totpPin"], input#totpPin
                   input[type="tel"][name="Pin"]
Next button:       button#totpNext
Label text:        "Enter the code from your authenticator app"

--- SMS (sms) ---
Code input:        input#smsCode, input[name="smsPin"]
Next button:       button#smsNext
Resend:            button containing "Resend" text
Label text:        "Enter the code we sent to" + masked phone

--- Backup Codes (bc) ---
Code input:        input#backupCodePin
Next button:       button#backupCodeNext
Label text:        "Enter one of your 8-digit backup codes"

--- Security Key (sk) ---
Prompt:            div[data-challengetype="6"] or platform-specific WebAuthn dialog
Label text:        "Use your security key" or "Insert your security key"
Note:              Browser's native WebAuthn dialog appears; not automatable

--- Recovery Email ---
Code input:        input[name="knowledgePreregisteredEmailResponse"]
```

**Detecting which 2FA is active**: Check the URL path (`/challenge/totp`, `/challenge/ipp`, etc.) or look for `data-challengetype` attributes. The challenge type numbers:
- 6 = Google prompt / security key
- 9 = Phone
- 13 = TOTP

#### Account Chooser (Multi-Account)

**URL**: `https://accounts.google.com/AccountChooser?...` or `https://accounts.google.com/v3/signin/identifier` with account list visible.

**DOM selectors**:
```
Account list:      div[data-identifier], div.JDAKTe
Account email:     div[data-identifier="user@gmail.com"]
"Use another":     div containing "Use another account"
                   or li[data-identifier="other"]
```

**Detection**: Multiple `div[data-identifier]` elements present, or `data-screenindex` attribute on the page container.

### 2. Google CAPTCHA Systems

#### reCAPTCHA v2 (Checkbox)

```
Container:         div.g-recaptcha, div[data-sitekey]
iframe:            iframe[src*="recaptcha/api2/anchor"]
Checkbox:          span.recaptcha-checkbox (inside iframe)
Challenge iframe:  iframe[src*="recaptcha/api2/bframe"] (image challenge)
```

#### reCAPTCHA v3 (Invisible)

No visible UI. Runs in background and assigns a score (0.0 to 1.0). Scores below threshold trigger a fallback (usually v2 checkbox or custom challenge).

```
Badge:             div.grecaptcha-badge (small badge in corner, often hidden via CSS)
Script:            script[src*="recaptcha/api.js?render="]
```

#### reCAPTCHA Enterprise

Used on Google's own login pages. Similar to v3 but with additional signals. No public selectors - integrated directly into Google's login JavaScript.

#### What triggers CAPTCHA on Google login:

- Headless browser detection (missing `window.chrome`, navigator.webdriver flag)
- Rapid form submission (typing speed too fast or instant)
- IP reputation (datacenter IPs, VPNs, Tor)
- Missing or inconsistent browser fingerprint
- No prior Google cookies (fresh browser profile)
- Multiple failed login attempts
- Geographic anomaly (IP location vs account's usual location)

### 3. Google Anti-Bot and Risk Analysis

#### "Verify it's you" Page

**URL**: `https://accounts.google.com/signin/v2/challenge/selection?...`

This page appears when Google's risk engine flags the login attempt. It asks the user to verify identity through one of several methods.

```
Challenge options: div[data-challengetype] (multiple, user picks one)
Selection list:    ul containing li with challenge descriptions
```

#### "Unusual Activity" / Blocked Page

**URL**: `https://accounts.google.com/signin/v2/disabled` or similar

```
Page text:         "suspicious activity", "unusual activity", "couldn't verify"
                   "This browser or app may not be secure"
Error container:   div.GQ8Pzc, div[jsname="B34EJ"]
```

The "This browser or app may not be secure" error specifically targets automation:
- Triggered by `navigator.webdriver = true`
- Triggered by missing GPU/WebGL fingerprint
- Triggered by Chrome launched with `--disable-blink-features=AutomationControlled` detection

#### Device Verification

Google may require the user to confirm from a trusted device:
```
URL path:          /challenge/dp (device prompt)
Content:           "We sent a notification to your [device]"
                   Shows device name and location
```

### 4. Google Cookies (Auth Success Indicators)

All set on domain `.google.com`:

| Cookie | Purpose | Reliability for Auth Detection |
|--------|---------|-------------------------------|
| `SID` | Primary session identifier | High - present after successful login |
| `HSID` | HTTP-only session ID | High |
| `SSID` | Secure session ID | High |
| `APISID` | API session ID | High |
| `SAPISID` | Secure API session ID | **Best indicator** - only set after full authentication |
| `NID` | Preferences, not auth | Low - set even without login |
| `__Secure-1PSID` | Partitioned session cookie | High (newer) |
| `__Secure-3PSID` | Partitioned session cookie | High (newer) |
| `LSID` | Login session (accounts.google.com only) | Medium |

**Recommended detection** (as used in web-ctl `providers.json`):
```json
{ "domain": ".google.com", "name": "SAPISID" }
```

The SAPISID cookie is the best single indicator because:
- Only set after completing full authentication (including 2FA if required)
- Not present for unauthenticated visitors
- Set on `.google.com` so visible from any Google property
- Long-lived (typically 2 years expiry)

### 5. Google Workspace vs Personal Accounts

**Personal accounts** (`@gmail.com`):
- Login at `accounts.google.com`
- Standard 2FA options
- No admin-enforced policies
- Success redirects to `myaccount.google.com` or the `continue` URL

**Google Workspace accounts** (`@company.com`):
- May redirect to a custom SSO/SAML provider before reaching Google
- URL may include `hd=company.com` (hosted domain) parameter
- Admin can enforce specific 2FA methods
- Admin can block "less secure apps"
- May use third-party IdP (Okta, Azure AD, etc.) - flow leaves Google entirely
- After SSO, returns to Google with SAML assertion

**Detection of Workspace SSO redirect**:
```
URL contains:      /o/saml2/initsso
                   or redirects to a non-Google domain
Parameter:         hd=<domain> in the identifier URL
DOM element:       Page may show "Sign in with your organization's credentials"
```

### 6. OAuth/SSO Flows

#### Google OAuth 2.0 Consent Screen

**URL**: `https://accounts.google.com/o/oauth2/v2/auth?...`

After authentication, if the app requests OAuth scopes, a consent screen appears:

```
Consent container: div#submit_approve_access
Allow button:      button#submit_approve_access
Deny button:       button#submit_deny_access
Scope list:        div.scope-list, ul.scope-list
App name:          div containing the requesting app's name
```

#### Google Sign-In on Third-Party Sites

Google One Tap / Sign in with Google:
```
One Tap popup:     div#credential_picker_container iframe
                   iframe[src*="accounts.google.com/gsi"]
Button:            div.g_id_signin, div[data-client_id]
Popup:             Window with accounts.google.com/o/oauth2/...
```

---

## Microsoft Authentication

### 1. Login Page Structure

Microsoft uses `login.microsoftonline.com` with tenant-specific routing.

#### URL Patterns

```
Common login:      https://login.microsoftonline.com/common/oauth2/v2.0/authorize
Organizations:     https://login.microsoftonline.com/organizations/...
Consumers only:    https://login.microsoftonline.com/consumers/...
Tenant-specific:   https://login.microsoftonline.com/{tenant-id}/...
                   https://login.microsoftonline.com/{domain.com}/...
```

Legacy endpoints (still active):
```
login.live.com              - Personal Microsoft accounts
login.windows.net           - Legacy Azure AD
login.microsoftonline.com   - Current standard
```

#### Step 1: Email/Identifier

**URL**: `https://login.microsoftonline.com/common/oauth2/v2.0/authorize?...` (initial load)

The page renders a single email input first:

**DOM selectors**:
```
Email input:       input[type="email"][name="loginfmt"]
Next button:       input[type="submit"]#idSIButton9
                   (value="Next")
Back button:       input#idBtn_Back
Error text:        div#usernameError
                   Content: "Enter a valid email address"
Page title:        div#loginHeader containing "Sign in"
```

After entering email, Microsoft determines account type:
- Personal (outlook.com, hotmail.com, live.com) -> stays on login.microsoftonline.com
- Work/school -> may redirect to tenant-specific page or federated IdP

#### Step 2: Password

**URL**: Same base URL, but the page content changes via JavaScript (SPA-like).

**DOM selectors**:
```
Password input:    input[type="password"][name="passwd"]#i0118
Submit button:     input[type="submit"]#idSIButton9
                   (value="Sign in")
Forgot password:   a#idA_PWD_ForgotPassword
Error text:        div#passwordError
                   Content: "Your account or password is incorrect"
"Sign in another": div#otherTile, a#otherTileText
```

#### Step 2 (Alternative): Passwordless

Microsoft now defaults to passwordless for many accounts (as of 2025+). Instead of a password field, the user sees:

```
Passwordless prompt:  div#idDiv_SAOTCS_Proofs
                      Content: "Approve sign in request" or "Enter the number shown"
Number match:         div.displaySign (shows a 2-digit number to match in Authenticator)
"Use password":       a#idA_SAOTCS_LookupHelp or link with text "Use your password instead"
```

**Detection**: After entering email, if `div#idDiv_SAOTCS_Proofs` appears instead of password input, the account is configured for passwordless.

#### Step 3: MFA/2FA (if required)

Microsoft's MFA page appears after successful password entry when MFA is enforced.

**URL**: Still on `login.microsoftonline.com`, content changes.

**DOM selectors by MFA method**:

```
--- Microsoft Authenticator Push ---
Container:         div#idDiv_SAOTCAS_Description
Content:           "Approve the sign in request" or "We've sent a notification to your..."
Number match:      div.displaySign (2-digit number to match in app)
"I can't use":     a#idA_SAOTCAS_LookupHelp

--- SMS Verification ---
Container:         div#idDiv_SAOTCC_Description
Code input:        input#idTxtBx_SAOTCC_OTC
Verify button:     input#idSubmit_SAOTCC_Continue
Content:           "Enter the code we texted to..." + masked phone number

--- Phone Call ---
Container:         div#idDiv_SAOTCA_Description
Content:           "We're calling..." + masked phone number

--- TOTP (Authenticator App Code) ---
Container:         div#idDiv_SAOTCC_Description (same as SMS container)
Code input:        input#idTxtBx_SAOTCC_OTC
Content:           "Enter the code from your authenticator app"

--- FIDO2 / Security Key ---
Content:           "Touch your security key" or browser WebAuthn dialog
Detection:         Browser native dialog, not automatable

--- "Other ways to sign in" ---
Link:              a#idA_SAOTCS_LookupHelp, a#signInAnotherWay
Options list:      div#idDiv_SAOTCS_Proofs containing radio buttons or links
```

**Detecting which MFA method**: Look for the container div IDs:
- `idDiv_SAOTCAS_*` = Authenticator push notification
- `idDiv_SAOTCC_*` = Code entry (SMS or TOTP)
- `idDiv_SAOTCA_*` = Phone call

#### "Stay signed in?" Prompt (KMSI)

After successful auth, Microsoft shows a "Stay signed in?" prompt:

```
URL:               Same login.microsoftonline.com
Container:         div#KmsiBanner or div.kmsi
Yes button:        input#idSIButton9 (value="Yes")
No button:         input#idBtn_Back (value="No")
Checkbox:          input#KmsiFlagCheck ("Don't show this again")
Text:              "Stay signed in?" / "Reduce the number of times you are asked to sign in"
```

**Important for automation**: This page must be handled or it blocks redirect to the target application. Click "Yes" or "No" to proceed.

### 2. Microsoft CAPTCHA Systems

Microsoft uses less visible CAPTCHA than Google. Historically:

```
Arkose Labs/FunCaptcha:  iframe[src*="arkoselabs.com"]
                         iframe[src*="funcaptcha"]
Custom challenges:       div#hipEnforcementContainer (Visual/audio puzzle)
```

Microsoft's CAPTCHA triggers:
- Suspicious IP addresses
- Bot-like behavior patterns
- Multiple failed sign-in attempts
- Account under attack (credential stuffing detected)

Microsoft relies more on conditional access and risk-based policies than CAPTCHAs for bot protection.

### 3. Microsoft Anti-Bot and Risk-Based Access

#### Conditional Access Policies

Tenant admins can configure policies that trigger additional requirements:

**Common policies an automation agent might encounter**:
- Require MFA from untrusted networks
- Block access from non-compliant devices
- Require managed device
- Block legacy authentication protocols
- Block access from specific countries/regions
- Require terms of use acceptance

**Conditional access block page**:
```
Container:         div#error_description or custom error page
Text:              "You cannot access this right now"
                   "Your sign-in was successful but does not meet the criteria"
                   "Your admin has configured a security policy"
Error codes:       AADSTS53003 (blocked by conditional access)
                   AADSTS50076 (MFA required)
                   AADSTS530034 (security defaults - MFA required)
```

#### "Proof Up" / MFA Registration

When a user has not yet registered MFA methods but the tenant requires it:

```
URL:               https://login.microsoftonline.com/.../combinedregistration
Container:         div#ProofUpDescription
Text:              "More information required" / "Your organization requires additional security"
Action required:   User must set up at least one MFA method
Skip (if allowed): a#proofUpSkipLink
```

This is a **hard block** for automation - the user must interactively set up MFA methods.

#### Risk-Based Sign-In

Microsoft Entra ID Protection evaluates sign-in risk in real-time:

**Risk levels**: Low, Medium, High

**Risk signals**:
- Anonymous IP address (Tor, VPN)
- Atypical travel (impossible travel between locations)
- Malware-linked IP
- Unfamiliar sign-in properties (new device, new location, new OS)
- Password spray detected
- Leaked credentials

**When risk is detected**:
```
Low risk:          May require MFA
Medium risk:       Require MFA + may require password change
High risk:         Block access entirely
```

### 4. Microsoft Cookies (Auth Success Indicators)

Cookies set on `login.microsoftonline.com`:

| Cookie | Purpose | Reliability |
|--------|---------|-------------|
| `ESTSAUTH` | Primary auth session token | **High** - confirms completed auth |
| `ESTSAUTHPERSISTENT` | Persistent auth token (KMSI=yes) | **High** - long-lived version |
| `ESTSAUTHLIGHT` | Lightweight auth indicator | Medium |
| `ESTSSC` | Session cookie | Medium |
| `buid` | Browser unique ID | Low - set before auth |
| `x-ms-gateway-slice` | Routing | Low - infrastructure |
| `stsservicecookie` | Service routing | Low |
| `CCState` | Cookie consent state | Low |

For Microsoft 365/Office post-login:
```
Domain: .office.com
Cookie: OIDCAuth (after redirect to Office)

Domain: .sharepoint.com
Cookie: FedAuth, rtFa (SharePoint-specific)

Domain: .teams.microsoft.com
Cookie: authtoken
```

**Recommended detection** for general Microsoft auth:

Option A - Cookie-based:
```json
{ "domain": "login.microsoftonline.com", "name": "ESTSAUTH" }
```

Option B - URL-based (current web-ctl approach):
```json
{ "successUrl": "https://www.office.com" }
```

**Problem with URL-based for enterprise**: Enterprise accounts may redirect to:
- `https://portal.office.com` (classic)
- `https://www.microsoft365.com` (newer)
- `https://myapps.microsoft.com` (app launcher)
- A tenant-specific SharePoint site
- A custom application the OAuth flow was for

Cookie-based detection on `login.microsoftonline.com` is more reliable across enterprise configurations.

### 5. Microsoft Personal vs Work/School Accounts

**Personal accounts** (`@outlook.com`, `@hotmail.com`, `@live.com`):
- Login at `login.live.com` (may redirect to `login.microsoftonline.com`)
- No tenant-specific policies
- No conditional access (unless using family safety)
- MFA optional, user-configured
- After auth, redirect to `outlook.live.com`, `onedrive.live.com`, etc.

**Work/school accounts** (`@company.com`, Entra ID):
- Login at `login.microsoftonline.com/{tenant-id}`
- Subject to tenant conditional access policies
- MFA often mandatory
- May use federated identity (ADFS, Okta, Ping)
- The email domain determines the tenant: Microsoft does home realm discovery (HRD)

**Home Realm Discovery (HRD)**:
After entering email, Microsoft queries:
```
GET https://login.microsoftonline.com/common/userrealm/<email>?api-version=2.1
```

Response indicates:
- `NameSpaceType: "Managed"` - password handled by Microsoft
- `NameSpaceType: "Federated"` - redirects to external IdP
- `AuthURL` - the federated IdP URL (ADFS, Okta, etc.)

**Detection of federated redirect**: After entering email and clicking next, if the URL changes to a non-Microsoft domain, the account uses federated auth. Common patterns:
```
ADFS:              https://sts.company.com/adfs/ls/...
Okta:              https://company.okta.com/app/microsoft_office365/...
Ping:              https://sso.company.com/idp/SSO.saml2/...
```

### 6. Azure AD B2C Custom Flows

Azure AD B2C allows completely custom authentication UIs:

**URL patterns**:
```
Standard:          https://{tenant}.b2clogin.com/{tenant}.onmicrosoft.com/{policy}/...
Custom domain:     https://login.company.com/{tenant}.onmicrosoft.com/{policy}/...
```

Policy names typically: `B2C_1_signupsignin`, `B2C_1A_signup_signin`

**Key difference from standard Entra ID**:
- The login page HTML/CSS/JS is fully customizable
- Standard selectors do NOT apply
- Each B2C tenant has its own page layouts
- May include social login buttons (Google, Facebook, Apple, etc.)
- Cannot predict DOM selectors without inspecting the specific tenant

**Note**: As of May 2025, Azure AD B2C is no longer available for new customers. Existing tenants continue to work. Microsoft recommends migrating to Microsoft Entra External ID.

### 7. Microsoft's Passwordless Default Flow

Microsoft has been rolling out passwordless-first sign-in since 2024. The flow:

1. User enters email
2. Instead of password field, user sees: "Approve sign in request"
3. A 2-digit number appears on screen
4. User opens Microsoft Authenticator app, matches the number, and approves
5. Auth completes

**DOM indicators for passwordless flow**:
```
Number display:    div.displaySign (contains the 2-digit match number)
Description:       "Open your Authenticator app and enter the number shown to sign in"
Alternative:       a containing "Use your password instead"
                   a containing "I can't use my Microsoft Authenticator app right now"
```

**For automation**: If the passwordless prompt appears, the agent cannot proceed automatically. It must either:
- Detect the passwordless prompt and instruct the user to approve on their phone
- Click "Use your password instead" to fall back to password entry (if allowed by tenant policy)

---

## Multi-Account / Account Picker Pages

### Google Account Chooser

**URL**: Contains `accounts.google.com` with account list visible.

```
Account tiles:     div[data-identifier]
                   Each tile shows email and profile picture
"Use another":     Content containing "Use another account"
Remove account:    Button within tile (usually hidden, appears on hover)
```

### Microsoft Account Picker

Triggered by `prompt=select_account` in OAuth URL or when multiple cached accounts exist.

```
Account tiles:     div.table-row (each account)
                   div[data-test-id="<email>"]
Username display:  small.table-cell.text-left (shows email)
Name display:      div.table-cell.text-left (shows display name)
"Use another":     div#otherTile, a#otherTileText
                   Content: "Use another account"
Forget account:    Not shown by default on picker
```

**Detection**: URL still on `login.microsoftonline.com` but page contains multiple account tiles or `div#otherTile`.

---

## Practical Recommendations for web-ctl

### Improving providers.json for Google

The current Google provider config only checks SAPISID cookie. Consider adding:

```json
{
  "slug": "google",
  "name": "Google",
  "loginUrl": "https://accounts.google.com",
  "successCookie": { "domain": ".google.com", "name": "SAPISID" },
  "captchaSelectors": [
    "iframe[src*=\"recaptcha\"]",
    "iframe[src*=\"recaptcha/api2\"]"
  ],
  "captchaTextPatterns": [
    "verify it's you",
    "this browser or app may not be secure",
    "unusual activity",
    "couldn't verify it's you",
    "suspicious activity"
  ],
  "twoFactorHint": "Google may prompt for phone verification, authenticator code, or security key. Check your phone for a Google prompt, or enter a code from your authenticator app.",
  "twoFactorSelectors": {
    "prompt": "div[data-challengetype='6']",
    "totp": "input#totpPin, input[name='totpPin']",
    "sms": "input#smsCode, input[name='smsPin']",
    "backup": "input#backupCodePin"
  }
}
```

### Improving providers.json for Microsoft

The current Microsoft provider only checks successUrl for office.com. Consider:

```json
{
  "slug": "microsoft",
  "name": "Microsoft",
  "loginUrl": "https://login.microsoftonline.com",
  "successCookie": { "domain": "login.microsoftonline.com", "name": "ESTSAUTH" },
  "successUrl": "https://www.office.com",
  "captchaSelectors": [
    "iframe[src*=\"arkoselabs\"]",
    "iframe[src*=\"funcaptcha\"]"
  ],
  "captchaTextPatterns": [
    "security check",
    "verify your identity"
  ],
  "twoFactorHint": "Microsoft may prompt for Authenticator app approval (match the number shown), SMS code, or phone call. If you see a number on screen, open your Authenticator app and enter that number.",
  "kmsiSelector": "input#idSIButton9[value='Yes'], input#idBtn_Back[value='No']",
  "conditionalAccessPatterns": [
    "AADSTS53003",
    "AADSTS50076",
    "does not meet the criteria",
    "Your admin has configured"
  ]
}
```

### Handling the "Stay Signed In?" (KMSI) Page

The auth-flow polling loop should detect and auto-dismiss the KMSI page:

```javascript
// In the polling loop, before checking auth success:
const kmsiYes = await page.$('input#idSIButton9[value="Yes"]');
const kmsiHeader = await page.$('div#KmsiTitle, div#kmsiTitle');
if (kmsiYes && kmsiHeader) {
  await kmsiYes.click();
  await page.waitForNavigation({ timeout: 5000 }).catch(() => {});
}
```

### Detecting Federated Auth Redirects

When a Microsoft work account uses ADFS/Okta/Ping:

```javascript
function isFederatedRedirect(currentUrl, originalUrl) {
  const current = new URL(currentUrl);
  const original = new URL(originalUrl);
  // If we left Microsoft's domain, it is a federated redirect
  return current.hostname !== original.hostname &&
    !current.hostname.endsWith('.microsoftonline.com') &&
    !current.hostname.endsWith('.microsoft.com');
}
```

The agent should inform the user that their organization uses a third-party login provider and the auth flow will appear different from standard Microsoft login.

---

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| "This browser or app may not be secure" on Google | Headless browser fingerprint detected | Always use headed mode for auth; use `--disable-blink-features=AutomationControlled` is no longer sufficient |
| KMSI page blocks Microsoft redirect | "Stay signed in?" not handled | Auto-detect and click through KMSI page in polling loop |
| Enterprise Microsoft redirects to unknown URL | Tenant-specific app landing page | Use cookie-based (ESTSAUTH) instead of URL-based success detection |
| Google Workspace SSO goes to Okta/ADFS | Federated identity | Detect domain change and inform user; cannot predict third-party IdP selectors |
| Microsoft conditional access blocks entirely | Tenant policy requires managed device | Inform user; no workaround from automation side |
| 2FA prompt times out | User did not respond in time | Increase timeout; provide clear instructions about which 2FA method to expect |
| Account chooser appears unexpectedly | Multiple cached accounts in browser profile | Detect account picker page; use fresh profile or specific account URL parameters |
| Google reCAPTCHA Enterprise invisible challenge | Risk score too low from automation signals | Cannot solve programmatically; human-in-the-loop is the only path |
| Azure AD B2C custom pages | Completely custom DOM | Cannot provide generic selectors; require per-tenant configuration |

---

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [Microsoft OAuth2 Auth Code Flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow) | Docs | Definitive reference for Microsoft auth endpoints and parameters |
| [Microsoft Authentication Methods](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-methods) | Docs | All MFA/passwordless methods supported by Entra ID |
| [Microsoft Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/overview) | Docs | Understanding what blocks automated access |
| [Microsoft Passwordless](https://learn.microsoft.com/en-us/entra/identity/authentication/concept-authentication-passwordless) | Docs | Passwordless flow details including passkeys/FIDO2 |
| [Google 2-Step Verification](https://support.google.com/accounts/answer/185839) | Docs | Google's official 2FA documentation |
| [reCAPTCHA v2/v3 docs](https://developers.google.com/recaptcha/docs/) | Docs | How reCAPTCHA works technically |
| web-ctl `providers.json` | Code | Current provider configurations in this codebase |
| web-ctl `auth-flow.js` | Code | Current auth flow implementation |
| `web-session-persistence-cli-agents.md` | Guide | Cookie formats and session management patterns |
| `cli-browser-automation-agents.md` | Guide | Browser automation approaches for agents |

---

*This guide was synthesized from 28 sources including Microsoft Learn documentation, Google developer documentation, community automation knowledge, and the web-ctl codebase. See `resources/google-microsoft-auth-flows-sources.json` for the full source list.*
