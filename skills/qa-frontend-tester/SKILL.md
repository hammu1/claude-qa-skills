---
name: qa-frontend-tester
description: Test the frontend UI of a feature — forms, validation messages, states, navigation, responsiveness, and accessibility. Optionally runs Playwright automation against a live URL for evidence capture.
argument-hint: [feature-name or URL]
disable-model-invocation: true
allowed-tools: Read Write Bash Grep Glob
effort: high
---

Run frontend QA on: **$ARGUMENTS**

```
Environment check: !`which playwright npx 2>/dev/null | head -5 || echo "playwright not installed"`
Project type: !`cat package.json 2>/dev/null | grep -E '"name"|"scripts"' | head -8 || echo "no package.json"`
```

---

## Manual Testing Checklist

Work through each section. Mark ✅ pass, ❌ fail (note TC ID), ⚠️ partial.

### Forms & Input Validation
- [ ] All required fields have clear indicator (`*` or label text)
- [ ] Validation fires on form submit (not just on blur, unless spec says otherwise)
- [ ] Each error message is specific and positioned near its field ("Email is required" below email, not a generic toast)
- [ ] Success state is visually confirmed after valid submission
- [ ] Submit button shows loading state (spinner, "Saving...") after click
- [ ] Submit button is disabled during loading — double-submit not possible
- [ ] Confirm password matches validation triggers correctly
- [ ] Password field has show/hide toggle
- [ ] Character counter shown if field has max length

### Dropdowns & Selects
- [ ] All options load on open
- [ ] Default placeholder shown ("Select an option...")
- [ ] Selection saves and persists
- [ ] Searchable dropdown: typing filters results correctly
- [ ] No results state shown for searchable dropdown when no match
- [ ] Multi-select shows selections as tags/chips

### Filters & Search
- [ ] Filter panel opens and closes correctly
- [ ] Applied filters shown as visible tags or indicators
- [ ] "Clear all" / "Reset" button removes all filters and restores full results
- [ ] Filter count badge updates when filters applied
- [ ] Empty state shows helpful message (not a blank page)
- [ ] URL params update if filters are URL-based (check browser address bar)
- [ ] Filter state persists on refresh if expected

### Pagination
- [ ] Active page highlighted clearly
- [ ] Previous button disabled on page 1
- [ ] Next button disabled on last page
- [ ] Page size selector changes visible item count
- [ ] Item count label is accurate: "Showing 1–10 of 47"
- [ ] Navigating pages scrolls back to top of list
- [ ] Pagination resets to page 1 when filter is applied

### Navigation & Routing
- [ ] All nav links go to correct pages
- [ ] Active link is highlighted in nav menu
- [ ] Breadcrumbs are correct and each link works
- [ ] Browser back/forward buttons work correctly (no blank screen)
- [ ] Deep links (direct URL access) load the correct content
- [ ] Unauthorized page redirects to login
- [ ] 404 page appears for invalid URLs (not a blank screen)

### Loading & Error States
- [ ] Skeleton screen or spinner shown during data fetch
- [ ] No layout shift (content jumping) when data loads
- [ ] Error state shown if API call fails (not a blank screen)
- [ ] Retry button or helpful message on error state

### Buttons & CTAs
- [ ] Primary button is visually distinct from secondary
- [ ] Disabled state is visually obvious and non-clickable
- [ ] Destructive actions (Delete, Cancel Order) have a confirmation dialog

### Responsiveness — Test All Breakpoints
- [ ] **320px** — No horizontal scroll, touch targets ≥ 44×44px
- [ ] **768px** — Tablet layout is correct, no broken components
- [ ] **1024px** — Laptop layout is correct
- [ ] **1440px** — Desktop layout, content not stretched excessively

### Basic Accessibility
- [ ] Form fields have labels (not just placeholders)
- [ ] Error messages announced to screen readers (aria-live or aria-invalid)
- [ ] Tab order follows logical visual flow
- [ ] Images have alt text
- [ ] Status not conveyed by color alone (icon or text alongside)

---

## Playwright Automation (Optional — for Evidence)

If Playwright is installed and a URL is provided, run this to capture screenshots:

```python
# Save as /tmp/qa_frontend_check.py and run: python /tmp/qa_frontend_check.py
from playwright.sync_api import sync_playwright

URL = "$ARGUMENTS"  # Replace with actual URL if not a URL

with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    
    # Desktop
    page = browser.new_page(viewport={"width": 1440, "height": 900})
    page.goto(URL)
    page.wait_for_load_state("networkidle")
    page.screenshot(path="/tmp/qa_desktop.png", full_page=True)
    
    # Mobile
    mobile = browser.new_page(viewport={"width": 375, "height": 812})
    mobile.goto(URL)
    mobile.wait_for_load_state("networkidle")
    mobile.screenshot(path="/tmp/qa_mobile.png", full_page=True)
    
    # Tablet
    tablet = browser.new_page(viewport={"width": 768, "height": 1024})
    tablet.goto(URL)
    tablet.wait_for_load_state("networkidle")
    tablet.screenshot(path="/tmp/qa_tablet.png", full_page=True)
    
    browser.close()
    print("Screenshots saved: /tmp/qa_desktop.png, /tmp/qa_mobile.png, /tmp/qa_tablet.png")
```

---

## Output

```markdown
# Frontend Test Results — [Feature]
**Date:** [YYYY-MM-DD]      **Environment:** [URL]      **Browser:** Chrome XX

## Results Table
| Test Case | Status | Notes |
|-----------|--------|-------|
| TC-FORM-01 | ✅ PASS | |
| TC-FORM-02 | ❌ FAIL | No error shown on empty email submit |

## Bugs Found
- BUG-001: [brief description] — will be documented in /qa-bug-reporter

## Screenshots
- Desktop (1440px): /tmp/qa_desktop.png
- Mobile (375px): /tmp/qa_mobile.png
- Tablet (768px): /tmp/qa_tablet.png

## Overall Status
- [ ] ✅ ALL PASS — ready for backend testing
- [ ] ⚠️ PARTIAL — [N] failures, not blocking
- [ ] ❌ BLOCKED — critical UI issues found
```
