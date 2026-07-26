# Email links silently skipped by screen reader after a global JS fix

## Context

A B2B client's portal built on a low-code/no-code software building platform, undergoing a WCAG accessibility audit. The portal contains email address links (`mailto:` href) embedded within paragraph text blocks across multiple screens. A global JavaScript function had previously been implemented to address a VoiceOver issue where surrounding paragraph text was being read alongside email links — this function set `aria-hidden="true"` on the parent `<span>` wrapping each email link, with the intention of hiding the span from VoiceOver's reading order whilst keeping the `<a>` tag itself accessible.

## Symptom as reported

The client submitted a video showing that email links were being skipped entirely by their screen reader during tab navigation. The links were visible on screen and functional with a mouse, but the screen reader did not announce them at any point — neither when tabbing nor when navigating by element.

## Investigation

The symptom differed from the inline link issue investigated in the previous case. There, the platform's component rendered anchor tags without `href`, causing NVDA to treat them as inert. Here, the email links had standard `href="mailto:"` attributes, which NVDA should recognise natively. The fact that NVDA was skipping them despite valid `href` values pointed to something actively suppressing them rather than a missing attribute.

I isolated the affected elements by reviewing the rendered HTML for the email link sections. The `<a>` tags themselves had `aria-hidden="false"` set explicitly, along with `aria-label` and `tabindex="0"` — all correct. However, inspecting one level up in the DOM revealed that the parent `<span>` had `aria-hidden="true"` set on it. This had been applied by a global JavaScript function — `fixEmailParentContext` — that was added earlier in the project to prevent VoiceOver from reading the full surrounding paragraph text as context when a user tabbed to an email link.

The function's logic was: set `aria-hidden="true"` on the parent span to hide it from VoiceOver's reading tree, then set `aria-hidden="false"` on the child `<a>` to keep the link itself visible. This approach works inconsistently across screen readers. VoiceOver respects the child override in some contexts; NVDA does not — when a parent element has `aria-hidden="true"`, NVDA excludes the entire subtree from its accessibility tree regardless of child overrides, meaning the email link was invisible to NVDA even though it appeared correctly in the DOM.

The fix was to remove `aria-hidden="true"` from the parent spans entirely, and remove the `fixEmailParentContext` function from the global JavaScript. The `aria-label` and `role="link"` attributes on the `<a>` tags themselves were sufficient for NVDA to announce the links correctly. The VoiceOver paragraph context issue that had originally prompted `fixEmailParentContext` was documented as a known platform limitation rather than something requiring a code workaround.

## Root cause

A global JavaScript function intended to improve VoiceOver behaviour was setting `aria-hidden="true"` on parent `<span>` elements wrapping email links. NVDA does not honour `aria-hidden="false"` overrides on child elements when a parent is hidden — the entire subtree is excluded from its accessibility tree. This caused NVDA to skip the email links entirely during tab navigation.

## Resolution & prevention

Removed the `fixEmailParentContext` global JavaScript function and stripped `aria-hidden="true"` from the affected parent spans in the HTML across the relevant screens. Verified that NVDA announced the email links correctly after the fix; client confirmed the same on their end.

For prevention: `aria-hidden` inheritance behaviour differs between VoiceOver and NVDA. Setting `aria-hidden="true"` on a parent and attempting to override it with `aria-hidden="false"` on a child is unreliable across screen readers and should be avoided. If an element needs to be accessible, the parent must not be hidden.

## Skills demonstrated

Cross-screen-reader ARIA inheritance analysis (VoiceOver vs NVDA), DOM inspection via browser dev tools, global JavaScript audit, `aria-hidden` subtree behaviour, conflict identification between two accessibility fixes, WCAG 4.1.2 name/role/value compliance.
