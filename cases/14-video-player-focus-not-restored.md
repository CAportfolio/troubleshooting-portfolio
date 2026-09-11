# Keyboard focus lost after closing an in-app video player

## Context

A B2B client's informational portal built on a low-code/no-code software building platform, undergoing a WCAG accessibility audit. The portal included a screen with a button that, when activated, played an introductory video using the platform's native video player. The video file was stored in the platform's built-in file manager and linked directly to the button's action. The screen had no custom JavaScript — all behaviour came from the platform's standard components.

## Symptom as reported

The client reported a keyboard navigation issue: after closing the video player, they could no longer tab through any elements on the screen. Focus did not return to the button that had triggered the video, nor to any other element. The only way to restore normal keyboard navigation was to click somewhere on the page with a mouse.

## Investigation

My first step was to reproduce the issue. I tested on both Google Chrome and Microsoft Edge, since the client's organisation uses Edge in some environments. The behaviour was consistent across both browsers — after closing the video player, keyboard focus was lost entirely on every attempt.

While reproducing the issue I also examined how the video had been set up. The file was stored in the platform's file manager and linked to a button action configured to "play a video." This is the standard approach for in-app video on this platform, and it works — the video plays correctly. However, the platform's native video player is intentionally lightweight. It is not a full-featured player in the way a browser-embedded YouTube or Vimeo player would be. Expecting it to handle focus management, accessibility hooks, or playback controls beyond the basics is a misalignment between client expectation and platform capability. This framing became relevant later.

I then attempted to identify what was happening at the DOM level when the video closed, with the goal of writing a JavaScript workaround to restore focus programmatically. Inspecting the rendered HTML while the video was playing showed a `<video>` element with `id="flLinkVideo"` rendered directly on the `<body>` — no wrapping modal or overlay container. This was significant: without a wrapper, there was no container element to watch for removal or visibility changes.

I attempted three successive approaches to catch the moment the video closed and fire a focus restoration call:

First, a `MutationObserver` watching for the `<video>` element being removed from the DOM. This did not fire — the element was not being removed when the player closed, only hidden or repositioned.

Second, a polling interval checking every 500ms for the element's presence and visibility. This also failed to detect the close event reliably.

Third, attaching event listeners directly to the video element for `ended` and `pause` events, with logic to distinguish a genuine close from a mid-playback pause. The browser console confirmed an uncaught error during playback — `screen.orientation.lock() is not available on this device` — which the platform's player was throwing internally. This error interrupted the player's close sequence in a way that made it impossible to intercept cleanly from outside.

At this point all client-side approaches had been exhausted. The consistent conclusion across every diagnostic attempt was that the platform's video player does not expose a close or dismiss event, and does not restore focus to the triggering element when dismissed. This is not detectable or patchable from custom JavaScript running on the same screen.

## Root cause

The platform's native video player does not manage keyboard focus on close. When the player is dismissed, focus is not returned to the triggering element or any other focusable element on the page, leaving keyboard-only users stranded. This is a platform-level gap rather than a configuration error or a bug in the client's implementation — the video file, storage method, and button configuration were all correct. The lightweight nature of the platform's player means focus management is simply not part of its feature set.

## Resolution & prevention

No client-side fix was possible. The issue was escalated to the platform's engineering team as a feature request, with a written description of the behaviour, the reproduction steps, and the app details. The client was informed that this was a platform limitation rather than something in their configuration, and that it had been raised for a future platform update.

For future projects: clients expecting full accessibility compliance from in-app video should be advised at scoping stage that the platform's native video player is lightweight and does not currently support focus management on close. If WCAG keyboard compliance for video is a hard requirement, the recommendation would be to embed video via an external provider whose player handles focus natively.

## Skills demonstrated

Cross-browser reproduction (Chrome and Edge), DOM inspection via browser dev tools, MutationObserver and polling-based diagnostic approaches, video event listener testing, platform capability assessment, feature gap escalation, client expectation management.
