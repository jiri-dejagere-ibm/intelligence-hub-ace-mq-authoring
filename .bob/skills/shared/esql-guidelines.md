---
name: esql-guidelines
description: Style and correctness guidelines for writing ESQL in IBM App Connect Enterprise Compute nodes. Read whenever creating or reviewing ESQL.
metadata:
  version: 1.0.0
---

# ACE ESQL Guidelines

## Purpose

This document defines ACE-focused guidance for creating and refining ESQL used by Compute nodes.

## When to use

Use this document when generating or reviewing `.esql` files for ACE Compute nodes.

## BROKER SCHEMA usage

**Default behavior: Do not use BROKER SCHEMA declarations unless explicitly required.**

### When NOT to use BROKER SCHEMA

- When the `.esql` file is placed in the project root directory
- When the user has not explicitly requested a specific schema organization
- For simple projects with a small number of ESQL files

### When to use BROKER SCHEMA

- Only when the user explicitly requests it
- When placing ESQL files in subdirectories for organizational purposes
- When the `.esql` file is in a Shared Library project with subdirectory structure

### File placement rules

- **No BROKER SCHEMA**: Place `.esql` file in project root
  - File: `ProjectName/ModuleName.esql`
  - ESQL: No `BROKER SCHEMA` declaration
  - Compute node reference: `esql://routine/#ModuleName.FunctionName`

- **With BROKER SCHEMA**: Place `.esql` file in subdirectory matching schema
  - File `ProjectName/com/ibm/example/ModuleName.esql`
  - ESQL: `BROKER SCHEMA com.ibm.example`
  - Compute node reference: `esql://routine/com.ibm.example#ModuleName.FunctionName`

### Example without BROKER SCHEMA (default)


CREATE COMPUTE MODULE MyCompute
    CREATE FUNCTION Main() RETURNS BOOLEAN
	BEGIN
	    -- implementation here
	    RETURN TRUE;
	END;
END MODULE;

Compute node property: `computeExpression="esql://routine/#MyCompute.Main"

## Compute node expression formats

The `computeExpression` property in a Compute node must match the ESQL file structure:

| ESQL Structure | computeExpression Format | Example |
| --- | --- | --- |
| No BROKER SCHEMA | `esql://routine/#Module.Function` | `esql://routine/#EmployeeAPI_Compute.CreateEmployee` |
| With BROKER SCHEMA | `esql://routine/schema.path#Module.Function` | `esql://routine/com.ibm.example#EmployeeAPI_Compute.CreateEmployee` |

## Performance and tree navigation

- Declare `REFERENCE` variables when navigating heavily nested logical trees to reduce repeated navigation cost.
- Avoid unnecessary full tree copying when a change is limited to a small part of the message tree.
- Avoid multiple consecutive Compute nodes where a simpler design would achieve the same result.
- Avoid using `CARDINALITY` inside loops when the value can be determined once outside the loop.
- Minimize expensive string manipulation where a simpler approach is available.
- Minimize the number of statements when doing so improves clarity and efficiency.

## Field existence checks

- Use `LASTMOVE` or `CARDINALITY` when checking whether a field exists.
- Prefer the most direct existence check that matches the tree navigation pattern being used.

## Tree construction

- Consider `CREATE ... PARSE` when it is more efficient than serializing and reparsing a copied logical tree.
- Be deliberate about Compute node copy behavior and avoid unnecessary message assembly duplication.

## Cross-domain transformation (XML → JSON)

**Never use a direct tree assignment to convert between message domains**, for example:

```esql
-- WRONG: copies XML tree structure verbatim, produces duplicate keys for repeated elements
SET OutputRoot.JSON.Data = InputRoot.XMLNSC.SomeElement;
```

### Step 1 — Analyse the XML structure before writing any ESQL

**Before writing a single line of ESQL**, examine the XML input and answer this question for every element at every level of nesting:

> "Can more than one of these elements appear as siblings under the same parent?"

If the answer is yes — even for a deeply nested element like `<Initial>` inside `<Invoice>` — that element **must** be mapped to a JSON array. There are no exceptions.

Build an explicit repeatability table before writing code:

| XML element        | Can repeat? | JSON mapping |
| ------------------ | ----------- | ------------ |
| `SaleList/Invoice` | yes         | JSON.Array   |
| `Invoice/Initial`  | yes         | JSON.Array   |
| `Invoice/Item`     | yes         | JSON.Array   |
| `Item/Code`        | yes         | JSON.Array   |
| `Invoice/Surname`  | no          | scalar       |

Do not start writing ESQL until this table is complete.

### Step 2 — Initialise the output domain correctly

Use `CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON'` followed by `CREATE FIELD OutputRoot.JSON.Data` to initialise the JSON output domain. **Do not use `SET OutputRoot.JSON.Data = NULL`** — that form does not correctly establish the JSON domain in ACE.

```esql
SET OutputRoot.Properties = InputRoot.Properties;
CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON';
CREATE FIELD OutputRoot.JSON.Data;
```

### Step 3 — Declare JSON arrays explicitly before populating them

Every repeating element **must** be declared as a typed JSON array using `CREATE FIELD ... IDENTITY (JSON.Array)` before any elements are added to it. Do not rely on indexed `SET` alone to create arrays.

```esql
CREATE FIELD OutputRoot.JSON.Data.SaleList TYPE JSON.Object;
CREATE FIELD OutputRoot.JSON.Data.SaleList.Invoice IDENTITY (JSON.Array)Invoice;
```

### Step 4 — Iterate with REFERENCE + WHILE LASTMOVE, not FOR loops

Use `DECLARE ... REFERENCE TO element[1]` + `WHILE LASTMOVE(...) DO` + `MOVE ... NEXTSIBLING NAME '...'` to iterate repeating elements. **Do not use `FOR srcX AS path[] DO` loops** — that form does not correctly traverse sibling nodes in the XMLNSC domain.

#### JSON array iteration — `MOVE NEXTSIBLING` without `NAME`

JSON array elements are anonymous siblings — they have no element name. Use `MOVE ref NEXTSIBLING` **without** a `NAME` argument:

```esql
DECLARE inItemRef REFERENCE TO inJson.lineItems.Item[1];
DECLARE itemIndex INTEGER 1;

WHILE LASTMOVE(inItemRef) DO
    -- process inItemRef
    SET itemIndex = itemIndex + 1;
    MOVE inItemRef NEXTSIBLING;  -- no NAME argument for JSON arrays
END WHILE;
```

Using `MOVE ref NEXTSIBLING NAME 'item'` will fail to advance the reference and produce an infinite loop.

#### JSON input arrays — start path must include `item[1]`

When the input is a JSON array (not XMLNSC), the array elements are stored in ACE's tree as children named `item` (lowercase). The start reference **must** navigate into the first child element, not point at the array node itself:

```esql
-- CORRECT — positions reference at the first array element
DECLARE lineRef REFERENCE TO inRef.lineItems.item[1];

-- WRONG — points at the array node; LASTMOVE returns FALSE immediately, loop never executes
DECLARE lineRef REFERENCE TO inRef.lineItems[1];
```

Similarly, `MOVE` must filter by `NAME 'item'` (lowercase) to stay within the array:

```esql
-- CORRECT
MOVE lineRef NEXTSIBLING NAME 'item';

-- WRONG — advances past the end of the array into the next field of the parent object
MOVE lineRef NEXTSIBLING;
```

For each object in an array, create it explicitly with `CREATE LASTCHILD ... TYPE JSON.Object` before assigning its fields.

```esql
DECLARE invoiceRef REFERENCE TO InputRoot.XMLNSC.SaleEnvelope.SaleList.Invoice[1];
DECLARE invoiceIndex INTEGER 1;
WHILE LASTMOVE(invoiceRef) DO
    CREATE LASTCHILD OF OutputRoot.JSON.Data.SaleList.Invoice TYPE JSON.Object;

    -- Read scalars with FIELDVALUE()
    SET OutputRoot.JSON.Data.SaleList.Invoice.Item[invoiceIndex].Surname = FIELDVALUE(invoiceRef.Surname);
    SET OutputRoot.JSON.Data.SaleList.Invoice.Item[invoiceIndex].Balance = FIELDVALUE(invoiceRef.Balance);

    -- Nested array: declare it, then iterate
    CREATE FIELD OutputRoot.JSON.Data.SaleList.Invoice.Item[invoiceIndex].Initial IDENTITY (JSON.Array)Initial;
    DECLARE initialIndex INTEGER 1;
    DECLARE initialRef REFERENCE TO invoiceRef.Initial[1];
    WHILE LASTMOVE(initialRef) DO
        SET OutputRoot.JSON.Data.SaleList.Invoice.Item[invoiceIndex].Initial.Item[initialIndex] = FIELDVALUE(initialRef);
        SET initialIndex = initialIndex + 1;
        MOVE initialRef NEXTSIBLING NAME 'Initial';
    END WHILE;

    SET invoiceIndex = invoiceIndex + 1;
    MOVE invoiceRef NEXTSIBLING NAME 'Invoice';
END WHILE;
```

### Step 5 — Always use FIELDVALUE() for scalar reads

Use `FIELDVALUE()` whenever reading a scalar leaf value from **any** message domain tree — XMLNSC, JSON, or MRM. A bare field reference returns a logical tree node, not a primitive value, and will produce incorrect output or a silent NULL.

```esql
-- WRONG — bare reference returns a tree node, not the string value
SET OutputRoot.JSON.Data.Surname = invoiceRef.Surname;
DECLARE vOrderId CHARACTER reqBody.orderId;

-- CORRECT
SET OutputRoot.JSON.Data.Surname = FIELDVALUE(invoiceRef.Surname);
DECLARE vOrderId CHARACTER FIELDVALUE(reqBody.orderId);
```

This applies equally when reading scalars from a JSON input tree (e.g. `InputRoot.JSON.Data`) into `CHARACTER` variables.

### Step 6 — Use DECLARE REFERENCE for navigation efficiency

Declare `REFERENCE` variables for deeply nested input paths to avoid repeated tree navigation:

```esql
DECLARE saleEnvelope REFERENCE TO InputRoot.XMLNSC.SaleEnvelope;
DECLARE saleList REFERENCE TO saleEnvelope.SaleList;
```

This rule applies whenever the source and target domains differ (XMLNSC→JSON, XMLNSC→JSONARRAY, etc.).

## Type mapping — BOOLEAN to XML xs:boolean

**Do not `CAST` a `BOOLEAN` variable to `CHARACTER` before writing it into an XMLNSC element that maps to `xs:boolean`.**

`CAST(TRUE AS CHARACTER)` produces the string `'TRUE'` (uppercase). The XML Schema type `xs:boolean` only accepts `true` or `false` (lowercase). A message containing `<PriorityFlag>TRUE</PriorityFlag>` will fail XSD validation at the receiving system.

Assign the native `BOOLEAN` value directly — the XMLNSC serialiser writes it as `true` or `false` automatically:

```esql
-- CORRECT — serialises as 'true' or 'false' (lowercase), valid xs:boolean
SET outDetails.kite:PriorityFlag = priorityFlag;

-- WRONG — serialises as 'TRUE' or 'FALSE' (uppercase), fails xs:boolean validation
SET outDetails.kite:PriorityFlag = CAST(priorityFlag AS CHARACTER);
```

This rule applies to any XML element whose XSD type is `xs:boolean`.

## Timestamp formatting — CAST to CHARACTER

When formatting a timestamp as an ISO 8601 string, use ACE's own format tokens — **not** Java `DateTimeFormatter` patterns.

**Java timezone tokens (`X`, `XX`, `XXX`, `Z`) are not valid in ACE ESQL `FORMAT` expressions** and cause `BIP3203S: not a valid FORMAT expression` at runtime.

```esql
-- WRONG — 'XXX' is a Java pattern, not an ACE format token; throws BIP3203S
CAST(CURRENT_TIMESTAMP AS CHARACTER FORMAT 'yyyy-MM-dd''T''HH:mm:ssXXX')

-- CORRECT — append literal 'Z' (valid when the runtime is UTC, as on IWHI)
CAST(CURRENT_TIMESTAMP AS CHARACTER FORMAT 'yyyy-MM-dd''T''HH:mm:ss') || 'Z'

-- CORRECT — use ACE's built-in IU format for a full ISO 8601 string
CAST(CURRENT_TIMESTAMP AS CHARACTER FORMAT 'IU')
```

Use `FORMAT 'IU'` when you want a complete ISO 8601 datetime string including timezone from the ACE runtime. Use the manual concatenation form only when you need to control the exact output shape.

## Transformation fidelity

- Preserve the requested message domain and target structure exactly.
- Do not change an XML-to-JSON request into XML-to-XML or any other domain transformation.
- Do not invent new element names, field names, wrapper objects, or business semantics unless the user explicitly requests them.
- Do not add calculations, enrichment, default values, formatting changes, aggregation, or derived values unless the user explicitly requests them.
- If the user provides an expected output shape, example ESQL, or reference implementation, match that structure as closely as possible.
- For mapping tasks, prefer faithful structural mapping over a redesigned example transformation.

## ESQL file structure

Every `.esql` file for a Compute node must use this skeleton. The module name must match the name given to the Compute node in the `.msgflow` file. The file must be placed in the same Application project as the `.msgflow`.

```esql
CREATE COMPUTE MODULE <ModuleName>_Compute
	CREATE FUNCTION Main() RETURNS BOOLEAN
	BEGIN
		-- implementation here
		RETURN TRUE;
	END;
END MODULE;
```

The `.esql` filename must match the module name, e.g. `XMLToJSON_Compute.esql` for module `XMLToJSON_Compute`.

## Header and body sequence (`BIP6062W` avoidance)

In ACE, header folders must precede body folders in the message tree hierarchy.

If setting `HTTPReplyHeader` (such as `X-Original-HTTP-Status-Code` or `Content-Type`), always create `HTTPReplyHeader` **before** creating the body domain (`XMLNSC`, `JSON`, `BLOB`, `MRM`):

```esql
-- CORRECT: Header created before body
CREATE LASTCHILD OF OutputRoot DOMAIN 'HTTPReplyHeader' NAME 'HTTPReplyHeader';
SET OutputRoot.HTTPReplyHeader."X-Original-HTTP-Status-Code" = 202;
SET OutputRoot.HTTPReplyHeader."Content-Type" = 'application/json';

CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON';
CREATE LASTCHILD OF OutputRoot.JSON TYPE JSON.Object NAME 'Data';
```

Creating `HTTPReplyHeader` after a body domain triggers:

```
BIP6062W: Invalid message body/header sequence '<bodyDomain>'/'WSREPHDR' encountered by node '<NodeName>'
```

## Namespace-qualified XMLNSC construction

When building an XML message with a target namespace, declare a `NAMESPACE` constant and use it on every `CREATE` and `SET` operation. Do **not** set `xmlns` as an attribute manually.

```esql
DECLARE kite NAMESPACE 'http://kite-sports.com/wms/order/v1';

-- Create root and position reference on it
CREATE LASTCHILD OF OutputRoot.XMLNSC NAMESPACE kite NAME 'WMSOrder';
DECLARE wmsRoot REFERENCE TO OutputRoot.XMLNSC;
MOVE wmsRoot LASTCHILD;

-- Create a child element and set namespace-qualified fields on it
CREATE LASTCHILD OF wmsRoot NAMESPACE kite NAME 'MessageHeader';
DECLARE hdr REFERENCE TO wmsRoot;
MOVE hdr LASTCHILD;
SET hdr.kite:CorrelationId    = UUIDASCHAR;
SET hdr.kite:MessageDateTime  = CAST(CURRENT_TIMESTAMP AS CHARACTER FORMAT 'IU');
SET hdr.kite:SchemaVersion    = '1.0';
SET hdr.kite:SourceSystem     = 'kitesports.com';

-- Nested child under a namespace-qualified element
CREATE LASTCHILD OF cust NAMESPACE kite NAME 'FullName';
DECLARE fn REFERENCE TO cust;
MOVE fn LASTCHILD;
SET fn.kite:FirstName = vFirstName;
SET fn.kite:LastName  = vLastName;
```

- Use `NAMESPACE kite NAME 'ElementName'` on every `CREATE LASTCHILD` call.
- Use `ref.kite:FieldName` syntax for all field assignments under the namespace.
- `DECLARE kite NAMESPACE '...'` must appear before any XML construction.

## REFERENCE vs FIELDVALUE vs direct assignment

Use the correct form depending on what you need:

| Intent | Correct form | Wrong form |
| --- | --- | --- |
| Navigate to a subtree for repeated access | `DECLARE x REFERENCE TO path;` | Repeating the full path each time |
| Read a scalar string/number from any domain | `FIELDVALUE(ref.Field)` | `ref.Field` (returns a tree node, not a value) |
| Copy a scalar from one domain to another | `SET outRef.Field = FIELDVALUE(inRef.Field)` | `SET outRef.Field = inRef.Field` |
| Declare a CHARACTER variable from a JSON/XMLNSC field | `DECLARE v CHARACTER FIELDVALUE(ref.Field);` | `DECLARE v CHARACTER ref.Field;` |
| Move a reference to the next named sibling | `MOVE ref NEXTSIBLING NAME 'X';` | n/a |
| Move to the next JSON array item | `MOVE ref NEXTSIBLING NAME 'item';` | `MOVE ref NEXTSIBLING;` (advances past the array boundary) |

Always use `FIELDVALUE()` when reading leaf values from **any** message domain tree (XMLNSC, JSON, MRM) into scalar variables or output fields.

## Flow design considerations

- Prefer clean flow design over wiring every failure terminal individually when a consistent catch strategy is more suitable.
- Consider whether a subflow or shared library is more appropriate when logic is repeated.

## Validity guardrails

- Do not invent ESQL built-in functions or statements.
- There is no `DOUBLE` data type in ESQL. ESQL does have `DECIMAL`, `FLOAT` and `INTEGER` numeric data types.
- There is no `RANDOM` function in ESQL although there is a `RAND` function.
- `CREATE OUTPUTROOT DOMAIN(...)` is not a valid ESQL statement — do not generate it.
- `SET OutputRoot.JSON.Data = NULL` does not correctly establish the JSON domain — use `CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON'` + `CREATE FIELD OutputRoot.JSON.Data` instead.
- Do not invent JSON scalar type syntax such as `JSON.String`, `JSON.Number`, or `JSON.Boolean`.
- If a requested function or syntax is uncertain, use known ACE ESQL constructs rather than guessing.

## Database SELECT — correct forms

### Reading multiple columns — use `THE(SELECT * ...)` into a ROW variable

The most efficient pattern for reading several columns from one matched row is a single `THE(SELECT * ...)` into a `ROW` variable. This issues one database round-trip and makes every column available as a field on the row:

```esql
DECLARE customerRow ROW;
SET customerRow = THE(
    SELECT * FROM Database.KITEDS.KITE.CUSTOMERS AS C
    WHERE C.CUSTOMER_ID = vCustomerId
);

DECLARE vFirstName CHARACTER COALESCE(customerRow.FIRST_NAME, '');
DECLARE vStatus    CHARACTER COALESCE(customerRow.ACCOUNT_STATUS, 'active');
```

### Multi-column assignment form is not valid ACE ESQL

The column-assignment or multi-variable `SELECT` form causes a `2401E` syntax error at startup:

```esql
-- WRONG — causes 2401E: Syntax error : expected 'DO' but found 'FIRST_NAME' or 'keyword Select'
SELECT
    firstName = T.FIRST_NAME,
    lastName  = T.LAST_NAME
FROM Database.DATASOURCE.SCHEMA.TABLE AS T
WHERE T.KEY = keyValue;
```

### Reading a single scalar value

```esql
-- CORRECT — one column, one row
SET vResult = THE (SELECT ITEM T.COL
                   FROM Database.DATASOURCE.SCHEMA.TABLE AS T
                   WHERE T.KEY = keyValue);
```

## UUIDASCHAR / UUIDASBLOB — standalone keyword / function

In ACE ESQL, `UUIDASCHAR` and `UUIDASBLOB` generate a new UUID without parentheses when used without arguments:

```esql
-- CORRECT — generates a new UUID as CHARACTER
SET outRef.CorrelationId = UUIDASCHAR;

-- Also valid when converting an existing BLOB UUID
SET outRef.CorrelationId = UUIDASCHAR(blobUuid);
```

## Scalar initialisation from a REFERENCE variable

Do **not** initialise a scalar `INTEGER` or `DECIMAL` variable inline from a `REFERENCE` path. The ACE ESQL parser does not accept inline initialisers of non-`CHARACTER` types from a reference expression, and the result is silently `NULL` or a type mismatch at runtime:

```esql
-- WRONG — inline initialiser from reference; may produce NULL or a type error
DECLARE qty INTEGER lineItemsRef.quantity;

-- CORRECT — always use CAST(FIELDVALUE(...) AS type) for numeric scalars from a reference
DECLARE qty INTEGER;
SET qty = CAST(FIELDVALUE(lineItemsRef.quantity) AS INTEGER);

-- Or inline with explicit CAST
DECLARE qty INTEGER CAST(FIELDVALUE(lineItemsRef.quantity) AS INTEGER);
```

## XMLNSC output — do not use CREATE DOMAIN or explicit folder creation

When building XMLNSC output, **do not call `CREATE LASTCHILD OF OutputRoot DOMAIN 'XMLNSC'`** or manually create intermediate folder nodes. The XMLNSC parser auto-creates the domain and all intermediate `XMLNSC.Folder` nodes the first time you `SET` a deep path:

```esql
-- CORRECT — Declare namespace prefix and use colon notation (prefix:ElementName)
DECLARE ns NAMESPACE 'http://example.com/v1';

-- Optional / Best Practice: Declare the namespace prefix on the root element
SET OutputRoot.XMLNSC.ns:Root.(XMLNSC.NamespaceDecl)xmlns:ns = ns;

-- SET auto-creates the domain and every intermediate folder
SET OutputRoot.XMLNSC.ns:Root.ns:Header.ns:Id = 'ABC';

-- WRONG — Do NOT use curly braces {ns} around declared namespace constants
-- SET OutputRoot.XMLNSC.{ns}Root.{ns}Header.{ns}Id = 'ABC';

-- WRONG — unnecessary and can interfere with subsequent SETs
CREATE LASTCHILD OF OutputRoot DOMAIN 'XMLNSC';
DECLARE doc REFERENCE TO OutputRoot.XMLNSC;
CREATE LASTCHILD OF doc AS doc TYPE XMLNSC.Folder NAMESPACE ns NAME 'Root';
CREATE LASTCHILD OF doc AS doc TYPE XMLNSC.Folder NAMESPACE ns NAME 'Header';
SET OutputRoot.XMLNSC.ns:Root.ns:Header.ns:Id = 'ABC';
```

The only time you need an explicit `CREATE LASTCHILD ... TYPE XMLNSC.Folder` is when you are building **repeating elements** (e.g. `LineItem` in a loop), where you need a new named child each iteration and `SET` alone would overwrite the previous one.

## Reserved variable names — do not use as identifiers

Some words are ESQL reserved keywords and will cause parse errors if used as variable or reference names. Common ones to avoid:

| Keyword | Why it appears | Safe alternative |
| --- | --- | --- |
| `OUT` | Parameter direction keyword in `CREATE PROCEDURE` | `outRef`, `wmsRoot`, `xmlOut`, etc. |
| `IN` | Parameter direction keyword | `inRef`, `reqBody`, etc. |
| `INOUT` | Parameter direction keyword | `ioRef`, etc. |

**Never use `out`, `in`, or `inout` as a `DECLARE` variable name**, even in lowercase — ESQL keywords are case-insensitive.

## DECLARE REFERENCE initialisation — `.*[<]` is not valid

The `.*[<]` (last-child) expression is valid inside `SET` and `IF` statements but **cannot** be used as the initialiser of a `DECLARE REFERENCE TO` statement. The parser rejects it with a syntax error.

```esql
-- WRONG — syntax error at startup
DECLARE hdr REFERENCE TO OutputRoot.XMLNSC.*[<];

-- CORRECT — declare against a known path, then MOVE to last child
DECLARE hdr REFERENCE TO OutputRoot.XMLNSC;
MOVE hdr LASTCHILD;
```

The two-statement form is the standard pattern for positioning a reference at a newly created child element immediately after a `CREATE LASTCHILD OF parent ...` statement.

## Known valid ACE ESQL constructs

- `FIELDVALUE(...)` is a valid ACE ESQL built-in and must not be replaced because it is described as unresolved.
- `LASTMOVE(...)` is a valid ACE ESQL built-in for reference navigation.
- `MOVE ref NEXTSIBLING NAME 'X';` is valid ACE ESQL syntax for named sibling traversal.
- `MOVE ref NEXTSIBLING NAME 'item';` is the correct form for advancing through a JSON input array — JSON array elements under the ACE HTTPInput/JSON parser are stored as siblings named `item` (lowercase). Using `MOVE ref NEXTSIBLING;` without `NAME` will advance past the array boundary into the next field of the parent object.
- `MOVE ref NEXTSIBLING;` (without `NAME`) is valid ACE ESQL syntax but is **not** used for JSON array traversal — use `MOVE ref NEXTSIBLING NAME 'item';` instead.
- `THE(SELECT * FROM Database.DS.SCHEMA.TABLE AS T WHERE ...)` is valid ACE ESQL for single-row database lookup.
- `DECLARE x ROW;` is a valid ACE ESQL declaration for holding a database result row.
- `UUIDASCHAR` is a valid ACE ESQL built-in that returns a UUID as a character string — used **without** parentheses when generating a new UUID.
- `DECLARE ns NAMESPACE 'uri';` is valid ACE ESQL syntax for declaring an XML namespace constant.
- `CREATE LASTCHILD OF ref NAMESPACE ns NAME 'Element'` is valid ACE ESQL for namespace-qualified XML element creation.
- `ref.ns:FieldName` is valid ACE ESQL syntax for namespace-qualified field access.
- `SET OutputRoot.XMLNSC.ns:Root.(XMLNSC.NamespaceDecl)xmlns:ns = ns;` is the correct way to declare a namespace prefix on a root XML element so the serialiser emits `xmlns:ns="..."` on the element rather than a prefix on every descendant.
- `TYPE JSON.Object` is valid ACE ESQL syntax for explicit JSON object creation.
- `CREATE FIELD ... IDENTITY (JSON.Array)` is valid ACE ESQL syntax for explicit JSON array creation.
- Create JSON scalar values by assigning them with `SET outRef.Field = value` rather than inventing typed scalar nodes.
- Do not claim that these constructs are invalid unless they are being used in a way that contradicts the ACE guidance in this file.

## Related files

- [`skills/ace-esql/SKILL.md`](../ace-esql/SKILL.md)
- [`skills/shared/review-checklist.md`](review-checklist.md)
