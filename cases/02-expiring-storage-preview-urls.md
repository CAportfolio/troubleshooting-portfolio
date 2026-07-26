# Case 2 — Videos silently failing to play in a custom media screen

**Context**

A different B2B client had a custom video-library screen on the same low-code platform. Video files lived in the platform's file manager; the screen referenced them by URL. The videos would not play. The client intended to move the video references into a data source but wanted to keep using file-manager URLs.

**Symptom as reported**

"I added videos to the file manager that should be referenced by URL, but the video doesn't play and I'm not sure what's happening."

**Investigation**

The browser console showed the video element failing with a "no supported source" error. That error usually means one of two things: the file genuinely isn't a playable format, or the URL isn't returning video at all. The videos were standard files the client had uploaded and played elsewhere, so I doubted the format and looked at what the URL was actually returning.

The URLs were the first thing that didn't fit. The working thumbnails on the same screen used the platform's own media URLs. The video URLs were a completely different shape — direct storage-provider links, on a different host, carrying storage-provider authentication parameters rather than the platform's. So the client had somehow pulled two different kinds of link out of the same file manager.

The storage links carried their own expiry, stated in the query string. I decoded the expiry timestamp on each of the four videos against the time they were signed. Every one had already passed — one had lapsed after a day, three after nearly a week. The storage provider was returning an access-denied response instead of video bytes, and the player was correctly reporting that an error document is not playable video. Nothing was wrong with the player, the format, or the component.

I then traced where the client had got those links, because the fix had to stop them being generated again. Clicking a video in the file manager opens a preview player, and that player streams from a short-lived, signed storage URL. Copying the address out of the preview player yields a link that works at the moment you paste it and dies within hours or days, with nothing in the interface indicating it is temporary.

One more thing to rule out: the screen's code passed every URL through the platform's authentication helper, so on paper it looked like authentication was handled. It wasn't. That helper only augments the platform's own media URLs; pointed at an external storage link it returns the link unchanged, which had masked the real problem by making the code appear to do something it didn't.

**Root cause**

The video URLs were short-lived signed storage links copied from the file manager's preview player, not durable media references. They expired on a fixed timer after copying. The authentication helper in the code gave a false impression of handling this, because it silently no-ops on non-platform URLs.

**Resolution & prevention**

I rebuilt the screen to reference videos and thumbnails by file name rather than URL, storing the names in the client's data source and resolving them to current, correctly-signed URLs at load time. This removed the dependency on copied links entirely and aligned with the client's plan to drive the screen from a data source. I also fixed an unrelated race in the player where playback was being triggered before the source had loaded, and moved a background image out of the stylesheet so it resolved the same durable way.

This was the second client I had seen break the same way through a different action — the first by copying a URL, this one by copying from the preview player — so I raised it internally alongside the first as a single pattern: the file manager hands out credentialed URLs that present as permanent, and users reasonably paste them into code. I included both clients' cases as evidence and proposed a check for the same URL patterns across other apps.

**Skills demonstrated**

Console error interpretation, network-response reasoning, decoding signed-URL expiry, tracing user actions to reproduce, identifying a no-op abstraction masking a fault, cross-incident pattern recognition, internal escalation with corroborating cases.
