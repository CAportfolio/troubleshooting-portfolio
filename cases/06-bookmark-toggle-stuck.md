# Bookmark toggle stuck on a single record for one user

## Context

A B2B client's end-user bookmarking feature on a low-code/no-code software building platform. The platform stores user bookmarks in a dedicated data source, separate from the main content data source. The bookmark toggle is rendered per document card in a dynamically generated list, with state determined at runtime by querying the bookmark data source filtered by the current user's ID.

## Symptom as reported

The client reported that one specific document appeared permanently bookmarked and could not be toggled off. Clicking or activating the bookmark icon on that document had no visible effect — the icon remained in its bookmarked state. All other bookmarks on the same screen were working correctly.

## Investigation

The fact that only one document was affected, while all others toggled correctly, ruled out a general rendering bug or a broken event handler. If the toggle logic were broken, it would fail consistently across all cards. I suspected instead that the data source contained duplicate or conflicting records for that specific document — duplicates are a known cause of silent failures in toggle operations, because the delete query removes one record but the find query still returns another, making the item appear permanently active.

To confirm, I added a temporary debug snippet to the screen's JavaScript to query the bookmark data source directly for that document's ID, running the query twice — once with the ID as a number type and once as a string — to rule out a type mismatch between how the record was written and how it was being queried. Both queries returned the same 8 records regardless of the filter parameter, which was the first concrete signal that something was wrong at the query level rather than the UI level.

Expanding the results in the browser console, I spotted that one of the 8 records belonged to a user from a different domain — not the current test account. That record referenced the exact document that was stuck. This meant the bookmark data source was not being filtered by user at all: the `find()` call in the `loadUserBookmarks` function was returning every record in the table across all users, not just the current user's bookmarks. The stuck document appeared bookmarked because another user had bookmarked it, and that record was being included in the current user's bookmark list.

Tracing back to the code, the `find()` call was written as:

```javascript
bookmarkConnection.find({ userId: currentUserId })
```

The platform's data source API requires filter parameters to be wrapped in a `where` object. Without it, the `{ userId: currentUserId }` argument was being ignored entirely, and the query was returning the full table. The correct syntax was:

```javascript
bookmarkConnection.find({ where: { userId: currentUserId } })
```

## Root cause

Incorrect query syntax in the `loadUserBookmarks` function. The missing `where:` wrapper meant the data source API silently ignored the filter parameter and returned all records across all users. The bug had gone undetected because in most cases each user only bookmarks different documents — the cross-user collision only became visible when two users from different accounts bookmarked the same document.

## Resolution & prevention

Updated the `find()` call to use the correct `where:` syntax. Verified the fix by testing the toggle on the previously stuck document — it toggled off correctly and the bookmarked state no longer persisted across sessions.

## Skills demonstrated

Browser console debugging, data source query analysis, type mismatch investigation, runtime log inspection, API parameter syntax validation, fix verification through UI behaviour testing.
