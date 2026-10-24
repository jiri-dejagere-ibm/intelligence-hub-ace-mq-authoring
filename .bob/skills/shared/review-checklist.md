---
name: review-checklist
description: Pre-commit review checklist applied by all ACE skills before producing final output. Read to verify correctness before delivering any ACE artifact.
metadata:
  version: 1.0.0
---

# ACE Skill Review Checklist

## Purpose

Use this checklist before returning generated ACE artifacts.

## When to use

Use this checklist after generating or updating ACE artifacts and before finalizing a response.

## Checklist

### Artifact correctness

- The correct artifact type was created.
- The generated files were placed in the correct project type.

### Project metadata — Application projects

When a new Application project is created, **all four of the following files must be present in the output** — not described in prose, actually created:

- `.project` — contains **exactly 2 natures**: `com.ibm.etools.msgbroker.tooling.applicationNature` and `com.ibm.etools.msgbroker.tooling.messageBrokerProjectNature`. Do not add extra natures.
- `.settings/org.eclipse.core.resources.prefs` — **must be created as a file and contain the following exact content verbatim:**

```properties
eclipse.preferences.version=1
encoding/<project>=UTF-8
```

- The `.msgflow` file(s) requested by the user.

### Project metadata — Policy projects

- Any new Policy project contains the required metadata files.

### Message flow nodes

- Every message flow node uses a validated `xmi:type`.
- HTTP Input and HTTP Reply nodes use `ComIbmWSInput.msgnode` and `ComIbmWSReply.msgnode` — **not** `ComIbmHTTPInput` or `ComIbmHTTPReply`.

### Connector-specific

- Connector-specific rules were applied where relevant.
- Required schema files were created where relevant.
- Required policy files were created where relevant.

### ESQL correctness

- No direct cross-domain tree assignment (`SET OutputRoot.JSON.Data = InputRoot.XMLNSC.*`) — this produces duplicate keys for repeated elements.
- A repeatability table was built for every element at every nesting depth before writing code. Every element that can repeat is mapped to a `JSON.Array`, not a scalar.
- Every repeating element is iterated with `DECLARE ... REFERENCE TO [1]` + `WHILE LASTMOVE` + `MOVE ... NEXTSIBLING NAME`, not `FOR ... AS ...[] DO`.
- Every JSON array is declared with `CREATE FIELD ... IDENTITY (JSON.Array)` before being populated.
- Every JSON array object is created with `CREATE LASTCHILD ... TYPE JSON.Object` before its fields are assigned.
- Every scalar read from the XMLNSC tree uses `FIELDVALUE()`.
- Output domain is initialised with `CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON'` + `CREATE FIELD OutputRoot.JSON.Data` — not `SET OutputRoot.JSON.Data = NULL`.
- `CREATE OUTPUTROOT DOMAIN(...)` does not appear anywhere — it is not valid ACE ESQL syntax.
- There is no such built-in ESQL function named `RANDOM` although there is a built-in function named `RAND`

### ESQL — JSON array iteration

- When iterating a **JSON input** array, the start reference uses `.item[1]` (e.g. `inRef.lineItems.item[1]`), **not** `arrayField[1]` — pointing at the array node leaves the reference unpositioned and the loop never executes.
- `MOVE` statements within a JSON array loop always specify `NAME 'item'` (lowercase) — omitting it allows the reference to advance past the end of the array into sibling fields of the parent object.

### ESQL — type mapping

- `BOOLEAN` variables written into XML elements typed `xs:boolean` are assigned **directly**, never via `CAST(... AS CHARACTER)`. `CAST` produces uppercase `'TRUE'`/`'FALSE'` which fails `xs:boolean` validation; the XMLNSC serialiser handles native `BOOLEAN` and writes lowercase `true`/`false` automatically.

### ESQL — database access

- Every Db2 `SELECT` uses the **scalar subquery form**: `SET var = (SELECT T.COL FROM Database.DS.SCHEMA.TABLE AS T WHERE T.KEY = val)`. The multi-column assignment form (`SELECT col1 = T.COL1, col2 = T.COL2 FROM ...`) is **not valid ACE ESQL** and causes `2401E` at startup.
- Every Db2 `SELECT` that may return zero rows is followed by null-safe handling. Use `THE(SELECT *)` into a `ROW` variable with `COALESCE` on extracted fields, or add an explicit null-check after the `SELECT`. Column-assignment `SELECT` that leaves variables `NULL` on a miss propagates `NULL` silently into the output with no error — this must not be left as-is in generated code.

### ESQL — built-in functions and scalar initialisation

- `UUIDASCHAR` must be called with parentheses: `UUIDASCHAR()`. Using it without parentheses (e.g. `CAST(UUIDASCHAR AS CHARACTER)`) causes a `2401E` syntax error at startup.
- Numeric scalar variables (`INTEGER`, `DECIMAL`) must **not** be initialised inline from a `REFERENCE` path. Always declare first, then assign with `CAST(FIELDVALUE(...) AS type)`.

### ESQL — XMLNSC output domain

- Do **not** call `CREATE LASTCHILD OF OutputRoot DOMAIN 'XMLNSC'` or manually create intermediate `XMLNSC.Folder` nodes for non-repeating elements. `SET` on a deep XMLNSC path auto-creates the domain and all intermediate folders. Explicit `CREATE LASTCHILD ... TYPE XMLNSC.Folder` is only needed for **repeating elements** built inside a loop.

### Simplicity

- The flow topology is the minimum required to satisfy the request. Do not add Compute nodes, ESQL files, or transformation logic unless the user explicitly asked for transformation. A direct wire from Input to Reply is correct for a pass-through or echo pattern.

### Final response

- Version-specific assumptions were either confirmed or clearly stated.
- The final response includes a concise summary of what was created.
- If a new Application project was created, include the ACE Toolkit import guidance.

## Related files

- [`skills/shared/ace-projects.md`](ace-projects.md)
- [`skills/shared/policy-projects.md`](policy-projects.md)
- [`skills/shared/message-flow-rules.md`](message-flow-rules.md)
