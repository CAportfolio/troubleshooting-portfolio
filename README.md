# Troubleshooting Portfolio

I'm a technical Customer Success specialist with 5+ years supporting a B2B low-code/no-code platform, working the kind of investigations that sit at the Tier 2/3 support boundary: reproducing vague reports, isolating root causes from console and log output, and escalating cleanly when an issue is bigger than one ticket.

This portfolio collects real investigations, anonymised. They span front-end behaviour, a data-integration failure, an API payload rejection, and two cases of expiring signed URLs that turned out to be the same platform-level pattern. In three of the use cases, the cause the user reported was plausible and wrong — the actual work was testing that explanation and rejecting it before acting, which is the habit these cases are meant to show.

Each case follows the same structure: the symptom as reported, the investigation in order with the reasoning behind each step, the root cause, and how it was resolved or escalated. They're written to be judged on whether the reasoning was sound.

## Cases

1. [Images breaking intermittently in a custom app screen](cases/01-expiring-signed-media-urls.md) — signed URLs frozen into source code; root cause isolated by confirming the file IDs never changed
2. [Videos silently failing to play](cases/02-expiring-storage-preview-urls.md) — short-lived storage URLs copied from a preview player; decoded the expiry to prove it
3. [Scheduled data sync failing with a timeout](cases/03-data-sync-agent-timeout.md) — an out-of-date integration agent identified from its runtime version string and changelog
4. [Directory screen showing no content after a login change](cases/04-directory-empty-after-login-change.md) — a reported cause disproved; two coincident issues separated
5. [Agenda list going blank when a field was hidden](cases/05-agenda-blank-on-hidden-field.md) — a plausible-but-wrong first hypothesis, caught before fixing the wrong thing
6. [Bookmark toggle stuck permanently on a single document](06-bookmark-toggle-stuck.md) — cross-user data bleeding into a filtered query; traced by inspecting the raw data source response in the browser console
7. [Inline links invisible to a Windows screen reader](07-nvda-inline-links-skipped.md) — platform renders anchor tags without href; NVDA requires explicit role="link" which the component never sets
8. [Email links silently skipped by screen reader after a global JS fix](08-aria-hidden-email-links/README.md) — a previous VoiceOver workaround set aria-hidden="true" on parent elements; NVDA does not honour child overrides and excluded the entire subtree
