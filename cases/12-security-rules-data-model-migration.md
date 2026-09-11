# 12. Data source security rules blocking reads after a data model migration

## Context

A crisis management app built on a low-code platform for a compliance-heavy client. The app allows users — both internal employees and external company contacts — to report live incidents. When an incident is triggered, the app runs a chain of operations across four data sources: Incident Log, Client Documents, Client Contacts, and Email and SMS Notifications. Each data source had security rules configured to scope access by company.

The app serves two distinct user types whose session data is structured differently: internal users have a `companyname` field in their session, external users have a `Company` field. During an unrelated fix to make the app work correctly for both user types, the decision was made to migrate all company-level identification to a `Company ID` field — a per-company GUID that exists in the session for all user types. The security rules on each data source were updated to require `Company ID` in the request data rather than `companyname`.

---

## Symptom as reported

After the data model migration, triggering a live incident report produced an error: "The security rules for the Data Source 'Incident Log' do not allow this app to read data." The incident was not being logged and the app was not navigating to the confirmation screen.

---

## Investigation

My first step when a security rule error appears is to go directly to the data source in Studio and read the rules as configured. This is faster than reasoning about the code because the rules are the authoritative source — if the rule says one thing and the code sends another, the rule wins regardless of what the code intends.

The Incident Log security rules showed two enabled rules: one for admin users with full read/write/update/delete and no request data requirement, and one for logged-in users with read/write/update, requiring `Company ID` equals `{{user.[Company ID]}}`.

The rules looked correct. I moved to the code to check what the queries were actually sending.

The incident chain starts in `reportRaid()`, which checks for `user['Company ID']` before calling `updateIncidentLog()`. Inside `updateIncidentLog()` I found a `find()` call at the top of the function:

```javascript
return connection.find({
  where: { companyname: user.companyname }
})
```

This query was still using `companyname` — it had not been updated during the migration. The security rule required `Company ID` in the request data, but this query was sending `companyname`. The rule blocked the read, which broke the promise chain before the insert could run, which is why the app never navigated to the confirmation screen.

I also found the same pattern in `updateClientDocuments()` and `updateClientContacts()`, which were both querying by `companyname`. For external users `companyname` would be undefined anyway — their session uses `Company` — so these queries would have returned empty results even if the security rules had allowed them through.

There was a further issue in `sendPushPlusInappNotifications()`: the Crisis Users query still contained a hardcoded string rather than a dynamic value:

```javascript
where: { companyname: 'COMPANY' }
```

This had been a placeholder that was never replaced, meaning push notifications and emails were being scoped to a literal company name rather than the reporting user's company. This bug predated the migration and was unrelated to the security rule error, but surfaced during the same investigation.

I updated all four functions to query by `Company ID`:

```javascript
where: { 'Company ID': user['Company ID'] }
```

I also updated the insert in `updateIncidentLog()` to include `Company ID` in the written data, which was required to satisfy the security rule on write as well as read.

After the code fix, a further error appeared on the welcome screen: the same "does not allow this app to read data" message on page load. This came from a separate function, `Check_Incident_Log()`, in the screen-level JS rather than the global JS. That function was also still querying by `companyname`. Updated to `Company ID` and the error cleared.

---

## Root cause

A data model migration updated security rules on four data sources to require `Company ID` in request data, but the corresponding queries in the application code were not fully updated. Several `find()` calls continued to send `companyname`, which the security rules rejected. A fifth location — a screen-level function separate from the main chain — was also missed. An unrelated pre-existing bug (hardcoded company name string in the notifications query) was discovered in the same pass.

---

## Resolution and prevention

Updated all `find()` and `commit()` calls across the global JS and screen JS to use `Company ID`. Updated the incident log insert to include `Company ID` in the written data to satisfy write-side security rules. Replaced the hardcoded `'COMPANY'` string with `user['Company ID']`.

The migration was complete and the incident chain ran end-to-end in testing. Emails were confirmed delivered to the correct company's users only.

The missed location (screen-level JS) is a known risk pattern on this platform: logic is split between global JS and per-screen JS, and a search across only one layer will miss references in the other. A complete migration of a field name requires searching both layers. I'd document this as a checklist item for any future data model changes on this app.

---

## Skills demonstrated

Data source security rule debugging; multi-layer JS architecture (global vs screen scope); data model migration; query/rule mismatch diagnosis; hardcoded value identification; promise chain failure tracing.
