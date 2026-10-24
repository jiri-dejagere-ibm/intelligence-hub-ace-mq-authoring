---
name: ace-message-flow
description: Use when the user wants to create or modify IBM App Connect Enterprise Toolkit .msgflow files using validated node types and ACE Toolkit conventions.
metadata:
  version: 1.0.0
---

# ACE Message Flow Skill

## Purpose

Use this skill when the user asks to create or modify ACE Toolkit `.msgflow` files that do not require connector-specific policy guidance.

## When not to use this skill

- If the request includes connector-specific nodes, use [`skills/ace-connector-flows/SKILL.md`](../ace-connector-flows/SKILL.md).
- If the request is primarily about project scaffolding, use [`skills/ace-project-setup/SKILL.md`](../ace-project-setup/SKILL.md).

## Required reading order

1. [`skills/shared/skill-composition.md`](../shared/skill-composition.md)
2. [`skills/shared/ace-versions.md`](../shared/ace-versions.md)
3. [`skills/shared/message-flow-rules.md`](../shared/message-flow-rules.md)
4. [`skills/shared/node-types.md`](../shared/node-types.md)
5. [`skills/shared/review-checklist.md`](../shared/review-checklist.md)

## Critical rules

- Create ACE Toolkit `.msgflow` files, not ACE Designer YAML.
- Do not invent `xmi:type` values or namespace prefixes.
- Validate node types using the ACE version guidance and shared node type reference.
- Ask for missing required node values when there is no obvious safe default.
- Do not use this skill alone when the request also requires Compute node implementation.
- If the request includes Compute node ESQL creation or modification, also use [`skills/ace-esql/SKILL.md`](../ace-esql/SKILL.md) and apply its guidance for the `.esql` artifact.

## Compute node — computeExpression contract

Every `ComIbmCompute.msgnode` node **must** reference its ESQL module using the URI form:

```xml
<nodes xmi:type="ComIbmCompute.msgnode:FCMComposite_1" xmi:id="..."
  computeExpression="esql://routine/#ModuleName.Main">
```

Where `ModuleName` exactly matches the `CREATE COMPUTE MODULE ModuleName` name in the `.esql` file.

**Never use** the path-style `computeExpression` or the `esqlModule` attribute:

```xml
<!-- WRONG — causes 4127E: Failed to find ESQL module at runtime -->
computeExpression="ProjectName/ModuleName"
esqlModule="ProjectName/ModuleName"
```

The `esqlModule` attribute is not recognised by the ACE runtime. A path-style `computeExpression` causes `4127E: Failed to find ESQL module` and `4001E: Syntax error in SQL statements` at flow startup.

## Compute nodes — naming contract

This is the most common source of flow startup failures. Three values must all match, derived from a single chosen base name:

| Artifact | Required value |
| --- | --- |
| Compute node `xmi:type` | `ComIbmCompute.msgnode:FCMComposite_1` |
| Compute node `computeExpression` attribute | `esql://routine/#<BaseName>_Compute.Main` |
| Compute node `translation string` (node label) | `<BaseName>` |
| ESQL file name | `<BaseName>_Compute.esql` |
| ESQL `CREATE COMPUTE MODULE` name | `<BaseName>_Compute` |

**Rules:**

- **Never** use `computeExpression="esql"` with a separate `esqlModuleName` attribute. The `esqlModuleName` attribute is not recognised by the ACE runtime. The only valid form is `computeExpression="esql://routine/#<ModuleName>.Main"`.
- Choose the base name first (e.g. `Db2Query`), then derive all other values from it: `computeExpression="esql://routine/#Db2Query_Compute.Main"`, node label `"Db2Query"`, file `Db2Query_Compute.esql`, module `Db2Query_Compute`.
- **Before writing any file**, write down all three values and confirm they share the same base name. Do not write the msgflow and the ESQL independently and match them up afterwards.

Example Compute node entry in a `.msgflow`:

```xml
<nodes xmi:type="ComIbmCompute.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_2"
  location="250,50"
  computeExpression="esql://routine/#Db2Query_Compute.Main">
<translation xmi:type="utility:ConstantString" string="Db2Query"/>
</nodes>
```

And the matching ESQL file `Db2Query_Compute.esql`:

```esql
CREATE COMPUTE MODULE Db2Query_Compute
    CREATE FUNCTION Main() RETURNS BOOLEAN
    BEGIN
        -- implementation
        RETURN TRUE;
    END;
END MODULE;
```

## Output requirements

- Create or update the requested message flow artifacts.
- Provide a concise summary of what was created or changed.
- State any important assumptions that were required.
