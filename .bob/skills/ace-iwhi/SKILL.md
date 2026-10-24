---
name: ace-iwhi
description: Use when the user wants to deploy a BAR file to IWHI (IBM App Connect as a Service), manage integration runtimes, retrieve flow credentials, or test a deployed toolkit flow on IWHI.
metadata:
  version: 1.1.0
---

# ACE IWHI Deployment Skill

## Purpose

This skill covers the full lifecycle of deploying an ACE toolkit flow to an IWHI (IBM App Connect as a Service) integration runtime: uploading a BAR, managing the IntegrationRuntime CR, retrieving credentials, and verifying the deployment.

## When to use

- The user asks to deploy a BAR file to IWHI.
- The user asks to create, update, or delete an IWHI integration runtime.
- The user asks to test or call a flow running on IWHI.
- This skill is referenced from `ace-bar-package` after the BAR is built.

## When not to use

- For building the BAR file itself, use [`skills/ace-bar-package/SKILL.md`](../ace-bar-package/SKILL.md).
- For CP4I on-prem operator deployments — this skill is IWHI-specific.

---

## Deployment procedure

If there are tool problems, consult [`skills/ace-debug-admin/SKILL.md`](../ace-debug-admin/SKILL.md).

### Step 1 — Check for an existing runtime

Use `integration_runtimes` (action: list) to check what runtimes already exist.

**Always filter out `default-integration-runtime` from all results.** This runtime is a system runtime managed internally. Never show it to the user in lists, never suggest deploying to it, and never attempt to delete or modify it.

- If a suitable non-system runtime already exists (e.g. it has `spec.flowType.toolkitFlow: true`) **prefer deploying to it** by updating its `barURL` rather than creating a new one.
- Only proceed to Step 2 if no suitable runtime exists (ignoring `default-integration-runtime`).

### Step 2 — Confirm before creating a new runtime

**Creating an integration runtime on IWHI incurs an ongoing cost while it is running.**

Before calling `integration_runtimes` (action: create), use `ask_followup_question` to warn the user:

> "Creating a new integration runtime on IWHI will incur an ongoing cost while it is running. Do you want to proceed?"

**Do not call `integration_runtimes` (action: create) until the user explicitly confirms.** If the user declines, stop and report that no runtime was created.

### Step 3 — Deploy policy projects as Configuration objects (not BARs)

If the flow requires an MQ endpoint policy or any other policy project, **do not** add the policy project as a BAR in `spec.barURL`. Instead:

1. Create a `policyproject` Configuration object — see [`skills/shared/policy-projects.md`](../shared/policy-projects.md) for the full procedure.
2. Add the Configuration name to `spec.configurations` on the Integration Runtime.

Adding a policy project BAR to `barURL` causes `1357E` and `2113E` errors at runtime.

**Configuration body must use `spec.data`, not `spec.contents`:**

```json
{
  "metadata": { "name": "my-policyproject" },
  "spec": {
    "type": "policyproject",
    "data": "<base64-encoded zip>"
  }
}
```

`configurations` (action: create) returns a misleading `"contents is required"` error. Always use `configurations` (action: replace) for both policy projects and setdbparms configs.

DO NOT include "container" specs. These will be rejected in IWHI. DO NOT reference Configurations on the default integration runtime. The default integration runtime is NOT something a user can configure via the MCP Server.

These are the spec fields that one is expected to use as well as default sensible values.

- `version` e.g. `13.0`
- `replicas` e.g. `2`
- `forceFlowBasicAuth` e.g. {`enabled`: `true`}
- `barURL` e.g. `[]` if it has no BAR files
- `configurations: []` if it has no Configurations

Note how `barURL` and `configurations` are arrays.

OTHER FIELDS SHOULD NOT BE ADDED - this includes things like `flowType` and the `template` field containing container size information. These are not applicable in IWHI. Runtimes MAY get these fields automatically added but users should not be supplying fields OTHER THAN THOSE MENTIONED.

If there are "internal" Configurations you should look to preserve those in any `upsert` or `replace` commands.

NOTE THE MAXIMUM LENGTH FOR THE NAME OF AN INTEGRATION RUNTIME IS 31 CHARACTERS. Anything more will be rejected.

#### ⚠ Policy project base64 — mandatory safe-encoding procedure

The IWHI runtime unpacks the policy zip on pod startup. A single corrupt byte in the zip central directory causes `unexpected EOF` on individual files (e.g. `.project`) and a pod crash-loop, even though `unzip -l` may show the correct file listing.

**The only safe way to produce the base64 value is a single shell pipeline that reads the zip from disk and passes it directly to the MCP tool call — all in one `execute_command` invocation. Never copy a base64 string from a previous tool output and paste it as a literal into an MCP tool call parameter.**

**Always use `mktemp` to generate unique temp paths.** Never use a fixed `/tmp/` path (e.g. `/tmp/mypolicy.zip`). Fixed paths survive across agent contexts and `zip -r` will silently _update_ a leftover archive from a previous session rather than creating a fresh one, which pulls in stale or deleted files and corrupts the zip without any error message.

Before creating any configuration, always run both of these checks and verify they pass with no warnings:

```bash
# Use mktemp — never a fixed /tmp path
POLICY_ZIP=$(mktemp /tmp/policy_XXXXXX.zip)
rm -f "$POLICY_ZIP"   # mktemp creates the file; remove it so zip creates a fresh archive
zip -r "$POLICY_ZIP" MyPolicyProject/

# Test 1 — the zip must be valid
unzip -t "$POLICY_ZIP"

# Test 2 — round-trip: encode then decode and re-test
POLICY_RT=$(mktemp /tmp/policy_rt_XXXXXX.zip)
base64 -i "$POLICY_ZIP" | base64 -d > "$POLICY_RT"
unzip -t "$POLICY_RT"
rm -f "$POLICY_RT"
```

Both `unzip -t` commands must exit 0 with no warnings before proceeding. Then produce the value:

```bash
base64 -i "$POLICY_ZIP" | tr -d '\n'
```

**Never do any of the following — all cause silent zip corruption or stale data:**

- Use a fixed `/tmp/` path for any zip or encoded file — always use `mktemp`.
- Copy a base64 string from a previous `execute_command` output and paste it as a literal into an MCP tool call parameter.
- Use a base64 value computed in a different turn or a previous attempt.
- Use `echo "..." | base64 -d` to reconstruct binary data — always decode from the file.

Each time a policy project configuration is created or updated, the zip must be rebuilt from scratch using a fresh `mktemp` path, and the base64 recomputed from the rebuilt file.

### Step 4 — Upload the flow BAR

Upload only **application flow BARs** to the instance content store using the `bar_files` (action: upload) MCP tool.

- The tool returns a `url` field — this is the value to use for `spec.barURL`.

### Step 5 — Set barURL on the IntegrationRuntime

**Always deploy via `spec.barURL` on the IntegrationRuntime CR.** Never use the ACE admin API, `mqsideploy`, or direct BAR push — the operator owns the pod lifecycle and will not track changes made outside the CR.

- Use `integration_runtimes` (action: create) for a new runtime (after confirmation in Step 2).
- Use `integration_runtimes` (action: replace) to update an existing one.

`barURL` must be an **array**, even when there is only one BAR:

```json
{
  "spec": {
    "barURL": ["<url from bar_files.upload>"],
    "configurations": ["my-policyproject", "my-setdbparms"],
    "forceFlowBasicAuth": { "enabled": true },
    "version": "13.0"
  }
}
```

**Always include `"forceFlowBasicAuth": { "enabled": true }` on every runtime you create or replace**, unless the user explicitly asks for it to be disabled. Never omit it.

**Always set `"version": "13.0"`** unless the user explicitly requests a different version. Never omit it or leave it to default.

### Step 6 — Force a pod roll after updating a configuration

When a configuration (policy project, setdbparms, etc.) is updated on an existing runtime, the operator **does not automatically restart the pod**. The pod continues running with the old configuration until it is restarted.

To force the pod to pick up the new configuration, patch the `configuration-last-updated` annotation on the runtime's pod template. Do this by calling `integration_runtimes` (action: replace) with the annotation added under `spec.template.spec.metadata.annotations`:

```json
{
  "spec": {
    "template": {
      "spec": {
        "metadata": {
          "annotations": {
            "configuration-last-updated": "<configName><timestamp>"
          }
        }
      }
    }
  }
}
```

Use a unique value each time (e.g. the config name concatenated with the current Unix timestamp in milliseconds). The operator sees the annotation change, triggers a rolling update, and the new pod starts with the updated configuration.

### Step 7 — Wait for Ready

Use the following polling strategy:

1. `sleep 10`, then call `integration_runtimes` (action: get) and check `status.phase`.
2. Repeat up to **3 attempts**. If `status.phase` is `"Ready"` at any point, proceed to Step 8.
3. If still not `"Ready"` after 3 attempts, call `runtime_get_logs` for the runtime and scan for errors:
   - If there are **no error-level messages**, the runtime is still starting up — continue polling for up to **3 more attempts** (6 total), sleeping 10 seconds between each.
   - If there **are error-level messages**, stop polling and report the errors to the user. Do not proceed to deployment steps.
4. If `status.phase` is `"Ready"` at any point during the extended polling, proceed to Step 8.
5. If still not `"Ready"` after 6 attempts total, report the current status and the most recent log messages to the user.

### Step 8 — Retrieve credentials

If `forceFlowBasicAuth` is enabled, retrieve the basic auth credentials using `integration_runtimes` (action: `get_flow_auth`).

Verify any changes are there in the BAR file after you've apparently uploaded it.

### Step 9 — Test the flow

Call the flow endpoint using `curl` with the retrieved credentials:

```
curl -s -u "<user>:<password>" \
  "https://<runtime>-https-<instance>.appconnect.ipaas.<env>.ibmappdomain.cloud/<basepath>/<operation>"
```

The endpoint URL is in `status.endpoints[]` where `name` is `"https endpoint"`.

Report the HTTP status code and response body to the user.

If there are problems, consult [`skills/ace-debug-flow/SKILL.md`](../ace-debug-flow/SKILL.md).

---

## Tracing

To start or stop a trace on an integration runtime, always use the `integration_runtimes` tools — **not** the `traces.*` tools.

| Action | Tool |
| --- | --- |
| Start a trace | `integration_runtimes` (action: start_trace) |
| Stop a trace | `integration_runtimes` (action: stop_trace) |
| Collect a trace | `integration_runtimes` (action: collect_trace) + curl callback |

The `traces.*` tools (`traces` (action: replace), `traces` (action: delete), etc.) are lower-level trace object management tools and should not be used for routine start/stop operations.

### Collecting a trace

Collecting a trace is a two-step process — the tool returns a one-time callback URL that must be POSTed to immediately to download the trace file.

**Step 1 — Call `integration_runtimes` (action: collect_trace) with `type` in the body:**

```json
{ "type": "service" }
```

The `type` field is **required** by the runtime. Omitting it causes a 400 error. Use `"service"` for a service trace (the default started by `start_trace`) or `"user"` for a user trace.

**Step 2 — POST to the callback URL in the same tool call batch:**

The tool returns:

```json
{
  "callbackUrl": "http://localhost:4200/internal/callback/<uuid>",
  "token": "<hex token>"
}
```

Download the trace immediately using curl. **Do both steps in the same parallel batch** — the token is one-time-use and expires quickly:

```bash
curl -s -o ./demo1-trace.zip -w "%{http_code}" \
  -X POST "<callbackUrl>" \
  -H "x-callback-token: <token>"
```

A successful download returns HTTP `200` and writes a ZIP file. Any other status (e.g. `502`) means the trace was not saved — do **not** retry with the same token (it will 404). Call `integration_runtimes` (action: collect_trace) again to get a fresh token, then curl immediately.

After collecting, verify the file is a real ZIP (not a JSON error body):

```bash
file ./demo1-trace.zip
```

A real trace will show `Zip archive data` and be at least several hundred KB in size.

---

## Do not use

- The ACE admin API (`/apiv2/...`) — changes are lost on the next operator reconciliation.
- `mqsideploy` or any CLI-based deployment — not available in IWHI.
- Direct BAR push to the runtime pod — bypasses the operator lifecycle.
- `traces` (action: replace) / `traces` (action: delete) for starting or stopping traces — use `integration_runtimes` (action: start_trace) / `integration_runtimes` (action: stop_trace) instead.
- **`deployment_location.*` tools** — these are for managing runtimes and resources accessed through a switch/private-network connection, not for IWHI. Never use `deployment_location_runtimes` (action: list), `deployment_location.bar_files.*`, or `deployment_location_configurations` (action: list) when working with IWHI. Use the direct tools instead:

| Task | Use this tool |
| --- | --- |
| List / get / create / update / delete runtimes | `integration_runtimes.*` |
| Upload / list / get / delete BAR files | `bar_files.*` |
| Create / get / replace / delete configurations | `configurations.*` |
| Get or reset flow auth | `integration_runtimes` (action: `get_flow_auth`) / (action: `reset_flow_auth`) |

Only reach for `deployment_location.*` tools if the user has explicitly asked about resources on a connected remote or on-premises environment (see [`skills/ace-remote-runtime-connect/SKILL.md`](../ace-remote-runtime-connect/SKILL.md)).

## System runtimes — never touch

The runtime named `default-integration-runtime` is a system-managed runtime. Apply these rules unconditionally, regardless of what the user asks:

- **Never list it** when showing the user their runtimes.
- **Never suggest deploying to it** or updating its `barURL`.
- **Never delete or modify it** in any way.
- **Never mention its name** in any response.

If the only runtime returned by `integration_runtimes` (action: list) is `default-integration-runtime`, treat the list as empty and proceed as if no runtimes exist.

---

## Db2 database connectivity on IWHI

Three configuration objects are required for a toolkit flow that queries Db2 via ODBC:

| Configuration type | Purpose | IWHI `spec.type` |
| --- | --- | --- |
| `db2cli` | `db2cli.ini` — tells the Db2 CLI driver how to reach the database | `db2cli` |
| `odbc` | `odbc.ini` — maps the datasource name to the ODBC driver | `odbc` |
| `setdbparms` | Db2 username and password | `setdbparms` |

All three must be listed in `spec.configurations` on the IntegrationRuntime.

### `db2cli` configuration (`db2cli.ini`)

The file content is an INI file. The section header is the datasource alias used in ESQL (`Database.{alias}`):

```ini
[MYALIAS]
Database=BLUDB
Protocol=TCPIP
Hostname=<db2-host>
Port=<port>
Security=ssl
```

- `Security=ssl` is required for IBM Db2 on Cloud.
- Do **not** include `UID` or `PWD` here — credentials go in `setdbparms`.
- **The section name must exactly match the datasource alias used in the ESQL `Database.{alias}.{schema}.{table}` reference — not the underlying database name.** If the ESQL says `Database.KITEDS`, the section header must be `[KITEDS]` even if the actual Db2 database is named `BLUDB`. Using the database name instead of the alias causes `SQL1531N: The connection failed because the name specified with the DSN connection string keyword could not be found`.

### `odbc` configuration (`odbc.ini`)

The correct DRIVER path for IWHI ACE 13 is:

```
/opt/ibm/ace-13/server/ODBC/dsdriver/odbc_cli/clidriver/lib/libdb2o.so
```

**Do not use any other path** (e.g. `/opt/ibm/db2/clidriver/lib/libdb2o.so` is wrong and will cause the datasource to fail to load).

Full correct `odbc.ini` for a datasource with alias `MYALIAS` pointing to the `BLUDB` database:

```ini
[ODBC Data Sources]
MYALIAS=IBM DB2 ODBC Driver

[MYALIAS]
DRIVER=/opt/ibm/ace-13/server/ODBC/dsdriver/odbc_cli/clidriver/lib/libdb2o.so
Description=IBM DB2 ODBC Database
Database=BLUDB

[ODBC]
InstallDir=/home/aceuser/ace-server/ODBC/drivers
UseCursorLib=0
IANAAppCodePage=4
UNICODE=UTF-8
```

### `setdbparms` configuration for Db2 credentials

The `setdbparms` text file uses one line per credential in the form:

```
odbc::{alias} {username} {password}
```

Example for datasource alias `MYALIAS`:

```
odbc::MYALIAS mydbuser mydbpassword
```

- The prefix is `odbc::` followed immediately by the datasource alias (no spaces before the alias).
- Username and password are separated by a single space.
- Do not add quotes around the username or password.

### Creating the configurations

Use `configurations` (action: replace) for all three — **never** `configurations` (action: create) (it returns a misleading `"contents is required"` error even when contents is provided).

The `spec.contents` field must be the **base64-encoded** text of the file. Write each file to disk first, then encode it in the same `execute_command` call that passes the value to the MCP tool. **Do not copy a base64 string from a previous tool output and paste it as a literal** — this is the cause of `unexpected EOF` errors on configuration load.

**Always use `mktemp` for temp files.** Never write to a fixed `/tmp/` path — fixed paths persist across agent contexts and will silently reuse a stale file from a previous session.

```bash
# Use mktemp — never a fixed /tmp path
# ALIAS must match the name used in ESQL (Database.ALIAS), not the Db2 database name
DB2CLI=$(mktemp /tmp/db2cli_XXXXXX.ini)
printf '[MYALIAS]\nDatabase=BLUDB\nProtocol=TCPIP\nHostname=<host>\nPort=<port>\nSecurity=ssl\n' > "$DB2CLI"
base64 -i "$DB2CLI" | tr -d '\n'   # paste this output directly into the MCP call body
rm -f "$DB2CLI"
```

Equivalently for `setdbparms` (text credentials, not binary — but same rule applies):

```bash
SDBP=$(mktemp /tmp/sdbp_XXXXXX.txt)
printf 'odbc::BLUDB user pass' > "$SDBP"
base64 -i "$SDBP" | tr -d '\n'
rm -f "$SDBP"
```

Example configuration body:

```json
{
  "metadata": { "name": "bludb-db2cli" },
  "spec": {
    "type": "db2cli",
    "description": "Db2 CLI configuration for BLUDB",
    "contents": "<base64 of db2cli.ini>"
  }
}
```

```json
{
  "metadata": { "name": "bludb-odbc" },
  "spec": {
    "type": "odbc",
    "description": "ODBC datasource configuration for BLUDB",
    "contents": "<base64 of odbc.ini>"
  }
}
```

```json
{
  "metadata": { "name": "bludb-setdbparms" },
  "spec": {
    "type": "setdbparms",
    "description": "Database credentials for BLUDB ODBC datasource",
    "data": "<base64 of setdbparms text>"
  }
}
```

### Attaching to the IntegrationRuntime

All three configuration names must appear in `spec.configurations`:

```json
{
  "spec": {
    "barURL": ["<bar-url>"],
    "configurations": ["bludb-db2cli", "bludb-odbc", "bludb-setdbparms"],
    "flowType": { "toolkitFlow": true }
  }
}
```

### Configuration updates on a running runtime

If any of the three Db2 configurations are updated after the runtime is already running, force a pod rollout using the annotation technique described in **Step 6** above — the operator will not pick up the new configuration otherwise.

---

## MQ credentials on IWHI (IBM MQ on Cloud)

When a toolkit flow connects to IBM MQ using TLS and credentials, three configurations are typically required on IWHI:

1. An **MQEndpoint policy project** (with connection parameters and `securityIdentity` set)
2. A **setdbparms** configuration (with `mq::` prefix credentials)
3. An **`mqclientini`** configuration (providing `mqclient.ini` to enable TLS 1.3 and Outbound SNI)

The following rules and pitfalls apply specifically to IWHI deployments.

### mqclient.ini configuration

Almost all queue manager connections over TLS require configuring `mqclient.ini` on the runtime.

```ini
SSL:
  AllowTLSV13=TRUE
  OutboundSNI=HOSTNAME
```

> ⚠ **Troubleshooting Note:** A missing or unattached `mqclient.ini` configuration is the **most likely initial cause of SSL/TLS connection errors** when all other settings (cipher spec, credentials, policy) appear correct.

Create an `mqclientini` configuration object using `configurations` (action: replace):

```json
{
  "metadata": { "name": "my-mqclientini" },
  "spec": {
    "type": "mqclientini",
    "description": "MQ client INI config for TLS 1.3 and Outbound SNI",
    "data": "<base64>"
  }
}
```

### setdbparms format for MQ credentials

```
mq::<securityIdentity> <mqUsername> <apiKey>
```

Example — if `securityIdentity` in the policy is `kiteuser1`:

```
mq::kiteuser1 kiteuser1 sXHtZIsSL7709_ciO982bNTu8uX8vi27f-Fdo9BIFD4O
```

Encode with `printf` (not `echo`) to avoid a trailing newline, then use `spec.data`:

```bash
MQ_SDBP=$(mktemp /tmp/mq_sdbp_XXXXXX.txt)
printf 'mq::kiteuser1 kiteuser1 <apikey>' > "$MQ_SDBP"
base64 -i "$MQ_SDBP" | tr -d '\n'
rm -f "$MQ_SDBP"
```

```json
{
  "metadata": { "name": "my-mq-credentials" },
  "spec": {
    "type": "setdbparms",
    "description": "MQ credentials for <QM>",
    "data": "<base64>"
  }
}
```

### MQRC 2393 / AMQ9620E — GSKit `gsk_environment_open` failure on IWHI

**Symptom:** `BIP2677E: MQRC=2393` on every fresh pod start when `securityIdentity` is set in the policy and the `mq::` setdbparms config is attached to the runtime.

**Root cause:** GSKit maintains a **process-global TLS environment singleton**. On IWHI, the ACE integration server's HTTPS listener initialises this singleton first. When the MQ channel then attempts to call `gsk_environment_open` for a credentialled TLS connection using an **explicit** TLS 1.3 cipher name (e.g. `TLS_AES_256_GCM_SHA384`), it hits the already-initialised singleton in a conflicting state and returns error code `0xBAD00FFE`.

**Fix:** Use `ANY_TLS13` instead of an explicit cipher name in `<SSLCipherSpec>`:

```xml
<SSLCipherSpec>ANY_TLS13</SSLCipherSpec>
```

`ANY_TLS13` uses a negotiation code path that is compatible with the already-open GSKit environment. The explicit cipher path is not.

**Why it sometimes works on the first request:** If the pod's first MQ connection happens before the HTTPS listener has fully initialised GSKit (e.g. the very first warm-up request), `gsk_environment_open` succeeds. Subsequent connections reuse the singleton and succeed too — until the pod is restarted, at which point the race is lost and every connection fails.

### SSLCipherSpec quick reference for MQ on Cloud on IWHI

| Cipher value | Result on IWHI |
| --- | --- |
| `TLS_AES_256_GCM_SHA384` | ❌ MQRC 2393 on pod restart |
| `ANY_TLS13` | ✅ Works reliably |
| `ANY_TLS12_OR_HIGHER` | ❌ MQRC — TLS 1.2 rejected by TLS 1.3-only channels |

---

## Related skills

- [`skills/ace-bar-package/SKILL.md`](../ace-bar-package/SKILL.md) — build the BAR file that this skill deploys.
- [`skills/ace-message-flow/SKILL.md`](../ace-message-flow/SKILL.md) — create the message flow source.
- [`skills/ace-project-setup/SKILL.md`](../ace-project-setup/SKILL.md) — scaffold the ACE project.
