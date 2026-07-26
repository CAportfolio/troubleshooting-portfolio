# Case 3 — Scheduled data sync failing with a timeout

**Context**

A client ran a scheduled job that synced a directory of user records from their internal database into the platform via the platform's data-integration agent, which runs on the client's own server. The job had started failing.

**Symptom as reported**

The client's team forwarded agent logs showing repeated errors: a sync to the platform's servers timing out after 60 seconds, then a critical error aborting the process.

**Investigation**

The three log lines the team highlighted looked like three failures but were one: the agent's error handler wraps and re-throws, so each line nested the previous one. The actual error was a single 60-second timeout on the push to the platform.

The timestamps told me more than the error text. The job fetched around 2,500 records, spent about sixteen minutes marking them, ran deletions, then hit the timeout exactly sixty seconds after the push began — and aborted one second later with no retry. Two things stood out: the whole record set plus deletions were going up in a single request, and there was no retry at all, which suggested the agent was old enough to predate automatic retries.

I looked for the agent's version in the log. It reports its version in the user-agent string it sets for its outgoing requests, near the top of the boot sequence. It was several years and several releases behind current. Checking that version against the agent's changelog, I found the specific release that would matter: a later version had introduced batch processing for exactly this scenario — large record counts in the sync mode this job used. Without it, everything went in one request, which was overrunning the 60-second window. The missing auto-retry was explained by the same age.

I also noticed in the logs that post-sync hooks were enabled and running one record at a time, which on a 2,500-record set is a large amount of serialised server-side work inside the same operation — a plausible secondary contributor, though the version gap was the clear primary cause.

**Root cause**

The client's data-integration agent was running a version several releases old, missing the batch-processing release that handles large record counts. The full record set was being sent in a single request that could not complete within the platform's 60-second timeout, and the old version did not retry.

**Resolution & prevention**

The agent runs on the client's infrastructure, so the update is theirs to apply. I wrote them clear upgrade steps — stop the service, update the package, restart — and flagged the one likely snag, that the update source needed to be reachable through their firewall. I gave them the changelog so their administrator could verify the version claims rather than take my word for it.

I was careful in the write-up not to frame this as the client ignoring warnings: the update-notification feature was itself added one release *after* the version they were on, so their agent had no way to tell them it was stale. I raised the broader question internally — how many other agent installations might be on old versions with no mechanism to surface it — since the same silent staleness would apply to any of them. I also flagged a data issue I spotted in the logs, a duplicate primary key differing only by letter case, which was causing one record to be skipped and was unrelated to the timeout.

**Skills demonstrated**

Log analysis, reading nested error wrapping, timestamp correlation, identifying software version from runtime output, changelog-driven diagnosis, distinguishing primary from secondary causes, client-facing remediation writeup, systemic-risk escalation.
