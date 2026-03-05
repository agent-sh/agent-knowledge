# Learning Guide: Authentication Flows for Browser Agents - Slack, Notion, AWS Console

**Generated**: 2026-02-22
**Sources**: Training knowledge (through May 2025) - WebFetch unavailable, selectors may have changed
**Depth**: deep

**IMPORTANT**: DOM selectors and URL patterns are based on observation through mid-2025. These services frequently update their UIs. Always verify selectors with a `snapshot` command before relying on them.

---

## Prerequisites

- Familiarity with the web-browse skill (`web-ctl.js run <session> <action>`)
- Understanding of OAuth 2.0, SAML, and cookie-based auth
- Knowledge of Playwright storageState format (see `cli-browser-automation-agents.md`)

---

## TL;DR

- **Slack**: Magic link (email code) is default. No password field unless workspace has password auth enabled. Workspace URL (`<slug>.slack.com`) is required before any auth. Enterprise Grid uses SAML/SSO redirect.
- **Notion**: Magic link (email code) is default. Password login available if user set one. Google/Apple OAuth buttons present. Single login page at `notion.so/login`.
- **AWS Console**: Multi-step flow. Account ID/alias first, then IAM username+password OR root email+password. MFA is nearly universal. SSO via IAM Identity Center has its own portal URL.
- All three services use bot detection. The `checkpoint` command (headed browser) is essential for initial auth. Post-auth, headless with cookies works.

---

## Slack Authentication

### 1. Login Page Structure

#### Pre-login: Workspace Selection

Slack requires knowing the workspace before login. There are two entry points:

**Path A: Direct workspace URL**
```
https://<workspace>.slack.com/signin
```

**Path B: Generic signin (workspace finder)**
```
https://slack.com/signin
```
This page asks for a workspace URL or email. The agent should use Path A when the workspace is known.

#### Workspace URL page selectors (slack.com/signin)

```
Form: Find your workspace
- Email input: css=input[data-qa="signin_domain_input"] or css=input#domain
  (In older versions: css=input[name="domain"])
- Continue button: role=button[name='Continue'] or css=button[data-qa="submit_team_domain_url"]
```

#### Magic Link Flow (Default - since ~2023)

After entering workspace URL, Slack defaults to "magic link" (email-based code):

1. **Email entry page** (`<workspace>.slack.com/signin`):
   ```
   - Email input: css=input[data-qa="login_email"] or css=input#email
   - "Sign In With Email" / "Continue": role=button[name='Sign In with Email']
     or css=button[data-qa="signin_with_email_btn"]
   ```

2. **Confirmation code sent page**:
   ```
   - Page shows: "Check your email" / "We've sent a code to..."
   - Detection text: text=Check your email
   - Code input: css=input[data-qa="confirmation_code_input"]
     or css=input.c-input_text (6-digit code field)
   - Confirm button: role=button[name='Confirm'] or css=button[data-qa="confirm_code_btn"]
   ```

   **Agent behavior**: When this page is detected, the agent MUST tell the user:
   ```
   "Slack sent a verification code to <email>. Please check your email and provide the code."
   ```

#### Password Login Flow (if enabled for workspace)

Some workspaces still allow password login. The page may show a "Sign in with a password instead" link:

```
- Password toggle: text=sign in with a password
  or css=a[data-qa="signin_with_password_link"]
- Password input: css=input[data-qa="login_password"] or css=input#password
  or css=input[type="password"]
- Sign In button: role=button[name='Sign In']
  or css=button[data-qa="signin_btn"]
```

#### Google/Apple OAuth Buttons

```
- Google: css=a[data-qa="sign_in_with_google"] or text=Sign in with Google
  or css=.c-google-signin-btn
- Apple: css=a[data-qa="sign_in_with_apple"] or text=Sign in with Apple
  or css=.c-apple-signin-btn
```

These redirect to Google/Apple OAuth pages. The agent should NOT attempt to automate Google/Apple OAuth - use `checkpoint` for user interaction.

### 2. CAPTCHA Systems

Slack uses **reCAPTCHA v2** (checkbox) and sometimes **reCAPTCHA v3** (invisible/score-based):

```
Detection selectors:
- css=iframe[src*="recaptcha"]
- css=div.g-recaptcha
- css=#captcha-container
```

**When detected**: Use `checkpoint` to let the user solve it in a headed browser.

reCAPTCHA typically appears after:
- Multiple failed login attempts
- Suspicious IP/user-agent patterns
- New device/location

### 3. 2FA/MFA Flows

#### Email Code (default "magic link")
Already covered above - this IS the default auth method.

#### TOTP (Authenticator App)
After password login (when workspace uses TOTP):

```
- Detection: URL contains "/signin/two-factor" or page contains text "Two-factor authentication"
- Code input: css=input[data-qa="two_factor_code"] or css=input#2fa_code
  or css=input[name="2fa_code"]
- Verify button: role=button[name='Confirm Code'] or css=button[data-qa="2fa_confirm_btn"]
```

#### SMS Code
Rare but possible:
```
- Detection: text=text message or text=SMS
- Same input pattern as TOTP
```

### 4. Success Detection

**URL patterns after successful auth**:
```
https://app.slack.com/client/<TEAM_ID>/<CHANNEL_ID>
https://app.slack.com/client/T[A-Z0-9]+/C[A-Z0-9]+
https://<workspace>.slack.com/messages/
```

**Key cookies** (domain: `.slack.com`):
```
d          - Main session cookie (long, encoded string). HttpOnly, Secure.
             This is the primary authentication cookie.
b          - Browser identifier cookie
lc         - Last channel
```

**Detection strategy**:
```bash
# After login attempt, check URL pattern
node web-ctl.js run slack evaluate "window.location.href"
# Success if matches: /app\.slack\.com\/client\/T[A-Z0-9]+/

# Or check for workspace-specific element
node web-ctl.js run slack snapshot
# Success if snapshot contains channel list, messaging UI
```

### 5. SSO/SAML (Enterprise Grid)

Enterprise workspaces redirect to their IdP:

1. User enters email at `<workspace>.slack.com/signin`
2. Slack detects SSO-configured domain
3. Redirect to: `<workspace>.enterprise.slack.com/sso/saml/start` or directly to IdP
4. After IdP auth, redirect back to: `<workspace>.slack.com/sso/callback`

**URL patterns indicating SSO**:
```
/sso/saml/start
/sso/saml/login
/sso/callback
enterprise.slack.com
Any redirect to Okta, Azure AD, OneLogin, Ping URLs
```

**Agent behavior**: When SSO redirect is detected, use `checkpoint` for the full IdP flow.

### 6. Free vs Pro vs Enterprise Differences

| Feature | Free/Pro | Enterprise Grid |
|---------|----------|-----------------|
| Magic link | Yes | Yes |
| Password auth | Optional per workspace | Usually disabled |
| Google/Apple OAuth | Yes | Often disabled |
| SSO/SAML | No (Pro has Google SSO only) | Yes, mandatory |
| 2FA | Optional per user | Can be org-enforced |

### 7. Anti-Bot Detection

- Rate limiting: ~5 login attempts per minute per IP
- After 5+ failures: account temporarily locked (15-30 min)
- Suspicious IP detection triggers additional verification
- Headless browser detection: Slack uses standard fingerprinting (navigator.webdriver, etc.)

---

## Notion Authentication

### 1. Login Page Structure

Single entry point: `https://www.notion.so/login`

#### Initial Email Entry

```
- Email input: css=input[type="email"] or css=input#notion-email-input-1
  or css=input[placeholder*="email"]
- Continue button: role=button[name='Continue with email']
  or css=div[role="button"] containing text "Continue with email"
```

Notion's UI is heavily React-based with dynamic class names. Prefer `role=` and `text=` selectors.

#### Magic Link Flow (Default)

After entering email, Notion sends a "login code":

1. **Code sent page**:
   ```
   Detection: text=paste login code or text=Enter login code
   - Code input: css=input[type="text"][placeholder*="Paste login code"]
     or css=input[aria-label*="login code"]
   - Continue button: role=button[name='Continue']
   ```

   **Agent behavior**: Tell user:
   ```
   "Notion sent a login code to <email>. Check your email for a message from Notion and provide the code."
   ```

2. The email contains a code like `horse-table-ocean-piano` (word-based) or a numeric code depending on the flow variant.

#### Password Login

If user has set a password:

```
- Password toggle: text=Continue with password
  or css=div[role="button"] containing text "password"
- Password input: css=input[type="password"]
- Continue button: role=button[name='Continue']
```

#### Google/Apple OAuth

```
- Google: role=button[name*='Google'] or text=Continue with Google
  or css=div[role="button"] with Google icon
- Apple: role=button[name*='Apple'] or text=Continue with Apple
  or css=div[role="button"] with Apple icon
```

These redirect to Google/Apple OAuth. Same advice: use `checkpoint`.

### 2. CAPTCHA Systems

Notion uses **Cloudflare Turnstile** (not traditional CAPTCHA):

```
Detection:
- css=iframe[src*="challenges.cloudflare.com"]
- css=div.cf-turnstile
- css=#cf-turnstile
```

Turnstile is usually invisible/automatic but can present a challenge on suspicious traffic. When detected and blocking, use `checkpoint`.

### 3. 2FA/MFA Flows

Notion does NOT have traditional TOTP/2FA as of mid-2025. Authentication relies on:
- Email verification (magic link/code) as the primary factor
- Google/Apple OAuth (delegated MFA to those providers)
- SAML SSO (delegated MFA to IdP)

If Notion adds TOTP in the future, expect:
```
Detection: text=two-factor or text=authenticator
```

### 4. Workspace/Space Selection

After login, if user belongs to multiple workspaces:

```
URL: https://www.notion.so/login/select-workspace
     or https://www.notion.so/context

Detection: text=Choose a workspace or text=Select a workspace
Workspace items: css=div[role="button"] containing workspace name
```

**Agent behavior**: If workspace selection appears, either:
- Click the known workspace name: `click "text=<workspace-name>"`
- Or ask the user which workspace to use

### 5. Success Detection

**URL patterns after successful auth**:
```
https://www.notion.so/<workspace-name>/<page-id>
https://www.notion.so/<workspace-slug>/...
https://notion.so/<anything-other-than-login>
```

**Key cookies** (domain: `.notion.so`):
```
token_v2    - Primary session token. HttpOnly, Secure. Long JWT-like string.
              This is THE authentication cookie.
notion_browser_id - Browser fingerprint UUID
notion_user_id    - User identifier
```

**Detection strategy**:
```bash
node web-ctl.js run notion evaluate "document.cookie.includes('token_v2')"
# Or check URL
node web-ctl.js run notion evaluate "!window.location.pathname.startsWith('/login')"
```

### 6. SSO/SAML

Notion SAML SSO (Business/Enterprise plans):

1. User enters email at `notion.so/login`
2. Notion detects SSO-configured email domain
3. Redirect to IdP (Okta, Azure AD, Google Workspace, OneLogin, etc.)
4. After IdP auth, redirect back to `notion.so/sso/callback`

**URL patterns**:
```
/sso/saml
/sso/callback
/login?sso=true
```

**Agent behavior**: When SSO redirect detected, use `checkpoint`.

### 7. Anti-Bot Detection

- Cloudflare protection (Turnstile + bot scoring)
- Rate limiting on login attempts (~10 per minute)
- Email code rate limiting (~3 codes per 5 minutes)
- Headless browser fingerprinting via Cloudflare

---

## AWS Console Authentication

### 1. Login Page Structure

Entry point: `https://signin.aws.amazon.com/signin` or `https://console.aws.amazon.com/`

AWS has the most complex login flow of the three, with multiple account types.

#### Step 1: Account Type Selection

The sign-in page first asks for account type:

```
URL: https://signin.aws.amazon.com/signin

Radio buttons or tabs:
- Root user: css=input#root_user_radio or role=radio[name='Root user']
  or text=Root user
- IAM user: css=input#iam_user_radio or role=radio[name='IAM user']
  or text=IAM user

Account ID / Alias field (for IAM):
- css=input#account or css=input[name="account"]
  or css=input[placeholder*="Account ID"]

Next button: role=button[name='Next'] or css=button#next_button
```

#### Step 2a: Root User Login

After selecting "Root user" and entering email:

```
Email input: css=input#resolving_input or css=input[name="email"]
  or css=input[type="email"]
Next: role=button[name='Next']

Password page:
- Password input: css=input#password or css=input[name="password"]
  or css=input[type="password"]
- Sign in: role=button[name='Sign in']
```

#### Step 2b: IAM User Login

After entering Account ID and clicking Next:

```
Username: css=input#username or css=input[name="username"]
Password: css=input#password or css=input[name="password"]
Sign in: role=button[name='Sign in'] or css=button#signin_button
```

#### AWS SSO / IAM Identity Center

Completely separate URL:

```
https://<directory-id>.awsapps.com/start
or
https://<custom-subdomain>.awsapps.com/start
```

This presents:
```
Username: css=input#username or css=input[name="username"]
Password: css=input#password or css=input[name="password"]
Sign in: role=button[name='Sign in']
```

After SSO login, an account/role selector appears:
```
Account list: css=div.instance-section or css=portal-application
Each account: clickable card with account name and ID
Role dropdown within each account: css=a containing role name (e.g., "AdministratorAccess")
```

### 2. CAPTCHA Systems

AWS uses its own CAPTCHA system (not reCAPTCHA):

```
Detection:
- css=img#captcha-image or css=img[alt*="captcha"]
- css=input#captchaGuess or css=input[name="captchaGuess"]
- text=Type the characters or text=solve the puzzle
```

AWS CAPTCHA appears after:
- 3+ failed login attempts
- Suspicious IP
- Some root account logins always show it

**Agent behavior**: Use `checkpoint` for CAPTCHA solving.

### 3. 2FA/MFA Flows

#### Virtual MFA (TOTP - most common)

After username+password:
```
URL may change to: /mfa or contain "mfa" parameter
Detection: text=MFA code or text=Authentication code or text=multi-factor

Code input: css=input#mfacode or css=input[name="mfacode"]
  or css=input[placeholder*="MFA"]
Submit: role=button[name='Submit'] or css=button#submitMfa
```

#### Hardware MFA (YubiKey)

```
Detection: text=hardware MFA or text=security key
- Triggers browser WebAuthn prompt
- Agent cannot handle this - use checkpoint
```

#### U2F/FIDO2 (Security Key)

```
Detection: text=security key or text=touch your security key
- Browser-level WebAuthn dialog
- Agent CANNOT automate this - must use checkpoint
```

#### MFA Device Selection (if multiple)

```
Detection: text=Select MFA device or text=Choose MFA
Device list: css=select#mfa-device-select or radio buttons
```

### 4. Success Detection

**URL patterns after successful auth**:
```
https://console.aws.amazon.com/console/home
https://<region>.console.aws.amazon.com/...
https://us-east-1.console.aws.amazon.com/console/home?region=us-east-1
```

Region-specific patterns:
```
https://us-east-1.console.aws.amazon.com/...
https://eu-west-1.console.aws.amazon.com/...
https://ap-southeast-1.console.aws.amazon.com/...
```

**Key cookies** (domain: `.aws.amazon.com` / `.console.aws.amazon.com`):
```
aws-creds             - Encrypted session credentials
aws-userInfo          - Base64-encoded user info JSON
aws-userInfo-signed   - Signed version
noflush_Region        - Selected region
```

**Session token in page** (for API calls):
```javascript
// AWS injects credentials into the page for console API calls
// These are NOT in cookies but in JavaScript variables/localStorage
```

**Detection strategy**:
```bash
# Check URL pattern
node web-ctl.js run aws evaluate "window.location.hostname.includes('console.aws.amazon.com')"

# Check for console UI elements
node web-ctl.js run aws snapshot
# Success if snapshot contains: services search bar, account dropdown, region selector
```

### 5. SSO/SAML

#### IAM Identity Center (AWS SSO)

Completely separate flow from console signin:

1. Navigate to: `https://<subdomain>.awsapps.com/start`
2. Login with directory credentials (or federated IdP)
3. Account/role selection portal appears
4. Click account -> click role -> redirected to console

**URL patterns**:
```
<subdomain>.awsapps.com/start
<subdomain>.awsapps.com/start/#/
<subdomain>.awsapps.com/start/#/saml/...
```

#### External IdP SAML

Some orgs configure AWS with Okta/Azure AD directly:
1. User goes to IdP portal
2. Clicks "AWS Console" tile
3. SAML assertion posted to `https://signin.aws.amazon.com/saml`
4. Role selection if multiple roles
5. Redirect to console

**Detection**: URL contains `/saml` or `SAMLResponse` in POST body

### 6. AWS Organizations Cross-Account Access

After initial login, switching roles:

```
URL: https://signin.aws.amazon.com/switchrole
     or console menu -> Switch Role

Account field: css=input#account or css=input[name="account"]
Role field: css=input#roleName or css=input[name="roleName"]
Display Name: css=input#displayName
Switch Role button: role=button[name='Switch Role']
```

After switch:
```
URL: https://console.aws.amazon.com/... (same, but role indicator changes)
Detection: Account/role indicator in top-right shows different account
```

### 7. AWS GovCloud Differences

GovCloud has separate endpoints:
```
Sign-in: https://signin.amazonaws-us-gov.com/
Console: https://console.amazonaws-us-gov.com/
Regions: us-gov-west-1, us-gov-east-1
```

Same login flow structure but completely isolated IAM - GovCloud accounts are separate from commercial AWS.

### 8. Anti-Bot Detection

- AWS CAPTCHA (proprietary) on failed attempts
- Account lockout after ~5 failed attempts (lockout duration escalates)
- IP-based rate limiting
- CloudFront-based bot detection on console pages
- Root account has stricter protections than IAM users

---

## Cross-Cutting Concerns

### Magic Link Detection Pattern

For both Slack and Notion, detect that a magic link/code was sent:

```bash
# After submitting email, check page content
node web-ctl.js run <session> snapshot

# Look for these text patterns in the snapshot:
# Slack: "Check your email", "confirmation code", "sent a code"
# Notion: "login code", "paste login code", "check your email"
```

**Agent state machine for magic link**:
```
EMAIL_ENTERED -> snapshot -> contains "check your email" or "code"?
  YES -> tell user to check email, wait for code input
  NO  -> check for password field, error message, or CAPTCHA
```

### SSO Redirect Detection (All Three)

Generic SSO detection across all providers:

```bash
# After submitting credentials, monitor URL changes
node web-ctl.js run <session> evaluate "window.location.href"

# SSO indicators in URL:
# - /saml/ or /sso/
# - login.microsoftonline.com (Azure AD)
# - *.okta.com
# - accounts.google.com/o/saml
# - *.onelogin.com
# - *.pingidentity.com
# - *.duosecurity.com
```

**Agent behavior on SSO redirect**: Always use `checkpoint` - SSO IdPs have their own MFA, CAPTCHA, and anti-bot systems that vary by organization.

### CAPTCHA Detection (Universal)

```bash
node web-ctl.js run <session> snapshot
# Check for:
# - iframe with "recaptcha" or "captcha" in src
# - iframe with "challenges.cloudflare.com" in src
# - img elements with captcha-like attributes
# - text containing "not a robot", "verify", "human"

# Alternative: check DOM directly
node web-ctl.js run <session> evaluate "
  !!document.querySelector('iframe[src*=\"recaptcha\"], iframe[src*=\"captcha\"], iframe[src*=\"challenges.cloudflare\"], .g-recaptcha, .cf-turnstile, #captcha-image')
"
```

**When CAPTCHA detected**:
```bash
# Switch to headed browser for user intervention
node web-ctl.js run <session> checkpoint --timeout 120
# Tell user: "A CAPTCHA challenge appeared. Please solve it in the browser window."
```

### Session Validity Checking

Before starting any authenticated workflow, verify the session is still valid:

```bash
# Slack
node web-ctl.js run slack goto "https://app.slack.com/client"
node web-ctl.js run slack evaluate "window.location.href"
# If redirected to signin -> session expired

# Notion
node web-ctl.js run notion goto "https://www.notion.so"
node web-ctl.js run notion evaluate "!window.location.pathname.startsWith('/login')"
# false -> session expired

# AWS
node web-ctl.js run aws goto "https://console.aws.amazon.com/console/home"
node web-ctl.js run aws evaluate "!window.location.hostname.includes('signin.aws')"
# false -> session expired
```

### Cookie Lifetimes

| Provider | Primary Cookie | Typical Lifetime | Notes |
|----------|---------------|-----------------|-------|
| Slack | `d` | 30 days | Refreshed on activity |
| Notion | `token_v2` | ~90 days | May rotate |
| AWS | `aws-creds` | 1-12 hours | Depends on IAM session duration setting |

AWS has the shortest sessions. For long-running AWS automation, the agent needs to handle re-auth or use CLI credentials (`aws sts assume-role`) instead of console cookies.

---

## Complete Auth Flow Diagrams

### Slack Auth Decision Tree

```
START
  |
  v
Navigate to <workspace>.slack.com/signin
  |
  v
Enter email -> Submit
  |
  +---> [CAPTCHA detected?] --YES--> checkpoint -> resume
  |
  v
Check page state:
  |
  +---> "Check your email" (magic link) --> tell user, wait for code --> enter code --> submit
  |
  +---> Password field visible --> enter password --> submit
  |         |
  |         +---> [2FA page?] --YES--> tell user, wait for TOTP code --> enter --> submit
  |
  +---> SSO redirect --> checkpoint (user completes IdP flow)
  |
  +---> Google/Apple button clicked --> checkpoint (user completes OAuth)
  |
  v
Check: URL matches app.slack.com/client/T*?
  YES --> [OK] Authenticated
  NO  --> Check error message, retry or escalate
```

### Notion Auth Decision Tree

```
START
  |
  v
Navigate to notion.so/login
  |
  v
Enter email -> "Continue with email"
  |
  +---> [Cloudflare challenge?] --YES--> checkpoint -> resume
  |
  v
Check page state:
  |
  +---> "Paste login code" --> tell user, wait for code --> enter code --> continue
  |
  +---> "Continue with password" link visible --> click it --> enter password --> continue
  |
  +---> SSO redirect --> checkpoint
  |
  v
Check: workspace selection page?
  YES --> select workspace
  |
  v
Check: URL is not /login?
  YES --> [OK] Authenticated
  NO  --> check error, retry
```

### AWS Console Auth Decision Tree

```
START
  |
  v
Determine auth type:
  IAM user --> need Account ID + username + password
  Root     --> need email + password
  SSO      --> navigate to awsapps.com/start portal
  |
  v
[IAM path]
Navigate to signin.aws.amazon.com
  -> Select "IAM user"
  -> Enter Account ID -> Next
  -> Enter username + password -> Sign in
  |
  +---> [CAPTCHA?] --YES--> checkpoint -> resume
  |
  v
[MFA page?]
  +---> TOTP --> tell user, wait for code --> enter --> submit
  +---> Hardware/FIDO2 --> checkpoint (must physically touch key)
  |
  v
Check: URL contains console.aws.amazon.com?
  YES --> [OK] Authenticated
  NO  --> check error, retry

[SSO path]
Navigate to <subdomain>.awsapps.com/start
  -> Enter credentials (may redirect to external IdP)
  -> checkpoint if external IdP
  -> Select account -> Select role
  -> Redirected to console
  --> [OK] Authenticated
```

---

## Recommended Agent Implementation Pattern

```bash
#!/bin/bash
# Generic auth flow for web-ctl

SESSION="$1"
PROVIDER="$2"  # slack, notion, aws

authenticate() {
  local session="$1"
  local provider="$2"

  # Step 1: Navigate to login
  case "$provider" in
    slack)  URL="https://${SLACK_WORKSPACE}.slack.com/signin" ;;
    notion) URL="https://www.notion.so/login" ;;
    aws)    URL="https://signin.aws.amazon.com/signin" ;;
  esac

  node web-ctl.js run "$session" goto "$URL"

  # Step 2: Take snapshot, analyze page
  SNAP=$(node web-ctl.js run "$session" snapshot)

  # Step 3: Check if already authenticated
  CURRENT_URL=$(node web-ctl.js run "$session" evaluate "window.location.href")
  if is_authenticated "$provider" "$CURRENT_URL"; then
    echo "[OK] Already authenticated to $provider"
    return 0
  fi

  # Step 4: CAPTCHA check
  HAS_CAPTCHA=$(node web-ctl.js run "$session" evaluate "
    !!document.querySelector('iframe[src*=\"recaptcha\"], iframe[src*=\"captcha\"], iframe[src*=\"challenges.cloudflare\"], .g-recaptcha, .cf-turnstile, #captcha-image')
  ")
  if [ "$HAS_CAPTCHA" = "true" ]; then
    echo "[WARN] CAPTCHA detected - opening headed browser"
    node web-ctl.js run "$session" checkpoint --timeout 120
  fi

  # Step 5: Provider-specific login flow
  # (Fill credentials, handle magic link, MFA, etc.)
  # ...

  # Step 6: Verify success
  CURRENT_URL=$(node web-ctl.js run "$session" evaluate "window.location.href")
  if is_authenticated "$provider" "$CURRENT_URL"; then
    echo "[OK] Authenticated to $provider"
    return 0
  else
    echo "[ERROR] Authentication failed for $provider"
    return 1
  fi
}

is_authenticated() {
  local provider="$1"
  local url="$2"
  case "$provider" in
    slack)  echo "$url" | grep -q "app.slack.com/client" ;;
    notion) echo "$url" | grep -qv "/login" ;;
    aws)    echo "$url" | grep -q "console.aws.amazon.com" ;;
  esac
}
```

---

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| Slack: trying to enter password when magic link is default | Workspace uses email code only | Check page state after email entry; don't assume password field exists |
| Notion: dynamic class names break CSS selectors | React-based UI with hashed classes | Use `role=` and `text=` selectors, not CSS classes |
| AWS: forgetting Account ID for IAM users | Agent assumes root login | Always ask user for Account ID/alias when doing IAM login |
| AWS: session expires in 1 hour | Default IAM session duration | Detect session expiry (redirect to signin) and re-auth |
| All: headless browser blocked by bot detection | Services fingerprint automation | Use `checkpoint` for initial auth, then use cookies headlessly |
| SSO: trying to automate IdP login | Each org's IdP config is different | Always use `checkpoint` for SSO/SAML flows |
| Magic link: agent waits forever for code | User didn't check email | Set timeout; remind user after 60s; offer to resend |
| AWS MFA: entering code too slowly | TOTP codes rotate every 30s | Prompt user for code and submit immediately |
| Notion: workspace selection not handled | User has multiple workspaces | Check for workspace selection page after login |
| Slack: workspace URL unknown | Agent doesn't know the slug | Ask user for workspace URL or use slack.com/signin finder |

---

## Self-Evaluation

| Metric | Score | Notes |
|--------|-------|-------|
| Coverage | 8/10 | All three providers covered with login, MFA, SSO, success detection |
| Accuracy | 7/10 | Based on training data through mid-2025; selectors may have changed |
| Examples | 8/10 | Full decision trees and implementation patterns |
| Actionability | 9/10 | Directly usable with web-ctl skill |
| Gaps | - | Exact selectors need verification via live snapshot; CAPTCHA solving strategies limited |

**Known limitations**:
- DOM selectors are from training data and WILL drift. Always `snapshot` first.
- Cookie names/formats may change without notice.
- AWS Console UI was redesigned multiple times; selectors are for the 2024-2025 version.

---

*Generated from training knowledge (through May 2025). Selectors and URL patterns should be verified against live pages before use in automation.*
