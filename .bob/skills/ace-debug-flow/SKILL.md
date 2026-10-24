---
name: ace-debug-flow
description: Use when the user has a problem with their flow or their
metadata:
  version: 1.0.0
---

# ACE Debug Flow skill

## Purpose

Help to determine and fix errors in App Connect integration flows.

## When to use

- The user asks to "fix" a flow for an ACE Application project.

### Procedure

- Check Runtime is Ready.
- Check Logs for errors.
- Turn on User Trace if needed.
- Invoke Flow and check it does what is expected.
- Collect User Trace.
- Determine changes needed to Runtime, Configuration or Flow.
- Suggest fixes needed.
- Prepare fixes and prompt user to confirm with a clear idea of how this should fix the problem and the expectation.
- Deploy any fixes as needed.
- Abide by rate limits and only check again IF things have changed to fix something.
