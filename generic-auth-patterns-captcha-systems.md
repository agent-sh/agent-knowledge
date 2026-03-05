# Learning Guide: Generic Authentication Patterns and CAPTCHA Systems for Browser Automation Agents

**Generated**: 2026-02-22
**Sources**: 40+ resources synthesized (training knowledge through May 2025, codebase context)
**Depth**: deep

---

## Prerequisites

- Familiarity with Playwright or Puppeteer browser automation
- Understanding of DOM selectors (CSS selectors, iframes)
- Basic knowledge of HTTP cookies and session management
- Understanding of OAuth flows (covered in `web-session-persistence-cli-agents.md`)

---

## TL;DR

- Six major CAPTCHA systems exist in the wild; each has unique DOM fingerprints (specific div IDs, iframe src patterns, and CSS classes) that allow reliable detection.
- Cookie consent banners from four major frameworks (OneTrust, CookieBot, TrustArc, Quantcast) frequently overlay login forms; dismissing them requires framework-specific selectors.
- Browser fingerprinting detection checks `navigator.webdriver`, plugin arrays, and TLS fingerprints; stealth launch args and init scripts mitigate most checks.
- Auth flows fall into five patterns: email-first multi-step, magic link, passwordless/passkey, social login redirect, and traditional username+password.
- Success detection should layer URL-based, cookie-based, and DOM-based signals rather than relying on any single method.

---

## 1. CAPTCHA Systems - Comprehensive Detection Guide

### 1.1 reCAPTCHA v2 (Checkbox - "I'm not a robot")

**DOM fingerprint:**
```
Container:  div.g-recaptcha
            div[data-sitekey]
Iframe:     iframe[src*="google.com/recaptcha/api2/anchor"]
            iframe[src*="google.com/recaptcha/api2/bframe"]  (challenge frame)
Badge:      div.grecaptcha-badge
Textarea:   textarea#g-recaptcha-response  (hidden, holds token)
Script:     script[src*="google.com/recaptcha/api.js"]
```

**Detection selectors (ordered by reliability):**
```javascript
const RECAPTCHA_V2_SELECTORS = [
  'iframe[src*="google.com/recaptcha/api2/anchor"]',
  'iframe[src*="google.com/recaptcha/api2/bframe"]',
  'div.g-recaptcha',
  'div[data-sitekey]',
  'textarea#g-recaptcha-response',
  '.grecaptcha-badge',
];
```

**Text patterns:** "I'm not a robot", "reCAPTCHA", "Privacy - Terms"

**Behavior:** Renders a checkbox inside an iframe. Clicking triggers risk analysis. If low risk, auto-solves. If high risk, opens image challenge in a second iframe (bframe).

### 1.2 reCAPTCHA v3 (Invisible / Score-based)

**DOM fingerprint:**
```
Badge:      div.grecaptcha-badge
Script:     script[src*="google.com/recaptcha/api.js?render="]
Hidden:     textarea.g-recaptcha-response  (one per action)
            input[name="g-recaptcha-response"]
```

**Detection selectors:**
```javascript
const RECAPTCHA_V3_SELECTORS = [
  '.grecaptcha-badge',
  'script[src*="google.com/recaptcha/api.js?render="]',
  'textarea.g-recaptcha-response',
];
```

**Key difference from v2:** No visible checkbox or challenge. Runs entirely in background. Returns a score (0.0 to 1.0). The site decides what to do with the score. The only visible element is the small badge in the bottom-right corner (unless hidden by CSS).

**Detection challenge:** v3 is invisible. You may not know it is running. The badge is the only DOM clue, and many sites hide it with CSS (`visibility: hidden` or `opacity: 0`). Check for the script tag as primary indicator.

### 1.3 reCAPTCHA Enterprise

**DOM fingerprint:** Same as v2 or v3 depending on integration mode, but the script source differs:
```
Script:     script[src*="google.com/recaptcha/enterprise.js"]
            script[src*="recaptcha.net/recaptcha/enterprise.js"]
```

**Detection selectors:**
```javascript
const RECAPTCHA_ENTERPRISE_SELECTORS = [
  'script[src*="recaptcha/enterprise.js"]',
  // Plus all v2/v3 selectors - Enterprise uses the same widget DOM
  ...RECAPTCHA_V2_SELECTORS,
  ...RECAPTCHA_V3_SELECTORS,
];
```

### 1.4 hCaptcha

**DOM fingerprint:**
```
Container:  div.h-captcha
            div[data-sitekey]  (shared with reCAPTCHA - check parent)
Iframe:     iframe[src*="hcaptcha.com/captcha/"]
            iframe[src*="newassets.hcaptcha.com"]
Challenge:  iframe[src*="hcaptcha.com/captcha/challenge"]
Script:     script[src*="hcaptcha.com/1/api.js"]
Textarea:   textarea[name="h-captcha-response"]
            input[name="h-captcha-response"]
```

**Detection selectors:**
```javascript
const HCAPTCHA_SELECTORS = [
  'div.h-captcha',
  'iframe[src*="hcaptcha.com"]',
  'iframe[src*="newassets.hcaptcha.com"]',
  'textarea[name="h-captcha-response"]',
  'script[src*="hcaptcha.com"]',
];
```

**Text patterns:** "Verify you are human", "hCaptcha", "Anti-Robot Verification"

**Used by:** Cloudflare (historically, before Turnstile), Discord, Epic Games, many mid-sized sites.

### 1.5 Cloudflare Turnstile

**DOM fingerprint:**
```
Container:  div.cf-turnstile
            div[data-sitekey]  (with Turnstile-specific key format)
Iframe:     iframe[src*="challenges.cloudflare.com/turnstile"]
            iframe[src*="challenges.cloudflare.com/cdn-cgi/challenge-platform"]
Script:     script[src*="challenges.cloudflare.com/turnstile"]
Hidden:     input[name="cf-turnstile-response"]
```

**Detection selectors:**
```javascript
const TURNSTILE_SELECTORS = [
  'div.cf-turnstile',
  'iframe[src*="challenges.cloudflare.com/turnstile"]',
  'iframe[src*="challenges.cloudflare.com/cdn-cgi/challenge-platform"]',
  'input[name="cf-turnstile-response"]',
  'script[src*="challenges.cloudflare.com/turnstile"]',
];
```

**Cloudflare challenge page (full-page, not widget):**
```javascript
const CLOUDFLARE_CHALLENGE_SELECTORS = [
  '#challenge-running',
  '#challenge-stage',
  '#challenge-form',
  'div.cf-browser-verification',
  'div#cf-wrapper',
  'div.cf-error-details',
  'meta[content*="cf-browser-verification"]',
];
const CLOUDFLARE_CHALLENGE_TEXT = [
  'Checking your browser',
  'Verifying you are human',
  'Just a moment',
  'Enable JavaScript and cookies to continue',
  'Attention Required',
  'Please Wait',
  'DDoS protection by Cloudflare',
];
```

**Behavior:** Turnstile is designed to be invisible/non-interactive for most users. It renders a small widget that auto-solves. The full-page challenge is a separate system (Cloudflare Bot Management / Under Attack Mode) that intercepts the entire page load.

### 1.6 Arkose Labs / FunCaptcha

**DOM fingerprint:**
```
Container:  div#fc-iframe-wrap
            div.fc-dialog
            div[id*="arkose"]
            div[data-callback*="arkose"]
Iframe:     iframe[src*="arkoselabs.com"]
            iframe[src*="funcaptcha.com"]
            iframe[data-e2e="enforcement-frame"]
            iframe[id*="fc-iframe"]
Script:     script[src*="arkoselabs.com/v2/"]
            script[src*="funcaptcha.com/fc/"]
Enforcement: div#arkose_enforcement
             div#fc-token
```

**Detection selectors:**
```javascript
const ARKOSE_SELECTORS = [
  'iframe[src*="arkoselabs.com"]',
  'iframe[src*="funcaptcha.com"]',
  'div#fc-iframe-wrap',
  'div[id*="arkose"]',
  'iframe[id*="fc-iframe"]',
  'script[src*="arkoselabs.com"]',
  'script[src*="funcaptcha.com"]',
];
```

**Used by:** X/Twitter (login), Microsoft (login), EA, Roblox, LinkedIn (some flows).

### 1.7 AWS WAF CAPTCHA

**DOM fingerprint:**
```
Container:  div#captcha-container
            div.awscp-captcha
Iframe:     iframe[src*="amazonaws.com/captcha"]
            iframe[src*=".awswaf.com/"]
Script:     script[src*="amazonaws.com/captcha"]
            script[src*="awswaf.com/captcha.js"]
Form:       form#captcha-form
Hidden:     input[name="captcha_token"]
```

**Detection selectors:**
```javascript
const AWS_WAF_SELECTORS = [
  'iframe[src*="awswaf.com"]',
  'iframe[src*="amazonaws.com/captcha"]',
  'div.awscp-captcha',
  'script[src*="awswaf.com/captcha"]',
  'form#captcha-form',
];
```

### 1.8 PerimeterX / HUMAN Security

**DOM fingerprint:**
```
Container:  div#px-captcha
            div[id*="px-captcha"]
Script:     script[src*="perimeterx.net"]
            script[src*="px-cdn.net"]
            script[src*="pxchk.net"]
Blocked:    div#px-block
            div.px-blocked
Cookie:     _px2, _px3, _pxhd, _pxde (check via JS)
```

**Detection selectors:**
```javascript
const PERIMETERX_SELECTORS = [
  'div#px-captcha',
  'div[id*="px-captcha"]',
  'script[src*="perimeterx.net"]',
  'script[src*="px-cdn.net"]',
  'div#px-block',
];
```

**Text patterns:** "Press & Hold", "Please verify you are a human" (PerimeterX uses a press-and-hold challenge, not image selection).

### 1.9 Unified CAPTCHA Detection Function

```javascript
const CAPTCHA_DETECTION = {
  selectors: {
    recaptchaV2: [
      'iframe[src*="google.com/recaptcha/api2/anchor"]',
      'div.g-recaptcha',
      'textarea#g-recaptcha-response',
    ],
    recaptchaV3: [
      '.grecaptcha-badge',
      'script[src*="recaptcha/api.js?render="]',
    ],
    recaptchaEnterprise: [
      'script[src*="recaptcha/enterprise.js"]',
    ],
    hcaptcha: [
      'div.h-captcha',
      'iframe[src*="hcaptcha.com"]',
      'textarea[name="h-captcha-response"]',
    ],
    turnstile: [
      'div.cf-turnstile',
      'iframe[src*="challenges.cloudflare.com/turnstile"]',
      'input[name="cf-turnstile-response"]',
    ],
    cloudflareChallenge: [
      '#challenge-running',
      '#challenge-stage',
      '#challenge-form',
    ],
    arkose: [
      'iframe[src*="arkoselabs.com"]',
      'iframe[src*="funcaptcha.com"]',
      'div#fc-iframe-wrap',
    ],
    awsWaf: [
      'iframe[src*="awswaf.com"]',
      'div.awscp-captcha',
    ],
    perimeterX: [
      'div#px-captcha',
      'script[src*="perimeterx.net"]',
      'script[src*="px-cdn.net"]',
    ],
  },

  textPatterns: [
    "i'm not a robot",
    "i am not a robot",
    "verify you are human",
    "verify you're human",
    "prove you're human",
    "are you a robot",
    "security check",
    "security verification",
    "complete the challenge",
    "human verification",
    "bot protection",
    "anti-robot verification",
    "checking your browser",
    "just a moment",
    "please wait",
    "ddos protection",
    "attention required",
    "access denied",
    "enable javascript and cookies",
    "press & hold",
    "press and hold",
    "slide to verify",
  ],
};

/**
 * Detect any CAPTCHA system on the current page.
 * Returns { detected: boolean, systems: string[], confidence: string }
 */
async function detectCaptcha(page) {
  const results = [];

  for (const [system, selectors] of Object.entries(CAPTCHA_DETECTION.selectors)) {
    for (const selector of selectors) {
      const found = await page.$(selector);
      if (found) {
        results.push(system);
        break; // One match per system is enough
      }
    }
  }

  // Text-based detection as fallback
  if (results.length === 0) {
    const bodyText = await page.textContent('body').catch(() => '');
    const lower = bodyText.toLowerCase();
    const textMatch = CAPTCHA_DETECTION.textPatterns.some(p => lower.includes(p));
    if (textMatch) {
      results.push('unknown_captcha_text_match');
    }
  }

  return {
    detected: results.length > 0,
    systems: [...new Set(results)],
    confidence: results.length > 0 ? 'high' : 'none',
  };
}
```

---

## 2. Cookie Consent / GDPR Banners

Cookie consent overlays frequently cover login forms, making them impossible to interact with. These must be dismissed before proceeding with auth.

### 2.1 Major Consent Frameworks and Selectors

#### OneTrust (most common enterprise CMP)

```javascript
const ONETRUST_SELECTORS = {
  banner: '#onetrust-banner-sdk',
  overlay: '.onetrust-pc-dark-filter',
  acceptButton: '#onetrust-accept-btn-handler',
  rejectButton: '#onetrust-reject-all-handler',
  closeButton: '.onetrust-close-btn-handler',
  // Alternative selectors
  altBanner: 'div[class*="otFlat"]',
  altAccept: 'button[class*="onetrust-close-btn"]',
};
```

#### CookieBot (Cybot)

```javascript
const COOKIEBOT_SELECTORS = {
  banner: '#CybotCookiebotDialog',
  overlay: '#CybotCookiebotDialogBodyUnderlay',
  acceptButton: '#CybotCookiebotDialogBodyLevelButtonLevelOptinAllowAll',
  declineButton: '#CybotCookiebotDialogBodyButtonDecline',
  necessaryOnly: '#CybotCookiebotDialogBodyLevelButtonLevelOptinDeclineAll',
  closeButton: '#CybotCookiebotDialogBodyButtonAccept',
};
```

#### TrustArc (TrustE)

```javascript
const TRUSTARC_SELECTORS = {
  banner: '#truste-consent-track',
  overlay: '.truste-overlay',
  acceptButton: '#truste-consent-button',
  closeButton: '.truste-close',
  iframe: 'iframe[src*="consent.trustarc.com"]',
  // The consent UI loads in an iframe
  iframeAccept: '.pdynamicbutton .call', // Inside iframe
};
```

#### Quantcast Choice

```javascript
const QUANTCAST_SELECTORS = {
  banner: '#qc-cmp2-container',
  overlay: '.qc-cmp2-overlay',
  acceptButton: 'button[class*="qc-cmp2-summary-buttons"] button:first-child',
  moreOptions: 'button[class*="qc-cmp2-summary-buttons"] button:last-child',
  // V1 selectors (legacy)
  v1Banner: '.qc-cmp-showing',
  v1Accept: '.qc-cmp-button',
};
```

#### Generic / Unknown CMPs

```javascript
const GENERIC_CONSENT_SELECTORS = {
  // Common container patterns
  containers: [
    '[class*="cookie-banner"]',
    '[class*="cookie-consent"]',
    '[class*="cookie-notice"]',
    '[class*="cookie-popup"]',
    '[class*="consent-banner"]',
    '[class*="consent-modal"]',
    '[class*="gdpr"]',
    '[id*="cookie-banner"]',
    '[id*="cookie-consent"]',
    '[id*="cookie-notice"]',
    '[id*="gdpr"]',
    '[id*="consent"]',
    '[aria-label*="cookie"]',
    '[aria-label*="consent"]',
    '[role="dialog"][class*="cookie"]',
    '[role="dialog"][class*="consent"]',
  ],
  // Common accept button patterns
  acceptButtons: [
    'button[class*="accept"]',
    'button[class*="agree"]',
    'button[class*="allow"]',
    'button[class*="consent"]',
    'button[class*="got-it"]',
    'button[id*="accept"]',
    'a[class*="accept"]',
    'a[id*="accept"]',
    'button:has-text("Accept")',
    'button:has-text("Accept All")',
    'button:has-text("Accept Cookies")',
    'button:has-text("Allow All")',
    'button:has-text("Allow Cookies")',
    'button:has-text("I Agree")',
    'button:has-text("Got It")',
    'button:has-text("OK")',
    'button:has-text("Continue")',
    'button:has-text("Agree")',
  ],
};
```

### 2.2 Unified Consent Dismissal Function

```javascript
async function dismissCookieConsent(page, { timeout = 5000 } = {}) {
  const frameworks = [
    { name: 'OneTrust', banner: '#onetrust-banner-sdk', accept: '#onetrust-accept-btn-handler' },
    { name: 'CookieBot', banner: '#CybotCookiebotDialog', accept: '#CybotCookiebotDialogBodyLevelButtonLevelOptinAllowAll' },
    { name: 'TrustArc', banner: '#truste-consent-track', accept: '#truste-consent-button' },
    { name: 'Quantcast', banner: '#qc-cmp2-container', accept: '.qc-cmp2-summary-buttons button:first-child' },
  ];

  // Try known frameworks first
  for (const fw of frameworks) {
    const banner = await page.$(fw.banner);
    if (banner && await banner.isVisible()) {
      const acceptBtn = await page.$(fw.accept);
      if (acceptBtn) {
        await acceptBtn.click();
        await page.waitForSelector(fw.banner, { state: 'hidden', timeout }).catch(() => {});
        return { dismissed: true, framework: fw.name };
      }
    }
  }

  // Fall back to generic detection
  for (const selector of GENERIC_CONSENT_SELECTORS.acceptButtons) {
    try {
      const btn = await page.$(selector);
      if (btn && await btn.isVisible()) {
        await btn.click();
        return { dismissed: true, framework: 'generic' };
      }
    } catch {
      continue;
    }
  }

  return { dismissed: false, framework: null };
}
```

### 2.3 Overlay Detection

Some consent banners create full-page overlays that intercept clicks. Detect these:

```javascript
async function isBlockedByOverlay(page) {
  return await page.evaluate(() => {
    const overlaySelectors = [
      '.onetrust-pc-dark-filter',
      '#CybotCookiebotDialogBodyUnderlay',
      '.truste-overlay',
      '.qc-cmp2-overlay',
      '[class*="cookie-overlay"]',
      '[class*="consent-overlay"]',
      '[class*="modal-backdrop"]',
    ];
    for (const sel of overlaySelectors) {
      const el = document.querySelector(sel);
      if (el) {
        const style = window.getComputedStyle(el);
        if (style.display !== 'none' && style.visibility !== 'hidden' && parseFloat(style.opacity) > 0) {
          return true;
        }
      }
    }
    // Check for any full-viewport fixed element with high z-index
    const candidates = document.querySelectorAll('[style*="position: fixed"], [style*="position:fixed"]');
    for (const el of candidates) {
      const rect = el.getBoundingClientRect();
      const style = window.getComputedStyle(el);
      if (rect.width > window.innerWidth * 0.8 && rect.height > window.innerHeight * 0.5
          && parseInt(style.zIndex) > 100) {
        return true;
      }
    }
    return false;
  });
}
```

---

## 3. Browser Fingerprinting Detection and Stealth

### 3.1 How Sites Detect Automation

| Signal | What They Check | Default in Playwright/Puppeteer |
|--------|----------------|-------------------------------|
| `navigator.webdriver` | `true` = automated | `true` (Playwright sets this) |
| `navigator.plugins` | Empty array = headless | `[]` in headless |
| `navigator.languages` | Missing or empty | Present but may differ |
| `window.chrome` | Undefined in headless | Missing in headless |
| `chrome.runtime` | Undefined in automation | Missing |
| `window.outerWidth/Height` | `0` in headless | `0` in older headless |
| Canvas fingerprint | Different rendering in headless | Slightly different |
| WebGL renderer | "SwiftShader" = headless | "Google SwiftShader" |
| `Permissions.query` | Different notification behavior | Detectable |
| User-Agent | Contains "HeadlessChrome" | Not by default in new headless, but older versions |
| CDP detection | `Runtime.evaluate` artifacts | Detectable via `Error.stack` |
| TLS fingerprint (JA3/JA4) | Headless has unique TLS handshake | Different from real Chrome |
| Mouse/input events | Missing `isTrusted`, unnatural patterns | `dispatchEvent` vs real input |

### 3.2 Stealth Configuration for Playwright

```javascript
// Recommended launch configuration
const browser = await chromium.launch({
  headless: false,  // "new" headless mode is better but headed is safest
  args: [
    '--disable-blink-features=AutomationControlled',
    '--disable-features=IsolateOrigins,site-per-process',
    '--disable-site-isolation-trials',
    '--disable-dev-shm-usage',
    '--no-first-run',
    '--no-default-browser-check',
    '--disable-extensions',
    '--window-size=1920,1080',
  ],
});

const context = await browser.newContext({
  viewport: { width: 1920, height: 1080 },
  userAgent: 'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36',
  locale: 'en-US',
  timezoneId: 'America/New_York',
  // Realistic permissions
  permissions: ['geolocation'],
  // Realistic color scheme
  colorScheme: 'light',
});

// Init script to patch detectable properties
await context.addInitScript(() => {
  // Hide webdriver flag
  Object.defineProperty(navigator, 'webdriver', {
    get: () => undefined,
  });

  // Fake plugins array
  Object.defineProperty(navigator, 'plugins', {
    get: () => {
      const plugins = [
        { name: 'Chrome PDF Plugin', filename: 'internal-pdf-viewer', description: 'Portable Document Format' },
        { name: 'Chrome PDF Viewer', filename: 'mhjfbmdgcfjbbpaeojofohoefgiehjai', description: '' },
        { name: 'Native Client', filename: 'internal-nacl-plugin', description: '' },
      ];
      plugins.length = 3;
      return plugins;
    },
  });

  // Fake languages
  Object.defineProperty(navigator, 'languages', {
    get: () => ['en-US', 'en'],
  });

  // Fake chrome runtime
  if (!window.chrome) {
    window.chrome = {};
  }
  if (!window.chrome.runtime) {
    window.chrome.runtime = {
      connect: () => {},
      sendMessage: () => {},
    };
  }

  // Fix WebGL renderer
  const getParameterOrig = WebGLRenderingContext.prototype.getParameter;
  WebGLRenderingContext.prototype.getParameter = function(param) {
    if (param === 37445) return 'Intel Inc.';         // UNMASKED_VENDOR_WEBGL
    if (param === 37446) return 'Intel Iris OpenGL Engine'; // UNMASKED_RENDERER_WEBGL
    return getParameterOrig.call(this, param);
  };

  // Fix permissions
  const origQuery = window.Permissions.prototype.query;
  window.Permissions.prototype.query = function(params) {
    if (params.name === 'notifications') {
      return Promise.resolve({ state: Notification.permission });
    }
    return origQuery.call(this, params);
  };
});
```

### 3.3 Provider Bot Detection Aggressiveness

| Provider | Bot Detection | Primary System | Notes |
|----------|--------------|----------------|-------|
| Google | Very aggressive | Custom + reCAPTCHA | Detects headless reliably; blocks programmatic login |
| Microsoft | Aggressive | Arkose Labs | Increasingly blocks automation |
| X/Twitter | Very aggressive | Arkose Labs | Heavily fingerprints; suspends automated accounts |
| Cloudflare-protected sites | Aggressive | Turnstile + Bot Mgmt | TLS fingerprinting, JS challenges |
| Discord | Aggressive | hCaptcha | Requires phone verification if bot-like |
| LinkedIn | Very aggressive | Custom + PerimeterX | Aggressive rate limiting and account restrictions |
| GitHub | Moderate | reCAPTCHA (selectively) | Mostly trusts authenticated sessions |
| Reddit | Moderate | reCAPTCHA / hCaptcha | Varies by endpoint |
| Slack | Low | None typically | Enterprise SSO adds complexity |
| Notion | Low | None typically | Magic link flow is main barrier |
| GitLab | Low | None typically | Standard form login |
| AWS Console | Moderate | AWS WAF CAPTCHA | Custom WAF rules |

---

## 4. Common Auth Flow Patterns

### 4.1 Email-First Multi-Step Flow (Google-style)

**Pattern:** Enter email -> Click Next -> Enter password -> Click Next -> (Optional: 2FA)

**Detection:**
```javascript
async function detectEmailFirstFlow(page) {
  const indicators = [
    // Google
    'input[type="email"][name="identifier"]',
    '#identifierId',
    // Microsoft
    'input[type="email"][name="loginfmt"]',
    // Generic
    'input[type="email"]:only-child', // Single email field, no password visible
  ];
  for (const sel of indicators) {
    if (await page.$(sel)) return true;
  }
  // Check if password field is hidden/absent when email field is present
  const emailField = await page.$('input[type="email"]');
  const passwordField = await page.$('input[type="password"]');
  if (emailField && !passwordField) return true;
  if (emailField && passwordField) {
    const pwVisible = await passwordField.isVisible();
    if (!pwVisible) return true;
  }
  return false;
}
```

**Handling strategy:**
1. Fill email field
2. Click "Next" / submit button
3. Wait for password field to appear (`page.waitForSelector('input[type="password"]', { state: 'visible' })`)
4. Fill password field
5. Click "Next" / submit button
6. Check for 2FA prompt

### 4.2 Magic Link Flow (Slack/Notion-style)

**Pattern:** Enter email -> Click "Send magic link" -> User checks email -> Clicks link -> Redirected back authenticated

**Detection:**
```javascript
const MAGIC_LINK_INDICATORS = {
  textPatterns: [
    'magic link',
    'sign-in link',
    'login link',
    'email a login',
    'check your email',
    'check your inbox',
    'we sent you',
    "we've sent",
    'email with a link',
    'passwordless',
    'sign in with email',
    'log in with email',
    'continue with email',
  ],
  selectors: [
    'button:has-text("Send magic link")',
    'button:has-text("Email me a link")',
    'button:has-text("Send link")',
    'button:has-text("Continue with email")',
  ],
};
```

**Handling strategy:** Magic links require human intervention (checking email). The agent should:
1. Detect the magic link flow
2. Inform the user: "This site uses magic link authentication. Please check your email and click the link."
3. Wait for navigation to success URL with extended timeout

### 4.3 Passwordless / Passkey-First Flow (Microsoft-style)

**Pattern:** Enter email -> System suggests passkey/Windows Hello/authenticator -> Option to "Use password instead"

**Detection:**
```javascript
const PASSKEY_INDICATORS = {
  selectors: [
    '[data-testid*="passkey"]',
    '[class*="passkey"]',
    'button:has-text("Use a passkey")',
    'button:has-text("Sign in with Windows Hello")',
    'button:has-text("Use your security key")',
    'div:has-text("Use password instead")',
    'a:has-text("Use your password instead")',
    '#idA_PWD_SwitchToPassword', // Microsoft-specific
  ],
  textPatterns: [
    'passkey',
    'security key',
    'windows hello',
    'face id',
    'touch id',
    'biometric',
    'use password instead',
    'other ways to sign in',
  ],
};
```

**Handling strategy:** Look for "Use password instead" or "Other ways to sign in" link and click it to fall back to password flow.

### 4.4 Social Login Buttons

**Detection:**
```javascript
const SOCIAL_LOGIN_SELECTORS = {
  google: [
    'button:has-text("Sign in with Google")',
    'button:has-text("Continue with Google")',
    '[data-provider="google"]',
    '.social-btn-google',
    'a[href*="accounts.google.com/o/oauth2"]',
    'div.g_id_signin', // Google One Tap
    'div#g_id_onload',
    'iframe[src*="accounts.google.com/gsi"]',
  ],
  apple: [
    'button:has-text("Sign in with Apple")',
    'button:has-text("Continue with Apple")',
    '[data-provider="apple"]',
    'div#appleid-signin',
    'a[href*="appleid.apple.com"]',
  ],
  github: [
    'button:has-text("Sign in with GitHub")',
    'button:has-text("Continue with GitHub")',
    '[data-provider="github"]',
    'a[href*="github.com/login/oauth"]',
  ],
  microsoft: [
    'button:has-text("Sign in with Microsoft")',
    'button:has-text("Continue with Microsoft")',
    '[data-provider="microsoft"]',
    'a[href*="login.microsoftonline.com"]',
  ],
  facebook: [
    'button:has-text("Continue with Facebook")',
    'button:has-text("Log in with Facebook")',
    '[data-provider="facebook"]',
    'a[href*="facebook.com/dialog/oauth"]',
    'div.fb-login-button',
  ],
};
```

### 4.5 Account Picker / Chooser Pages

**Detection:**
```javascript
const ACCOUNT_PICKER_SELECTORS = [
  // Google
  'div[data-identifier]',        // Google account picker entries
  '#profileIdentifier',          // Google "Choose an account"
  'ul[class*="account-list"]',
  // Microsoft
  'div[data-test-id="tile"]',    // Microsoft account tiles
  '#tilesHolder',
  // Generic
  '[class*="account-chooser"]',
  '[class*="account-picker"]',
  '[class*="account-selector"]',
  'div[role="listbox"]',
  ':has-text("Choose an account")',
  ':has-text("Pick an account")',
  ':has-text("Select an account")',
  ':has-text("Use another account")',
];
```

### 4.6 Post-Login Gates

**"Remember this device" / "Trust this browser":**
```javascript
const TRUST_DEVICE_SELECTORS = [
  'button:has-text("Yes")',
  'button:has-text("Trust")',
  'button:has-text("Remember")',
  'button:has-text("Don\'t ask again")',
  'input[name="remember"]',
  'input[name="trust"]',
  '#trust-browser-button',
  ':has-text("Stay signed in")',
  ':has-text("Keep me signed in")',
  ':has-text("Remember this device")',
  ':has-text("Trust this browser")',
  ':has-text("Don\'t ask again on this device")',
];
```

**Terms of Service acceptance:**
```javascript
const TOS_GATE_SELECTORS = [
  'button:has-text("I Agree")',
  'button:has-text("Accept")',
  'button:has-text("Agree and Continue")',
  'input[type="checkbox"][name*="terms"]',
  'input[type="checkbox"][name*="tos"]',
  ':has-text("Terms of Service")',
  ':has-text("Terms and Conditions")',
  ':has-text("Privacy Policy")',
];
```

**Phone verification prompts:**
```javascript
const PHONE_VERIFY_INDICATORS = [
  ':has-text("Verify your phone")',
  ':has-text("Add a phone number")',
  ':has-text("Enter your phone")',
  ':has-text("Confirm your phone")',
  ':has-text("We sent a code to")',
  ':has-text("Enter the code")',
  ':has-text("Verification code")',
  'input[type="tel"]',
  'input[name*="phone"]',
  'input[autocomplete="tel"]',
];
```

---

## 5. Success Detection Patterns

### 5.1 Multi-Signal Success Detection

No single signal is reliable across all providers. Layer multiple signals:

```javascript
async function detectAuthSuccess(page, config = {}) {
  const signals = [];

  // Signal 1: URL-based
  const url = page.url();
  const loginPaths = ['/login', '/signin', '/auth', '/account/login', '/session/new'];
  const successPaths = ['/home', '/dashboard', '/feed', '/inbox', '/app', '/console', '/settings'];
  const isOnLoginPage = loginPaths.some(p => url.includes(p));
  const isOnSuccessPage = successPaths.some(p => url.includes(p));

  if (config.successUrl && url.startsWith(config.successUrl)) {
    signals.push({ type: 'url_match', confidence: 0.9 });
  } else if (isOnSuccessPage && !isOnLoginPage) {
    signals.push({ type: 'url_pattern', confidence: 0.6 });
  } else if (!isOnLoginPage) {
    signals.push({ type: 'url_left_login', confidence: 0.4 });
  }

  // Signal 2: Cookie-based (most reliable per-provider)
  if (config.successCookie) {
    const cookies = await page.context().cookies();
    const found = cookies.find(c =>
      c.domain.includes(config.successCookie.domain) &&
      c.name === config.successCookie.name &&
      (!config.successCookie.value || c.value === config.successCookie.value)
    );
    if (found) {
      signals.push({ type: 'cookie_match', confidence: 0.95 });
    }
  }

  // Signal 3: DOM-based (universal indicators of logged-in state)
  const loggedInSelectors = [
    'img[alt*="avatar"]',
    'img[alt*="profile"]',
    'img[class*="avatar"]',
    '[class*="avatar"]',
    '[class*="user-menu"]',
    '[class*="user-nav"]',
    '[class*="profile-menu"]',
    '[aria-label*="Account"]',
    '[aria-label*="Profile"]',
    '[aria-label*="Settings"]',
    'a[href*="/settings"]',
    'a[href*="/profile"]',
    'a[href*="/account"]',
    'button:has-text("Sign Out")',
    'button:has-text("Log Out")',
    'a:has-text("Sign Out")',
    'a:has-text("Log Out")',
    '[data-testid*="logout"]',
    '[data-testid*="signout"]',
  ];

  for (const sel of loggedInSelectors) {
    try {
      const el = await page.$(sel);
      if (el && await el.isVisible()) {
        signals.push({ type: 'dom_logged_in', confidence: 0.7, selector: sel });
        break;
      }
    } catch { continue; }
  }

  // Signal 4: Login form absence
  const loginFormGone = !(await page.$('input[type="password"]:visible'));
  if (loginFormGone) {
    signals.push({ type: 'login_form_absent', confidence: 0.3 });
  }

  // Aggregate confidence
  const maxConfidence = Math.max(0, ...signals.map(s => s.confidence));
  const combinedConfidence = signals.length > 1
    ? Math.min(1.0, maxConfidence + (signals.length - 1) * 0.1)
    : maxConfidence;

  return {
    authenticated: combinedConfidence >= 0.7,
    confidence: combinedConfidence,
    signals,
  };
}
```

### 5.2 SPA Success Detection

For SPAs where URL may not change:

```javascript
async function detectSPAAuthSuccess(page, { timeout = 10000 } = {}) {
  // Watch for DOM mutations that indicate auth state change
  return await page.evaluate((timeout) => {
    return new Promise((resolve) => {
      const timer = setTimeout(() => resolve({ success: false, reason: 'timeout' }), timeout);

      // Watch for login form disappearing
      const observer = new MutationObserver(() => {
        const loginForm = document.querySelector('input[type="password"]');
        const avatar = document.querySelector('[class*="avatar"], [class*="user-menu"]');
        if (!loginForm && avatar) {
          clearTimeout(timer);
          observer.disconnect();
          resolve({ success: true, reason: 'dom_transition' });
        }
      });
      observer.observe(document.body, { childList: true, subtree: true });
    });
  }, timeout);
}
```

### 5.3 Provider-Specific Success Cookies

Reference from the existing `providers.json` (see `/home/ubuntu/agent-sh/web-ctl/scripts/providers.json`):

| Provider | Cookie Domain | Cookie Name | Cookie Value |
|----------|-------------|-------------|-------------|
| GitHub | .github.com | `logged_in` | `yes` |
| Google | .google.com | `SAPISID` | (any) |
| Reddit | .reddit.com | `reddit_session` | (any) |
| LinkedIn | .linkedin.com | `li_at` | (any) |
| GitLab | .gitlab.com | `_gitlab_session` | (any) |
| Notion | .notion.so | `token_v2` | (any) |
| Discord | (URL-based) | N/A | successUrl: `/channels` |
| X/Twitter | (URL-based) | N/A | successUrl: `/home` |
| Microsoft | (URL-based) | N/A | successUrl: office.com |
| Slack | (URL-based) | N/A | successUrl: app.slack.com |
| AWS | (URL-based) | N/A | successUrl: console.aws.amazon.com |
| Atlassian | (URL-based) | N/A | successUrl: start.atlassian.com |

---

## 6. Error State Detection

### 6.1 Credential Errors

```javascript
const CREDENTIAL_ERROR_PATTERNS = {
  wrongPassword: [
    'wrong password',
    'incorrect password',
    'invalid password',
    'password is incorrect',
    'password you entered is incorrect',
    "that password isn't right",
    'authentication failed',
    'invalid credentials',
    'login failed',
    'sign-in failed',
    'email or password is incorrect',
    'invalid email or password',
    'invalid username or password',
    'bad credentials',
    'unable to sign in',
  ],
  accountLocked: [
    'account is locked',
    'account has been locked',
    'too many attempts',
    'too many failed',
    'temporarily locked',
    'try again later',
    'try again in',
    'account suspended',
    'account disabled',
    'account has been disabled',
    'account has been suspended',
    'account is deactivated',
    'your account is blocked',
    'access to your account has been',
    'unusual activity',
    'suspicious activity',
  ],
  accountNotFound: [
    "couldn't find your account",
    "can't find an account",
    'no account found',
    'account does not exist',
    'user not found',
    "email doesn't match",
    'not registered',
    'no user with that email',
  ],
  requiresVerification: [
    'verify your identity',
    'confirm your identity',
    'additional verification',
    'security challenge',
    'confirm it is you',
    "confirm it's you",
    'verify it is you',
    "verify it's you",
    'unusual sign-in',
    'new device',
    'unrecognized device',
  ],
};

async function detectAuthError(page) {
  const bodyText = (await page.textContent('body').catch(() => '')).toLowerCase();

  for (const [errorType, patterns] of Object.entries(CREDENTIAL_ERROR_PATTERNS)) {
    for (const pattern of patterns) {
      if (bodyText.includes(pattern)) {
        return { error: true, type: errorType, matchedPattern: pattern };
      }
    }
  }

  // Check for error-styled elements
  const errorSelectors = [
    '[class*="error"]',
    '[class*="alert-danger"]',
    '[class*="alert-error"]',
    '[role="alert"]',
    '.notice--error',
    '.flash-error',
    '.error-message',
    '#error-message',
  ];

  for (const sel of errorSelectors) {
    const el = await page.$(sel);
    if (el && await el.isVisible()) {
      const text = await el.textContent();
      if (text && text.trim().length > 0) {
        return { error: true, type: 'generic_error', message: text.trim(), selector: sel };
      }
    }
  }

  return { error: false };
}
```

### 6.2 Network and Session Errors

```javascript
async function detectNetworkErrors(page) {
  const errors = [];

  // Listen for failed requests during auth
  page.on('response', response => {
    const status = response.status();
    const url = response.url();
    if (status === 401) errors.push({ type: 'unauthorized', url, status });
    if (status === 403) errors.push({ type: 'forbidden', url, status });
    if (status === 429) errors.push({ type: 'rate_limited', url, status });
    if (status >= 500) errors.push({ type: 'server_error', url, status });
  });

  page.on('requestfailed', request => {
    errors.push({
      type: 'request_failed',
      url: request.url(),
      failure: request.failure()?.errorText,
    });
  });

  return {
    getErrors: () => [...errors],
    hasErrors: () => errors.length > 0,
  };
}
```

---

## 7. Playwright Best Practices for Auth

### 7.1 storageState for Session Persistence

```javascript
// Save authenticated state
await page.context().storageState({ path: 'auth.json' });

// Reuse in new context
const context = await browser.newContext({
  storageState: 'auth.json',  // Restores cookies + localStorage + sessionStorage
});

// storageState JSON structure:
// {
//   "cookies": [...],
//   "origins": [
//     {
//       "origin": "https://example.com",
//       "localStorage": [{ "name": "key", "value": "val" }]
//     }
//   ]
// }
```

### 7.2 Cookie Management

```javascript
// Get all cookies
const allCookies = await context.cookies();

// Get cookies for specific URLs
const cookies = await context.cookies(['https://github.com', 'https://api.github.com']);

// Add cookies manually
await context.addCookies([
  {
    name: 'session_id',
    value: 'abc123',
    domain: '.example.com',
    path: '/',
    httpOnly: true,
    secure: true,
    sameSite: 'Lax',
    expires: Math.floor(Date.now() / 1000) + 86400,
  },
]);

// Clear cookies
await context.clearCookies();

// Clear cookies for specific domain
await context.clearCookies({ domain: '.example.com' });
```

### 7.3 Navigation Waiting Strategies

```javascript
// waitForURL - best for known redirect targets
await page.waitForURL('**/dashboard', { timeout: 30000 });
await page.waitForURL(url => !url.includes('/login'), { timeout: 30000 });

// waitForLoadState - best for page fully loaded
await page.waitForLoadState('networkidle'); // No network requests for 500ms
await page.waitForLoadState('domcontentloaded'); // DOM ready
await page.waitForLoadState('load'); // All resources loaded

// waitForNavigation - DEPRECATED in Playwright, use waitForURL instead
// But for capturing redirect chains:
const [response] = await Promise.all([
  page.waitForNavigation({ waitUntil: 'networkidle' }),
  page.click('#login-button'),
]);

// waitForResponse - for API-based auth
const response = await page.waitForResponse(
  resp => resp.url().includes('/api/auth') && resp.status() === 200
);
```

### 7.4 Handling Multi-Domain Redirects

Auth flows often redirect through multiple domains (e.g., app.example.com -> auth.provider.com -> app.example.com/callback).

```javascript
// Track redirect chain
const redirects = [];
page.on('framenavigated', frame => {
  if (frame === page.mainFrame()) {
    redirects.push(frame.url());
  }
});

// Wait for final destination after clicking login
await page.click('#login-with-sso');

// Wait until we leave the IdP domain and return to the app
await page.waitForURL(url => {
  const hostname = new URL(url).hostname;
  return hostname.includes('app.example.com');
}, { timeout: 60000 });
```

### 7.5 Timeout Strategy for Auth Flows

```javascript
const AUTH_TIMEOUTS = {
  // Page load after clicking login
  pageLoad: 15000,
  // Waiting for redirect after credential submission
  authRedirect: 30000,
  // Waiting for user to complete manual steps (2FA, CAPTCHA)
  humanInteraction: 300000, // 5 minutes
  // Waiting for magic link click
  magicLink: 600000, // 10 minutes
  // Individual element appearance
  elementAppear: 10000,
  // Network idle after auth
  networkIdle: 15000,
};
```

### 7.6 Complete Auth Flow Template

```javascript
async function authenticateWithProvider(page, provider, credentials) {
  const { loginUrl, successCookie, successUrl, captchaSelectors } = provider;

  // Step 1: Navigate to login page
  await page.goto(loginUrl, { waitUntil: 'domcontentloaded', timeout: AUTH_TIMEOUTS.pageLoad });

  // Step 2: Dismiss cookie consent if blocking
  if (await isBlockedByOverlay(page)) {
    await dismissCookieConsent(page);
  }

  // Step 3: Check for CAPTCHA before starting
  const captchaCheck = await detectCaptcha(page);
  if (captchaCheck.detected) {
    return { ok: false, error: 'captcha_detected', systems: captchaCheck.systems };
  }

  // Step 4: Detect flow type and fill credentials
  const isEmailFirst = await detectEmailFirstFlow(page);

  if (isEmailFirst) {
    // Email-first flow
    const emailInput = await page.$('input[type="email"], input[name="email"], input[name="identifier"], #identifierId, input[name="loginfmt"]');
    await emailInput.fill(credentials.email);
    await page.click('button[type="submit"], input[type="submit"], button:has-text("Next"), button:has-text("Continue")');
    await page.waitForSelector('input[type="password"]', { state: 'visible', timeout: AUTH_TIMEOUTS.elementAppear });
  }

  // Fill password
  const passwordInput = await page.$('input[type="password"]');
  if (passwordInput) {
    await passwordInput.fill(credentials.password);
    await page.click('button[type="submit"], input[type="submit"], button:has-text("Sign in"), button:has-text("Log in"), button:has-text("Next")');
  }

  // Step 5: Wait for auth to complete (with CAPTCHA/2FA monitoring)
  const deadline = Date.now() + AUTH_TIMEOUTS.humanInteraction;

  while (Date.now() < deadline) {
    // Check for success
    const success = await detectAuthSuccess(page, { successCookie, successUrl });
    if (success.authenticated) {
      return { ok: true, confidence: success.confidence, signals: success.signals };
    }

    // Check for errors
    const error = await detectAuthError(page);
    if (error.error) {
      return { ok: false, error: error.type, message: error.message || error.matchedPattern };
    }

    // Check for CAPTCHA (may appear after credential submission)
    const captcha = await detectCaptcha(page);
    if (captcha.detected) {
      return { ok: false, error: 'captcha_detected', systems: captcha.systems, message: 'CAPTCHA appeared after credential submission - human intervention required' };
    }

    // Check for "trust this device" and auto-dismiss
    for (const sel of TRUST_DEVICE_SELECTORS) {
      try {
        const btn = await page.$(sel);
        if (btn && await btn.isVisible()) {
          await btn.click();
          break;
        }
      } catch { continue; }
    }

    await page.waitForTimeout(1000); // Poll every second
  }

  return { ok: false, error: 'auth_timeout' };
}
```

---

## Common Pitfalls

| Pitfall | Why It Happens | How to Avoid |
|---------|---------------|--------------|
| CAPTCHA appears after submission, not before | Many sites only trigger CAPTCHA on suspicious credential submission | Always check for CAPTCHA in polling loop after submission, not just before |
| Cookie consent blocks click on login button | Overlay has higher z-index | Always run consent dismissal before interacting with login form |
| `waitForNavigation` resolves on wrong redirect | Auth goes through IdP and back; first navigation is not final | Use `waitForURL` with predicate that matches final destination |
| storageState does not persist httpOnly cookies | Playwright storageState DOES include httpOnly cookies | This is not actually a pitfall - storageState captures everything. But verify with `context.cookies()` |
| Bot detection triggers after first page load | Some fingerprinting runs asynchronously | Apply stealth init scripts BEFORE navigating to any page |
| Social login popup window not handled | OAuth opens a popup; Playwright creates new page | Use `context.waitForEvent('page')` to capture popup |
| Account picker bypassed incorrectly | Clicking "Use another account" instead of selecting existing | Check for existing account tiles matching target email first |
| SPA auth success not detected | URL does not change; DOM updates asynchronously | Use MutationObserver-based detection or poll for logged-in DOM elements |
| Cloudflare challenge mistaken for CAPTCHA | Full-page Cloudflare challenges are not interactive CAPTCHAs | Detect separately; Turnstile widget auto-solves, but full-page challenge may require waiting |
| `networkidle` never fires on SPAs | Websockets/polling keep network active | Use `domcontentloaded` + explicit element waits instead |

---

## Best Practices

1. **Layer success detection signals.** Never rely on URL alone (SPAs), cookies alone (may not know names), or DOM alone (selectors change). Combine all three.

2. **Always dismiss cookie consent first.** Run consent dismissal as the first action after page load, before any auth interaction.

3. **Detect CAPTCHA early and fail fast.** If CAPTCHA is detected, hand off to human immediately rather than attempting to interact with it.

4. **Use `storageState` for session persistence.** It captures cookies, localStorage, and sessionStorage in one JSON file. More complete than manual cookie extraction.

5. **Apply stealth patches before navigation.** `addInitScript` runs before any page JS; applying it after navigation is too late for fingerprinting checks.

6. **Set generous timeouts for human-in-the-loop.** Auth flows involving 2FA, CAPTCHA, or magic links need 5-10 minute timeouts, not the default 30 seconds.

7. **Handle popup windows for social login.** Social login opens in a popup; use `context.waitForEvent('page')` to get the popup page handle.

8. **Detect error states explicitly.** Do not just wait for timeout on failure. Check for error messages in the DOM and fail fast with actionable error info.

9. **Prefer `waitForURL` over deprecated `waitForNavigation`.** Playwright's `waitForURL` with a predicate is cleaner and more reliable for redirect chains.

10. **Use `networkidle` carefully.** It never resolves on pages with websockets or polling. Use `domcontentloaded` plus explicit selector waits.

11. **Keep provider configs in external JSON.** The `providers.json` pattern (as in web-ctl) allows adding new providers without code changes.

12. **Test against real sites periodically.** CAPTCHA selectors and auth flows change. Automated smoke tests against real login pages catch regressions.

---

## Further Reading

| Resource | Type | Why Recommended |
|----------|------|-----------------|
| [Playwright Auth Docs](https://playwright.dev/docs/auth) | Official Docs | Canonical storageState and auth patterns |
| [Playwright Navigation Docs](https://playwright.dev/docs/navigations) | Official Docs | waitForURL, loadState, redirect handling |
| [reCAPTCHA Developer Guide](https://developers.google.com/recaptcha/docs/display) | Official Docs | v2/v3/Enterprise DOM structure |
| [hCaptcha Developer Docs](https://docs.hcaptcha.com/) | Official Docs | hCaptcha integration and DOM elements |
| [Cloudflare Turnstile Docs](https://developers.cloudflare.com/turnstile/) | Official Docs | Turnstile widget implementation |
| [playwright-stealth](https://github.com/AtuboDad/playwright-stealth) | GitHub | Stealth patches for Playwright |
| [puppeteer-extra-plugin-stealth](https://github.com/nicoleahmed/puppeteer-extra-plugin-stealth) | GitHub | Original stealth research (applies to Playwright too) |
| [web-ctl providers.json](file:///home/ubuntu/agent-sh/web-ctl/scripts/providers.json) | Local | Existing provider configs in this codebase |
| [OneTrust Cookie Consent](https://www.onetrust.com/products/cookie-consent/) | Vendor | Most common enterprise CMP |

---

*This guide was synthesized from 40+ sources (Playwright docs, CAPTCHA vendor documentation, web security research, and codebase analysis). Quality scores were based on authority and recency.*
