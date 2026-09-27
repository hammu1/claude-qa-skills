# Claude QA Skills — AI agent workflows for software testing

Ten [Claude Code](https://claude.com/claude-code) skills that turn an LLM into an autonomous QA engineer across **web apps, mobile apps, browser extensions, and native desktop applications**.

These are not prompt templates. Each skill is a complete operating procedure: the agent launches the real application, looks at it through screenshots, works out what the app *is*, decides what is worth testing, executes the tests against live software, and hands back evidence a human reviewer can audit.

```
"test my Android app"        →  AppTester discovers every screen, maps flows,
                                generates scenarios, runs them on a real device
"run QA on this website"     →  webapp-testing crawls the site, writes Playwright
                                specs, executes them, saves screenshots
"QA this Chrome extension"   →  webextension-testing loads the unpacked extension,
                                exercises popup/options/content scripts
```

---

## Why this exists

Traditional test automation breaks on contact with change: a selector moves and the suite goes red for reasons unrelated to product quality. Record-and-replay tools are worse — they encode a single path through the app and know nothing about intent.

An agent-driven approach inverts it. The agent is given the *goal* ("verify checkout works") and the *means* (a browser, a device, a screenshot tool), and it discovers the path itself. When the UI changes, it adapts, because it was never holding a brittle script in the first place.

These skills encode the discipline that makes that reliable:

| Principle | How it's enforced |
|---|---|
| **Discovery before assertion** | The agent must map screens and flows before it is allowed to write a single test case |
| **Visual grounding** | Every decision is made from an actual screenshot, not from a DOM guess |
| **Evidence or it didn't happen** | Failures require a screenshot, numbered repro steps, and console/network logs |
| **Unrun ≠ passed** | A test that never executed is reported `BLOCKED` and excluded from the pass rate |
| **Isolated workspaces** | Each target gets its own folder, so prior QA work is never overwritten |

---

## The skills

### Platform QA agents

| Skill | Platform | What the agent does |
|---|---|---|
| **`AppTester`** | Android · iOS | Detects connected devices, launches the app, screenshots every screen for visual analysis, discovers all UI flows via `mobile-mcp`, maps them into named flows, then generates and runs maximum test scenarios per flow |
| **`webapp-testing`** | Web | Initializes a per-domain workspace, opens the URL with Playwright, discovers all pages and flows, generates scenarios, then **writes and executes real Playwright scripts** and saves evidence |
| **`webextension-testing`** | Chrome · Firefox · Edge | Loads the unpacked extension in a live browser, discovers popup / options / content-script surfaces, and writes runnable tests against `chrome.*` extension APIs |
| **`native-qa`** | macOS desktop | Launches any native app, discovers UI sections visually, asks clarifying questions per section, then generates flow documents and detailed test cases organized by flow |

### QA process agents

| Skill | What it produces |
|---|---|
| **`qa-automation`** | Full pipeline for a feature or ticket: reviews the task, writes positive and negative cases, tests frontend/backend/design, reproduces bugs with evidence, emits a QA report |
| **`qa-task-reviewer`** | A QA scope document *before* testing starts — acceptance criteria, test scope, risk flags, entry/exit criteria |
| **`qa-test-case-creator`** | Comprehensive manual cases covering functional, UI, API, security, and boundary testing, with mock data auto-applied for dropdowns, filters, pagination, payments, and profiles |
| **`qa-test-planner`** | Test plans, regression suites, and bug reports — with Figma MCP integration for design-vs-build validation |
| **`qa-frontend-tester`** | Frontend verification: forms, validation messages, states, navigation, responsiveness, accessibility — optionally driving Playwright against a live URL for evidence |
| **`senior-qa`** | Jest + React Testing Library stubs for React/Next.js, Istanbul/LCOV coverage-gap analysis, Playwright scaffolding from Next.js routes, MSW API mocks |

### Router

**`agents/QAAgent.md`** — a master QA agent that introduces itself, asks what kind of application you're testing, and routes to the correct specialized skill above. The entry point when you don't want to remember ten skill names.

---

## Install

Claude Code reads skills from `~/.claude/skills/` (user-wide) or `.claude/skills/` (per project).

```bash
git clone https://github.com/hammu1/claude-qa-skills.git
cd claude-qa-skills

# all of them, user-wide
cp -r skills/*  ~/.claude/skills/
cp -r agents/*  ~/.claude/agents/

# or just one
cp -r skills/webapp-testing ~/.claude/skills/
```

Restart Claude Code, then invoke by name or by intent:

```
/webapp-testing https://staging.example.com
```
```
test my Android app on the connected device
```

### Dependencies per skill

| Skill | Needs |
|---|---|
| `webapp-testing`, `webextension-testing`, `qa-frontend-tester` | Playwright — `pip install playwright && playwright install chromium` |
| `AppTester` | [`@mobilenext/mobile-mcp`](https://github.com/mobile-next/mobile-mcp) + a connected device or emulator (`adb devices`) |
| `native-qa` | macOS host with screen-recording permission granted to the terminal |
| `qa-test-planner` | Figma MCP server (optional — only for design validation) |
| `senior-qa` | A React/Next.js project with Jest configured |

The process skills (`qa-automation`, `qa-task-reviewer`, `qa-test-case-creator`) need nothing beyond Claude Code.

---

## Repository layout

```
claude-qa-skills/
├── skills/
│   ├── AppTester/              mobile QA agent          (~710 lines)
│   ├── native-qa/              macOS desktop QA agent  (~1040 lines)
│   ├── qa-test-planner/        planning + Figma MCP     (~760 lines)
│   ├── webapp-testing/         web QA agent             (~500 lines)
│   ├── webextension-testing/   extension QA agent       (~440 lines)
│   ├── senior-qa/              React/Next.js testing    (~330 lines)
│   ├── qa-automation/          full feature pipeline    (~290 lines)
│   ├── qa-test-case-creator/   manual case generation   (~170 lines)
│   ├── qa-frontend-tester/     UI + a11y verification   (~155 lines)
│   └── qa-task-reviewer/       pre-test scope doc       (~110 lines)
└── agents/
    └── QAAgent.md              router across all platforms
```

Each skill is a self-contained `SKILL.md` with YAML frontmatter declaring its name, description, and trigger phrases. Skills with supporting assets keep them alongside.

---

## Writing your own

The pattern that makes these work, in order:

1. **Gate on capability first.** Check the device/browser/display is actually there and report what's missing with the command to fix it. Never start a run you can't finish.
2. **Discover, then plan, then execute.** Three distinct phases. Skipping discovery produces tests for an app the agent imagined.
3. **Force visual confirmation.** Require a screenshot at each decision point; an agent reasoning over a DOM dump will confidently test elements no user can see.
4. **Make the output a contract.** Fixed result shape means one reporter serves every platform.
5. **Refuse to fake success.** Distinguish *failed* from *never ran*, and make the pass rate honest about coverage.

---

## Related

- [`ai-qa-agent`](https://github.com/hammu1/ai-qa-agent) — the standalone multi-platform framework these agents drive
- [`appium-framework`](https://github.com/hammu1/appium-framework) — Java + TestNG Android automation with a generic APK smoke crawler

---

## License

**All Rights Reserved** — see [LICENSE](LICENSE).

This code is published for portfolio and evaluation purposes. You're welcome to read it;
reuse in your own projects requires written permission.

Built by [Syed Hammad Ali](https://github.com/hammu1) · AI QA Engineer
