# Case 9 — App-wide crash traced to a silently dropped package dependency

**Context**

An event app used a shared global script that ran on every screen, handling user session lookup, menu personalisation, and several cross-screen utilities. One screen had previously included a form-based export feature that was later removed from the UI.

**Symptom as reported**

"The app just stops working. I have to keep refreshing, screens disappear, and I have no idea which screen is causing it." No JavaScript changes had knowingly been made to the app by our team; the client had independently edited screens in the interim, including removing UI elements from at least one of them.

The browser console showed a single, generic error: `Cannot read properties of undefined (reading 'get')`, with no useful stack trace — the platform's own error-reporting wrapper caught the exception and logged only the message text, discarding the file and line number.

**Investigation**

The first working theory was that the error was screen-specific and would need to be reproduced and isolated one screen at a time. That approach stalled: the error appeared inconsistently, "the whole app" was too vague a symptom to chase directly, and standard browser debugging (pause-on-exceptions, blackboxing vendor scripts) kept surfacing unrelated internal framework code rather than anything in the app's own script, because the platform's editor environment threw its own internal exceptions constantly.

Rather than continue live-debugging blind, the full global script was read end to end, looking specifically for code that ran unconditionally on every screen load — since an app-wide symptom pointed at something in the shared script, not a single screen. Two candidates were tested by reasoning through what object each depended on and whether that dependency was guaranteed to be present on every screen: a session-lookup call and a notifications-setup call. Both were patched defensively as a precaution, but neither explained the crash — the error persisted unchanged after both fixes, ruling them out.

The client then provided a screen-level script that, when deleted, made the error stop. Reading that script showed a call to the platform's form API, made unconditionally at the top of the file, with no error handling: `platform.FormBuilder.get().then(...)`. The screen this script belonged to had originally included a CSV-export feature built on a form component; the client had since deleted that form from the screen's UI while leaving the script that depended on it in place.

The mechanism: removing a widget from a screen can also drop the underlying package dependency it required, if nothing else on that screen still needs it. With the form gone, the API namespace it belonged to was no longer loaded on that screen. Calling `.get()` on an undefined object threw synchronously, with no `.catch()` to contain it — matching the exact error text reported, and explaining why the failure read as "the whole app," since an uncaught synchronous error early in a shared script can prevent the rest of that script's setup from completing.

**Root cause**

A screen's custom script called a platform API tied to a form component that had been removed from the screen's UI. Removing the form had also dropped the package dependency that API relied on, so the call failed with an undefined-object error. The call had no error handling, so the failure was uncaught and disrupted the screen rather than failing quietly.

**Resolution & prevention**

Rewrote the affected screen's script to remove the entire dead feature — the form-dependent export logic and everything that referenced it — rather than patching around the missing dependency, since the feature itself was no longer wanted. Confirmed the fix by testing in the platform's preview environment before publishing.

For prevention: when a widget or form is removed from a screen, any script still referencing it needs to be checked and removed in the same pass, since the dependency it required may silently disappear from that screen without any warning at build time. The two defensive fixes applied earlier in the investigation (guarding the session and notifications calls with existence checks and `.catch()` handlers) were kept regardless, since an unguarded top-level API call with no error handling is a latent risk independent of whether it was the actual cause this time.

**Skills demonstrated**

Diagnosing a vague, system-wide symptom by reading source rather than continuing to live-debug an uncooperative environment, identifying unconditional top-level code as the right place to look for an app-wide failure, ruling out plausible candidates by testing them rather than assuming, connecting a UI change (form deletion) to a non-obvious downstream effect (dependency removal), and distinguishing a genuine fix from defensive patches that didn't address the actual cause.
