---
name: webapp-testing
description: >
  Autonomous end-to-end QA skill for any web application. Use this skill when the user wants to test a website or web app. Triggers on: "test this website", "run QA on my web app", "/webapp-testing", "end-to-end test", "test this URL", "write test cases for my web app", "check if my site works", "automated testing for web", or any mention of testing a website or web application. The skill initializes a QA workspace per domain (qa-web/{domain}/), opens the URL with Playwright, takes screenshots for visual analysis, discovers all pages and flows, generates test scenarios, writes and runs actual Playwright test scripts, and saves evidence. Each website gets its own isolated folder so previous work is never lost.
---

# Web App QA Skill

An autonomous QA engineer for **any web application or website**. Opens your URL, takes screenshots, discovers every page and user flow, generates test scenarios, and runs real Playwright end-to-end tests with evidence.

**Works on**: localhost ✅ | staging URLs ✅ | live websites ✅

**Multi-site safe**: Each website saved in its own `qa-web/{domain}/` folder — previous work never overwritten.

---

## Step 0: Mode Detection

**First action every time** — derive the workspace from the URL, then check its state.

### 0.1 Get the URL

If the user provided a URL in their message, use it directly.
If not, ask: "What URL would you like to test?"

### 0.2 Derive Workspace Directory

```python
from urllib.parse import urlparse
import re

url = "USER_PROVIDED_URL"
domain = urlparse(url).netloc.replace("www.", "")      # e.g. "gsmarena.com"
domain_slug = re.sub(r'[^a-z0-9]', '-', domain.lower())  # e.g. "gsmarena-com"
workspace = f"qa-web/{domain_slug}"
print(workspace)  # → "qa-web/gsmarena-com"
```

**Every path in this skill uses `{workspace}` — never `qa-web/` directly.**
This keeps each site isolated:

```
qa-web/
├── gsmarena-com/        ← gsmarena.com ka sab kaam
├── purevpn-com/         ← purevpn.com ka sab kaam
└── localhost-3000/      ← local dev app ka sab kaam
```

### 0.3 Check Workspace State

```python
import os
config_path = f"{workspace}/.qa-config.json"

if not os.path.isdir(workspace) or not os.path.isfile(config_path):
    print("INIT")
elif not os.path.isdir(f"{workspace}/flows") and not os.path.isdir(f"{workspace}/features"):
    print("CONFIGURED_NO_FLOWS")
else:
    print("HAS_WORKSPACE")
```

| Result | Action |
|--------|--------|
| `INIT` | → **Step 1: INIT MODE** |
| `CONFIGURED_NO_FLOWS` | → **Step 2: URL Check** |
| `HAS_WORKSPACE` | → **Step 10: UPDATE MODE** |

---

## Step 1: INIT MODE — Initialize Workspace

### 1.1 Framework Selection

Tell the user:
> "Welcome to webapp-testing! Setting up QA workspace for **[domain]**.
>
> How would you like your test cases organized?
>
> **1. Flow-based** *(recommended)*
>    `{workspace}/flows/F-001-login/`, `{workspace}/flows/F-002-checkout/`
>    Best for apps with clear user journeys.
>
> **2. Feature-based**
>    `{workspace}/features/auth/`, `{workspace}/features/dashboard/`
>    Best for large apps with many independent modules.
>
> **3. Risk-based**
>    `{workspace}/test-cases/P1-critical/`, `{workspace}/test-cases/P2-high/`
>    Best for regression suites or release deadlines."

Wait for choice. Default to **flow-based** if unclear.

### 1.2 Create Directory Structure

```bash
mkdir -p {workspace}/planning
mkdir -p {workspace}/knowledgebase/screenshots
mkdir -p {workspace}/evidence
```

For flow-based:    `mkdir -p {workspace}/flows`
For feature-based: `mkdir -p {workspace}/features`
For risk-based:    `mkdir -p {workspace}/test-cases/P1-critical {workspace}/test-cases/P2-high {workspace}/test-cases/P3-medium`

### 1.3 Write `{workspace}/.qa-config.json`

```json
{
  "version": "1.0",
  "framework": "flow-based",
  "base_url": "USER_URL",
  "app_name": null,
  "domain_slug": "DOMAIN_SLUG",
  "workspace": "qa-web/DOMAIN_SLUG",
  "created": "YYYY-MM-DD",
  "last_discovery": null,
  "flows_count": 0,
  "test_cases_count": 0,
  "tests_passed": 0,
  "tests_failed": 0
}
```

Proceed to **Step 2**.

---

## Step 2: Connectivity Check

### 2.1 Verify Playwright is Installed

```bash
python3 -c "from playwright.sync_api import sync_playwright; print('ready')" 2>/dev/null || echo "not_installed"
```

If not installed: `pip install playwright && python3 -m playwright install chromium`

### 2.2 Check URL is Reachable

```python
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True, args=['--no-sandbox'])
    context = browser.new_context(
        user_agent='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
    )
    page = context.new_page()
    try:
        page.goto('BASE_URL', timeout=30000, wait_until='domcontentloaded')
        print("✅ reachable:", page.title())
    except Exception as e:
        print("❌ unreachable:", e)
    browser.close()
```

If unreachable, tell the user and ask for a different URL.

Update `{workspace}/.qa-config.json` with `app_name` from page title.
Write `{workspace}/planning/scope.md` with URL, date, environment type.

Proceed to **Step 3**.

---

## Step 3: Prior Knowledge Interview

Ask the user:
> "Do you have existing knowledge about **[domain]** I should start from?
> - Key flows (login → dashboard → checkout)
> - Features to prioritize or skip
> - Test credentials needed?
> - Known bugs?
>
> Say **'no'** and I'll discover everything visually."

If credentials needed — store in `.env.qa` (gitignored), use `os.getenv()` in scripts. Never hardcode.

Proceed to **Step 4**.

---

## Step 4: Visual Discovery — Homepage

### 4.1 Take Screenshot and Collect UI Inventory

```python
from playwright.sync_api import sync_playwright
import json, os

WORKSPACE = "{workspace}"  # replace with actual derived workspace
BASE_URL = "USER_URL"

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True, args=['--no-sandbox'])
    context = browser.new_context(
        viewport={"width": 1440, "height": 900},
        user_agent='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
    )
    page = context.new_page()
    console_errors = []
    page.on("console", lambda msg: console_errors.append(msg.text) if msg.type == "error" else None)

    page.goto(BASE_URL, wait_until='domcontentloaded', timeout=30000)
    page.wait_for_timeout(3000)
    page.screenshot(path=f'{WORKSPACE}/knowledgebase/screenshots/01-homepage.png', full_page=True)

    nav_links = []
    for a in page.locator('nav a, header a, [role="navigation"] a').all()[:30]:
        text = a.inner_text().strip()
        href = a.get_attribute('href') or ''
        if text and len(text) < 50:
            nav_links.append({'text': text, 'href': href})

    inputs = []
    for inp in page.locator('input, textarea, select').all()[:20]:
        inputs.append({'type': inp.get_attribute('type') or 'text', 'name': inp.get_attribute('name') or '', 'id': inp.get_attribute('id') or ''})

    buttons = [b.inner_text().strip() for b in page.locator('button, input[type="submit"]').all()[:15] if b.inner_text().strip()]
    headings = [h.inner_text().strip() for h in page.locator('h1, h2').all()[:10] if h.inner_text().strip()]

    inventory = {'title': page.title(), 'url': page.url, 'nav_links': nav_links, 'inputs': inputs, 'buttons': buttons, 'headings': headings, 'console_errors': console_errors[:5]}
    with open(f'{WORKSPACE}/knowledgebase/ui-inventory.json', 'w') as f:
        json.dump(inventory, f, indent=2)

    print(json.dumps(inventory, indent=2))
    browser.close()
```

### 4.2 Visual Analysis

**USE the Read tool** on `{workspace}/knowledgebase/screenshots/01-homepage.png`.

Analyze: layout, navigation, forms, CTAs, app type. Tell user what you found.

Proceed to **Step 5**.

---

## Step 5: Deep Exploration — Navigate All Pages

For each nav link discovered:

```python
page.goto(absolute_url, wait_until='domcontentloaded')
page.wait_for_timeout(2000)
page.screenshot(path=f'{WORKSPACE}/knowledgebase/screenshots/{slug}.png', full_page=False)
```

**USE the Read tool** on each screenshot. Document what each page does.

**Always explore:**
- Homepage, Login/Signup, Dashboard (if auth needed), Search results, 404 page, any nav links

After exploring, present page inventory table to user and ask 2-3 clarifying questions.

Proceed to **Step 6**.

---

## Step 6: Flow Mapping

### Universal flows (every app):
- `F-001-homepage-load`
- `F-002-navigation`
- `F-003-responsive-layout`
- `F-004-404-error-handling`

### App-specific flows (from discovery):
- `F-005-user-login`, `F-006-search`, `F-007-signup`, `F-008-[core-feature]`, etc.

Create directories:
```bash
mkdir -p {workspace}/flows/F-NNN-[slug]/test-cases
```

Write `{workspace}/flows/F-NNN-[slug]/flow.md` for each (see Flow Template).

---

## Step 7: Scenario Generation

For each flow write `{workspace}/flows/F-NNN-[slug]/scenarios.md`.

| Category | Priority | When |
|----------|----------|------|
| Happy Path | P1 | Every flow |
| Negative / Invalid Input | P1 | Forms, search |
| Empty State | P1 | Lists, results |
| Unauthorized Access | P1 | Protected pages |
| Broken Links | P2 | Navigation |
| Mobile Responsive | P2 | All visual flows |
| Console Errors | P2 | All page loads |
| Boundary Values | P2 | Input fields |
| Cross-browser | P3 | Core flows |

---

## Step 8: Test Case Generation and Execution

For each scenario write **and immediately run** a test script.

### File placement:
`{workspace}/flows/F-NNN-[slug]/test-cases/TC-NNN-[slug].py`

### Test script template:

```python
"""
TC-NNN: [Title]
Flow: F-NNN | Priority: P1
"""
from playwright.sync_api import sync_playwright
import os

BASE_URL = "https://example.com"
WORKSPACE = "qa-web/domain-slug"
EVIDENCE_DIR = f"{WORKSPACE}/evidence"

def run_test():
    results = []
    with sync_playwright() as p:
        browser = p.chromium.launch(headless=True, args=['--no-sandbox', '--disable-blink-features=AutomationControlled'])
        context = browser.new_context(
            viewport={"width": 1440, "height": 900},
            user_agent='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36'
        )
        page = context.new_page()
        console_errors = []
        page.on("console", lambda msg: console_errors.append(msg.text) if msg.type == "error" else None)

        try:
            page.goto(BASE_URL, wait_until='domcontentloaded', timeout=30000)
            page.wait_for_timeout(2000)

            # ASSERTIONS
            if [condition]:
                results.append(("PASS", "[description]"))
            else:
                page.screenshot(path=f"{EVIDENCE_DIR}/TC-NNN-fail.png")
                results.append(("FAIL", "[description]"))

        except Exception as e:
            page.screenshot(path=f"{EVIDENCE_DIR}/TC-NNN-exception.png")
            results.append(("FAIL", f"Exception: {str(e)[:100]}"))

        if console_errors:
            results.append(("WARN", f"Console errors: {console_errors[:2]}"))

        browser.close()
    return results

if __name__ == "__main__":
    os.makedirs(EVIDENCE_DIR, exist_ok=True)
    results = run_test()
    passed = sum(1 for s, _ in results if s == "PASS")
    total = sum(1 for s, _ in results if s != "WARN")
    print(f"\nTC-NNN: {passed}/{total} passed")
    for status, msg in results:
        print(f"  {'✅' if status=='PASS' else '⚠️' if status=='WARN' else '❌'} {msg}")
```

### Run immediately after writing:
```bash
python3 {workspace}/flows/F-NNN-[slug]/test-cases/TC-NNN-[slug].py
```

If FAIL → fix → re-run before moving to next TC.

### Responsive check (add to every visual TC):
```python
for name, w, h in [("mobile", 375, 812), ("tablet", 768, 1024), ("desktop", 1440, 900)]:
    ctx = browser.new_context(viewport={"width": w, "height": h}, user_agent='...')
    pg = ctx.new_page()
    pg.goto(BASE_URL, wait_until='domcontentloaded')
    pg.screenshot(path=f"{EVIDENCE_DIR}/TC-NNN-{name}.png")
    ctx.close()
```

---

## Step 9: Finalize and Report

### Run all tests:
```bash
find {workspace} -name "TC-*.py" | sort | while read f; do python3 "$f"; done
```

### Update config:
```python
import json, glob, os, datetime

with open(f"{WORKSPACE}/.qa-config.json") as f:
    config = json.load(f)

config["last_discovery"] = datetime.datetime.now().isoformat()
config["flows_count"] = len([d for d in glob.glob(f"{WORKSPACE}/flows/F-*") if os.path.isdir(d)])
config["test_cases_count"] = len(glob.glob(f"{WORKSPACE}/**/TC-*.py", recursive=True))

with open(f"{WORKSPACE}/.qa-config.json", "w") as f:
    json.dump(config, f, indent=2)
```

### Print summary:
```
╔══════════════════════════════════════════════════════════╗
  WEB QA COMPLETE — [App Name]
  URL: [base_url]
  Workspace: qa-web/[domain-slug]/
╠══════════════════════════════════════════════════════════╣
  Flows: [N] | Test Scripts: [N]
  ✅ Passed: [N] | ❌ Failed: [N] | ⚠️ Warnings: [N]
  Evidence: {workspace}/evidence/
╚══════════════════════════════════════════════════════════╝
```

---

## Step 10: UPDATE MODE

```
Found existing QA workspace for [domain]:
  Workspace: qa-web/[domain-slug]/
  Flows: [N] | Test cases: [N] | Last run: [date]

1. Re-run all tests
2. Re-discover (detect UI changes)
3. Add new flow
4. Full refresh
```

---

## Key Rules

- **Domain slug = workspace** — always derive from URL, never use `qa-web/` directly
- **domcontentloaded + wait_for_timeout(2000)** — always wait before DOM inspection
- **Realistic user-agent** — many sites block headless browsers
- **Screenshot on FAIL** — save to `{workspace}/evidence/TC-NNN-fail.png`
- **Check console errors** — JS errors are bugs even if page looks fine
- **Run tests immediately** — write, run, fix before moving to next TC
- **Credentials in .env.qa only** — never hardcode

---

## Flow Template

```markdown
# F-[NNN]: [Flow Name]

| Field | Value |
|-------|-------|
| **Flow ID** | F-[NNN] |
| **URL(s)** | [URLs involved] |
| **Description** | [One sentence goal] |
| **Start State** | [e.g., "Not logged in, on homepage"] |
| **End State** | [e.g., "Logged in, on dashboard"] |
| **Priority** | P1 / P2 / P3 |
| **Auth Required** | Yes / No |
| **Created** | [YYYY-MM-DD] |

## Steps
| Step | Action | Expected Result |
|------|--------|-----------------|
| 1 | [Action] | [Result] |

## Key Selectors
| Element | Selector | Type |
|---------|----------|------|
| [Name] | `#id` | input/button |

## Evidence
`{workspace}/knowledgebase/screenshots/0N-[slug].png`
```

---

## Scenarios Template

```markdown
# Scenarios — F-[NNN]: [Flow Name]
Total: [N] | Date: [YYYY-MM-DD]

## S-01: Happy Path
| Priority | P1 |
| Steps | [1-line] |
| Expected | [outcome] |
| Script | [TC-NNN.py](test-cases/TC-NNN.py) |

## Coverage
| Category | Count |
|----------|-------|
| Happy Path | [N] |
| Negative | [N] |
| Responsive | [N] |
| **Total** | **[N]** |
```
