# Case 4 — Directory screen showing no content after a login change

**Context**

A custom directory screen on an event app pulled records from a data source and personalised the view for the logged-in user. After the login method on the app was changed, the screen showed no content and some interface elements disappeared.

**Symptom as reported**

"The check-in and scan buttons are gone from the home screen and I don't know why" — and separately, the directory list showing "no upcoming sessions" where it should have shown records.

**Investigation**

The console had a clear chain in it, and the trick was reading it in the right order. At the top: a data-source query returning a 400 Bad Request. Immediately after: a type error — "cannot read properties of null" — on a user object. The screen's code took the user object returned by a session lookup and immediately read a property from it; if that object was null, the read threw, and the surrounding error handler caught the exception and returned nothing. With nothing returned, the list rendered zero records and the conditionally-shown buttons never appeared. One failure, several visible symptoms.

My first hypothesis was that the login change had moved the user's session data to a different location, so the lookup was returning null. That would have been a tidy explanation, so I tested it rather than assuming it. I logged the session object and inspected it: the session was fully populated and the user data was exactly where the lookup expected it. My hypothesis was wrong — the login change hadn't moved anything.

That sent me back to the 400. The response body said "invalid request data" — the platform rejecting the query payload, not a permissions or connectivity problem. But when I looked at the timing, the errors only appeared on loads where no user was established yet. Re-running the checks once properly logged in, the 400 and the null error were both gone. The earlier errors had been from a load that happened before the session was ready — a timing artefact, not the live fault.

With that cleared, the disappearing buttons turned out to be a separate issue that the login change had merely coincided with. The buttons were hidden by default in the stylesheet and only revealed when a class was added by script. The script that added that class — which had previously gated the buttons on user role — was no longer present. Nothing was adding the class, so the buttons stayed hidden.

**Root cause**

Two distinct issues that looked like one. The console errors were a load-order timing artefact from before the session was ready, not a persistent fault. The missing buttons were caused by revealing logic that was absent: the stylesheet hid them pending a class that no script was applying.

**Resolution & prevention**

For the buttons, the client confirmed all attendees should see them, with no role gating, so the fix was to add the revealing class unconditionally on load — the styling for the two-button layout was already in place and worked once the elements were shown. I also added a guard so that a null user object degrades gracefully — the personalisation is skipped but the list still renders — rather than throwing and taking out the whole screen, which is what had made a minor timing issue look like a total failure.

**Skills demonstrated**

Reading a causal chain from console output, hypothesis testing and rejection, distinguishing a transient artefact from a live fault, separating coincident issues, defensive-coding fix (null guarding), CSS-and-script interaction debugging.
