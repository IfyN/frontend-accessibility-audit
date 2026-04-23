# Frontend Accessibility Audit

A Claude skill that turns accessibility reviews into shippable engineering tickets. Paste a component, a page, or a screenshot; get back a prioritised list of WCAG-mapped findings with code fixes, reproduction steps, and named user impact.

Built from patterns encountered doing real accessibility work across web and Flutter/iOS apps. Opinionated, code-inclusive, and designed to be used by both engineers and accessibility specialists.

---

## What this skill does

When triggered in a Claude conversation, it changes Claude's behaviour in three concrete ways:

1. **Runs a structured 7-step audit** , page structure, keyboard, screen reader, interactive, content, visual, edge cases , in an order that surfaces issues efficiently.
2. **Matches findings to a pattern library** of 22 web + 8 Flutter/iOS production issues, each with WCAG mapping, detection heuristics, severity, code fixes, and common false positives.
3. **Outputs shippable tickets** rather than loose observations , every finding comes out with title, affected users, WCAG criteria, severity, reproduction steps, code fix, rationale, and false-positive notes.

The result: an audit you can hand directly to engineering, or read as a learning artefact if you're growing into this work.

---

## Who this is for

**Frontend engineers** doing pre-ship reviews, PR reviews, or inherited-codebase audits. You'll get tickets that drop straight into Jira with code you can implement.

**Accessibility specialists** who want structured findings with user-impact grounding. The ticket format includes WCAG mapping, severity justification, reproduction steps, and named affected user groups , the specialist essentials.

**Design engineers and design system maintainers** building reusable components. Useful for validating that components meet accessibility requirements once, so consumers get it for free.

**Engineers learning accessibility.** The skill doubles as a playbook, read `SKILL.md` and the reference files as standalone documentation and you'll learn the methodology without ever triggering the skill.

**Product managers and non-technical stakeholders** can use it as a translation tool , paste a screenshot, get tickets, take them to your engineering team, but the methodology and output format are engineer-shaped. A PM-focused companion skill may come later.

---

## What's in the repo

```
frontend-accessibility-audit/
├── SKILL.md                        # Main skill: methodology, conventions, 22-pattern library
├── flutter-accessibility-skill.md  # Flutter/iOS patterns + 8 additional production issues
└── references/
    └── ticket-examples.md         # Three fully worked tickets showing the format
```

- **`SKILL.md`** , the audit methodology, team conventions, ticket format, severity guidance, and the 22-pattern web library. This is what Claude reads first when the skill triggers.
- **`flutter-accessibility-skill.md`** , Flutter-specific mental model (`Semantics`, `ExcludeSemantics`, `MergeSemantics`), mapping the 22 patterns to Flutter, and 8 additional Flutter/iOS production patterns (A–H) covering dropdowns, text field focus, VoiceOver rotor, date pickers, text scaling, merged card semantics, and screen-load focus.
- **`references/ticket-examples.md`** , three fully worked tickets (Blocker, Major, Minor) showing the format in practice.

---

## Installation

> Skills are a Claude capability. You need to be using a Claude interface that supports custom skills to install this.

1. Clone this repo, or download the `frontend-accessibility-audit/` folder.
2. Place the folder in your Claude skills directory (typically `/mnt/skills/user/` in Claude Code environments, or the equivalent path in your setup).
3. The skill will be available the next time you start a Claude session.

To use it, start a conversation and paste a component, screenshot, or URL, and ask for an accessibility audit. Trigger phrases include "audit this for a11y", "check this for WCAG issues", "write accessibility tickets for this", or anything mentioning screen readers, focus order, contrast, alt text, or ARIA.

---

## Example usage

**Input:**

```jsx
<div className="modal" onClick={handleClose}>
  <h2>Delete account</h2>
  <p>This action cannot be undone.</p>
  <span onClick={handleDelete}>Delete</span>
  <span onClick={handleClose}>Cancel</span>
</div>
```

**Output (abbreviated):**

Three tickets produced, prioritised by severity:

- **Blocker** , Modal lacks dialog semantics and focus management (WCAG 2.4.3, 4.1.2)
- **Blocker** , Destructive and cancel actions use `<span>` not `<button>`, not keyboard-accessible (WCAG 2.1.1, 4.1.2)
- **Major** , Modal container's `onClick` creates an inaccessible click target on the overlay, likely dismissing the modal unintentionally when users try to read content

Each ticket includes the full format: affected users, WCAG criteria with level, severity with justification, reproduction steps, expected behaviour, code fix, why it works, and common false positives.

See `frontend-accessibility-audit/references/ticket-examples.md` for three complete worked examples.

---

## Methodology (at a glance)

The skill's audit follows seven steps, in this order:

1. **Page structure** , title, headings, landmarks, language
2. **Keyboard navigation** , tab order, focus visibility, focus traps
3. **Screen reader pass** , VoiceOver and TalkBack, rotor navigation
4. **Interactive components** , buttons, modals, accordions, forms
5. **Content accessibility** , images, videos, tables, PDFs
6. **Visual accessibility** , contrast, text scaling, responsive behaviour
7. **Edge cases** , animations, orientation, dynamic content

The order matters. Structural issues cascade into interactive and content issues, so fixing them first prevents re-auditing later.

Full methodology is in `frontend-accessibility-audit/SKILL.md`.

---

## Design principles

A few opinions baked into this skill, made explicit so you can judge whether they fit your context:

- **ARIA is a fallback, not a first choice.** Semantic HTML first (`<button>` over `<div role="button">`); every ARIA attribute is a maintenance liability.
- **Severity is about user impact, not fix difficulty.** A one-line `alt=""` fix can still be a blocker.
- **Tickets, not observations.** Findings are only useful if they can be worked on , so the output format is always a shippable ticket, never a loose list.
- **Code fixes, not just descriptions.** Even specialist tickets include code, because specifying behaviour in prose leaves too much room for misinterpretation.
- **Name affected users specifically.** "Screen reader users" is OK; "Screen reader users on iOS (VoiceOver), keyboard-only users, and users with cognitive disabilities" is better. Specificity drives prioritisation and grounds issues in real human impact.

---

## What this skill doesn't do

Being honest about the limits:

- **It can't replace real screen reader testing.** The skill tells Claude what to look for in code, but "does VoiceOver actually announce this correctly on iOS 17" requires a physical device.
- **It can't catch visual-only issues from code alone.** Contrast, focus ring visibility, and reflow need the rendered output. Screenshots help, live testing helps more.
- **It's not WCAG-exhaustive.** The 22 + 8 patterns cover real production issues but don't touch every WCAG success criterion. Things like 2.4.1 Bypass Blocks (skip links), 2.5.8 Target Size, or 3.3.7 Redundant Entry aren't in the library yet.
- **It's opinionated.** Team conventions (ARIA as fallback, severity defaults, ticket format) are baked in. If your house style differs, fork and adapt.
- **It's web-primary with a Flutter reference file.** Native iOS (UIKit, SwiftUI) and native Android (Jetpack Compose) patterns aren't covered.

---

## Contributing

Issues and pull requests are welcome. Most useful contributions:

- **New patterns** for the pattern library, with detection heuristic + WCAG mapping + code fix + false positives
- **Corrections** to existing patterns (Flutter APIs evolve, web best practices shift)
- **New ticket examples** for under-represented issue types
- **Improvements to the methodology** , if you find the 7-step order misses something important in your context, say so

Please open an issue to discuss larger changes before sending a PR. This is a single-maintainer project maintained in spare time, so response times will vary , typically within a week or two.

---

## Maintenance status

Maintained on a best-effort basis. I'll respond to issues and PRs when I can , typically within a week or two, sometimes longer. If the skill falls out of date (Flutter APIs change, WCAG versions update), open an issue and I'll work through them.

---

## Licence

MIT. See `LICENSE`.

---

## Background

Built by Ifeoma Nwosu, a frontend engineer with a background in accessibility engineering and design systems. Also the author of two [LinkedIn Learning courses](https://www.linkedin.com/learning/instructors/ifeoma-nwosu), a frequent conference speaker, and technical insturctor at Code First Girls where she helps Junior engineers upsill to Mid-level engineers.

If you use this skill and find it useful, I'd love to hear about it, open an issue or reach out on LinkedIn.
