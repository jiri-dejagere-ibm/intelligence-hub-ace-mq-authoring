---
name: ace-esql
description: Use when the user wants to create or refine IBM App Connect Enterprise ESQL for Compute nodes using ACE-focused best practices.
metadata:
  version: 1.0.0
---

# ACE ESQL Skill

## Purpose

Use this skill when the user asks to create or modify `.esql` files for ACE Compute nodes.

## When not to use this skill

- If the main request is to create or modify `.msgflow` structure, use [`skills/ace-message-flow/SKILL.md`](../ace-message-flow/SKILL.md) or [`skills/ace-connector-flows/SKILL.md`](../ace-connector-flows/SKILL.md).
- If the main request is JavaCompute logic, use [`skills/ace-java-compute/SKILL.md`](../ace-java-compute/SKILL.md).

## Required reading order

1. [`skills/shared/esql-guidelines.md`](../shared/esql-guidelines.md)
2. [`skills/shared/review-checklist.md`](../shared/review-checklist.md)

## Critical rules

- Do not add BROKER SCHEMA declarations unless explicitly requested by the user or required by file placement in subdirectories
- For ESQL files in the project root, use the simple module format without BROKER SCHEMA
- Match the computeExpression format in .msgflow files to the ESQL file structure (with or without BROKER SCHEMA)
- Generate ACE-compatible ESQL.
- Prefer simple, efficient, and maintainable ESQL.
- Avoid inventing nonexistent ESQL functions.
- Keep the response focused on the requested ESQL change.
- Preserve the requested transformation contract exactly; do not substitute a different output schema, domain, or business behavior.
- When the user provides a target pattern, expected output shape, or reference ESQL, follow that pattern instead of generating a generic example transformation.
- Preserve valid ACE ESQL constructs unless the requested change or the shared guidance requires otherwise.
- Do not invent compiler, parser, or unresolved identifier errors for valid ACE ESQL built-ins.
- Treat `FIELDVALUE(...)`, `LASTMOVE(...)`, `MOVE ... NEXTSIBLING NAME '...'`, `TYPE JSON.Object`, and `CREATE FIELD ... IDENTITY (JSON.Array)` as valid ACE ESQL constructs.
- **Never use `out`, `in`, or `inout` as variable or reference names** — they are ESQL reserved keywords (parameter direction modifiers) and cause parse errors. Use alternatives such as `outRef`, `wmsRoot`, `xmlOut`, `reqBody`, `inRef`.
- **Never initialise a `DECLARE REFERENCE TO` with a `.*[<]` last-child expression** — that syntax is only valid inside `SET`/`IF`. Instead use two statements: `DECLARE ref REFERENCE TO parent;` then `MOVE ref LASTCHILD;`.
- Do not invent ACE JSON scalar type syntax such as `TYPE JSON.String`, `TYPE JSON.Number`, or `TYPE JSON.Boolean`.
- Create JSON scalar values with `SET` assignments rather than by inventing typed scalar nodes.
- Do not describe ACE ESQL fixes as replacing "SQL-like syntax" when the original syntax is valid ACE ESQL.
- For XMLNSC-to-JSON transformations, follow [`skills/shared/esql-guidelines.md`](../shared/esql-guidelines.md) exactly.
- For XMLNSC repeating elements, map every repeating element to a JSON array; do not flatten repeated values into concatenated strings.
- For XMLNSC-to-JSON array traversal, use `DECLARE ... REFERENCE TO ...[1]` with `WHILE LASTMOVE(...)` and `MOVE ... NEXTSIBLING NAME '...'`; do not use `FOR ... AS path[] DO`.
- Before populating any JSON array, create it explicitly with `CREATE FIELD ... IDENTITY (JSON.Array)`.
- Always use `FIELDVALUE(...)` when reading any scalar leaf value from **any** message domain tree (XMLNSC, JSON, MRM) — this includes reading `CHARACTER` variables from a JSON input body (`InputRoot.JSON.Data`), not only XMLNSC-to-JSON transformations.
- When iterating a JSON input array (e.g. `InputRoot.JSON.Data.lineItems`), the array elements are stored as siblings named `item` (lowercase) by the ACE HTTPInput/JSON parser. The reference must start at `.item[1]` and advance with `MOVE ref NEXTSIBLING NAME 'item'`. Using `MOVE ref NEXTSIBLING` without `NAME` advances past the array boundary.
- If a user provides an ESQL file that contains a BROKER SCHEMA declaration, you MUST ensure that the ESQL file is placed in a subdirectory of the project of the correct name
- For example, if an ESQL file named `Example.esql` has a declaration `BROKER SCHEMA com.ibm.dev.test` and it is to be placed inside a Shared Library project named `ExampleSharedLibrary`, then within ExampleSharedLibrary there must be a subdirectory structure of `com/ibm/dev/test/Example.esql`

## Database access

**Do not use `PASSTHRU`** for database queries or DML. `PASSTHRU` bypasses the ACE SQL parser and is not recommended — avoid it entirely.

Use the native ACE ESQL `SELECT` and `INSERT` statement forms instead.

- `Database.DATASOURCE_NAME` must match the ODBC datasource alias configured on the runtime.
- Omit columns that have a `DEFAULT` or `GENERATED ALWAYS` constraint.

### SELECT — reading multiple columns from one row

**Always use `THE(SELECT * FROM ...)` into a `ROW` variable** when reading multiple columns from a single matched row — one database round-trip, all columns available as fields on the row variable:

```esql
DECLARE customerRow ROW;
SET customerRow = THE(
    SELECT * FROM Database.DATASOURCE_NAME.SCHEMA.TABLENAME AS T
    WHERE T.KEY_COL = vKeyValue
);

DECLARE vFirstName CHARACTER COALESCE(customerRow.FIRST_NAME, '');
DECLARE vStatus    CHARACTER COALESCE(customerRow.STATUS, 'active');
```

**Never** use multi-variable assignments (`SELECT v1 = T.COL1, v2 = T.COL2 FROM ...`) or column-list assignments (`SELECT C.COL1, C.COL2 INTO ...`) — these are **not valid ACE ESQL syntax** and cause `2401E: Syntax error : expected 'DO' but found '<COLUMN_NAME>'` or `2401E: Syntax error: expected 'END' but found 'keyword Select'` at startup.

### SELECT — reading a single scalar value

```esql
SET vResult = THE (SELECT ITEM T.COL
                   FROM Database.DATASOURCE_NAME.SCHEMA.TABLENAME AS T
                   WHERE T.KEY_COL = vKeyValue);
```

### INSERT (write)

```esql
INSERT INTO Database.DATASOURCE_NAME.SCHEMA.TABLENAME (COL1, COL2)
  VALUES (vValue1, vValue2);
```

## XMLNSC namespace-qualified element access

**Never use `(namespace:FieldName)` parenthesis notation to access or set XMLNSC fields.** This syntax does not exist in ACE ESQL and causes a parse error at startup.

To build a namespace-qualified XMLNSC output tree, use `CREATE LASTCHILD` with `NAMESPACE` and `NAME` clauses for every element:

```esql
DECLARE kiteNS NAMESPACE 'http://example.com/v1';

DECLARE parent REFERENCE TO OutputRoot.XMLNSC;
CREATE LASTCHILD OF OutputRoot.XMLNSC AS parent TYPE XMLNSC.Element
    NAMESPACE kiteNS NAME 'Root';

-- Correct: create each child element with its namespace and value
CREATE LASTCHILD OF parent TYPE XMLNSC.Element NAMESPACE kiteNS NAME 'FieldName'
    VALUE vSomeValue;
```

**Never write:**

```esql
-- WRONG — syntax error
SET parent.(kiteNS:FieldName) = vSomeValue;
```

The `(namespace:Name)` form is not valid ACE ESQL. Always use `CREATE LASTCHILD ... NAMESPACE ... NAME ... VALUE ...` to produce namespaced elements.

---

## Header/Body sequence rule (`BIP6062W` avoidance)

In ACE, the root message tree structure mandates that **header folders must precede body folders**.

If you need to construct or modify `HTTPReplyHeader` (e.g. setting `X-Original-HTTP-Status-Code` or `Content-Type`), you must create `HTTPReplyHeader` **before** creating any body domain folder (`XMLNSC`, `JSON`, `BLOB`, `MRM`).

```esql
-- CORRECT: Header created before body
CREATE LASTCHILD OF OutputRoot DOMAIN 'HTTPReplyHeader' NAME 'HTTPReplyHeader';
SET OutputRoot.HTTPReplyHeader."X-Original-HTTP-Status-Code" = 202;
SET OutputRoot.HTTPReplyHeader."Content-Type" = 'application/json';

CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON';
CREATE LASTCHILD OF OutputRoot.JSON TYPE JSON.Object NAME 'Data';
```

Creating `HTTPReplyHeader` **after** `OutputRoot.XMLNSC` or `OutputRoot.JSON` causes:

```
BIP6062W: Invalid message body/header sequence '<bodyDomain>'/'WSREPHDR' encountered by node 'HTTP Input'
```

If a flow puts a message to MQ or a backend service and then returns an independent HTTP response to the caller, use a second Compute node after `MQ Output` to set the HTTP reply headers and response body.

---

## Compute node wiring — computeExpression URI contract

When generating or referencing a Compute node in a `.msgflow`, the `computeExpression` attribute **must** use the URI form:

```xml
computeExpression="esql://routine/#ModuleName.Main"
```

Where `ModuleName` exactly matches the `CREATE COMPUTE MODULE ModuleName` name in this `.esql` file.

**Never** use the path-style form or the `esqlModule` attribute — both cause `4127E: Failed to find ESQL module` at runtime. See [`skills/ace-message-flow/SKILL.md`](../ace-message-flow/SKILL.md) for full details.

## Output requirements

- Create or update the requested ESQL artifacts.
- Provide a concise summary of what was created or changed.
