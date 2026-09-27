---
name: webextension-testing
description: >
  Autonomous QA skill for browser extensions (Chrome, Firefox, Edge). Use this skill when the user wants to QA test any browser extension or web extension. Triggers on: "test my Chrome extension", "QA my browser extension", "test this extension", "write test cases for my web extension", "run webextension-testing", "test manifest.json app", or any mention of testing a browser plugin or add-on. The skill loads the extension in a browser via Playwright, takes screenshots, discovers all popup/options/content-script flows, generates comprehensive test scenarios, and writes runnable Playwright tests with chrome.extension APIs.
---

# WebExtension QA Skill

An autonomous QA engineer for **Chrome / Firefox / Edge browser extensions**. Loads your unpacked extension, screenshots every surface (popup, options page, content scripts), maps flows, and generates full test coverage.

**Supported**: Chrome (MV2/MV3) ✅ | Firefox WebExtension ✅ | Edge ✅

**Requires**: Node.js, Playwright (`npx playwright install chromium`)

---

## Step 0: Mode Detection

```bash
if [ ! -d "qa-ext" ] || [ ! -f "qa-ext/.qa-config.json" ]; then
  echo "INIT"
elif [ ! -d "qa-ext/flows" ]; then
  echo "CONFIGURED_NO_FLOWS"
else
  echo "HAS_WORKSPACE"
fi
```

| Result | Action |
|--------|--------|
| `INIT` | → **Step 1: INIT MODE** |
| `CONFIGURED_NO_FLOWS` | → **Step 2: Extension Selection** |
| `HAS_WORKSPACE` | → **Step 9: UPDATE MODE** |

---

## Step 1: INIT MODE — Initialize QA Workspace

### 1.1 Framework Selection

Tell the user:
> "Welcome to WebExtension QA! Let me set up your extension testing workspace.
>
> How would you like your test cases organized?
>
> **1. Flow-based** *(recommended)*
>    `qa-ext/flows/F-001-popup/`, `qa-ext/flows/F-002-options/`, etc.
>
> **2. Feature-based**
>    `qa-ext/features/popup/`, `qa-ext/features/content-script/`, etc.
>
> **3. Risk-based**
>    `qa-ext/test-cases/P1-critical/`, `qa-ext/test-cases/P2-high/`, etc."

Wait for choice. Default to **flow-based** if unclear.

### 1.2 Create Directory Structure

```bash
mkdir -p qa-ext/planning qa-ext/guardrails qa-ext/credentials qa-ext/scope
mkdir -p qa-ext/knowledgebase/screenshots qa-ext/flows qa-ext/evidence
```

### 1.3 Write `.qa-config.json`

```json
{
  "version": "1.0",
  "skill": "webextension-testing",
  "framework": "flow-based",
  "extension_name": null,
  "extension_path": null,
  "manifest_version": null,
  "browser": "chromium",
  "created": "YYYY-MM-DD",
  "last_discovery": null,
  "flows_count": 0,
  "test_cases_count": 0
}
```

---

## Step 2: Extension Selection

Ask the user:
> "What extension are you testing?
> - Provide the **path to your unpacked extension folder** (contains `manifest.json`)
>   Example: `./my-extension/` or `/Users/you/projects/my-ext/dist/`
> - Or a **CRX file path** if already packaged
>
> Which browser? (chromium / firefox / edge) — default: chromium"

### 2.1 Read manifest.json

```bash
cat [extension_path]/manifest.json
```

Extract:
- `name` → extension name
- `manifest_version` → MV2 or MV3
- `permissions` → what APIs it uses (storage, tabs, scripting, etc.)
- `action` / `browser_action` → popup HTML path
- `options_page` / `options_ui` → options page
- `content_scripts` → content script matches
- `background` → service worker / background script

Tell user the summary:
> "Found **[ExtensionName]** (Manifest V[2/3])
> - Permissions: [list]
> - Surfaces: [popup / options page / content scripts on: URLs]
> - Background: [service worker / background page]"

### 2.2 Update config

Update `qa-ext/.qa-config.json` with all extension metadata.

---

## Step 3: Prior Knowledge Interview

Ask:
> "Do you have existing knowledge about **[ExtensionName]** I should start from?
>
> For example:
> - Known flows (e.g., 'user clicks icon, popup opens, they toggle feature')
> - Features to prioritize or skip
> - Known bugs or edge cases
>
> Or say **'no'** and I'll discover by loading the extension visually."

---

## Step 4: Launch Extension and Take Screenshots

### 4.1 Create Playwright launch script

Write `qa-ext/scripts/launch-ext.js`:

```javascript
const { chromium } = require('playwright');
const path = require('path');

(async () => {
  const extensionPath = path.resolve(process.argv[2]);
  const outputDir = process.argv[3] || 'qa-ext/knowledgebase/screenshots';

  const context = await chromium.launchPersistentContext('', {
    headless: false,
    args: [
      `--disable-extensions-except=${extensionPath}`,
      `--load-extension=${extensionPath}`,
    ],
  });

  // Get extension ID
  let [background] = context.serviceWorkers();
  if (!background) {
    background = await context.waitForEvent('serviceworker');
  }
  const extensionId = background.url().split('/')[2];
  console.log('Extension ID:', extensionId);

  // Screenshot popup
  const popupPage = await context.newPage();
  await popupPage.goto(`chrome-extension://${extensionId}/popup.html`);
  await popupPage.waitForLoadState('networkidle');
  await popupPage.screenshot({ path: `${outputDir}/01-popup.png`, fullPage: true });

  // Screenshot options page if exists
  try {
    const optionsPage = await context.newPage();
    await optionsPage.goto(`chrome-extension://${extensionId}/options.html`);
    await optionsPage.waitForLoadState('networkidle');
    await optionsPage.screenshot({ path: `${outputDir}/02-options.png`, fullPage: true });
  } catch (e) { console.log('No options page found'); }

  // Screenshot on a real page (content script)
  const contentPage = await context.newPage();
  await contentPage.goto('https://example.com');
  await contentPage.waitForLoadState('networkidle');
  await contentPage.screenshot({ path: `${outputDir}/03-content-script.png`, fullPage: true });

  console.log('Screenshots saved to', outputDir);
  await context.close();
})();
```

### 4.2 Run it

```bash
node qa-ext/scripts/launch-ext.js [extension_path] qa-ext/knowledgebase/screenshots
```

### 4.3 Visual Analysis

**USE the Read tool** on each screenshot. Analyze:
- Popup layout, buttons, toggles, inputs
- Options page sections
- Content script overlay/injection on pages

Tell user:
> "I can see **[ExtensionName]**. Here's my analysis:
> - **Popup**: [description of UI elements]
> - **Options page**: [found/not found — description]
> - **Content script**: [visible on pages / not visible]
> - **Flows to explore**: [N] identified — [names]"

---

## Step 5: Deep Exploration — Navigate Every Surface

### 5.1 Extension Surfaces to Cover

| Surface | How to Access | What to Test |
|---------|--------------|-------------|
| Popup | `chrome-extension://[id]/popup.html` | All buttons, toggles, inputs |
| Options page | `chrome-extension://[id]/options.html` | All settings, save/reset |
| Content script | Load on matching URLs | Injection, UI overlay, DOM manipulation |
| Background/SW | DevTools → Service Workers | Message handling, storage |
| Context menu | Right-click on page | If extension adds context menu items |
| New tab override | Open new tab | If extension overrides new tab page |

### 5.2 For each surface — Screenshot + Analyze

For every interactive element found:

```javascript
// Get all interactive elements
const elements = await page.$$eval(
  'button, input, select, a, [role="button"], [role="checkbox"], [role="switch"]',
  els => els.map(el => ({
    tag: el.tagName,
    text: el.textContent?.trim(),
    type: el.type,
    id: el.id,
    class: el.className,
    bounds: el.getBoundingClientRect()
  }))
);
```

Screenshot after each interaction.

### 5.3 Compile Extension Inventory

Write `qa-ext/knowledgebase/extension-inventory.md`:

```markdown
# Extension Inventory — [ExtensionName]

## Metadata
- Name: [name]
- Version: [version]
- Manifest: V[2/3]
- Permissions: [list]

## Surfaces
| Surface | URL | Elements Found | Screenshots |
|---------|-----|---------------|------------|
| Popup | chrome-extension://[id]/popup.html | [N] | 01-popup.png |
| Options | chrome-extension://[id]/options.html | [N] | 02-options.png |
| Content Script | Injected on [URL patterns] | [N] | 03-content-script.png |

## Flows Identified
1. [Flow name] — [description]
2. ...
```

---

## Step 6: Flow Creation

### 6.1 Universal Extension Flows

- `F-001-extension-install` — Install, permissions grant, first launch
- `F-002-popup-main` — Open popup, primary action
- `F-003-options-settings` — Open options, change settings, persist
- `F-004-content-script` — Navigate to matching URL, verify injection
- `F-005-enable-disable` — Disable extension, verify content removed, re-enable
- `F-006-storage-persistence` — Set preference, reload browser, verify persisted
- `F-007-permissions` — Permission prompts, grant/deny behavior

### 6.2 App-Specific Flows (from discovery)

Add flows based on what the extension actually does.

### 6.3 Create flow directories and write `flow.md`

```bash
mkdir -p "qa-ext/flows/F-NNN-[slug]/test-cases"
```

---

## Step 7: Scenario Generation

### 7.1 Extension-Specific Scenario Categories

| Category | Min | Priority |
|----------|-----|----------|
| Happy Path | 1 | P1 |
| Popup interaction | 2+ | P1 |
| Options save/reset | 2 | P1 |
| Content script on/off | 2 | P1 |
| Extension disable/enable | 1 | P1 |
| Storage persistence (reload) | 1 | P2 |
| Incognito mode | 1 | P2 |
| Multiple tabs | 1 | P2 |
| Slow network | 1 | P2 |
| Permission denied | 1 | P2 |
| Browser restart persistence | 1 | P2 |
| Cross-browser compatibility | 1 | P3 |

### 7.2 Write `scenarios.md`

Write `qa-ext/flows/F-NNN-[slug]/scenarios.md`.

---

## Step 8: Test Case Generation

### 8.1 Test Case Template

Each `TC-NNN-*.md` must have:
- Metadata: TC ID, flow, priority, browser, manifest version, extension version
- Preconditions: extension loaded state, browser state
- Playwright script block (runnable)
- Pass criteria checklist
- Evidence path: `qa-ext/evidence/TC-NNN-S1.png`

### 8.2 Playwright Test Structure

```javascript
// TC-NNN-[slug].spec.js
const { test, expect, chromium } = require('@playwright/test');
const path = require('path');

const EXTENSION_PATH = path.resolve('./[extension_path]');

test.describe('TC-NNN: [Test Name]', () => {
  let context, extensionId;

  test.beforeAll(async () => {
    context = await chromium.launchPersistentContext('', {
      headless: false,
      args: [
        `--disable-extensions-except=${EXTENSION_PATH}`,
        `--load-extension=${EXTENSION_PATH}`,
      ],
    });
    let [background] = context.serviceWorkers();
    if (!background) background = await context.waitForEvent('serviceworker');
    extensionId = background.url().split('/')[2];
  });

  test.afterAll(async () => { await context.close(); });

  test('[scenario description]', async () => {
    const page = await context.newPage();
    await page.goto(`chrome-extension://${extensionId}/popup.html`);

    // [Test steps]
    await page.screenshot({ path: 'qa-ext/evidence/TC-NNN-S1.png' });

    // Assertions
    await expect(page.locator('[data-testid="result"]')).toBeVisible();
  });
});
```

---

## Step 9: Finalize and Summary

### 9.1 Update config

```python
import json, datetime, glob, os

with open("qa-ext/.qa-config.json") as f:
    config = json.load(f)

flows = [d for d in os.listdir("qa-ext/flows") if d.startswith("F-")] if os.path.isdir("qa-ext/flows") else []
tcs = glob.glob("qa-ext/**/TC-*.md", recursive=True)

config["last_discovery"] = datetime.datetime.now().isoformat()
config["flows_count"] = len(flows)
config["test_cases_count"] = len(tcs)

with open("qa-ext/.qa-config.json", "w") as f:
    json.dump(config, f, indent=2)
```

### 9.2 Print Summary

```
╔════════════════════════════════════════════════════════════════╗
  WebExtension QA Workspace Ready — [ExtensionName]
╠════════════════════════════════════════════════════════════════╣
  Extension:       [name] v[version] (MV[2/3])
  Browser:         [chromium / firefox / edge]
  Surfaces:        [N] (popup, options, content-script, ...)
  Permissions:     [list]

  Discovery:
    Screenshots:   [N] in qa-ext/knowledgebase/screenshots/
    Elements:      [N] enumerated

  Flows Created:   [N]
  Scenarios:       [N total] ([N] P1, [N] P2, [N] P3)
  Test Cases:      [N files]
  Playwright Tests: [N] .spec.js files ready to run
╚════════════════════════════════════════════════════════════════╝

Run tests:
  npx playwright test qa-ext/
```

---

## Step 10: UPDATE MODE

When `HAS_WORKSPACE` detected:

```bash
cat qa-ext/.qa-config.json
ls qa-ext/flows/ | wc -l
find qa-ext -name "TC-*.md" | wc -l
```

> "Found existing extension QA workspace for **[ExtensionName]**:
> - Flows: [N] | Test cases: [N] | Last discovery: [date]
>
> What would you like to do?
> 1. **Re-discover** — Reload extension, take new screenshots, detect UI changes
> 2. **Add flow** — New feature coverage
> 3. **Add scenarios** — More scenarios to an existing flow
> 4. **Full refresh** — Regenerate everything"
