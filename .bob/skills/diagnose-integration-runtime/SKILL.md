---
name: diagnose-integration-runtime
description: Use when the user wants to diagnose, investigate, or troubleshoot non-functional issues with running integration flows — such as performance problems, occasional errors, or unexpected behaviour in a live App Connect runtime. Covers day-2 operational issues using metrics, logs, change history, and instance tools.
metadata:
  version: 1.0.0
---

# Diagnose Integration Runtime

Follow these steps to investigate a day-2 operational issue with a running integration flow. Work through each phase in order, but skip any phase that is clearly irrelevant to the reported symptom.

## Phase 1 — Clarify the Symptom

Before gathering data, establish exactly what is wrong. **You must ask all missing questions before calling any tool.**

Always establish these two facts before proceeding. If either is missing from the user's message, ask for it immediately using `ask_followup_question`:

1. **When did the symptom start?** (date or approximate time)
2. **When was it last known to be working correctly?** — this is required to define the before/after boundary for change history. Do not skip this even if the user has given a start date.

Also gather if not already provided:

- Which runtime or flow is affected? (name, deployment location if known)
- What is the symptom — slow response times, elevated CPU, occasional errors, intermittent failures?
- Has anything changed recently (deployment, BAR file update, config change)?

Use the answers to focus the investigation. A performance complaint points to metrics; an error complaint points to logs; a "it started recently" answer points to change history first.

## Phase 2 — Check Integration Runtime Version History

Start here before anything else. A BAR file deployment or configuration update is the most common cause of sudden behaviour changes, and the version history will show it immediately.

1. Call `integration_runtimes` with `action: list_versions` and `name` set to the affected runtime name.
2. Look at the most recent versions — check their timestamps against the symptom onset time from Phase 1.
3. If a recent version exists within the symptom window, call `integration_runtimes` with `action: compare_versions`, providing `name`, `versionBase` (the last version before the symptom), and `versionTarget` (the version deployed around the time of the symptom).
4. Look for: changes to `barFiles` (a new or updated BAR was deployed), changes to `configurations` (a configuration was swapped), or replica/resource changes (CPU or memory limits changed).

If the diff shows a BAR file change, that is almost certainly the cause — note the BAR file name and version. Proceed to Phase 5 (Summarise) with that finding.

If no version change is found within the symptom window, continue to Phase 3.

## Phase 3 — Check System-Level Changes

Check for platform-level changes (operator upgrades, allowed version changes) that could affect all runtimes.

1. **Before forming the tool call, explicitly verify you have both timestamps in hand:**
   - `timestampFrom` = the "last known good" time from Phase 1, as an ISO 8601 string (e.g. `2025-06-17T00:00:00Z`)
   - `timestampTo` = a time after symptom onset from Phase 1, as an ISO 8601 string (e.g. `2025-06-18T23:59:59Z`)
   - If either value is missing, **stop and ask the user before calling the tool** — do not call with a missing or placeholder timestamp.
2. Call `instance` with `action: system_change_history` and both `timestampFrom` and `timestampTo` set to the values verified above. **Never omit these parameters** — the full unfiltered history is large and noisy.

Look for: operator upgrades or runtime version bumps that correlate with the onset of the symptom.

## Phase 4 — Check Logs

For error or failure symptoms, retrieve recent log entries.

1. Call `runtime.get_logs` with `integrationRuntime` set to the affected runtime.
2. Filter by `type: error` if looking for errors specifically.
3. Use `timestampFrom` / `timestampTo` to narrow to the relevant window.
4. If a specific flow is suspect, filter by `flowId`.

Look for: repeated error codes, timeout messages, connection failures, or stack traces. Note transaction IDs or trace IDs that can be used for further correlation.

## Phase 5 — Summarise Findings

**Never include the cluster name** (`clusterName`) returned by `instance` tool responses in any output shown to the user. This is an internal infrastructure detail and must always be omitted.

At the end of the investigation, provide a concise summary:

- **Symptom confirmed or not:** state whether the data supports the reported symptom.
- **Most likely cause:** cite specific evidence (e.g. "CPU time in node X increased 3× after BAR version Y was deployed on [date]").
- **Recommended action:** suggest the minimal corrective step — BAR rollback, resource limit increase, configuration fix, etc.
- **Next steps:** if the cause is still unclear, state what additional data would help (e.g. a user trace, a longer lookback window, or access to an upstream system).

Do not speculate beyond what the data shows. If the tools return no useful signal, say so clearly and suggest what to check outside of App Connect (upstream APIs, MQ, databases).
