---
name: cli-duo-auth
description: >
  Implement Duo SSO authentication in CLI tools using browser automation
  (Playwright, chromedp, Selenium, etc.) to intercept tokens (SAML, OAuth, OIDC)
  after Duo MFA. Use when user mentions "Duo authentication in CLI", "Duo SSO for CLI tool",
  "Duo MFA in terminal", "CLI Duo login", "Duo browser auth", "intercept SAML after Duo",
  "browser-based Duo token capture", "CLIツールにDuo認証を追加", or wants to add
  Duo SSO authentication flow to a command-line application.
---

# Duo SSO Authentication Pattern for CLI Tools

## Overview

A language-agnostic pattern for CLI tools that need to authenticate users through
Duo SSO with MFA, by launching a real browser and intercepting the authentication
token from network traffic. The user completes Duo MFA (push, passcode, etc.)
in the browser while the tool captures the resulting token automatically.

## Architecture

```
CLI Tool
  |
  +-- Launch browser (headless=false, with profile dir)
  |
  +-- Navigate to IdP login URL
  |
  +-- (Optional) Auto-fill username/email
  |
  +-- User completes MFA in browser manually
  |
  +-- Intercept network request to token endpoint
  |     |
  |     +-- Match URL (e.g., AWS SAML URN, OAuth redirect)
  |     +-- Extract token from POST body or URL params
  |
  +-- Close browser
  |
  +-- Use token (e.g., assume AWS role, call API)
```

## Implementation Steps

### Step 1: Choose a browser automation library

| Language | Recommended | Alternative |
|----------|-------------|-------------|
| Python | Playwright (`playwright`) | Selenium + selenium-wire |
| TypeScript/JS | Playwright (`playwright`) | Puppeteer |
| Go | chromedp | go-rod |
| Rust | chromiumoxide | fantoccini |

Playwright is preferred for Python/JS/TS because `page.on("request")` provides
direct access to POST data without extra decoding. chromedp is the standard for Go.

### Step 2: Browser launch configuration

Configure the browser with these key options:

```
Required:
  - headless: false          # User must see and interact with MFA
  - user_data_dir: <path>    # Profile directory for session persistence
  - channel: "chrome"        # Use system Chrome (passkey/FIDO2 support)

Stealth (critical for Duo):
  - args: ["--disable-blink-features=AutomationControlled"]
  - Override navigator.webdriver to undefined via init script

Optional:
  - executable_path: <path>  # Custom browser binary (Edge, Chrome, etc.)
  - timeout: 600s            # 10 min timeout for MFA completion
  - disable_extensions: true # Sandbox mode for temporary profiles
  - disable_cache: true      # Clean state for temporary profiles
```

**System Chrome vs Playwright Chromium (passkey/FIDO2 support):**

Playwright ships its own Chromium binary, which does NOT have access to the user's
registered passkeys (FIDO2/WebAuthn). If the user has set up passkey authentication
with Duo in their regular Chrome, Playwright's Chromium will not recognize it,
forcing a full MFA challenge every time.

**Solution:** Use Playwright's `channel` parameter to launch the system Chrome instead.

| | Playwright Chromium (default) | System Chrome (`channel="chrome"`) |
|---|---|---|
| Binary | `~/.cache/ms-playwright/chromium-*` | `/Applications/Google Chrome.app` etc. |
| Passkeys | Not registered → full MFA every time | User's passkeys available → auto-auth |
| Duo "Remember me" | Synthetic fingerprint → not recognized | Real device fingerprint → recognized |
| Extensions | None | System extensions available |

**Playwright (Python) — system Chrome detection and launch:**

```python
import sys
from pathlib import Path

def _find_system_chrome() -> bool:
    if sys.platform == "darwin":
        return Path("/Applications/Google Chrome.app").exists()
    elif sys.platform == "win32":
        import shutil
        return shutil.which("chrome.exe") is not None
    else:
        import shutil
        return (shutil.which("google-chrome") is not None
                or shutil.which("google-chrome-stable") is not None)
```

**Stealth configuration (avoiding repeated MFA):**

Even with system Chrome, Duo may detect automation markers and force extra MFA.
Apply these stealth measures:

| Detection Vector | Default Behavior | Fix |
|---|---|---|
| `navigator.webdriver` | `true` in Playwright | Override to `undefined` via init script |
| `AutomationControlled` | Enabled in Chromium | Disable via `--disable-blink-features` |
| Playwright globals | `__playwright__binding__` exposed | Cannot fully remove; stealth args mitigate |
| Device fingerprint | Synthetic in Playwright, real in chromedp | Use system Chrome + persistent profile |

**Playwright (Python) — recommended launch configuration:**

```python
_stealth_args = [
    "--disable-blink-features=AutomationControlled",
]
# Use system Chrome for passkey support; fall back to Playwright Chromium
_channel = "chrome" if _find_system_chrome() else None

launch_kwargs = dict(
    user_data_dir=str(profile_dir),
    headless=False,
    args=_stealth_args,
)
if _channel:
    launch_kwargs["channel"] = _channel

context = p.chromium.launch_persistent_context(**launch_kwargs)
page = context.new_page()

# Hide navigator.webdriver flag from Duo detection
page.add_init_script("""
    Object.defineProperty(navigator, 'webdriver', { get: () => undefined });
""")
```

**chromedp (Go) — equivalent setup:**

```go
opts := append(chromedp.DefaultExecAllocatorOptions[:],
    chromedp.Flag("disable-blink-features", "AutomationControlled"),
    chromedp.UserDataDir(profileDir),
    chromedp.Flag("headless", false),
)
```

chromedp uses the real system Chrome binary by default, so passkeys work
out of the box without extra configuration.

**Verified behavior (circuit-cli):**
- Playwright Chromium (no stealth): Duo requires full MFA every time
- Playwright Chromium + stealth + persistent profile: Duo remembers device after first MFA
- System Chrome + stealth + persistent profile: passkey auto-auth, email entry only

**Profile modes:**

- **Temporary** (default): Create a temp directory, delete on exit. Clean sandbox.
- **Persistent**: Reuse a fixed directory. Caches login state, skips repeated prompts.

When using persistent mode, skip sandbox flags (disable_cache, etc.) so the browser
retains cookies and session data between runs.

### Step 3: Navigate and auto-fill credentials

1. Navigate to the IdP SSO URL
2. If email/username is known (from config):
   - Wait for the email input field with a short timeout (3-5s)
   - If found, fill it and click the submit/next button
   - If not found (persistent profile cached it), skip silently
3. Handle "Is this your device?" confirmation if Duo shows it:
   - Wait for "Yes" button with a short timeout (2-3s)
   - Click it if found, skip silently if not
4. Let the user handle the rest (passkey, MFA push) in the browser

**Important:** Use short timeouts for ALL optional UI elements. With persistent
profiles and passkeys, most prompts are skipped — the entire flow may complete
automatically without user interaction.

**Playwright (Python) — auto-fill implementation:**

```python
def _autofill_sso(page, config: dict, logger) -> None:
    """Auto-fill email and click through SSO prompts."""
    email = config.get("sso_email")
    if not email:
        return

    # Wait for email field (may not appear if cached)
    try:
        page.wait_for_selector('input[type="email"]', timeout=5_000)
    except Exception:
        logger.debug("Email field not found (cached), skipping")
        return

    logger.debug("Auto-filling email: %s", email)
    page.fill('input[type="email"]', email)

    # Click Next/Submit button
    try:
        page.click('button[type="submit"], input[type="submit"]', timeout=2_000)
    except Exception:
        try:
            page.locator("button").first.click(timeout=2_000)
        except Exception:
            logger.debug("Submit button not found")

    # Handle "Is this your device?" confirmation
    try:
        page.locator('button:has-text("Yes")').click(timeout=3_000)
        logger.debug("Clicked 'Yes' on device confirmation")
    except Exception:
        logger.debug("Device confirmation not shown")
```

**chromedp (Go) — equivalent from duo-sso:**

```go
func setUsernameInBrowserLogin(ctx context.Context, email string) {
    emailSelector := `input[type="email"]`

    // Short timeout — field may be cached
    emailCtx, cancel := context.WithTimeout(ctx, 3*time.Second)
    defer cancel()
    if err := chromedp.Run(emailCtx, chromedp.WaitVisible(emailSelector)); err != nil {
        return // cached, skip
    }

    chromedp.Run(ctx, chromedp.SendKeys(emailSelector, email))
    chromedp.Run(ctx, chromedp.Click("button", chromedp.NodeVisible))

    // "Is this your device?" — may not appear
    deviceCtx, cancel := context.WithTimeout(ctx, 2*time.Second)
    defer cancel()
    yesBtn := `//button[contains(normalize-space(text()), "Yes")]`
    if err := chromedp.Run(deviceCtx, chromedp.WaitVisible(yesBtn)); err == nil {
        chromedp.Run(deviceCtx, chromedp.Click(yesBtn, chromedp.NodeVisible))
    }
}
```

**Configuration:** Store the email in the tool's config file so users set it once:
```
# circuit-cli example:
/config set sso_email user@company.com

# duo-sso uses --email flag or config file
duo-sso --email user@company.com
```

### Step 4: Intercept the authentication token

This is the core pattern. Register a network request listener BEFORE navigation:

```
BEFORE navigating to the IdP URL:
  Register listener for all outgoing requests
    ON each request:
      IF request.url matches token_endpoint AND method is POST:
        Parse POST body as form-urlencoded
        Extract token field (e.g., "SAMLResponse", "code", "id_token")
        Signal the main flow with the token value
```

**Token types and where to find them:**

| Protocol | Token Field | Where |
|----------|-------------|-------|
| SAML | `SAMLResponse` | POST body to ACS URL |
| OAuth 2.0 | `code` | Redirect URL query param |
| OIDC | `id_token` | POST body or redirect URL fragment |

### Step 5: Wait for token and clean up

Two approaches to wait:

**A. Event-driven (preferred for JS/TS):**
Use `page.waitForRequest(predicate, { timeout })` to block until the matching
request is detected.

**B. Channel/polling (Go, Python):**
Use a channel (Go) or asyncio event/polling loop (Python) to wait for the
listener callback to signal token receipt.

After receiving the token (or detecting auth redirect):
1. Save state (cookies, tokens) immediately
2. Close the browser context (see performance notes below)
3. Delete temporary profile directory if applicable
4. Return the token/state to the caller

**Performance: browser teardown with persistent contexts**

When using system Chrome with `launch_persistent_context`, `context.close()` can
be slow (10+ seconds) because Chrome syncs the user profile to disk.

Mitigations:
- **Use `domcontentloaded` instead of `networkidle`** for post-auth wait.
  SPAs often never reach `networkidle`, causing unnecessary timeouts.
  Cookies are set on redirect, so `domcontentloaded` (or a short sleep) is sufficient.
- **Close the page before closing the context.** This reduces the profile sync overhead.
- **Keep wait timeouts short** (3-5s) for post-auth load states.

```python
# GOOD: fast teardown
try:
    page.wait_for_load_state("domcontentloaded", timeout=5_000)
except Exception:
    pass
context.storage_state(path=str(auth_state_file))  # save cookies
page.close()       # close page first
context.close()    # then close context (faster with page already closed)

# BAD: slow teardown
page.wait_for_load_state("networkidle", timeout=15_000)  # may never fire on SPAs
context.storage_state(path=str(auth_state_file))
context.close()    # slow: Chrome syncs full profile with page still open
```

### Step 6: Use the token

Depending on the protocol:
- **SAML**: Parse the assertion to extract roles/attributes, then call
  `sts:AssumeRoleWithSAML` or equivalent
- **OAuth**: Exchange the authorization code for access/refresh tokens
- **OIDC**: Validate the id_token and extract claims

## Language-Specific Examples

### Go (chromedp)

```go
// Listen for network events before navigation
chromedp.ListenTarget(ctx, func(ev interface{}) {
    if req, ok := ev.(*network.EventRequestWillBeSent); ok {
        if req.Request.URL == tokenEndpoint {
            postData := decodePostData(req.Request.PostDataEntries)
            form, _ := url.ParseQuery(postData)
            tokenChan <- form.Get("SAMLResponse")
        }
    }
})
```

Note: chromedp requires base64 decoding of `PostDataEntries[].Bytes` before parsing.

### Python (Playwright)

```python
def on_request(request):
    if request.url == token_endpoint and request.method == "POST":
        form = parse_qs(request.post_data)
        token_event.set(form["SAMLResponse"][0])

page.on("request", on_request)
await page.goto(idp_url)
```

`request.post_data` is a plain string - no extra decoding needed.

### TypeScript (Playwright)

```typescript
page.on("request", (request) => {
    if (request.url() === tokenEndpoint && request.method() === "POST") {
        const form = new URLSearchParams(request.postData() ?? "");
        resolve(form.get("SAMLResponse")!);
    }
});

// Or use waitForRequest for cleaner flow:
const req = await page.waitForRequest(
    (r) => r.url() === tokenEndpoint && r.method() === "POST",
    { timeout: 600_000 }
);
```

## Error Handling

Handle these failure modes:

1. **Browser not found**: Check `executable_path` exists or fall back to auto-detection.
   Provide a helpful error message with install instructions.
2. **Timeout**: User did not complete MFA within the time limit. Use 10 minutes as default.
3. **Navigation failure**: IdP URL unreachable. Check network connectivity.
4. **Token not intercepted**: The listener URL pattern may have changed.
   Log all requests in debug mode to help diagnose.
5. **Profile directory error**: Cannot create temp dir or access persistent dir.

## Security Considerations

- **Temporary profiles**: Delete on exit (`defer os.RemoveAll` / `finally: shutil.rmtree`).
  Do not leak session cookies.
- **Persistent profiles**: Store in a user-controlled directory with restricted permissions.
  Warn users that cached sessions reduce MFA frequency.
- **Token handling**: Keep tokens in memory only. Do not log tokens at INFO level.
  Use DEBUG level with explicit opt-in.
- **Sandbox flags**: In temporary mode, disable extensions, cache, and background networking
  to minimize attack surface.

## Fallback: Browserless Mode

For CI/CD or headless environments, consider a fallback HTTP client-based flow:

1. Scrape the login page, submit credentials via HTTP POST
2. Handle MFA via API (e.g., Duo Auth API for push/passcode)
3. Follow redirects to capture the token

This is more fragile (breaks when IdP changes HTML) but works without a display.
Use browser mode as the default and browserless as an opt-in alternative.
