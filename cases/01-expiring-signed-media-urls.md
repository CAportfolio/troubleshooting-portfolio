# Case 1 — Images breaking intermittently in a custom app screen

**Context**

A B2B client had built a custom screen on a low-code platform that referenced images by pasting their URLs — copied from the platform's file manager — directly into the screen's JavaScript. The images displayed correctly at first, then broke at unpredictable intervals. The client had hit this before; each time, they had manually re-copied the URLs to patch it. They came back asking whether there was a more durable fix, and reported that all the images had broken again. The app was not yet live.

**Symptom as reported**

"The URLs change unexpectedly and the images break, so I have to manually update the code with the new URLs. I think the platform changed the file structure of the images."

**Investigation**

I started from the client's own explanation — that the file structure had changed — because if it were true, it would decide the fix. But it was a claim to test, not to accept, so I went looking for evidence before agreeing with it.

The pasted URLs had three parts bundled together: a file ID, the file path, and a signature parameter. My first question was which of those had actually changed, because each implied a different cause. If the file IDs had changed, files had genuinely been deleted and re-uploaded, and the client's theory would hold. If only the signature had changed, nothing had happened to the files at all.

So I compared the file IDs hard-coded in the client's JavaScript against the IDs currently in the file manager for the same images. Every one matched. The files had not moved, been renamed, or been re-uploaded — they were sitting exactly where they always had been, under the same IDs. The only part of each URL that had changed was the signature.

That reframed the problem. This was not a file-structure change; it was the signature expiring. Before recommending anything, I confirmed the current behaviour of the platform's media API against our internal documentation rather than relying on memory, since signing behaviour is the kind of version-sensitive detail I don't trust myself on. That confirmed the intended pattern: resolve the file reference at runtime and let the platform attach a fresh signature on each load, rather than freezing one copied signature into the code.

I also checked what the signature actually contained, because it governed how bad the problem was. It was not a static file link — it was a time-limited credential tied to the session of whoever had originally copied the URL. That meant the client had, in effect, pasted one person's session credential into the app's source, and every user loading the screen had been fetching images using it until it lapsed. That explained the intermittency perfectly: different people copying URLs at different times, each freezing a credential with its own expiry.

**Root cause**

Not a platform change and not a file-structure change. The URLs copied from the file manager carried a time-limited, session-bound signature. Pasting them into source code froze that credential in place; it worked at the moment of copying and failed once it expired. The recurring, random-looking breakage was multiple copied URLs each lapsing on their own schedule.

**Resolution & prevention**

I rewrote the screen to stop storing URLs entirely. Instead of hard-coded links, the code holds the image file names and resolves them to current URLs at load time, letting the platform attach a valid signature on each load. Because nothing is persisted in the source, there is no signature left in the code to expire.

I then wrote the client a plain-language explanation correcting the "file structure changed" framing, since that framing would have led them to keep patching the wrong thing. I was explicit that the files had never moved and that the real issue was expiring credentials in copied URLs.

Separately, I recognised this as a pattern rather than a one-off: any URL copied from the file manager and pasted into code would fail the same way, and the signature being a session credential made it a mild security concern as well as a reliability one. I raised this internally with engineering, with a reproduction and a proposed check to find other apps with the same pasted-URL pattern in their code, so the fix could be applied fleet-wide rather than one client at a time.

**Skills demonstrated**

DevTools inspection, root-cause isolation by hypothesis elimination, distinguishing correlation from causation, internal documentation verification, client communication, pattern recognition across incidents, internal escalation with reproduction.
