# Inline links not detected by screen reader on Windows

## Context

A B2B client's portal built on a low-code/no-code software building platform, undergoing a WCAG accessibility audit. The portal uses the platform's native inline link component to render clickable text links within paragraph blocks. These links navigate to other screens within the app or open external URLs, and are rendered at runtime as `<a>` tags by the platform's component engine — without a standard `href` attribute, since the platform handles navigation internally.

## Symptom as reported

The client submitted a video showing that their screen reader was skipping inline links entirely when tabbing through the portal. The links were visible on screen and clickable with a mouse, but the screen reader passed over them without announcing them.

## Investigation

My first test was to replicate the issue using VoiceOver on macOS. VoiceOver did pick up the links, though inconsistently — sometimes skipping them, other times reading surrounding paragraph text alongside the link rather than the link label alone. This told me the issue was not a straightforward missing-element problem, but potentially a screen reader compatibility issue rather than a universal rendering failure.

I asked the client which screen reader they were using. They confirmed NVDA on Windows. This was significant: NVDA and VoiceOver handle non-standard anchor elements differently. VoiceOver is more forgiving of anchor tags that lack an `href` attribute, while NVDA relies more strictly on the presence of semantic role information to identify interactive elements.

Inspecting the rendered HTML in browser dev tools confirmed that the platform's inline link component outputs `<a>` tags with no `href` — navigation is handled by a JavaScript click handler instead. NVDA requires either a valid `href` or an explicit `role="link"` to recognise an anchor as a link. Without either, it treats the element as inert text and skips it during tab navigation.

I also confirmed that the links had `tabindex="0"` set in some cases but not consistently, and that no `aria-label` was being applied to communicate link purpose to the screen reader.

I identified that the fix needed to be applied globally rather than per screen — the inline link component is used across dozens of screens in the portal, and patching each screen individually would be unscalable and difficult to maintain. A single global JavaScript function targeting all rendered inline link anchor tags was the correct architectural approach.

The global fix injected `role="link"`, `tabindex="0"`, and an `aria-label` (appending context such as "navigates to another page" or "opens email client" depending on link type) onto every inline link anchor tag at runtime via a `MutationObserver`, ensuring it applied to dynamically rendered content as well as static content.

## Root cause

The platform's inline link component renders anchor tags without an `href` attribute, relying on JavaScript for navigation. NVDA requires either a valid `href` or an explicit `role="link"` to identify an element as a link during tab navigation. Without this, NVDA treated the elements as non-interactive and skipped them entirely.

## Resolution & prevention

Implemented a global JavaScript function using a `MutationObserver` to inject `role="link"`, `tabindex="0"`, and contextual `aria-label` values onto all inline link anchor tags across the portal at runtime. Applied globally rather than per screen to ensure scalability and consistent coverage. Verified the fix with VoiceOver on macOS; client verified with NVDA on Windows.

This is a known limitation of the platform's component rendering approach for screen reader compatibility — the fix is a client-side workaround rather than a platform-level resolution, and should be documented as such for any future accessibility audits on the same platform.

## Skills demonstrated

Cross-platform screen reader compatibility analysis (VoiceOver vs NVDA), DOM inspection via browser dev tools, ARIA attribute implementation (`role`, `tabindex`, `aria-label`), MutationObserver pattern for dynamic content, global vs per-screen fix architectural decision, WCAG 2.4.4 link purpose compliance.
