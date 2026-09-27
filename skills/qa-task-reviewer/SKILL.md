---
name: qa-task-reviewer
description: Review a ticket or feature request and produce a QA scope document. Extracts acceptance criteria, identifies test scope (frontend/backend/design), flags risks, and defines entry/exit criteria before testing begins.
argument-hint: [ticket-id or feature description]
disable-model-invocation: true
allowed-tools: Read Grep Glob
effort: medium
---

Review this task for QA: **$ARGUMENTS**

## Step 1 — Find and Read the Task

- Search for ticket ID or feature name in: `docs/`, `.claude/`, `README.md`, `CHANGELOG.md`
- If $ARGUMENTS is a description, use it directly

```
Project files: !`find . -name "*.md" | head -20 2>/dev/null`
Recent changes: !`git log --oneline -10 2>/dev/null || echo "no git"`
```

## Step 2 — Extract These Fields

**Feature Summary** — What does this feature do? (2–3 sentences max)

**Acceptance Criteria** — What does "done" look like? List each criterion as a checkbox.

**User Roles** — Who uses this feature? (Admin / Regular User / Guest / other)

**Entry Points** — Which screens/pages/URLs are affected?

**API Endpoints** — Which backend endpoints are involved? (GET/POST/PUT/DELETE)

**Dependencies** — What other features/modules does this touch?

## Step 3 — Define Test Scope

For each area, list what specifically needs testing:

**Frontend:**
- UI components and screens
- Forms and validation
- Navigation flows
- Responsive breakpoints (320px, 768px, 1024px, 1440px)

**Backend:**
- API endpoints with methods
- Authentication and authorization rules
- Data validation rules

**Design:**
- Is a Figma/design spec available? (yes/no/link)

**Integration:**
- What other modules does this touch?

## Step 4 — Risk Assessment

Flag these automatically:
- Any auth or permission logic → HIGH RISK
- Any payment or financial data → CRITICAL RISK
- Any user data read/write → HIGH RISK
- No design spec provided → MEDIUM RISK (design testing blocked)
- Acceptance criteria vague → MEDIUM RISK (scope unclear)

## Step 5 — Entry and Exit Criteria

**Entry Criteria (before QA starts):**
- [ ] Requirements documented
- [ ] Design spec available (if UI feature)
- [ ] Test environment deployed
- [ ] Test accounts available

**Exit Criteria (before QA approves):**
- [ ] All test cases executed
- [ ] Pass rate ≥ 90%
- [ ] Zero Critical bugs open
- [ ] Zero High bugs open (or documented with fix plan)

## Output — QA Scope Document

```markdown
# QA Scope — [Feature Name]
**Ticket:** [ID]        **Date:** [YYYY-MM-DD]        **Risk Level:** [CRITICAL/HIGH/MEDIUM/LOW]

## Feature Summary
[2-3 sentences]

## Acceptance Criteria
- [ ] [criterion 1]
- [ ] [criterion 2]

## Test Scope
### Frontend: [list screens/components]
### Backend: [list endpoints]
### Design: [link or "not available"]
### User Roles: [list roles]

## Risk Areas
- [HIGH] [risk description]

## Entry Criteria
- [ ] [item]

## Exit Criteria
- [ ] [item]

## Questions / Blockers
- [unclear item that needs answer before testing]
```
