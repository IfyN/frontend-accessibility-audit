# Flutter / iOS Accessibility Reference

Mobile accessibility differs from web in important ways. Flutter renders its own UI layer rather than using native widgets, which means accessibility is exposed through the `Semantics` tree rather than HTML/ARIA. Read this whole file when auditing a Flutter app.

## The mental model

Flutter's `Semantics` tree is the mobile equivalent of the accessibility DOM. Every widget can either:

- **Add semantics** (most interactive widgets do this automatically , `ElevatedButton`, `TextField`, `Switch`)
- **Merge semantics** (combine child nodes into one for the screen reader)
- **Exclude semantics** (hide a subtree from assistive tech, e.g. for decorative visuals)
- **Override semantics** (provide custom labels, hints, or actions)

When you see a `Container`, `GestureDetector`, or custom painter, assume it has no semantics unless wrapped in `Semantics`.

## Key widgets and properties

### `Semantics`

The general-purpose widget for adding or overriding semantic information.

```dart
Semantics(
  label: 'Close dialog',
  button: true,
  onTap: () => Navigator.pop(context),
  child: Icon(Icons.close),
)
```

Common properties:
- `label` , the accessible name (what VoiceOver announces)
- `hint` , additional context ("Double tap to activate")
- `button: true` , announces as a button
- `header: true` , announces as a heading (rotor-navigable)
- `image: true` , announces as an image
- `excludeSemantics: true` , removes child semantics entirely

### `ExcludeSemantics`

Use for decorative content that should be invisible to screen readers.

```dart
ExcludeSemantics(
  child: Image.asset('assets/decorative_swirl.png'),
)
```

### `MergeSemantics`

Combines a subtree into a single node , useful for icon + label pairs.

```dart
MergeSemantics(
  child: Row(
    children: [
      Icon(Icons.star),
      Text('Favourite'),
    ],
  ),
)
```

Without `MergeSemantics`, VoiceOver reads the icon and label as separate nodes.

## Mapping the 22 patterns to Flutter

### #1 Modal focus (Flutter)

**Detection:** Custom dialogs built with `showDialog` or `showGeneralDialog` often miss focus management.

**Fix:** Ensure the first focusable element in the dialog receives focus. Use `FocusScope` and `autofocus: true`:

```dart
showDialog(
  context: context,
  builder: (context) => AlertDialog(
    title: Text('Confirm'),
    content: Text('Are you sure?'),
    actions: [
      TextButton(
        autofocus: true,
        onPressed: () => Navigator.pop(context),
        child: Text('Cancel'),
      ),
    ],
  ),
);
```

For custom dialogs, wrap in `FocusScope` and call `FocusScope.of(context).requestFocus()` on open.

### #2 Accordion state (Flutter)

Flutter's `ExpansionTile` handles state announcement automatically via `ExpansionTileController`. For custom accordions, announce state changes with `SemanticsService.announce`:

```dart
SemanticsService.announce('Section expanded', TextDirection.ltr);
```

### #3–#4 Images (Flutter)

```dart
// Meaningful image
Image.asset('assets/chart.png', semanticLabel: 'Q3 revenue chart')

// Decorative image
ExcludeSemantics(child: Image.asset('assets/divider.png'))
```

### #5–#6 Form fields (Flutter)

`TextField` and `DropdownButton` expose their `decoration.labelText` or `hint` as their accessible name. Always set one:

```dart
TextField(
  decoration: InputDecoration(labelText: 'Full name'),
)
```

For complex fields, wrap in `Semantics` with an explicit `label`.

### #8 Icon button missing name (Flutter)

`IconButton` requires `tooltip` for its accessible name:

```dart
IconButton(
  icon: Icon(Icons.download),
  tooltip: 'Download report',
  onPressed: _download,
)
```

A tooltip-less `IconButton` is unlabelled to VoiceOver.

### #10 Contrast (Flutter)

Contrast rules are identical to web. Use Flutter's colour tools or WebAIM's checker. The `ThemeData` colour scheme should define accessible pairs; don't hard-code colours at the widget level.

### #12 Animation (Flutter)

Respect the OS reduce-motion preference:

```dart
final reduceMotion = MediaQuery.of(context).disableAnimations;

AnimatedContainer(
  duration: reduceMotion ? Duration.zero : Duration(milliseconds: 300),
  ...
)
```

### #17 Text scaling (Flutter) , the big one

This is a common finding in production Flutter apps. Flutter does not respect iOS Dynamic Type by default unless you explicitly use `MediaQuery.textScaleFactor` (now `textScaler` in newer Flutter versions).

**Broken:**
```dart
Text('Hello', style: TextStyle(fontSize: 16))
// Renders at 16pt regardless of the user's system setting
```

**Fixed (modern Flutter, 3.16+):**
```dart
Text(
  'Hello',
  style: TextStyle(fontSize: 16),
  textScaler: MediaQuery.textScalerOf(context),
)
```

Or set it globally via `MaterialApp.builder`:

```dart
MaterialApp(
  builder: (context, child) {
    return MediaQuery(
      data: MediaQuery.of(context).copyWith(
        textScaler: MediaQuery.textScalerOf(context).clamp(
          minScaleFactor: 1.0,
          maxScaleFactor: 2.0,
        ),
      ),
      child: child!,
    );
  },
)
```

Also test layouts at maximum text scale , many Flutter screens break at 200% text size because of fixed-height containers.

### #19 Graph not tabbable (Flutter)

Custom-painted charts have no semantics. Wrap them in `Semantics` with a `label` summarising the data, or provide an alternative text table.

### #22 Orientation (Flutter)

**Broken:**
```dart
// in main.dart
SystemChrome.setPreferredOrientations([
  DeviceOrientation.portraitUp,
]);
```

**Fix:** Remove the orientation lock unless there's a specific WCAG-exempted reason. Build layouts that work in both orientations using `OrientationBuilder` or `LayoutBuilder`.

## VoiceOver-specific gotchas

- **Reading order follows widget tree order, not visual position.** A `Stack` can visually reorder children but VoiceOver reads them in declaration order. Use `Semantics(sortKey: OrdinalSortKey(n))` to override.
- **`GestureDetector` does not announce as tappable.** Wrap in `Semantics(button: true, onTap: ...)` or use a proper button widget.
- **Custom tab bars often miss `selected: true`.** Screen readers can't tell which tab is active without it.
- **Form validation errors** need to be announced via `SemanticsService.announce` , otherwise VoiceOver users won't know the form failed.

## Flutter-specific production issues

These eight patterns were found in real Flutter/iOS audits and extend the main 22-pattern library. They're Flutter-specific because they arise from how Flutter renders its own UI rather than using native widgets , so they don't map cleanly onto the web patterns.

### A. Dropdown announced only as "Button"

**WCAG:** 4.1.2 Name, Role, Value (A)
**Severity:** Blocker , users don't know what the control is for or what's selected.

**Detection:** Focus a `DropdownButton` with VoiceOver on. If it announces "button" with no label and no current value, this is the issue. Same applies to `DropdownMenu` in older Flutter versions.

**Broken:**
```dart
DropdownButton<String>(
  value: _country,
  items: _countries.map((c) => DropdownMenuItem(value: c, child: Text(c))).toList(),
  onChanged: (v) => setState(() => _country = v),
)
```

**Fixed:**
```dart
Semantics(
  label: 'Country',
  value: _country ?? 'Not selected',
  hint: 'Double tap to choose',
  child: ExcludeSemantics(
    child: DropdownButton<String>(
      value: _country,
      items: _countries.map((c) => DropdownMenuItem(value: c, child: Text(c))).toList(),
      onChanged: (v) => setState(() => _country = v),
    ),
  ),
)
```

**Why:** `DropdownButton` doesn't expose a label or current value on iOS by default. Wrapping in `Semantics` gives it a proper name and announces the selection; `ExcludeSemantics` on the inner button prevents double announcement. Alternative (cleaner if you control the form layout): pair the dropdown with a visible `Text` label and rely on that visually, plus explicit `Semantics` for the screen reader.

**False positives:** `DropdownMenu` in Flutter 3.19+ has better default semantics , check the rendered announcement before wrapping.

### B. Text field focus doesn't move to the keyboard

**WCAG:** 2.4.3 Focus Order (A), 3.3.2 Labels or Instructions (A)
**Severity:** Major , friction rather than blocker, but multiplies across every form.

**Detection:** Activate a `TextField` with VoiceOver. Does the keyboard appear *and* does VoiceOver focus move into the input? If the keyboard opens but VoiceOver stays on the previous element, the field and input focus are desynced.

**Root cause:** On mobile, accessibility focus (VoiceOver cursor) is decoupled from input focus by design. Calling `FocusNode.requestFocus()` moves input focus and opens the keyboard but does not move the VoiceOver cursor.

**Fix:**
```dart
// When you programmatically focus a field (e.g. after validation):
_emailFocusNode.requestFocus();

// Also request accessibility focus:
SemanticsService.announce('Email field focused', TextDirection.ltr);
// Or, for precise focus:
_emailFieldKey.currentContext
    ?.findRenderObject()
    ?.sendSemanticsEvent(const FocusSemanticEvent());
```

For the common pattern of moving to the next field on submit, rely on `textInputAction: TextInputAction.next` and `onFieldSubmitted` , these correctly move both input and accessibility focus on recent Flutter versions.

**Why:** Explicitly sending a semantics focus event bridges the gap between input focus and the screen reader's focus ring.

### C. VoiceOver rotor doesn't show links

**WCAG:** 2.4.5 Multiple Ways (AA), 4.1.2 (A)
**Severity:** Major , rotor navigation is how screen reader users skim a page; missing links means they have to swipe through everything linearly.

**Detection:** On iOS with VoiceOver, two-finger rotate to "Links" in the rotor. If the rotor shows "No items" on a screen that clearly has links (Terms & Conditions, Privacy Policy, inline help), the links aren't exposed as links.

**Root cause:** Flutter often styles tappable text using `GestureDetector` + `TextStyle(color: Colors.blue, decoration: underline)` , which looks like a link but has no link semantics. The `Semantics.link` flag exists but is commonly omitted.

**Broken:**
```dart
GestureDetector(
  onTap: _openPrivacyPolicy,
  child: Text(
    'Privacy Policy',
    style: TextStyle(color: Colors.blue, decoration: TextDecoration.underline),
  ),
)
```

**Fixed:**
```dart
Semantics(
  link: true,
  child: GestureDetector(
    onTap: _openPrivacyPolicy,
    child: Text(
      'Privacy Policy',
      style: TextStyle(color: Colors.blue, decoration: TextDecoration.underline),
    ),
  ),
)
```

For rich text with inline links, use `Text.rich` with `TextSpan` and `recognizer: TapGestureRecognizer()..onTap = ...`, then wrap the whole `Text.rich` in `Semantics(link: true)` , or split each link into its own widget.

**Why:** `Semantics(link: true)` adds the link trait that the rotor filters on. On Android, TalkBack historically announces `link: true` as "button" , check your target platforms.

**False positives:** Internal navigation (e.g. "Go to Settings") is arguably a button, not a link. Use `link: true` for navigation to external URLs or separate content regions, `button: true` for in-app actions.

### D. Date picker focus moves to background

**WCAG:** 2.4.3 Focus Order (A), 3.2.1 On Focus (A)
**Severity:** Blocker , users can't complete the date selection task.

**Detection:** Open a `showDatePicker` or `CupertinoDatePicker` with VoiceOver on. Swipe right , does focus stay inside the picker, or does it leak to the form/page behind?

**Root cause:** `showDatePicker` opens a modal route that traps *input* focus but VoiceOver can still navigate to background content if the background isn't marked as inert. Custom inline pickers (e.g. `CupertinoDatePicker` placed in a `BottomSheet` without `isDismissible` and `enableDrag` configured correctly) often don't set up the inertness at all.

**Fix for custom pickers:**
```dart
showModalBottomSheet(
  context: context,
  isDismissible: true,
  enableDrag: true,
  builder: (context) => FocusScope(
    autofocus: true,
    child: Semantics(
      scopesRoute: true,
      namesRoute: true,
      label: 'Select date',
      child: _buildDatePicker(),
    ),
  ),
);
```

**Why:** `scopesRoute: true` tells the accessibility tree that this is a new route scope. `namesRoute: true` + a `label` names it. Combined with `FocusScope(autofocus: true)`, focus is both set and contained.

**Additional fix:** For the underlying page, if the modal is non-standard, wrap the background content in `ExcludeSemantics` while the picker is open , but this is fragile, prefer the route-based fix.

### E. Country names truncate / don't flow with enlarged text

**WCAG:** 1.4.4 Resize Text (AA), 1.4.10 Reflow (AA)
**Severity:** Blocker at max scale , users with low vision can't read what they're selecting.

**Detection:** Set iOS text size to the largest Accessibility setting. Open the country picker. Are long names like "Democratic Republic of the Congo" or "Saint Vincent and the Grenadines" truncated with ellipsis? Do they overflow their row?

**Root cause:** Fixed-height list items with `Text` widgets that default to `maxLines: 1` and `overflow: TextOverflow.ellipsis`, combined with no text scaler clamping.

**Broken:**
```dart
SizedBox(
  height: 44,
  child: Text(country.name, style: TextStyle(fontSize: 16)),
)
```

**Fixed:**
```dart
ConstrainedBox(
  constraints: BoxConstraints(minHeight: 44),
  child: Padding(
    padding: EdgeInsets.symmetric(vertical: 8, horizontal: 16),
    child: Text(
      country.name,
      style: TextStyle(fontSize: 16),
      softWrap: true,
      // no maxLines , let it wrap
    ),
  ),
)
```

**Why:** Replacing fixed `height` with `minHeight` + generous padding lets the row grow with text scale. Removing `maxLines` and `overflow: ellipsis` allows wrapping. If you must keep a maximum, use `maxLines: 2` with visible overflow handled by the container.

**False positives:** Some design systems legitimately cap row height for scannability , in those cases, make the full name available on activation (e.g. detail screen or tooltip) and keep the truncated version in the list only.

### F. Country flags overflow when font size increases

**WCAG:** 1.4.4 Resize Text (AA), 1.4.10 Reflow (AA)
**Severity:** Major , visual breakage, often pushes interactive elements off-screen.

**Detection:** Same test as E , max text scale. Do flag icons overlap with text? Do they push content outside the viewport?

**Root cause:** Flag icons sized in logical pixels that don't scale with text, placed in a `Row` without proper flex. When text grows, it pushes into the flag.

**Broken:**
```dart
Row(
  children: [
    SvgPicture.asset(country.flagPath, width: 32, height: 20),
    SizedBox(width: 12),
    Text(country.name),
  ],
)
```

**Fixed:**
```dart
Row(
  crossAxisAlignment: CrossAxisAlignment.center,
  children: [
    ExcludeSemantics(
      child: SvgPicture.asset(
        country.flagPath,
        width: 32 * MediaQuery.textScalerOf(context).scale(1),
        height: 20 * MediaQuery.textScalerOf(context).scale(1),
      ),
    ),
    SizedBox(width: 12),
    Expanded(
      child: Text(country.name, softWrap: true),
    ),
  ],
)
```

**Why:** Scaling the flag with the text scaler keeps the visual ratio consistent. `Expanded` on the text gives it flexible width so it wraps rather than pushing the row wider than the screen. `ExcludeSemantics` on the flag prevents VoiceOver announcing it separately from the country name.

**False positives:** Flags that convey information the text doesn't (e.g. a currency-selection screen where the flag is the primary identifier) should not be excluded from semantics , label them with the country name and exclude the `Text` instead.

### G. Two cards treated as one accessibility item

**WCAG:** 1.3.1 Info and Relationships (A), 4.1.2 (A)
**Severity:** Major , users can't act on the individual items; they hear one long announcement and have no way to select just one.

**Detection:** In a list or grid of cards, swipe right with VoiceOver. If two visually separate cards are announced together as one long string, or activating one triggers the other, their semantics are merged incorrectly.

**Root cause:** One of three common causes:
1. Cards are wrapped in a shared `MergeSemantics` or inherit one from a parent.
2. Cards use `GestureDetector` wrapping multiple children without establishing each as its own semantics node.
3. A `ListTile` is nested inside another `ListTile` or tappable parent, and Flutter merges their semantics.

**Fix:**
```dart
// Give each card its own semantics boundary:
Column(
  children: cards.map((card) => Semantics(
    container: true,  // forces a new semantics node
    button: true,
    label: card.title,
    hint: card.summary,
    onTap: () => _openCard(card),
    child: ExcludeSemantics(  // prevent child widgets adding noise
      child: _buildCardVisual(card),
    ),
  )).toList(),
)
```

**Why:** `container: true` forces Flutter to create a distinct semantics node for this widget rather than merging it with siblings or parents. Combined with a clear `label` and `onTap`, each card becomes its own focusable, activatable item.

**False positives:** A card composed of a title + subtitle + icon *should* be merged into one node (use `MergeSemantics`). The issue is only when two *separate* cards merge with each other.

### H. VoiceOver focus doesn't move to page items on load

**WCAG:** 2.4.3 Focus Order (A), 3.2.1 On Focus (A)
**Severity:** Major , users don't know the page has changed; they have to manually swipe up to find the new content.

**Detection:** With VoiceOver on, navigate from one screen to another via `Navigator.push`. After the transition, does VoiceOver focus land on the new screen's heading/first element? Or does it stay where the previous tap was?

**Root cause:** Flutter's `Navigator` doesn't reliably move VoiceOver focus on `push`, especially on iOS. Known framework issue , see Flutter issue #118397 and related.

**Fix:**
```dart
class NewScreen extends StatefulWidget {
  @override
  State<NewScreen> createState() => _NewScreenState();
}

class _NewScreenState extends State<NewScreen> {
  final GlobalKey _headingKey = GlobalKey();

  @override
  void initState() {
    super.initState();
    // Move VoiceOver focus to the heading after the screen is rendered:
    WidgetsBinding.instance.addPostFrameCallback((_) {
      final context = _headingKey.currentContext;
      if (context != null) {
        context.findRenderObject()?.sendSemanticsEvent(
          const FocusSemanticEvent(),
        );
      }
    });
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Semantics(
        key: _headingKey,
        header: true,
        child: Text('Screen title', style: Theme.of(context).textTheme.headlineMedium),
      ),
    );
  }
}
```

Also wrap the screen's content in `Semantics(scopesRoute: true, namesRoute: true, label: 'Screen name')` to give the route a proper announcement.

**Why:** `addPostFrameCallback` ensures the widget tree is built before we request focus. `FocusSemanticEvent` moves the VoiceOver cursor specifically (not input focus). `header: true` marks the destination as a heading, making it rotor-navigable.

**False positives:** Modal routes (`showDialog`, `showModalBottomSheet`) should handle this automatically. Only apply this pattern when you've verified the issue actually occurs.

---

## Testing on iOS

1. Settings → Accessibility → VoiceOver → On
2. Settings → Accessibility → Display & Text Size → Larger Text (test at max)
3. Settings → Accessibility → Motion → Reduce Motion (test animations)
4. Rotate the device to test orientation
5. Use the rotor (two-finger rotate) to navigate headings, links, form controls

## Flutter tooling

- **`flutter analyze`** , catches some a11y issues
- **`flutter test` + `SemanticsTester`** , write widget tests that assert semantics tree structure
- **Accessibility Inspector** (Xcode → Open Developer Tool) , inspect the live semantics tree on an iOS simulator
- **Accessibility Scanner** (Android) , for TalkBack testing
