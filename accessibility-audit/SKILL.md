---
name: frontend-accessibility-audit
description: Run a structured frontend accessibility audit and produce shippable engineering tickets. Use this skill whenever the user asks for an accessibility review, a11y audit, WCAG check, screen reader review, or ticket-writing for accessibility findings , on web (HTML/React) or Flutter/iOS. Trigger this even when the user just pastes a component and asks "is this accessible?" or "any a11y issues here?", and whenever they mention VoiceOver, TalkBack, focus order, contrast, alt text, ARIA, or semantic HTML. Produces findings in a Jira-ready ticket format grounded in a 22-pattern library from real production audits.
---

# Frontend Accessibility Audit

A software engineer's methodology for auditing frontend accessibility and writing shippable tickets. Based on real production audit work across web and Flutter/iOS apps.

This skill does three things:

1. **Runs a structured audit** in a specific order that surfaces issues efficiently
2. **Matches findings to a 22-pattern library** of real issues with detection heuristics and fixes
3. **Outputs findings as Jira-ready tickets** rather than loose observations

## When to use this skill

Use this whenever the user asks for an accessibility review of code, a component, a screenshot, or a page. Common triggers:

- "Audit this for a11y"
- "Is this component accessible?"
- "Check this for WCAG issues"
- "Write accessibility tickets for this page"
- "Review this for VoiceOver / TalkBack"
- Any mention of: focus order, contrast, alt text, ARIA, semantic HTML, screen reader behaviour

For Flutter/iOS-specific audits, always read `flutter-accessibility-skill.md` before starting , detection heuristics and fixes differ from web, and the reference includes 8 additional Flutter-specific production patterns (A–H) that don't map cleanly onto the web-focused main library.

## Audit methodology

Work through these seven steps **in order**. The order matters: structural issues cascade into interactive and content issues, so fixing them first prevents re-auditing later.

### 1. Page structure

Check the scaffolding before anything else:

- `<title>` is present and descriptive (not empty, not just the app name)
- Headings are in logical order (`h1` → `h2` → `h3`, no skipped levels)
- Landmarks exist: `<header>`, `<nav>`, `<main>`, `<footer>`, or ARIA equivalents
- Language is declared: `<html lang="en">`

### 2. Keyboard navigation

Tab through the entire page without a mouse:

- Every interactive element is reachable
- Tab order follows visual/logical order
- Focus is visible at every step
- Focus is trapped inside modals and released on close
- No keyboard traps

### 3. Screen reader pass

Run the page through VoiceOver (iOS/Mac) and TalkBack (Android). Use rotor/quick nav to check:

- Headings make sense as an outline
- Links are descriptive out of context (no "click here")
- Landmarks are announced
- Form fields have accessible names
- State changes are announced (expanded/collapsed, loading, errors)

### 4. Interactive components

Inspect each interactive pattern against the pattern library (issues #1, #2, #5–#9, #13, #19, #20):

- Buttons, links, inputs, selects
- Modals, accordions, tabs, menus
- Custom controls (graphs, animations, drag-and-drop)

### 5. Content accessibility

Check non-interactive content (issues #3, #4, #11, #14, #15, #18):

- Images have meaningful or empty alt
- Videos have captions (including for ambient/non-speech audio)
- Tables have captions and proper headers
- PDFs are tagged and readable

### 6. Visual accessibility

Run contrast checks and resize testing (issues #10, #17, #21):

- Normal text ≥ 4.5:1 contrast (WCAG AA)
- Large text and UI components ≥ 3:1
- Text scales up to 200% without breaking layout
- No typography issues (spacing, typos, unclear hierarchy)

### 7. Edge cases

Finally, the easy-to-miss ones (issues #12, #22):

- Animations respect `prefers-reduced-motion` or have a pause control
- Orientation is not locked , page works in portrait and landscape
- Dynamic content changes are announced via `aria-live`

## Tools used

- **VoiceOver** (iOS + macOS) and **TalkBack** (Android) , primary screen readers
- **Chrome DevTools** Accessibility tree + contrast picker
- **Axe DevTools** for automated first-pass
- **WebAIM Contrast Checker** for colour verification
- **Adobe Acrobat Pro** for PDF audits

Automated tools catch roughly 30% of issues. Always follow up with manual keyboard and screen reader testing.

## Team conventions

These are the defaults , deviate only with a stated reason.

- **Decorative images**: `alt=""` and `aria-hidden="true"`. Do not describe them.
- **Meaningful images**: descriptive `alt` that conveys the information or function.
- **Contrast thresholds**: 4.5:1 for normal text, 3:1 for large text and UI components.
- **Interactive elements** must be keyboard-reachable and have both a visible label and a programmatic name.
- **ARIA is a fallback, not a first choice.** Prefer semantic HTML (`<button>` over `<div role="button">`). Every ARIA attribute is a maintenance liability.
- **Focus management**: moved deliberately on route changes, modal opens, and after destructive actions.

## Ticket format

Every finding produced by this skill uses this template. Do not output loose observations , always a ticket.

````markdown
**Title:** [Concise issue summary]

**WCAG Criteria:**

- [Criterion number and name] (Level A/AA/AAA)

**Affected Users:**
[Specific user groups impacted, e.g. screen reader users on iOS (VoiceOver), keyboard-only users, users with low vision, users with cognitive disabilities. Be specific , this drives prioritisation and grounds the issue in real human impact.]

**Severity:** [Blocker / Major / Minor] , [one-line justification]

**Description:**
[What is happening and why it's a problem]

**Steps to Reproduce:**

1. [Step]
2. [Step]
3. [Observed behaviour]

**Expected Behaviour:**
[What should happen instead]

**Proposed Solution:**

```[language]
[Code example showing the fix]
```
````

**Why This Fix Works:**
[1–2 sentence explanation of the underlying principle]

**Common False Positives:**
[Cases that look like this issue but aren't, if any]

````

See `references/ticket-examples.md` for fully worked examples.

## Severity guidance

- **Blocker** , the feature is unusable for someone using assistive tech (e.g. a modal that traps focus incorrectly, an unlabelled primary CTA, a form that can't be submitted via keyboard). Ship-stopper.
- **Major** , the feature works but is significantly degraded (e.g. state changes not announced, low but near-threshold contrast, missing captions on a content table). Fix this sprint.
- **Minor** , polish issues that affect experience but not task completion (e.g. non-descriptive link text when context makes it clear, decorative image missing `aria-hidden`). Backlog.

Default to the higher severity when in doubt , accessibility bugs tend to be underestimated.

---

## Pattern library , 22 production issues

When the audit surfaces a finding, match it to a pattern below. The pattern gives you the WCAG criterion, detection heuristic, and fix. Use it to write the ticket.

### 1. Modal focus issues

**Platforms:** Web + Flutter · VoiceOver
**WCAG:** 2.4.3 Focus Order (A), 4.1.2 Name Role Value (A)
**Severity:** Blocker , modal is unusable

**Detection:** Open the modal, check: does focus move into the modal? Does Tab cycle within the modal? Does the title announce correctly? Does Escape close it?

**Broken:**
```html
<div class="modal">
  <h2>Title</h2>
</div>
````

**Fixed:**

```html
<div role="dialog" aria-modal="true" aria-labelledby="modal-title">
  <h2 id="modal-title">Title</h2>
  <button autofocus>Close</button>
</div>
```

**Why:** `role="dialog"` + `aria-labelledby` gives screen readers the semantic contract. `autofocus` moves focus into the modal on open. `aria-modal="true"` tells assistive tech to treat content outside as inert.

**False positives:** Static, non-interactive overlays (e.g. a loading scrim) don't need dialog semantics.

### 2. Accordion accessibility

**Platforms:** Web + Flutter · VoiceOver
**WCAG:** 4.1.2 Name Role Value (A)
**Severity:** Major

**Detection:** Does the trigger announce its expanded/collapsed state? Is closed content still read by the screen reader?

**Broken:**

```html
<div onclick="toggle()">Section</div>
<div class="panel">...</div>
```

**Fixed:**

```html
<button aria-expanded="false" aria-controls="panel1">Section</button>
<div id="panel1" hidden>...</div>
```

**Why:** `aria-expanded` exposes state. `hidden` removes collapsed content from the accessibility tree , not just visually. Using `<button>` gives you keyboard support for free.

### 3. Image without alt

**WCAG:** 1.1.1 Non-text Content (A)
**Severity:** Blocker (for meaningful images)

**Detection:** Any `<img>` missing the `alt` attribute entirely.

**Fixed:**

```html
<img src="img.png" alt="Robot illustration welcoming new users" />
```

**False positives:** Decorative images should use `alt=""` (empty), not omit the attribute.

### 4. Linked image without label

**WCAG:** 2.4.4 Link Purpose (A)
**Severity:** Blocker

**Broken:**

```html
<a href="/"><img src="logo.png" /></a>
```

**Fixed:**

```html
<a href="/"><img src="logo.png" alt="Home" /></a>
```

**Why:** The link's accessible name comes from its content. An unlabelled image means an unlabelled link.

### 5. Input without label

**WCAG:** 3.3.2 Labels or Instructions (A), 4.1.2 (A)
**Severity:** Blocker

**Fixed:**

```html
<label for="name">Name</label> <input id="name" name="name" />
```

**False positives:** Inputs wrapped in a `<label>` don't need `for`/`id`. Inputs labelled via `aria-labelledby` or `aria-label` are also valid (use visible labels where possible).

### 6. Select without label/name

**WCAG:** 3.3.2 (A), 4.1.2 (A)
**Severity:** Blocker

**Fixed:**

```html
<label for="country">Country</label>
<select id="country" name="country">
  ...
</select>
```

### 7. Missing ARIA for expand/collapse

**WCAG:** 4.1.2 (A)
**Severity:** Major

**Detection:** Any toggle control (menu, disclosure, tree) that changes content visibility without announcing state.

**Fix:** Add `aria-expanded` to the trigger and `aria-controls` pointing to the target region.

### 8. Button missing accessible name

**WCAG:** 4.1.2 (A)
**Severity:** Blocker

**Broken:**

```html
<button><img src="icon.png" /></button>
```

**Fixed:**

```html
<button aria-label="Download report"><img src="icon.png" alt="" /></button>
```

**Why:** Icon-only buttons need an explicit accessible name. The inner image should then be `alt=""` to avoid double announcement.

### 9. Redundant aria-label

**WCAG:** 2.5.3 Label in Name (A)
**Severity:** Minor to Major (depends on mismatch severity)

**Issue:** `aria-label` overrides visible text. If they mismatch, voice-control users can't activate the control by its visible name.

**Fix:** Remove `aria-label` when visible text is sufficient, or ensure `aria-label` includes the visible text verbatim.

### 10. Text contrast issues

**WCAG:** 1.4.3 Contrast Minimum (AA), 1.4.11 Non-text Contrast (AA)
**Severity:** Major

**Detection:** Run WebAIM Contrast Checker on all text/background pairs. Normal text needs 4.5:1, large text (18pt+ or 14pt+ bold) and UI components need 3:1.

**Fix:** Darken text or lighten background until the ratio passes. Never rely on colour alone to convey information.

### 11. Empty title element

**WCAG:** 2.4.2 Page Titled (A)
**Severity:** Major

**Fixed:**

```html
<title>Dashboard , AppName</title>
```

**Why:** Screen readers read the title on page load. Empty or generic titles destroy orientation in tab-switching scenarios.

### 12. Animation cannot be stopped

**WCAG:** 2.2.2 Pause Stop Hide (A), 2.3.3 Animation from Interactions (AAA)
**Severity:** Major

**Fixed:**

```css
@media (prefers-reduced-motion: reduce) {
  .animated {
    animation: none;
    transition: none;
  }
}
```

**Why:** Vestibular disorders are real. Respect the OS-level preference; add explicit pause controls for animations longer than 5 seconds.

### 13. Robot animation , no interaction feedback

**WCAG:** 4.1.3 Status Messages (AA)
**Severity:** Major

**Detection:** Any custom interactive control (mascot, Easter egg, gamified element) where the visual state change isn't announced.

**Fix:** Add a live region that updates on interaction:

```html
<div aria-live="polite" class="sr-only">Robot is waving</div>
```

### 14. Table missing caption

**WCAG:** 1.3.1 Info and Relationships (A)
**Severity:** Major

**Fixed:**

```html
<table>
  <caption>
    Active users by region, Q3 2024
  </caption>
  <thead>
    ...
  </thead>
</table>
```

**Why:** Captions give screen reader users the table's purpose before they navigate its cells.

### 15. Video without captions (including ambient audio)

**WCAG:** 1.2.2 Captions (A)
**Severity:** Blocker (if dialogue), Major (if ambient)

**Fix:** Provide `.vtt` captions. For non-speech audio, include descriptions like `[ambient sound]`, `[keyboard clacking]`, `[music swells]`.

**Why:** Deaf and hard-of-hearing users need to know audio is present even when there's no dialogue.

### 16. Footer focus skips links

**WCAG:** 2.4.3 Focus Order (A), 2.4.7 Focus Visible (AA)
**Severity:** Major

**Detection:** Tab into the footer. If focus jumps past links, check for `aria-hidden="true"`, `tabindex="-1"`, `display: none`, or `visibility: hidden` on the links or their parent.

**Fix:** Remove the hidden attributes. If content must be hidden from some users but not others, rethink the pattern.

### 17. Mobile text scaling

**WCAG:** 1.4.4 Resize Text (AA)
**Severity:** Major

**Fix (web):** Use relative units (`rem`, `em`) instead of `px` for font sizes. Ensure containers don't clip at 200% zoom.

**Fix (Flutter):** Respect `MediaQuery.textScaleFactor` , see `flutter-skill.md` for the full pattern.

### 18. Table icons not announced

**WCAG:** 1.1.1 (A)
**Severity:** Major

**Detection:** Status icons in tables (, , warning triangle) without text labels.

**Fix:**

```html
<img src="check.svg" alt="Correct" />
```

Or, better, combine an icon with sr-only text:

```html
<span aria-hidden="true"></span><span class="sr-only">Correct</span>
```

### 19. Graph not tabbable

**WCAG:** 2.1.1 Keyboard (A)
**Severity:** Major (if interactive), N/A (if static)

**Fix:**

```html
<div role="img" aria-label="Revenue chart, Q1 to Q4 2024" tabindex="0">...</div>
```

For static charts, provide a text summary or data table alternative. For interactive charts, ensure all interactions work with keyboard.

### 20. Span used as link

**WCAG:** 4.1.2 (A), 2.1.1 (A)
**Severity:** Blocker

**Broken:**

```html
<span onclick="navigate()">Read more</span>
```

**Fixed:**

```html
<a href="/article">Read more</a>
```

**Why:** `<a>` gives you keyboard support, correct role announcement, right-click menus, and middle-click-to-new-tab for free. Never reinvent links.

### 21. Typography issues

**WCAG:** 1.4.12 Text Spacing (AA)
**Severity:** Minor

**Fix:** Proofread content. Ensure line-height ≥ 1.5× font size, paragraph spacing ≥ 2× font size, letter-spacing ≥ 0.12× font size, word-spacing ≥ 0.16× font size when users override.

### 22. Orientation locked

**WCAG:** 1.3.4 Orientation (AA)
**Severity:** Major

**Fix (web):** Don't use CSS to restrict orientation. Design layouts that work in both portrait and landscape.

**Fix (Flutter):** Remove `SystemChrome.setPreferredOrientations` unless there's a WCAG-exempted reason (e.g. a piano app). See `flutter-accessibility-skill.md`.

**Why:** Users with mounted devices (wheelchairs, stands) can't rotate their screen.

---

## How to use this skill in a session

When auditing, follow this flow:

1. **Scope the audit.** Ask what's being audited (single component, page, flow, whole app) and on what platform.
2. **Walk the methodology.** Go through steps 1–7 in order, noting findings as you go.
3. **Match each finding to the pattern library.** If a finding doesn't match any of the 22 patterns, still write a ticket , the library is a starting point, not an exhaustive list.
4. **Write tickets in the exact format above.** One ticket per finding. Group related findings only when they share a single root cause.
5. **Prioritise by severity.** Deliver blockers first, then majors, then minors.

When in doubt, err toward the user's assistive tech experience. A "probably fine" on an audit is a "broken" in production.

For Flutter-specific detection and fixes , including 8 additional production patterns (A–H) covering dropdowns, text field focus, VoiceOver rotor links, date pickers, text scaling with country lists and flags, merged card semantics, and screen-load focus , read `flutter-accessibility-skill.md`. For fully worked ticket examples, read `ticket-examples.md`.
