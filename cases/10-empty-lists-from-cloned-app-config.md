# Case 10 — Lists showing no results while their data sources were full

**Context**

An internal staff directory app, built by cloning an earlier app and then adapting it. One screen listed committees from a data source and, when a committee was tapped, opened a directory screen filtered to that committee's members. The list screen showed "no matches found" despite its data source being populated and verified.

**Symptom as reported**

"It always shows no matches found when I'm logged in, but the data source definitely has data. If I comment out the screen's custom JavaScript I can see the records again, and there are no console errors."

**Investigation**

The reported cause was compelling and I took it seriously: custom code in, list empty; custom code out, list populated. That is close to a controlled experiment, and it pointed squarely at the screen's JavaScript, which deduplicated records in one lifecycle hook and intercepted taps in another.

My first hypothesis was the deduplication. It kept the first record for each committee name and discarded the rest, so if the column it keyed on didn't exist, every record would evaluate to the same empty key. I worked through what that would actually produce and rejected it: collapsing on a missing key leaves one record, not zero. It could explain a list that was too short, not a list that was empty.

Second hypothesis: the code used a utility library that the platform had recently made an opt-in dependency rather than bundling by default. If it wasn't loaded, the hook would throw, and because these hooks run inside a promise chain, the rejection would be swallowed rather than surfacing — which would also explain the absence of console errors. That was testable, so I rewrote the logic in plain JavaScript with no library dependency. No change.

At this point I had spent two hypotheses on the assumption that "no console errors" was a real observation, so I checked it. It wasn't. The builder previews apps inside an iframe, and the browser console defaults to the top frame. Evaluating the platform's global object returned "not defined", confirming we had been reading the wrong context throughout. Screen JavaScript logs were going somewhere nobody was looking. Several apparently informative results — no errors, no output, code appears not to run — were artefacts of the tool, not findings about the app.

Rather than fight the console, I made the diagnostics visible in the page itself: a small function appending coloured bars to the top of the screen, reporting record counts at each stage of the list's lifecycle. That removed the tooling from the equation entirely.

The result was immediate and unambiguous. Twenty-five records fetched, with the expected columns. Zero records rendering. The loss sat between two specific lifecycle hooks, and reading the component's source, the only operation in that gap was the record-preparation step — which, among other things, applies filters supplied through the screen's URL query parameters.

The screen's URL carried a prefilter: column `Type`, operator `==`, value `Committee`. The data source had no `Type` column. Every record failed a test against a field that didn't exist, so all twenty-five were discarded.

The parameter was not configured on the list component, which is where I and the app's owner had both looked. It was attached to the menu item that opened the screen, in an optional query-parameter field on the link action — invisible from the component settings, invisible from the screen's code, and not something anyone had knowingly set. It came from the app this one was cloned from, where committees and other record types shared a single data source distinguished by a `Type` column. When the data was later split into separate sources, the column disappeared and the parameter became a filter that could never match.

That also dissolved the original evidence. Commenting out the JavaScript hadn't fixed anything. Previewing the screen directly opens it without query parameters; reaching it through the app's menu applies the broken one. The two tests had differed by navigation path, not by code.

The same diagnostic was then run on a second screen in the app that showed identical symptoms. Eighty-four records fetched, zero rendering, same stale parameter with a different value. That took minutes rather than hours, and confirmed the finding was a pattern in how the app had been cloned rather than a one-off.

**Root cause**

A query parameter inherited from the source app during cloning, attached to menu links rather than to the components it affected, filtering on a column that no longer existed in the destination app's data. The filter was applied silently — no error, no warning, no indication in the component's own configuration that a filter was active.

**Secondary finding**

The same screen contained a hard-coded numeric data source identifier, also inherited from the source app. Because data sources are scoped to the organisation rather than to a single app, the query succeeded: it returned several hundred records from a data source belonging to a different app, with no error and no permissions failure. The code appeared to work while operating on the wrong data entirely. This was replaced with the platform's name-based lookup, which resolves correctly after a clone because names are preserved and numeric identifiers are not.

**Resolution & prevention**

The stale parameters were removed from the menu items on both screens, and the lists rendered correctly. Hard-coded identifiers were replaced with name-based lookups. I then swept the rest of the app for both patterns.

The generalisable lesson is about provenance: a cloned app carries configuration that is valid in its origin and meaningless in its destination, and the most dangerous of it fails silently rather than loudly. After any clone it is worth auditing hard-coded identifiers, and query parameters that reference columns, screens or fields which may not have survived the copy.

The secondary lesson is about instrumentation. Two hypotheses were spent on the strength of "there are no console errors", an observation that had never actually been made. When a negative result is doing significant work in an investigation, it is worth confirming that the tool producing it is pointed at the right thing.

**Skills demonstrated**

Hypothesis testing and rejection, distinguishing correlation from causation in a user-reported reproduction, recognising and working around a tooling blind spot, instrumenting an application to localise a fault between lifecycle stages, reading component source to determine what runs between two hooks, identifying configuration inherited through app cloning, recognising a silent wrong-data failure, generalising a single fault into an auditable pattern.
