---
name: kepion-app-instance-names-accept-special-characters
description: Since Kepion 6.1 the workflow app-instance Name accepts / . and other metadata-invalid characters; before 6.1 it blocked Next/Save with "The name contains invalid character" and forced a rename that split the instance's name across workflow emails
type: reference
classification: public
scope: public
repo: public-knowledge
---

**Symptom (before 6.1):** on the Save, Submit, Approve, Reject or Recall app instance
window, a Name containing `/`, `.` or another character from the metadata
invalid-character list stops Next/Save with *"The name contains invalid character '…'"*.
The message concatenates every offending character, so `/.` means both a slash and a
period. Dates (`16/09/2026`) and version-like strings (`9.41`) are the usual triggers.

**Why it happened:** the window ran the same validation used for metadata identifiers
(object, table and cube names) against the instance Name. The Name is a display label:
it is stored as the submission's tag and substituted into workflow emails as the
submission name. No schema object is derived from it, so identifier rules never applied.

**Fixed in:** 6.1, from build 6.1.26261, and in every later build including the 6.2 line.
The window now only checks that the Name is present. The server never re-validated it.

**Upgrade consequence worth knowing:** instances whose names were written before the fix,
for example by a custom SQL trigger that sets the tag, can be approved unchanged after
upgrading. If the name was edited to get past the old error, emails sent before the
edit show the old name and emails sent after it show the new one. Nothing back-fills
the old name.

**How it was verified (2026-11-03, build 6.2.26302):** a scripted browser run opened the
Save app instance window, typed `16/09/2026 Verify-1018`, saved, and recorded no
invalid-character warning with the window closed. Reopening the window showed the name
stored as typed. The original name was then restored. The approve leg was not run
separately: Save and Approve share the window and the validation that was removed.

**For agents diagnosing a report of this message:** check the product version first.
On 6.1 or later, this message on an app instance Name is not expected, so suspect a
different window or a custom validation script. Below 6.1, the fix is to upgrade. The
workaround is to remove the restricted characters before Next, accepting the email-name
split described above.
