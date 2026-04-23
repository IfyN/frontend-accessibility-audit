# Ticket Examples

Three fully worked tickets showing the format in practice. Use these as models when writing new tickets.

---

## Example 1: Modal focus (Blocker)

**Title:** Sign-up confirmation modal does not receive focus on open

**WCAG Criteria:**
- 2.4.3 Focus Order (Level A)
- 4.1.2 Name, Role, Value (Level A)

**Affected Users:** Screen reader users on iOS (VoiceOver) and Android (TalkBack) , primary impact. Keyboard-only users cannot locate the new modal content without visual cues. Users with cognitive disabilities lose context when the screen changes without a clear focus shift.

**Severity:** Blocker , VoiceOver users cannot locate the modal after it opens. Focus remains on the trigger button behind a visual overlay, and the modal content is announced as generic div content.

**Description:**
When a user taps "Create account" on the sign-up form, a confirmation modal appears asking them to verify their email. VoiceOver does not move focus into the modal, and the modal is not announced as a dialog. Users hear the previous screen's content being read as if nothing changed, even though interaction is now blocked by the overlay. This makes the sign-up flow impossible to complete with a screen reader.

**Steps to Reproduce:**
1. Enable VoiceOver on iOS
2. Navigate to the sign-up screen
3. Complete the form and tap "Create account"
4. Observe that focus stays on the "Create account" button
5. Swipe right to explore , the trigger button is still read before any modal content

**Expected Behaviour:**
On modal open, focus should move to the first focusable element inside the modal (the "Close" button or the primary action). VoiceOver should announce the modal title and its role as a dialog. Focus should be trapped inside the modal until it's dismissed.

**Proposed Solution:**
```html
<div role="dialog" aria-modal="true" aria-labelledby="confirm-title">
  <h2 id="confirm-title">Confirm your email</h2>
  <p>We've sent a verification link to your inbox.</p>
  <button autofocus>Got it</button>
</div>
```

For Flutter:
```dart
showDialog(
  context: context,
  builder: (context) => AlertDialog(
    title: Text('Confirm your email'),
    content: Text("We've sent a verification link to your inbox."),
    actions: [
      TextButton(
        autofocus: true,
        onPressed: () => Navigator.pop(context),
        child: Text('Got it'),
      ),
    ],
  ),
);
```

**Why This Fix Works:**
`role="dialog"` + `aria-modal="true"` tells assistive tech to treat the modal as a focus-trapping surface and announce it as a dialog. `aria-labelledby` gives the modal its accessible name. `autofocus` on the primary button moves focus into the modal on open, so the user lands in the right place.

**Common False Positives:**
Non-interactive overlays (loading scrims, image lightboxes) don't need full dialog semantics , they just need to announce their state change via a live region.

---

## Example 2: Contrast on form helper text (Major)

**Title:** Helper text under form fields fails WCAG AA contrast

**WCAG Criteria:**
- 1.4.3 Contrast (Minimum) (Level AA)

**Affected Users:** Users with low vision or moderate visual impairments , primary impact. Users in bright sunlight or on uncalibrated/low-quality displays. Older users (age-related vision changes are common from 40+). Users with colour vision deficiencies, who often rely more heavily on luminance contrast.

**Severity:** Major , Helper text conveys important input requirements (password rules, format hints). Users with low vision, in sunlight, or on uncalibrated screens cannot read it, leading to form errors and abandonment.

**Description:**
Helper text below form fields uses `#9E9E9E` on a white (`#FFFFFF`) background, which measures 2.85:1 , below the 4.5:1 AA threshold for normal body text. This pattern is used on the sign-up, login, and profile-edit screens, affecting multiple critical flows.

**Steps to Reproduce:**
1. Navigate to any form with helper text (e.g. sign-up screen)
2. Inspect the helper text colour using the DevTools colour picker
3. Run the pair through WebAIM Contrast Checker
4. Observe the ratio is 2.85:1, below the 4.5:1 AA threshold

**Expected Behaviour:**
Helper text should meet a minimum 4.5:1 contrast ratio against its background. A darker grey such as `#595959` (7.0:1) or `#666666` (5.74:1) would pass.

**Proposed Solution:**
Update the design token:
```css
:root {
  /* Before */
  --color-text-helper: #9E9E9E;
  /* After */
  --color-text-helper: #595959;
}
```

For Flutter, update the theme:
```dart
theme: ThemeData(
  inputDecorationTheme: InputDecorationTheme(
    helperStyle: TextStyle(color: Color(0xFF595959)),
  ),
)
```

**Why This Fix Works:**
4.5:1 is the WCAG AA threshold for normal text and represents the minimum contrast at which most users with moderate low vision can read text comfortably. Darkening the grey to `#595959` passes the ratio without losing the intentional visual hierarchy between primary and helper text.

**Common False Positives:**
Large text (18pt+ or 14pt+ bold) only needs 3:1, so larger headings in the same grey may pass. Disabled UI states are exempt from contrast requirements under WCAG 1.4.3, though the WCAG working group recommends meeting the threshold anyway.

---

## Example 3: Decorative image incorrectly described (Minor)

**Title:** Decorative onboarding illustrations announced as "graphic" with no context

**WCAG Criteria:**
- 1.1.1 Non-text Content (Level A)

**Affected Users:** Screen reader users on iOS (VoiceOver) and Android (TalkBack) , primary impact, particularly affecting first-time users during the onboarding flow. Users with cognitive disabilities who are slowed down by extraneous announcements.

**Severity:** Minor , Screen reader users hear "graphic" announced three times during onboarding with no useful content. It doesn't block task completion but adds noise that slows navigation.

**Description:**
The onboarding carousel uses three decorative illustrations (a waving character, a map with pins, a stylised calendar). These illustrations are purely decorative , the meaningful content is in the heading and body text beside each one. However, each `<img>` has no `alt` attribute, so VoiceOver reads them as "graphic, image" with no way to skip.

**Steps to Reproduce:**
1. Enable VoiceOver
2. Open the app for the first time to trigger onboarding
3. Swipe right through the carousel
4. Observe that each slide announces "graphic, image" before the heading

**Expected Behaviour:**
Decorative images should be hidden from the accessibility tree entirely. VoiceOver should move straight from the previous slide's content to the next slide's heading with no image announcement in between.

**Proposed Solution:**
```html
<img src="onboarding-1.svg" alt="" aria-hidden="true">
```

For Flutter:
```dart
ExcludeSemantics(
  child: Image.asset('assets/onboarding-1.png'),
)
```

**Why This Fix Works:**
`alt=""` plus `aria-hidden="true"` (belt and braces , some older screen readers respect one but not the other) removes the image from the accessibility tree. The screen reader skips it entirely rather than announcing a placeholder.

**Common False Positives:**
If the illustration *does* carry meaning not present in surrounding text (e.g. an infographic conveying data), it's not decorative , it needs a descriptive `alt`. The test: if you removed the image and replaced it with nothing, would the user lose information? If no, it's decorative.

---

## Tips for writing good tickets

- **Reproduce with specifics.** "VoiceOver doesn't read it" is vague. "VoiceOver on iOS 17, iPhone 14, swiping right from the heading, announces 'button' with no label" is actionable.
- **Name the affected users specifically.** "Screen reader users" is OK; "Screen reader users on iOS (VoiceOver), keyboard-only users, and users with cognitive disabilities" is better. Specificity drives prioritisation and grounds the issue in real human impact , which is what gets the ticket fixed.
- **Give the fix, don't just describe it.** Engineers ship faster when they can copy-paste.
- **Explain the principle.** The "Why this fix works" section is where knowledge transfers. One well-explained ticket teaches the whole team.
- **Flag false positives upfront.** It saves a round-trip when a reviewer asks "but what about decorative images?"
- **Match severity to user impact, not effort.** A one-line `alt=""` fix can still be a blocker.
