# 13. App-wide null crash traced to wrong authentication method in security rule

## Context
Event management app built on a low-code/no-code platform. The app uses 
email/password authentication, storing user session data in a connected 
data source. On load, the Agenda screen runs a hook that calls a global 
`getUser()` function to retrieve the logged-in user's session before 
fetching and rendering list content. Several subsequent functions depend 
on the returned user object being non-null.

## Symptom as reported
The Agenda screen was blank and crashing immediately on load. The client 
had already received advice to log out and back in, which had not helped.

## Investigation
The first visible error was a JavaScript alert on the device:

> Cannot read properties of null (reading 'Type')

The error fired immediately on load, before any data had rendered. My 
first step was to find every reference to `.Type` in the Agenda screen JS. 
The most likely candidate was `filterSessionsForUser`, a global function 
that filters agenda records based on user properties. I read through it 
expecting to find an unguarded `session.data.Type` access on an empty row.

It wasn't there. The function doesn't reference `.Type` at all. None of 
the filter rules touch that property.

I widened the search to Global JS and found the actual site of the crash 
in `checkIfUserHasSpeakerType`:

```javascript
function checkIfUserHasSpeakerType(user) {
  const userType = user['Type'];
  return typeof user.Type == 'string' && user.Type.indexOf('Speaker') > -1
    || Array.isArray(user.Type) && user.Type.includes('Speaker');
}
```

This function is called immediately after `getUser()` resolves in 
`flListDataBeforeGetData`. If `getUser()` returns null, 
`checkIfUserHasSpeakerType(null)` tries to read `null['Type']` and 
crashes before anything else runs.

The question was why `getUser()` was returning null for a logged-in user. 
I added a temporary debug log inside `getUser()`:

```javascript
console.log('SESSION DEBUG:', JSON.stringify(session));
```

The log confirmed the session object existed and contained a valid 
`dataSource` entry with the user's email, name and admin flag. The user 
was authenticated. `getUser()` should have returned their data.

Reading `getUser()` more carefully:

```javascript
function getUser() {
  return Fliplet.Session.get().then(function(session) {
    if (session && session.entries) {
      if (session.entries.dataSource) {
        return { id: session.entries.dataSource.id, 
                 ...session.entries.dataSource.data };
      }
      if (session.entries.saml2) { ... }
      if (session.entries.flipletLogin) { ... }
    } else {
      return null;
    }
  });
}
```

The function checks `session.entries.dataSource` first, which is correct 
for email/password auth. But if the session was created under a different 
auth method, `session.entries.dataSource` would be absent and the function 
would fall through all three conditions and return `undefined` implicitly 
— which behaves identically to null at the call site.

The app security rule was set to require a valid Fliplet login rather than 
email/password. This meant sessions were being created under 
`flipletLogin` rather than `dataSource`, so `session.entries.dataSource` 
was never populated. `getUser()` fell through to an implicit undefined 
return on every call, for every user, regardless of whether they had 
successfully logged in.

I can't reconstruct the exact sequence of steps that led me to check the 
app security rule rather than the data source security rules — that detail 
is not recoverable. Based on the symptom pattern (everything null, no data 
loading, no obvious logic error in the code itself), checking 
authentication configuration was the next reasonable move.

After fixing the security rule to email/password, `getUser()` began 
returning user data correctly. The `.Type` crash was gone. A second error 
immediately appeared:

> Cannot read properties of null (reading 'Email')

This was in the attendance records processing in `flListDataAfterGetData`:

```javascript
const goingRecords = attendanceRecords.filter(r => 
  r.data['RSVP Status'] === 'Going');
const userRSVPAttendanceRecords = goingRecords.filter(r => 
  r.data.Email === loggedInUser.Email);
const userCheckInAttendanceRecords = attendanceRecords.filter(r => 
  r.data.Email === loggedInUser.Email);
```

Empty rows in the Attending Logs data source were returning records with 
`null` data objects. With a valid user now returned by `getUser()`, 
processing had advanced far enough to reach this code. Null guards were 
added to all three lines.

## Root cause
The app security rule was configured to require a valid Fliplet login 
rather than email/password. This caused `getUser()` to return undefined 
on every call because it only checks `session.entries.dataSource`, which 
is only populated for email/password sessions. The null return cascaded 
through every function that depended on the user object being non-null, 
producing immediate crashes on load.

A secondary issue — empty rows in the Attending Logs data source returning 
null data objects — was masked until the primary authentication issue was 
resolved.

## Resolution & prevention
- App security rule changed from "valid Fliplet login" to "email/password"
- Onboarding screen added as a security rule exception to prevent 
  unauthenticated users being caught in a redirect loop before reaching 
  the login screen
- Null guard added to `checkIfUserHasSpeakerType` in Global JS
- Null guards added to attendance record filtering in Agenda screen JS

The null guards are defensive improvements but do not address the class 
of problem. If the security rule is misconfigured again, `getUser()` will 
return undefined silently. A more robust fix would be to add an explicit 
null check and early exit inside `getUser()` itself, with a logged warning, 
so the failure is visible and localised rather than cascading.

## Skills demonstrated
Auth method debugging, session object inspection, null cascade tracing, 
app security rule configuration, defensive null guarding in JS hooks
