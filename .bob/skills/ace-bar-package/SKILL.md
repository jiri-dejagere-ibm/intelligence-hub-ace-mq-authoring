---
name: ace-bar-package
description: Use when the user wants to package an ACE Application project into a BAR file without using the ACE Toolkit or mqsipackagebar — builds a valid BAR using only the zip command.
metadata:
  version: 1.0.0
---

# ACE BAR Packaging Skill

## Purpose

Package an existing ACE Toolkit Application project into a deployable `.bar` file using only the `zip` command — no ACE Toolkit installation or `mqsipackagebar` required.

## When to use

- The user asks to "package", "build a BAR", or "create a BAR file" for an ACE Application project.
- `mqsipackagebar` / `mqsicreatebar` is unavailable or fails to run (e.g. no ACE profile sourced, crash on macOS ARM).
- The project files already exist on disk (`.msgflow`, `application.descriptor`, optional `.esql`).

## BAR file structure

A BAR file is a ZIP with this two-level structure:

```
<FlowName>.bar                  ← outer ZIP (the BAR)
├── META-INF/
│   ├── manifest.mf             ← BAR-level flags (fixed content, see below)
│   ├── service.log             ← human-readable build log
│   └── bar-refresh.links       ← deployment descriptor XML
└── <ProjectName>.appzip        ← inner ZIP (the compiled application)
    ├── META-INF/
    │   ├── manifest.mf         ← same fixed flags
    │   ├── broker.xml          ← configurable properties for mqsiapplybaroverride
    │   └── service.log         ← inner build log (same fixed flags string)
    ├── <FlowName>.msgflow      ← the message flow
    ├── application.descriptor  ← ACE application descriptor
    └── *.esql                  ← any ESQL files (if present)
```

## Step-by-step procedure

### Step 1 — Identify project inputs

Collect:

- **Project directory** — the folder containing `.project`, `application.descriptor`, and `.msgflow` files.
- **Flow name** — the `.msgflow` filename without extension (e.g. `GoogleProxy`).
- **Project name** — the Eclipse project folder name (e.g. `GoogleProxyApp`).
- **Output path** — where to write the `.bar` file (default: inside the project directory).
- **Source files** — list all `.msgflow` and `.esql` files to include.

If any are missing, ask the user before proceeding.

### Step 2 — Inspect the flow for configurable properties

Read the `.msgflow` file and identify:

- HTTP Input node `URLSpecifier` values
- HTTP Request node `URLSpecifier` values (note: same attribute name as HTTP Input — **not** `URL`)
- MQ Input node `queueName` values
- Any other node properties likely to need overrides at deploy time

These become `<ConfigurableProperty>` entries in `broker.xml`.

### Step 3 — Build the inner appzip

Use `execute_command` to run a shell script that:

1. Creates a temp staging directory, e.g. `APPZIP_DIR=$(mktemp -d /tmp/<ProjectName>-appzip-XXXXXX)`.
2. Writes `META-INF/manifest.mf` with exact content:

   ```
   srcSelected=false;showSrc=true;esql21VersionSelected=false;
   ```

   > **Note:** no trailing newline — use `printf`, not `echo`.

3. Writes `META-INF/service.log` with the same fixed content as manifest.mf.

4. Writes `META-INF/broker.xml` — see the template in the **broker.xml template** section below.

5. Copies `application.descriptor`, `<FlowName>.msgflow`, and any `.esql` files into the staging dir.

6. Runs `zip -r <ProjectName>.appzip .` from inside the staging dir, writing the output appzip to a known path outside the staging dir (e.g. `/tmp/<ProjectName>.appzip`).

7. **Immediately** runs `rm -rf "$APPZIP_DIR"` to remove the staging directory.

### Step 4 — Build the outer BAR

1. Creates a second staging directory, e.g. `BAR_DIR=$(mktemp -d /tmp/<ProjectName>-bar-XXXXXX)`.
2. Copies `<ProjectName>.appzip` into it.
3. Writes `META-INF/manifest.mf` with the same fixed content as above.
4. Writes `META-INF/service.log` — a human-readable log line, e.g.:
   ```
   ==============================================

   !<timestamp>

   Successfully added file <ProjectName> to BAR file.
   ```
5. Writes `META-INF/bar-refresh.links` — see the **bar-refresh.links template** section below.
6. Runs `zip -r <FlowName>.bar .` from inside the staging dir, writing the output BAR to the final output path.
7. **Immediately** runs `rm -rf "$BAR_DIR"` and `rm -f /tmp/<ProjectName>.appzip` to remove the staging directory and the intermediate appzip.

### Step 5 — Confirm no temp files remain

Run:

```bash
ls /tmp/<ProjectName>* 2>/dev/null || echo "clean"
```

Confirm the output is `clean` (no leftover staging dirs or intermediate zips). If any remain, remove them with `rm -rf` before proceeding.

### Step 6 — Verify

Run `unzip -l <FlowName>.bar` and confirm:

- `GoogleProxyApp.appzip` (or equivalent) is present
- `META-INF/manifest.mf`, `META-INF/service.log`, `META-INF/bar-refresh.links` are present
- Total file count matches expectations

Report the output path and the `unzip -l` listing to the user.

---

### Step 7 — Deploy to IWHI

If the user wants to deploy the BAR to IWHI, follow [`skills/ace-iwhi/SKILL.md`](../ace-iwhi/SKILL.md) for the full deployment procedure.

---

## broker.xml template

Replace `<FlowName>` with the actual flow name (no extension). Add one `<ConfigurableProperty>` for each node property identified in Step 2. Standard properties to always include:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<Broker>
<CompiledApplication>
<ConfigurableProperty uri="startMode"/>
<ConfigurableProperty uri="javaIsolation"/>
</CompiledApplication>
<CompiledMessageFlow name="<FlowName>">
<ConfigurableProperty uri="<FlowName>#additionalInstances"/>
<ConfigurableProperty uri="<FlowName>#notificationThresholdMsgsPerSec"/>
<ConfigurableProperty uri="<FlowName>#maximumRateMsgsPerSec"/>
<ConfigurableProperty uri="<FlowName>#processingTimeoutSec"/>
<ConfigurableProperty uri="<FlowName>#processingTimeoutAction"/>
<ConfigurableProperty uri="<FlowName>#commitCount"/>
<ConfigurableProperty uri="<FlowName>#commitInterval"/>
<ConfigurableProperty uri="<FlowName>#coordinatedTransaction"/>
<ConfigurableProperty uri="<FlowName>#startMode"/>
<!-- Add per-node overridable properties below, e.g.: -->
<!-- <ConfigurableProperty override="/test" uri="<FlowName>#HTTP Input.URLSpecifier"/> -->
<!-- <ConfigurableProperty override="http://example.com" uri="<FlowName>#HTTP Request.URLSpecifier"/> -->
<!-- <ConfigurableProperty uri="<FlowName>#HTTP Input.useHTTPS"/> -->
</CompiledMessageFlow>
</Broker>
```

---

## bar-refresh.links template

Replace `<ProjectName>` and `<FlowName>` with the actual names.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<deploy:Deploy xmlns:deploy="http://www.ibm.com/xmlns/prod/wmb/deploy/6.0.0" includeDepFromLibs="false">
  <options name="uteSyncKey" value="manual_build"/>
  <deployables>
    <deployable deployed="true" workspaceResource="/<ProjectName>/.project">
      <deployableStatus code="0" key="DEPLOYABLE_LAST_COMPILE_STATUS"
        message="Application file <ProjectName>.appzip successfully added to the BAR." severity="0"/>
      <barEntry name="<ProjectName>.appzip"/>
    </deployable>
  </deployables>
</deploy:Deploy>
```

---

## Important notes

- **`manifest.mf` must not end with a newline.** Use `printf` not `echo` when writing it, otherwise ACE may reject the BAR.
- **The inner appzip must not contain the project `.project` or `.settings/` files** — only `application.descriptor`, `.msgflow`, and `.esql` files go inside.
- **broker.xml drives `mqsiapplybaroverride`** — only list properties the user actually wants to override at deploy time. **Do not add a `<ConfigurableProperty>` for a node property unless you know its exact ACE property name.** An unrecognised name causes ACE to silently apply an empty string override, which produces a `2211E` startup error.
- **HTTP Request node URL** — the `URL` attribute in the `.msgflow` is the flow-level setting. Do **not** add `<ConfigurableProperty uri="...#HTTP Request.URL"/>` or `<ConfigurableProperty uri="...#HTTP Request.URLSpecifier"/>` to `broker.xml` unless you explicitly intend to override it at deploy time. Listing it with no `override` value causes ACE to apply an empty string, failing with: `2211E: Invalid configuration attribute value '', is not valid for target attribute 'URLSpecifier'`
- If the project has multiple flows, each gets its own `<CompiledMessageFlow>` block in `broker.xml`.
- **Baking a property value into the BAR** — to set a flow-level property at build time (e.g. `additionalInstances`), add the `override` attribute directly on `<ConfigurableProperty>`. The value is baked into the BAR and applied at deploy time without needing a separate `mqsiapplybaroverride` step:
  ```xml
  <ConfigurableProperty override="3" uri="OrderFulfilment#additionalInstances"/>
  ```
  Without `override`, the property is declared as overridable but keeps its flow default.

## Related skills

- [`skills/ace-message-flow/SKILL.md`](../ace-message-flow/SKILL.md) — create the `.msgflow` source files that this skill packages.
- [`skills/ace-esql/SKILL.md`](../ace-esql/SKILL.md) — create ESQL files that this skill includes in the appzip.
- [`skills/ace-project-setup/SKILL.md`](../ace-project-setup/SKILL.md) — scaffold the Application project that this skill packages.
- [`skills/ace-iwhi/SKILL.md`](../ace-iwhi/SKILL.md) — deploy the resulting BAR to IWHI.
