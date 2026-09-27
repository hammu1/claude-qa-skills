---
name: qa-test-case-creator
description: Generate comprehensive manual test cases (positive + negative) for any feature. Covers functional, UI, API, security, and boundary testing. Auto-applies mock data for dropdowns, filters, pagination, payments, and user profiles.
argument-hint: [feature-name or ticket-id]
disable-model-invocation: true
allowed-tools: Read Write Grep Glob
effort: high
---

Create test cases for: **$ARGUMENTS**

Read the QA scope document if it exists, otherwise use the feature description in $ARGUMENTS.

---

## Rules — Always Follow

1. **Every element gets BOTH**: Positive (valid, happy path) AND Negative (invalid, edge case, error)
2. **One behavior per test case** — if the name has "and", split it into two
3. **Specific steps** — no vague steps like "interact with form"; say exactly what to click/type
4. **Include test data** — specify the exact value to use, not just "valid email"
5. **Expected result is measurable** — "shows error message" is bad; "shows 'Email is required' below email field" is good

---

## Test Case Format

```markdown
## TC-[NNN]: [Behavior being tested]

**Category:** Functional / UI / API / Security / Boundary / Permissions
**Type:** Positive / Negative
**Priority:** P0-Critical / P1-High / P2-Medium / P3-Low
**Estimated Time:** [N] min

### Preconditions
- User is logged in as [role]
- [any required setup]

### Test Data
- [field]: [exact value]

### Steps
1. Navigate to [URL or screen name]
2. [Exact action — click/type/select/scroll]
3. [Next action]

### Expected Result
[Specific, measurable outcome]

### Post-conditions
[System state after test completes]
```

---

## Standard Test Cases — Include ALL Applicable Sections

### Forms & Input Fields

| ID | Test | Type | Priority |
|----|------|------|----------|
| TC-FORM-01 | Submit with all valid required fields filled | Positive | P0 |
| TC-FORM-02 | Submit with all required fields empty | Negative | P0 |
| TC-FORM-03 | Submit with only required fields, optionals empty | Positive | P1 |
| TC-FORM-04 | Input exactly at max character limit | Boundary | P1 |
| TC-FORM-05 | Input one character over max limit | Negative | P1 |
| TC-FORM-06 | Input XSS payload: `<script>alert(1)</script>` | Security | P0 |
| TC-FORM-07 | Input SQL injection: `' OR 1=1 --` | Security | P0 |
| TC-FORM-08 | Input only whitespace in required field | Negative | P1 |
| TC-FORM-09 | Paste value vs manual type | UI | P2 |
| TC-FORM-10 | Tab navigation order through all fields | UI/A11y | P2 |
| TC-FORM-11 | Submit button disabled during loading | UI | P1 |
| TC-FORM-12 | Error message shown near field, not generic toast | UI | P1 |

### Dropdowns & Selects

| ID | Test | Type | Priority |
|----|------|------|----------|
| TC-DD-01 | Select valid option — value saved and displayed | Positive | P0 |
| TC-DD-02 | Submit without selecting (required dropdown) | Negative | P0 |
| TC-DD-03 | Select first option in list | Boundary | P1 |
| TC-DD-04 | Select last option in list | Boundary | P1 |
| TC-DD-05 | Search inside searchable dropdown — results filter | Functional | P1 |
| TC-DD-06 | Search with no match — "no results" state shown | Boundary | P1 |
| TC-DD-07 | Selection persists after page refresh | Functional | P1 |

### Filters & Search

| ID | Test | Type | Priority |
|----|------|------|----------|
| TC-FILTER-01 | Apply single filter — results update correctly | Positive | P0 |
| TC-FILTER-02 | Apply multiple filters simultaneously | Positive | P1 |
| TC-FILTER-03 | Filter produces zero results — empty state shown | Boundary | P1 |
| TC-FILTER-04 | Clear single filter — results revert | Functional | P1 |
| TC-FILTER-05 | Clear all filters — all results shown | Functional | P1 |
| TC-FILTER-06 | Search with special chars: `!@#$%^&*()<>` | Security | P0 |
| TC-FILTER-07 | Search empty string — all results or empty state | Boundary | P1 |
| TC-FILTER-08 | Date range: end date before start date | Negative | P1 |
| TC-FILTER-09 | Date range: same start and end date | Boundary | P2 |
| TC-FILTER-10 | Combined filter + search simultaneously | Integration | P1 |
| TC-FILTER-11 | Filter state persists after navigating away and back | Functional | P2 |

### Pagination

| ID | Test | Type | Priority |
|----|------|------|----------|
| TC-PAGE-01 | Page 1 — Previous/Back button is disabled | Boundary | P1 |
| TC-PAGE-02 | Last page — Next button is disabled | Boundary | P1 |
| TC-PAGE-03 | Navigate to middle page | Functional | P1 |
| TC-PAGE-04 | Change page size (10 → 25 → 50 → 100) | Functional | P1 |
| TC-PAGE-05 | Single item total — pagination hidden or disabled | Boundary | P2 |
| TC-PAGE-06 | Zero items — pagination hidden, empty state shown | Boundary | P1 |
| TC-PAGE-07 | Direct URL with page=0 or page=-1 | Negative | P2 |
| TC-PAGE-08 | Direct URL with page=99999 (beyond last page) | Negative | P2 |
| TC-PAGE-09 | Item count label shows correct range ("1–10 of 47") | UI | P2 |
| TC-PAGE-10 | Filter applied — pagination resets to page 1 | Integration | P1 |

### User Profiles

| ID | Test | Type | Priority | Test Data |
|----|------|------|----------|-----------|
| TC-PROF-01 | View complete profile — all fields displayed | Positive | P0 | admin@test.com |
| TC-PROF-02 | View incomplete profile — gracefully handled | Negative | P1 | incomplete@test.com |
| TC-PROF-03 | Update name with valid value | Positive | P0 | user@test.com |
| TC-PROF-04 | Update email with invalid format | Negative | P0 | `notanemail` |
| TC-PROF-05 | Update email with already-used email | Negative | P0 | admin@test.com |
| TC-PROF-06 | Upload avatar: valid PNG under size limit | Positive | P1 | test.png (<2MB) |
| TC-PROF-07 | Upload avatar: file over size limit | Negative | P1 | large.png (>5MB) |
| TC-PROF-08 | Upload avatar: non-image file | Negative | P1 | document.pdf |
| TC-PROF-09 | Admin views another user's profile | Permissions | P0 | admin@test.com |
| TC-PROF-10 | Regular user cannot see admin-only fields | Permissions | P0 | user@test.com |
| TC-PROF-11 | User cannot edit another user's profile | Security | P0 | IDOR attempt |

### Payment Methods

| ID | Test | Type | Priority | Card |
|----|------|------|----------|------|
| TC-PAY-01 | Pay with valid Visa | Positive | P0 | 4242 4242 4242 4242 / 12/26 / 123 |
| TC-PAY-02 | Pay with valid Mastercard | Positive | P0 | 5555 5555 5555 4444 / 12/26 / 123 |
| TC-PAY-03 | Declined card — clear error shown | Negative | P0 | 4000 0000 0000 0002 |
| TC-PAY-04 | Insufficient funds | Negative | P0 | 4000 0000 0000 9995 |
| TC-PAY-05 | Expired card | Negative | P0 | 4000 0000 0000 0069 |
| TC-PAY-06 | Wrong CVV | Negative | P0 | 4000 0000 0000 0127 |
| TC-PAY-07 | 3D Secure — completes redirect and payment | Positive | P0 | 4000 0025 0000 3155 |
| TC-PAY-08 | Empty card number field | Negative | P0 | — |
| TC-PAY-09 | Invalid card number format | Negative | P1 | `1234 5678` |
| TC-PAY-10 | Past expiry date entered | Negative | P1 | `01/20` |
| TC-PAY-11 | Double-click submit — no duplicate charge | Security | P0 | valid card |
| TC-PAY-12 | Payment amount matches order total | Functional | P0 | — |

---

## Coverage Summary Table

After writing all test cases, output this summary:

```markdown
## Test Coverage Summary — [Feature Name]
**Total Cases:** N     **Positive:** N     **Negative:** N

| Category | Total | P0 | P1 | P2 | P3 |
|----------|-------|----|----|----|----|
| Functional | N | N | N | N | N |
| UI/UX | N | N | N | N | N |
| API | N | N | N | N | N |
| Security | N | N | N | N | N |
| Boundary | N | N | N | N | N |
| Permissions | N | N | N | N | N |
| **Total** | **N** | **N** | **N** | **N** | **N** |
```

Save output as: `TEST_CASES_$ARGUMENTS_$(date +%Y-%m-%d).md`
