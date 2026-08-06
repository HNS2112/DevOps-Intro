# Runbook — HighErrorRate

**Alert:** `HighErrorRate`
**Severity:** `page`
**Expression:** 4xx+5xx ratio > 5% sustained for 5 minutes

---

## What this alert means

More than one in twenty requests to QuickNotes is failing, and has been for at
least five minutes — users are being turned away right now.

---

## Triage steps

1. **Confirm the service is up at all.**
A non-200 or a container not in `healthy` state means this is an outage, not
   an error-rate problem — skip to Mitigation 1.

2. **Find out which status code dominates.** Open the Errors panel on the
   *QuickNotes — Golden Signals* dashboard, or query Prometheus directly:
4xx means clients are sending bad requests — a broken caller, a bad deploy on
   their side, or someone probing. 5xx means QuickNotes itself is failing.

3. **Check whether a deploy correlates with the onset.** Compare the time the
   ratio started climbing against the container start time:
If the error onset lines up with a restart, treat the deploy as the cause
   until proven otherwise.

4. **Check disk and the data volume.** The store writes to `/data`; a full or
   read-only volume produces 5xx on every write:
---

## Mitigations

1. **Roll back to the previous image.** Fastest fix when the onset correlates
   with a deploy. Stop the current container first — starting a second instance
   on the same port fails with `bind: address already in use`:
Verify with `curl -s http://localhost:8080/health` before declaring recovery.

2. **Restart the container.** Clears wedged state such as an exhausted
   connection pool or a stuck file handle. Data survives — it lives in the named
   volume, not the container:
3. **If the errors are 4xx from a single caller**, the application is behaving
   correctly and the fix is upstream. Rate-limit or block that client at the
   proxy rather than changing QuickNotes.

---

## Post-incident

Write a blameless postmortem covering: timeline (first bad request → alert fired
→ mitigation → recovery), user impact in requests and minutes, root cause, and
the specific change that prevents recurrence. Follow the structure used in
`submissions/lab4.md` §2.4 — summary, timeline, what went wrong, why it was hard
to see, what prevents it.

Then check this alert itself: if it fired and no user was actually affected,
that is a tuning bug, and it belongs in the postmortem's action items.
