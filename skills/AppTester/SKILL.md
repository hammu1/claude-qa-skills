---
name: AppTester
description: >
  Autonomous QA skill for Android and iOS native apps using @mobilenext/mobile-mcp. Use this skill when the user wants to QA test any Android or iOS application. Triggers on: "test my Android app", "QA my iOS app", "run AppTester", "test this APK", "generate test cases for Android", "write test scenarios for my mobile app", "init mobile QA", or any mention of testing a native Android or iOS application. The skill detects available devices, launches the target app, takes screenshots for visual LLM analysis, discovers all UI screens and flows using mobile-mcp tools, maps every screen into named flows, generates maximum test scenarios per flow, and creates detailed mobile test case files.
---

# AppTester — Mobile QA Skill

An autonomous QA engineer for **Android and iOS apps**. Uses `@mobilenext/mobile-mcp` to launch your app, take screenshots, enumerate UI elements, and generate comprehensive test cases.

**Supported platforms**: Android ✅ | iOS ✅ | macOS → use `native-qa` skill instead

**Requires**: `.mcp.json` with `@mobilenext/mobile-mcp` configured (already present in this repo).

---

## Step 0: Mode Detection

**First action every time** — determine which mode to run:

```bash
if [ ! -d "qa" ] || [ ! -f "qa/.qa-config.json" ]; then
  echo "INIT"
elif [ ! -d "qa/flows" ] && [ ! -d "qa/features" ] && [ ! -d "qa/test-cases/P1-critical" ]; then
  echo "CONFIGURED_NO_FLOWS"
else
  echo "HAS_WORKSPACE"
fi
```

| Result | Action |
|--------|--------|
| `INIT` | → **Step 1: INIT MODE** |
| `CONFIGURED_NO_FLOWS` | → **Step 2: App Selection** |
| `HAS_WORKSPACE` | → **Step 10: UPDATE MODE** |

---

## Step 1: INIT MODE — Initialize QA Workspace

### 1.1 Framework Selection

Tell the user:
> "Welcome to AppTester! Let me set up a mobile QA workspace.
>
> How would you like your test cases organized?
>
> **1. Flow-based** *(recommended)*
>    One directory per user journey: `qa/flows/F-001-login/`, `qa/flows/F-002-home/`, etc.
>
> **2. Feature-based**
>    One directory per feature: `qa/features/authentication/`, `qa/features/dashboard/`, etc.
>
> **3. Risk-based**
>    By severity: `qa/test-cases/P1-critical/`, `qa/test-cases/P2-high/`, etc."

Wait for choice. Default to **flow-based** if unclear.

### 1.2 Create Directory Structure

```bash
mkdir -p qa/planning qa/guardrails qa/credentials qa/scope
mkdir -p qa/knowledgebase/screenshots
```

Flow-based: `mkdir -p qa/flows`
Feature-based: `mkdir -p qa/features`
Risk-based:
```bash
mkdir -p qa/test-cases/P1-critical qa/test-cases/P2-high
mkdir -p qa/test-cases/P3-medium qa/test-cases/P4-low
```

### 1.3 Write `.qa-config.json`

```json
{
  "version": "1.0",
  "skill": "AppTester",
  "framework": "flow-based",
  "app_name": null,
  "app_package": null,
  "app_version": null,
  "platform": null,
  "device_id": null,
  "os_version": null,
  "created": "YYYY-MM-DD",
  "last_discovery": null,
  "flows_count": 0,
  "test_cases_count": 0
}
```

Proceed to **Step 2**.

---

## Step 2: Platform & App Selection

### 2.1 Ask platform

> "Which platform are you testing?
> 1. Android (emulator or physical device)
> 2. iOS (simulator or physical device)"

### 2.2 List available devices

Call:
```
mobile_list_available_devices
```

Show the list to the user. Ask them to confirm which device to use.

If no devices found:

**Android — start emulator:**
```bash
# List available AVDs
emulator -list-avds

# Start one
emulator -avd Pixel_7_API_34 &
sleep 15

# Verify
adb devices
```

**iOS — start simulator:**
```bash
open -a Simulator
sleep 10
xcrun simctl list devices | grep Booted
```

### 2.3 Ask for app details

> "What app would you like to test?
> - Android: provide the package name (e.g., `com.example.myapp`) or APK path
> - iOS: provide the bundle ID (e.g., `com.example.MyApp`) or .app path"

### 2.4 Get device screen size

```
mobile_get_screen_size
  deviceId: "[selected device]"
```

Store this for coordinate calculations in test cases.

### 2.5 Update config and write planning files

Update `qa/.qa-config.json` with `app_name`, `app_package`, `platform`, `device_id`, `os_version`.

Write planning files (see Templates section at end).

Proceed to **Step 3**.

---

## Step 3: Prior Knowledge Interview

Ask the user:
> "Do you have existing knowledge about **[AppName]** I should start from?
>
> For example:
> - Known screens or user flows (e.g., 'users open app, login, see dashboard')
> - Features to prioritize or skip
> - Known bugs or edge cases
>
> Or say **'no'** and I'll discover by exploring the app visually."

If prior knowledge provided → extract flows/screens, store for Step 6.
If none → "Got it — I'll explore [AppName] visually."

Proceed to **Step 4**.

---

## Step 4: Launch App and Take First Screenshot

### 4.1 Launch the app

```
mobile_launch_app
  deviceId: "[device_id]"
  packageName: "[app_package]"    # Android
  # bundleId: "[bundle_id]"       # iOS
```

Wait 3 seconds for app to load.

### 4.2 Capture main screen

```
mobile_save_screenshot
  deviceId: "[device_id]"
  outputPath: "qa/knowledgebase/screenshots/01-main-screen.png"
```

### 4.3 Visual Analysis

**USE the Read tool** to open `qa/knowledgebase/screenshots/01-main-screen.png`.

Analyze and tell the user:
> "I can see [AppName]'s main screen. Here's my initial analysis:
> - **Layout**: [bottom nav / tab bar / drawer / single screen / etc.]
> - **Navigation areas**: [list with names]
> - **Primary actions**: [list key buttons/controls]
> - **App type**: [inferred — social, utility, e-commerce, VPN, etc.]
> - **Screens to explore**: [N] areas identified — [names]"

Proceed to **Step 5**.

---

## Step 5: Deep Exploration — Navigate, Screenshot, Analyze

For each major screen/section identified in Step 4:

### 5.1 Get UI element list

```
mobile_list_elements_on_screen
  deviceId: "[device_id]"
```

Parse the output to find:
- Navigation elements (bottom bar items, tabs, drawer items)
- Primary action buttons
- Text fields and forms
- Lists and scrollable areas

### 5.2 Deep Navigation Loop

For each navigation item found:

**Tap the navigation element** (use bounds from element list — center = x + width/2, y + height/2):
```
mobile_click_on_screen_at_coordinates
  deviceId: "[device_id]"
  x: [center_x]
  y: [center_y]
```

**Wait 1 second**, then screenshot:
```
mobile_save_screenshot
  deviceId: "[device_id]"
  outputPath: "qa/knowledgebase/screenshots/0N-[screen-slug].png"
```

**USE the Read tool** on the screenshot. Analyze:
- What does this screen show?
- What user actions are available?
- Are there sub-screens or nested navigation?
- What data is visible?
- Any forms, lists, cards, toggles?

**Get elements for this screen:**
```
mobile_list_elements_on_screen
  deviceId: "[device_id]"
```

Repeat for: Settings screen, any tab or drawer items, modals reachable from the main view.

### 5.3 Scroll and discover hidden content

For screens with scrollable content:
```
mobile_swipe_on_screen
  deviceId: "[device_id]"
  startX: [screen_center_x]
  startY: [screen_height * 0.75]
  endX: [screen_center_x]
  endY: [screen_height * 0.25]
  duration: 500
```

Screenshot after each scroll to discover more elements.

### 5.4 Compile Screen Inventory

After exploring all screens, present to user:

> "I've explored [AppName] and found **[N] screens**:
>
> | # | Screen | Type | Key Elements |
> |---|--------|------|-------------|
> | 1 | [Name] | [nav/feature/settings/auth] | [N buttons, N fields] |
> ...
>
> Quick question about each significant screen before I map flows."

### 5.5 Clarifying Questions

Batch 2-3 screens per message:
> "Quick questions:
> **[Screen A]**: What's the main user goal here? Any states I should know?
> **[Screen B]**: Are there multiple ways users reach this screen?"

---

## Step 6: Flow Creation

Map discovered screens to named flows.

### 6.1 Identify Distinct Flows

**Universal flows** (every app):
- `F-001-app-launch-and-startup` — Cold launch, splash, first screen
- `F-002-navigation` — Bottom nav / tab bar / drawer navigation
- `F-003-settings` — Open settings, change preferences, persist
- `F-004-app-backgrounding` — Background and foreground, state preserved

**App-specific flows** (from discovery):
- `F-005-[primary-feature]` — The app's core value action
- `F-006-[secondary-feature]`
- `F-00N-[screen-slug]`

### 6.2 Create Flow Directory

Flow-based: `mkdir -p "qa/flows/F-NNN-[slug]/test-cases"`
Feature-based: `mkdir -p "qa/features/[name]/test-cases"`

### 6.3 Write `flow.md`

Use the **Flow Template** from the Templates section.

---

## Step 7: Scenario Generation

For each flow, generate **maximum coverage** test scenarios.

### 7.1 Scenario Categories

| Category | Min | Priority | When |
|----------|-----|----------|------|
| Happy Path | 1 | P1 | Every flow |
| Alternative Happy Path | 1+ | P1 | Multiple valid paths |
| Negative / Invalid Input | 2+ | P1 | Any input flow |
| Empty / Null Input | 1 | P1 | Required fields |
| Boundary Values | 2 | P2 | Length/range fields |
| State Persistence | 1 | P2 | App backgrounded/killed |
| Interrupted Flow | 1 | P2 | Multi-step flows |
| No Network | 1 | P2 | Network-dependent flows |
| Permission Denied | 1 | P2 | Camera, location, etc. |
| Orientation Change | 1 | P2 | Portrait ↔ Landscape |
| Back Navigation | 1 | P2 | All screens |
| Accessibility | 1 | P3 | All flows |

### 7.2 Scenario Count Targets

| Flow Complexity | Min Scenarios |
|----------------|---------------|
| Simple (launch, nav) | 3–5 |
| Medium (settings, lists) | 6–10 |
| Complex (auth, core feature) | 10–20 |

### 7.3 Write `scenarios.md`

Write `qa/flows/F-NNN-[slug]/scenarios.md` using the Scenarios Template.

---

## Step 8: Test Case Generation

For each scenario, write a full test case file.

### 8.1 File Placement

Flow-based: `qa/flows/F-NNN-[slug]/test-cases/TC-NNN-[scenario-slug].md`
Feature-based: `qa/features/[name]/test-cases/TC-NNN-[scenario-slug].md`
Risk-based: `qa/test-cases/P[1-4]-[name]/TC-NNN-[scenario-slug].md`

TC numbers are globally sequential. Zero-pad to 3 digits: TC-001, TC-002…

### 8.2 Quality Standards

Each `TC-NNN-*.md` must have:
- **Metadata table**: TC ID, flow, priority, platform, OS version, device, automation method
- **Preconditions**: specific app/device state required
- **Setup commands**: mobile-mcp calls to establish preconditions
- **Steps table**: Step | Action | mobile-mcp Command | Expected Result
- **mobile-mcp block**: full runnable command sequence for happy path
- **Pass criteria checklist**: binary observable outcomes
- **Evidence path**: `qa/evidence/TC-NNN-S1.png`
- **Teardown**: restore device state after test

### 8.3 mobile-mcp Command Standards

- Always save a screenshot after each significant action for visual verification
- Use `mobile_list_elements_on_screen` before tapping to get exact coordinates
- Pattern for every interaction:
  ```
  1. mobile_list_elements_on_screen  ← find element bounds
  2. mobile_click_on_screen_at_coordinates  ← tap
  3. mobile_save_screenshot  ← verify state
  4. Read tool on screenshot  ← visual assertion
  ```
- Include pass/fail assertions by comparing screenshot content visually

Use `templates/test-case-mobile.md` as the source template.

---

## Step 9: Finalize and Update Config

### 9.1 Update `qa/.qa-config.json`

```bash
python3 - <<'PYEOF'
import json, datetime, os, glob

with open("qa/.qa-config.json", "r") as f:
    config = json.load(f)

flows = [d for d in os.listdir("qa/flows") if d.startswith("F-")] if os.path.isdir("qa/flows") else []
tcs = glob.glob("qa/**/TC-*.md", recursive=True)

config["last_discovery"] = datetime.datetime.now().isoformat()
config["flows_count"] = len(flows)
config["test_cases_count"] = len(tcs)

with open("qa/.qa-config.json", "w") as f:
    json.dump(config, f, indent=2)

print(f"Updated: {len(flows)} flows, {len(tcs)} test cases")
PYEOF
```

### 9.2 Print Summary

```
╔═══════════════════════════════════════════════════════════════╗
  AppTester QA Workspace Ready — [AppName] on [Platform]
╠═══════════════════════════════════════════════════════════════╣
  Framework:       [flow-based / feature-based / risk-based]
  App:             [AppName] ([package/bundle-id])
  Platform:        [Android / iOS]
  Device:          [device_id]
  OS Version:      [version]

  Discovery:
    Screenshots:   [N] in qa/knowledgebase/screenshots/
    Screens:       [N] explored
    UI Elements:   [N] enumerated

  Flows Created:   [N]
  Scenarios:       [N total] ([N] P1, [N] P2, [N] P3)
  Test Cases:      [N files]
╚═══════════════════════════════════════════════════════════════╝
```

---

## Step 10: UPDATE MODE

When `HAS_WORKSPACE` detected.

```bash
cat qa/.qa-config.json
ls qa/flows/ 2>/dev/null | wc -l
find qa -name "TC-*.md" 2>/dev/null | wc -l
```

Tell user:
> "Found existing QA workspace for **[AppName]** ([platform]):
> - Flows: [N] | Test cases: [N] | Last discovery: [date]
>
> What would you like to do?
> 1. **Re-discover** — Relaunch app, take new screenshots, detect UI changes
> 2. **Add flow** — Add coverage for a new feature
> 3. **Add scenarios** — Add more scenarios to an existing flow
> 4. **Full refresh** — Regenerate everything from scratch"

---

## Templates

### `qa/planning/platforms.md`

```markdown
# Testing Platforms & Environments

## Application Under Test
- **App Name**: [AppName]
- **Package / Bundle ID**: [com.example.myapp]
- **Version**: [version]
- **Platform**: [Android / iOS]

## Device Environment
- **Device**: [Emulator name / Physical device model]
- **Device ID**: [emulator-5554 / iPhone 15 Pro]
- **OS Version**: [Android 14 / iOS 17.4]
- **Screen Size**: [width x height px]

## Test Environments
| Environment | Details | Notes |
|-------------|---------|-------|
| Android Emulator | Pixel 7 API 34 | Primary |
| iOS Simulator | iPhone 15 iOS 17 | Primary |

## QA Framework
- **Organization**: [flow-based / feature-based / risk-based]
- **Automation Method**: @mobilenext/mobile-mcp via Claude MCP
- **Test Case Format**: Markdown + mobile-mcp commands

## Last Updated
- **Date**: [date]
- **Updated by**: AppTester QA Agent
```

---

### `qa/guardrails/do-and-dont.md`

```markdown
# QA Guardrails — [AppName] Mobile

## ✅ DO

- Use dedicated test accounts — never personal or production accounts
- Terminate and relaunch app before each test scenario for clean state
- Call `mobile_list_elements_on_screen` before every tap to get current coordinates
- Save a screenshot after every significant action for evidence
- Re-enable any changed device settings (Wi-Fi, GPS) after each test
- Test both portrait and landscape orientation where relevant
- Verify element is enabled before tapping

## ❌ DO NOT

- Never hardcode coordinates — always get fresh bounds from `mobile_list_elements_on_screen`
- Never use real payment methods or production accounts
- Never commit credentials, tokens, or API keys to tracked files
- Never leave device in modified state (Wi-Fi off, GPS off) after test
- Never assume app state — always verify with a screenshot

## ⚠️ Important Cautions

- **Permission dialogs** — Camera, location, notification dialogs may appear on first launch; handle in test preconditions
- **OS-level interruptions** — Calls, notifications may interrupt tests; document as known interference
- **Emulator vs real device** — Some features (camera, GPS, biometric) behave differently on emulators
- **Deep links** — Use `mobile_open_url` for deep link testing, not manual navigation
```

---

### `qa/credentials/access.md`

```markdown
# Credentials & Access

> ⚠️ Structure only. Never store real values here. Use `.env.qa` (gitignored).

## Required Test Accounts

| Role | Purpose | How to Obtain |
|------|---------|--------------|
| Standard user | Core functional testing | Dedicated test account |
| Premium user | Premium feature testing | Test subscription |
| New user | Onboarding flows | Fresh account |

## `.env.qa` Structure

```env
QA_APP_NAME=[AppName]
QA_PLATFORM=android|ios
QA_DEVICE_ID=
QA_USERNAME=
QA_PASSWORD=
QA_TEST_EMAIL=
QA_ACCOUNT_TIER=free|premium
QA_SANDBOX_MODE=true
```

## Device Prerequisites

- [ ] Android: USB Debugging enabled / Emulator booted
- [ ] iOS: Simulator booted / Developer mode enabled on device
- [ ] `.mcp.json` present with `@mobilenext/mobile-mcp` configured
- [ ] `npx @mobilenext/mobile-mcp` runs without error
```

---

### Flow Template

**File**: `qa/flows/F-NNN-[slug]/flow.md`

```markdown
# F-[NNN]: [Flow Name]

## Summary

| Field | Value |
|-------|-------|
| **Flow ID** | F-[NNN] |
| **Application** | [AppName] |
| **Platform** | Android / iOS / Both |
| **Description** | [One sentence: what goal does this flow accomplish?] |
| **Start State** | [App state before flow begins] |
| **End State** | [App state when flow completes successfully] |
| **Screen** | [e.g., Home / Login / Settings > Notifications] |
| **Priority** | P1 / P2 / P3 |
| **Discovered via** | mobile-mcp screenshot analysis + element enumeration |
| **Created** | [YYYY-MM-DD] |

---

## UI Elements Involved

| Element Type | Label / Description | Approx Bounds | Role in Flow |
|-------------|--------------------|----|---|
| button | "Connect" | x:100 y:400 w:200 h:60 | Initiates action |

---

## User Journey

### Preconditions
- [ ] App installed and launched
- [ ] [Required account state / permissions / network]

### Steps

| Step | User Action | mobile-mcp Command | Expected Result |
|------|------------|-------------------|----------------|
| 1 | Launch app | `mobile_launch_app` | Splash then home screen |
| 2 | [Action] | `mobile_click_on_screen_at_coordinates x=... y=...` | [Response] |

### Success Outcome
> [What the user sees when flow completes]

### Failure Outcomes

| What Goes Wrong | Expected App Behavior |
|----------------|----------------------|
| No network | Error message shown, app stable |

---

## Discovery Evidence

- Screenshot: `qa/knowledgebase/screenshots/0N-[screen-slug].png`

---

## Notes / Observations

- [Anything unusual — timing, animations, inconsistencies]
```

---

### Scenarios Template

**File**: `qa/flows/F-NNN-[slug]/scenarios.md`

```markdown
# Scenarios — F-[NNN]: [Flow Name]

**Flow**: F-[NNN]
**Application**: [AppName]
**Platform**: Android / iOS / Both
**Total scenarios**: [N]
**Generated**: [YYYY-MM-DD]

---

## S-[NNN]-01: Happy Path — [Short Description]

| Field | Value |
|-------|-------|
| **Priority** | P1 |
| **Category** | Happy Path |
| **Platform** | Android / iOS / Both |
| **Preconditions** | [Specific state] |
| **Steps summary** | [1-line description] |
| **Expected result** | [Success state] |
| **Test Case** | [TC-NNN-[slug].md](test-cases/TC-NNN-[slug].md) |

---

## Coverage Summary

| Category | Count | TC Files |
|----------|-------|---------|
| Happy Path | [N] | TC-NNN, ... |
| Negative / Invalid Input | [N] | TC-NNN, ... |
| State Persistence | [N] | TC-NNN, ... |
| Orientation Change | [N] | TC-NNN, ... |
| No Network | [N] | TC-NNN, ... |
| **Total** | **[N]** | |
```

---

## Key Reminders

- **Step 0 is mandatory** — always detect mode first
- **`mobile_list_available_devices` before anything** — no device = no automation
- **Screenshot → Read → Analyze** is the discovery loop for every new screen
- **Always get fresh element bounds** — never hardcode coordinates
- **Terminate + relaunch** between scenarios for clean state
- **Read `references/mobile-automation.md`** for all mobile-mcp tool signatures
- **Read `references/test-patterns-native.md`** for scenario patterns by UI element and app category
- **`.env.qa` is always gitignored** — never write real credentials to tracked files
