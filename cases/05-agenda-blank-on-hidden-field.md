# Case 5 — Agenda list going blank when a field was hidden

**Context**

An event app used a list component in an agenda layout to display sessions from a data source. The client wanted to stop displaying each session's end time.

**Symptom as reported**

"If I hide the end time, the agenda stops displaying content — it just says 'no upcoming sessions'. How do I remove the end time without breaking it?"

**Investigation**

The obvious explanation was that the agenda layout used the end time to work out which sessions were upcoming, so removing the field had broken that logic. It was a reasonable first guess and I nearly ran with it — but the client had been specific about *what* they changed, and it didn't quite match, so I checked before committing.

The client had only removed the end time from the detail-view display settings. I confirmed how the agenda layout actually decides what to show: it filters on the time fields configured in the summary settings, not the detail-view settings. Those are different configuration areas. So the change the client made shouldn't have touched the field the filter depends on — which meant the empty list probably wasn't caused by the field removal at all, and I was about to fix the wrong thing.

I asked for the console output on a broken load. It showed an error in a custom timezone-conversion step running on the screen, failing before the list rendered — and the list receiving zero records as a result. The blank agenda traced back to that error, not to the hidden end time. Confirming the field removal and the empty list were unrelated saved me from stripping out display settings that were never the problem.

**Root cause**

The empty agenda was caused by a failure in a custom timezone-conversion step on the screen, not by hiding the end time. The two had coincided, and the reported symptom pointed at the wrong cause.

**Resolution & prevention**

For the client's actual goal — hiding the end time visually — the correct approach was to keep the field configured where the component needs it and suppress it in display with a targeted style rule, so the filtering logic keeps its data while the client gets the display they want. My first selector matched the wrong element because it counted element position rather than the specific field; I corrected it to target the field unambiguously and confirmed against the rendered markup. The underlying conversion error was logged as a separate issue to address on its own terms rather than being masked by the display change.

**Skills demonstrated**

Resisting a plausible-but-wrong first hypothesis, distinguishing configuration areas with different responsibilities, tracing a symptom to an unrelated cause, precise CSS selector targeting, verifying against rendered output, separating a display fix from an underlying fault.
