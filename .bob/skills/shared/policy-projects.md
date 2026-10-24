---
name: policy-projects
description: Structure and descriptor rules for IBM App Connect Enterprise Policy projects. Read when creating or validating any ACE policy project.
metadata:
  version: 1.0.0
---

# ACE Policy Projects

## Purpose

This document defines the required structure for IBM App Connect Enterprise (ACE) Policy projects.

## When to use

Use this document when a connector node requires a policy project or when reviewing existing ACE Policy project scaffolding.

## When to create a Policy project

- Create a Policy project when a connector node requires a policy and the user has not identified an existing suitable Policy project.
- Policy files must be created in a Policy project, not in an Application project.

## Required project files

An ACE Policy project must contain:

- a `.project` file in the project root
- a `.settings` directory
- a `.settings/org.eclipse.core.resources.prefs` file
- a `policy.descriptor` file in the project root

## `.project` requirements

ACE Policy projects **MUST** contain the following natures section:

```xml
<natures>
  <nature>com.ibm.etools.mft.policy.ui.Nature</nature>
</natures>
```

ACE Policy projects **MUST** contain the following buildSpec section:

```xml
<buildSpec>
  <buildCommand>
    <name>com.ibm.etools.mft.policy.ui.policybuilder</name>
    <arguments>
    </arguments>
  </buildCommand>
</buildSpec>
```

## `.settings` requirements

The `.settings/org.eclipse.core.resources.prefs` file must contain:

```properties
eclipse.preferences.version=1
encoding/<project>=UTF-8
```

## `policy.descriptor` requirements

The `policy.descriptor` file must exist in the project root and use the validated ACE structure.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns2:policyProjectDescriptor xmlns="http://com.ibm.etools.mft.descriptor.base" xmlns:ns2="http://com.ibm.etools.mft.descriptor.policyProject">
  <references/>
</ns2:policyProjectDescriptor>
```

## Policy file format

Policy `.policyxml` files **must** wrap the `<policy>` element inside a `<policies>` root element.

> **⚠ `<policies>` wrapper is required.** A bare `<policy ...>` as the root element causes the ACE runtime to silently load the project as empty (`hasChildren: false`) — the policy file is present on disk but no policies are registered. The flow fails with `1361E: requires Policy 'X' which is not deployed`.

### MQEndpoint policy — correct field structure

Use **separate fields** for hostname and port. Do **not** use `<connectionName>host(port)</connectionName>` — that field is not recognised by the runtime and leaves `queueManagerHostname` blank, causing `BIP2214E` at startup.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<policies>
<policy policyType="MQEndpoint" policyName="MyPolicy" policyTemplate="MQEndpoint">
    <connection>CLIENT</connection>
    <destinationQueueManagerName>QMNAME</destinationQueueManagerName>
    <queueManagerHostname>your.qm.host.example.com</queueManagerHostname>
    <listenerPortNumber>1414</listenerPortNumber>
    <channelName>CLOUD.APP.SVRCONN</channelName>
    <CCDTUrl></CCDTUrl>
    <securityIdentity>mqcred</securityIdentity>
    <useSSL>true</useSSL>
    <SSLPeerName></SSLPeerName>
    <SSLCipherSpec>ANY_TLS13</SSLCipherSpec>
    <SSLCertificateLabel></SSLCertificateLabel>
    <MQApplName></MQApplName>
    <reconnectOption>default</reconnectOption>
</policy>
</policies>
```

**Required fields** (all must be present even if empty):

- `policyTemplate="MQEndpoint"` on the `<policy>` element
- `<connection>CLIENT</connection>`
- `<destinationQueueManagerName>` — queue manager name
- `<queueManagerHostname>` — hostname only, no port
- `<listenerPortNumber>` — port as a plain number
- `<channelName>`
- `<securityIdentity>` — matches the resource name in `mqsisetdbparms` (e.g. `mq::mqcred`)

**Do not use:**

- `<connectionName>host(port)</connectionName>` — not a valid MQEndpoint field
- `policyVersion="1"` — use `policyTemplate="MQEndpoint"` instead
- `<channel>` — the correct element name is `<channelName>`
- `<SecurityIdentity>` (capital S) — use lowercase `<securityIdentity>`

**IBM MQ on Cloud — TLS cipher spec and peer name:**

IBM MQ on Cloud queue managers (MQ 9.3+) have TLS 1.3 enabled on their application channels. Three rules apply:

1. **Cipher spec on IWHI** — use **`ANY_TLS13`** rather than an explicit cipher name such as `TLS_AES_256_GCM_SHA384`. On IWHI pods, using an explicit TLS 1.3 cipher name with a `securityIdentity` / `mq::` setdbparms credential causes `AMQ9620E: gsk_environment_open` (MQRC 2393) on every fresh pod start. This is a GSKit process-singleton conflict — the HTTPS listener and the MQ channel both try to initialise the GSKit TLS environment, and the explicit cipher path fails on the second call. `ANY_TLS13` uses a negotiation path that avoids this conflict.

   ```xml
   <SSLCipherSpec>ANY_TLS13</SSLCipherSpec>
   ```

   > **Note:** `ANY_TLS12_OR_HIGHER` negotiates TLS 1.2 which is rejected by a TLS 1.3-only channel (`AMQ9616E`). Do not use it for MQ on Cloud.

2. **Peer name** — leave `<SSLPeerName>` **empty**. MQ on Cloud certificates do not match a pattern suitable for peer name verification; setting `CN=<hostname>` causes `AMQ9637E`.

3. **Truststore** — when deploying to IWHI, do **not** provide a custom truststore. IWHI includes a built-in MQ on Cloud truststore.

**IBM MQ on Cloud — MQ Output node `messageContextMode`:** When the MQEndpoint policy targets an IBM MQ on Cloud queue manager, the MQ Output node in the `.msgflow` file **must** have `messageContextMode="none"`. MQ on Cloud application channels do not hold `setall` or `setidentity` context authority; omitting this attribute (or leaving it at `default`) causes the put to fail with `MQRC_NOT_AUTHORIZED (2035)`.

## Policy file placement

- Policy files should be created in the root of the Policy project.
- If the user does not specify a policy file name, derive one from the node type and a numeric suffix.

## Validation

- Validate policy XML structure using the Policy schema for the applicable ACE version.
- For schema locations, refer to [`skills/shared/ace-versions.md`](ace-versions.md).

## Related files

- [`skills/shared/ace-versions.md`](ace-versions.md)
- [`skills/shared/connector-index.md`](connector-index.md)
