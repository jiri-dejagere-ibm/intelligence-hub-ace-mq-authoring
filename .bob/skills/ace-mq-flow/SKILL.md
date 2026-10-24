---
name: ace-mq-flow
description: Use when the user wants to create an IBM App Connect Enterprise message flow that uses MQ Input, MQ Output, or MQ Get nodes — including the Application project scaffolding, MQEndpoint policy project, and all IBM MQ on Cloud (MQOC) specific rules.
metadata:
  version: 1.0.0
---

# ACE MQ Flow Skill

## Purpose

This skill covers everything needed to produce a working ACE MQ message flow from scratch:

- Application project scaffolding
- MQEndpoint Policy project scaffolding
- MQ Input / MQ Output / MQ Get node attributes
- IBM MQ on Cloud (MQOC) specific rules

Apply every rule in this document without exception. Do not invent attribute names or node types.

---

## Step 1 — Gather required inputs

Before generating any files, confirm the following with the user if not already provided:

| Input | Required? | Notes |
| --- | --- | --- |
| Input queue name | Yes | MQ Input node `queueName` |
| Output queue name | Yes (if MQ Output node needed) | MQ Output node `queueName` |
| Queue manager name | Yes | Used in both the MQ Output node and the MQEndpoint policy |
| Queue manager hostname | Yes | Hostname only — no port |
| Listener port | Yes | Plain number e.g. `1414` |
| Channel name | Yes | e.g. `CLOUD.APP.SVRCONN` |
| Security identity name | Yes | Matches `mqsisetdbparms` resource, e.g. `mqcred` → `mq::mqcred` |
| SSL cipher spec | Yes if using TLS | See MQ on Cloud note below — use `ANY_TLS13` for IBM MQ on Cloud |
| SSL peer name | Yes if using TLS | Leave **empty** for IBM MQ on Cloud (see note below) |
| Target is IBM MQ on Cloud? | Yes | Determines cipher spec, peer name, `messageContextMode`, and truststore guidance |
| Application project name | If not provided, derive from flow name | e.g. `PutQueueApp` |
| Policy project name | If not provided, derive from QM name | e.g. `MQCloudPolicyProject` |
| Policy name | If not provided, derive from QM name | e.g. `MQCloudEndpoint` |
| Flow name | If not provided, ask | e.g. `PutQueue` |
| ACE version | If schema validation needed | Default to ACE 13 if not stated |

---

## Step 2 — Application project scaffolding

An ACE Application project requires exactly these four files. **All must be generated as file output** — never described in prose only.

### `<AppProjectName>/.project`

Use the template below verbatim. Replace `<AppProjectName>` with the actual project name. The file must contain **exactly two natures** — do not add or remove any.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<projectDescription>
	<name><AppProjectName></name>
	<comment></comment>
	<projects>
	</projects>
	<buildSpec>
		<buildCommand>
			<name>com.ibm.etools.mft.applib.applibbuilder</name>
			<arguments>
			</arguments>
		</buildCommand>
		<buildCommand>
			<name>com.ibm.etools.mft.applib.applibresourcevalidator</name>
			<arguments>
			</arguments>
		</buildCommand>
		<buildCommand>
			<name>com.ibm.etools.mft.connector.policy.ui.PolicyBuilder</name>
			<arguments>
			</arguments>
		</buildCommand>
		<buildCommand>
			<name>com.ibm.etools.mft.applib.mbprojectbuilder</name>
			<arguments>
			</arguments>
		</buildCommand>
	</buildSpec>
	<natures>
		<nature>com.ibm.etools.msgbroker.tooling.applicationNature</nature>
		<nature>com.ibm.etools.msgbroker.tooling.messageBrokerProjectNature</nature>
	</natures>
</projectDescription>
```

### `<AppProjectName>/.settings/org.eclipse.core.resources.prefs`

Replace `<AppProjectName>` with the actual project name in the encoding key — do **not** use the literal text `<project>`.

```properties
eclipse.preferences.version=1
encoding/<AppProjectName>=UTF-8
```

### `<AppProjectName>/application.descriptor`

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns2:appDescriptor xmlns="http://com.ibm.etools.mft.descriptor.base" xmlns:ns2="http://com.ibm.etools.mft.descriptor.app">
  <references/>
</ns2:appDescriptor>
```

⚠ **The element name is `ns2:appDescriptor` and the `ns2` namespace must be `http://com.ibm.etools.mft.descriptor.app`** — not `.base`. Using `applicationDescriptor` as the element name, or giving `ns2` the same URI as the default namespace, causes the ACE Toolkit to reject the descriptor at packaging time with `1357E: Source file could not be processed`.

**Never** add a `<reference>` pointing to the Policy project here. Policy projects are server-scoped and are available to all flows automatically. Adding a reference causes `1357E: Source file could not be processed` at deployment time.

### `<AppProjectName>/<FlowName>.msgflow`

See Step 4 for the `.msgflow` template.

---

## Step 3 — Policy project scaffolding

A Policy project requires exactly these files. **All must be generated as file output.**

### `<PolicyProjectName>/.project`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<projectDescription>
	<name><PolicyProjectName></name>
	<comment></comment>
	<projects>
	</projects>
	<buildSpec>
		<buildCommand>
			<name>com.ibm.etools.mft.policy.ui.policybuilder</name>
			<arguments>
			</arguments>
		</buildCommand>
	</buildSpec>
	<natures>
		<nature>com.ibm.etools.mft.policy.ui.Nature</nature>
	</natures>
</projectDescription>
```

### `<PolicyProjectName>/.settings/org.eclipse.core.resources.prefs`

```properties
eclipse.preferences.version=1
encoding/<PolicyProjectName>=UTF-8
```

### `<PolicyProjectName>/policy.descriptor`

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns2:policyProjectDescriptor xmlns="http://com.ibm.etools.mft.descriptor.base" xmlns:ns2="http://com.ibm.etools.mft.descriptor.policyProject">
  <references/>
</ns2:policyProjectDescriptor>
```

### `<PolicyProjectName>/<PolicyName>.policyxml`

Use **separate fields** for hostname and port. Do **not** use `<connectionName>host(port)</connectionName>` — that field is not recognised and leaves `queueManagerHostname` blank, causing `BIP2214E`.

The `<policy>` element **must** be wrapped in a `<policies>` root element. A bare `<policy>` root causes the runtime to silently discard the project (`hasChildren: false`), and the flow fails with `1361E: requires Policy 'X' which is not deployed`.

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<policies>
<policy policyType="MQEndpoint" policyName="<PolicyName>" policyTemplate="MQEndpoint">
    <connection>CLIENT</connection>
    <destinationQueueManagerName><QueueManagerName></destinationQueueManagerName>
    <queueManagerHostname><HostnameOnly></queueManagerHostname>
    <listenerPortNumber><Port></listenerPortNumber>
    <channelName><ChannelName></channelName>
    <CCDTUrl></CCDTUrl>
    <securityIdentity><SecurityIdentityName></securityIdentity>
    <useSSL>true</useSSL>
    <SSLPeerName><SSLPeerName></SSLPeerName>
    <SSLCipherSpec><SSLCipherSpec></SSLCipherSpec>
    <SSLCertificateLabel></SSLCertificateLabel>
    <MQApplName></MQApplName>
    <reconnectOption>default</reconnectOption>
</policy>
</policies>
```

**Required fields checklist:**

- `policyTemplate="MQEndpoint"` on the `<policy>` element — not `policyVersion="1"`
- `<connection>CLIENT</connection>`
- `<destinationQueueManagerName>` — queue manager name, not hostname
- `<queueManagerHostname>` — hostname **only**, no port
- `<listenerPortNumber>` — port as a plain integer
- `<channelName>` — not `<channel>`
- `<securityIdentity>` — lowercase `s`; matches `mqsisetdbparms` resource name (e.g. `mq::mqcred` → use `mqcred` here)

**Do not use:**

- `<connectionName>host(port)</connectionName>`
- `policyVersion="1"` — use `policyTemplate="MQEndpoint"` instead
- `<channel>` — must be `<channelName>`
- `<SecurityIdentity>` (capital S)

### IBM MQ on Cloud — TLS cipher spec and peer name

IBM MQ on Cloud queue managers run MQ **9.3+** and have TLS 1.3 enabled on their channels. This has two important consequences:

**Cipher spec — use `ANY_TLS13`**

IBM MQ on Cloud channels are configured with TLS 1.3 cipher specs. `ANY_TLS12_OR_HIGHER` negotiates TLS 1.2 suites, which are rejected with `AMQ9616E` (reason code `(e)` — incompatible CipherSpec). Use `ANY_TLS13` instead, which allows the MQ client to negotiate any TLS 1.3 cipher suite and matches the server-side configuration:

```xml
<SSLCipherSpec>ANY_TLS13</SSLCipherSpec>
```

**Peer name — leave empty**

MQ on Cloud certificates do not present a CN that matches what `SSLPeerName` expects. Setting `<SSLPeerName>CN=<hostname></SSLPeerName>` causes the connection to fail with peer name verification errors (`AMQ9637E`). Always leave `<SSLPeerName>` **empty**:

```xml
<SSLPeerName></SSLPeerName>
```

**Truststore — IWHI provides one built-in**

When deploying to IWHI (IBM App Connect as a Service), **do not provide a custom truststore** for IBM MQ on Cloud. IWHI has a built-in MQ on Cloud truststore that covers all IBM MQ on Cloud queue managers. Providing a custom truststore configuration is unnecessary and can cause deployment errors.

---

## Step 4 — Message flow

### MQ Input node attributes

Node type: namespace prefix of `ComIbmMQInput.msgnode`

Required attributes:

- `queueName="<InputQueueName>"` — always required; ask if not provided
- `policyUrl="{<PolicyProjectName>}:<PolicyName>"` — required when using an MQEndpoint policy

### MQ Output node attributes

Node type: namespace prefix of `ComIbmMQOutput.msgnode`

Required attributes:

- `queueName="<OutputQueueName>"` — required
- `destinationQueueManagerName="<QueueManagerName>"` — **always required**, even when `policyUrl` is set. Omitting it causes `BIP2214E: Attribute value mandatory ['queueManagerHostname']` at startup because the runtime validates this field before policy resolution.
- `policyUrl="{<PolicyProjectName>}:<PolicyName>"` — required when using an MQEndpoint policy

⚠ **`policyUrl` not `policyName`** — `policyName` is not recognised by the runtime and causes `1361E: requires Policy 'Default' which is not deployed`.

⚠ **`persistenceMode` valid values** — `no`, `automatic`, `asQdef`, `yes`. The value `asPublished` is **not valid** and causes `BIP2211E` at startup. Use `asQdef` for queue-defined persistence.

⚠ **`transactionMode` valid values** — `no`, `automatic`, `yes`.

⚠ **IBM MQ on Cloud — `messageContextMode="none"` is mandatory** — IBM MQ on Cloud (MQOC) application channels do not hold `setall` or `setidentity` context authority. If `messageContextMode` is left at its default (`default`) or set to `setIdentityContext` / `setAllContext`, the MQPUT will fail with `MQRC_NOT_AUTHORIZED (2035)`. Always add `messageContextMode="none"` to every MQ Output node when the target queue manager is hosted on IBM MQ on Cloud.

### HTTP Input / HTTP Reply node types (REST API flows)

When the flow is triggered by an HTTP REST API call rather than an MQ Input, use:

| Node       | Type in `.msgflow`      |
| ---------- | ----------------------- |
| HTTP Input | `ComIbmWSInput.msgnode` |
| HTTP Reply | `ComIbmWSReply.msgnode` |

⚠ **Do not use `ComIbmWSInputResponse.msgnode`** — that type does not exist and will cause the flow to fail at startup.

HTTP Input key attributes:

- `URLSpecifier="/<path>"` — the path after the base URL, e.g. `"/putQueue"`
- `httpMethodRestriction="PUT"` (or `GET`, `POST`, etc.)

HTTP Reply has no required extra attributes beyond `xmi:type` and `xmi:id`.

### HTTP Request-Reply over MQ Patterns

#### Pattern A — Pass-through / Echo Reply (single Compute)

When the message body put onto the MQ queue is also returned directly as the HTTP response:

HTTP Input → MQ Output → HTTP Reply `.msgflow` skeleton:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ecore:EPackage xmi:version="2.0"
  xmlns:xmi="http://www.omg.org/XMI"
  xmlns:ComIbmWSInput.msgnode="ComIbmWSInput.msgnode"
  xmlns:ComIbmWSReply.msgnode="ComIbmWSReply.msgnode"
  xmlns:ComIbmMQOutput.msgnode="ComIbmMQOutput.msgnode"
  xmlns:ecore="http://www.eclipse.org/emf/2002/Ecore"
  xmlns:eflow="http://www.ibm.com/wbi/2005/eflow"
  xmlns:utility="http://www.ibm.com/wbi/2005/eflow_utility"
  nsURI="<FlowName>.msgflow"
  nsPrefix="<FlowName>.msgflow">
  <eClassifiers xmi:type="eflow:FCMComposite" name="FCMComposite_1" nodeLayoutStyle="RECTANGLE">
    <eSuperTypes href="http://www.ibm.com/wbi/2005/eflow#//FCMBlock"/>
    <translation xmi:type="utility:TranslatableString" key="<FlowName>" bundleName="<FlowName>" pluginId="<AppProjectName>"/>
    <composition>
      <nodes xmi:type="ComIbmWSInput.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_1"
        location="50,50"
        URLSpecifier="/<path>"
        httpMethodRestriction="PUT">
        <translation xmi:type="utility:ConstantString" string="HTTP Input"/>
      </nodes>
      <nodes xmi:type="ComIbmMQOutput.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_2"
        location="250,50"
        queueName="<OutputQueueName>"
        destinationQueueManagerName="<QueueManagerName>"
        policyUrl="{<PolicyProjectName>}:<PolicyName>"
        messageContextMode="none">
        <translation xmi:type="utility:ConstantString" string="MQ Output"/>
      </nodes>
      <nodes xmi:type="ComIbmWSReply.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_3"
        location="450,50">
        <translation xmi:type="utility:ConstantString" string="HTTP Reply"/>
      </nodes>
      <connections xmi:type="eflow:FCMConnection" xmi:id="FCMConnection_1"
        targetNode="FCMComposite_1_2"
        sourceNode="FCMComposite_1_1"
        sourceTerminalName="OutTerminal.out"
        targetTerminalName="InTerminal.in"/>
      <connections xmi:type="eflow:FCMConnection" xmi:id="FCMConnection_2"
        targetNode="FCMComposite_1_3"
        sourceNode="FCMComposite_1_2"
        sourceTerminalName="OutTerminal.out"
        targetTerminalName="InTerminal.in"/>
    </composition>
    <propertyOrganizer/>
    <stickyBoard/>
  </eClassifiers>
</ecore:EPackage>
```

#### Pattern B — Decoupled Response (e.g. 202 Accepted + empty JSON or acknowledgment)

When placing a message on the queue (e.g. XMLNSC) but returning a custom response to the HTTP caller (e.g. `202 Accepted` with an empty JSON body `{}`), place a second Compute node after `MQ Output`:

`HTTP Input` → `BuildMQMessage_Compute` → `MQ Output` → `SetHTTPReply_Compute` → `HTTP Reply`

In `SetHTTPReply_Compute.esql`:

```esql
CREATE COMPUTE MODULE SetHTTPReply_Compute
    CREATE FUNCTION Main() RETURNS BOOLEAN
    BEGIN
        -- Preserve HTTP reply identifier so HTTP Reply node knows which client connection to reply to
        SET OutputLocalEnvironment.Destination.HTTP.RequestIdentifier = InputLocalEnvironment.Destination.HTTP.RequestIdentifier;

        -- Create HTTPReplyHeader FIRST (before body) to avoid BIP6062W header sequence errors
        CREATE LASTCHILD OF OutputRoot DOMAIN 'HTTPReplyHeader' NAME 'HTTPReplyHeader';
        SET OutputRoot.HTTPReplyHeader."X-Original-HTTP-Status-Code" = 202;
        SET OutputRoot.HTTPReplyHeader."Content-Type" = 'application/json';

        -- Return empty JSON object {}
        CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON';
        CREATE LASTCHILD OF OutputRoot.JSON TYPE JSON.Object NAME 'Data';

        RETURN TRUE;
    END;
END MODULE;
```

### MQ Get node attributes (if needed)

Node type: namespace prefix of `ComIbmMQGet.msgnode`

Attributes: `queueName="<QueueName>"`, `policyUrl="{<PolicyProjectName>}:<PolicyName>"`

### `.msgflow` template

The example below is a minimal MQ Input → MQ Output flow. Add or remove nodes to match the user's request. Adjust `xmi:id` values, `location` coordinates, and namespace declarations for all nodes actually used.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ecore:EPackage xmi:version="2.0"
  xmlns:xmi="http://www.omg.org/XMI"
  xmlns:ComIbmMQInput.msgnode="ComIbmMQInput.msgnode"
  xmlns:ComIbmMQOutput.msgnode="ComIbmMQOutput.msgnode"
  xmlns:ecore="http://www.eclipse.org/emf/2002/Ecore"
  xmlns:eflow="http://www.ibm.com/wbi/2005/eflow"
  xmlns:utility="http://www.ibm.com/wbi/2005/eflow_utility"
  nsURI="<FlowName>.msgflow"
  nsPrefix="<FlowName>.msgflow">
  <eClassifiers xmi:type="eflow:FCMComposite" name="FCMComposite_1" nodeLayoutStyle="RECTANGLE">
    <eSuperTypes href="http://www.ibm.com/wbi/2005/eflow#//FCMBlock"/>
    <translation xmi:type="utility:TranslatableString" key="<FlowName>" bundleName="<FlowName>" pluginId="<AppProjectName>"/>
    <composition>
      <nodes xmi:type="ComIbmMQInput.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_1"
        location="50,50"
        queueName="<InputQueueName>"
        policyUrl="{<PolicyProjectName>}:<PolicyName>">
        <translation xmi:type="utility:ConstantString" string="MQ Input"/>
      </nodes>
      <nodes xmi:type="ComIbmMQOutput.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_2"
        location="350,50"
        queueName="<OutputQueueName>"
        destinationQueueManagerName="<QueueManagerName>"
        policyUrl="{<PolicyProjectName>}:<PolicyName>"
        messageContextMode="none">
        <translation xmi:type="utility:ConstantString" string="MQ Output"/>
      </nodes>
      <connections xmi:type="eflow:FCMConnection" xmi:id="FCMConnection_1"
        targetNode="FCMComposite_1_2"
        sourceNode="FCMComposite_1_1"
        sourceTerminalName="OutTerminal.out"
        targetTerminalName="InTerminal.in"/>
    </composition>
    <propertyOrganizer/>
    <stickyBoard/>
  </eClassifiers>
</ecore:EPackage>
```

**Only declare namespace prefixes for nodes that actually appear in the flow.** Remove unused `xmlns:` declarations from the `<ecore:EPackage>` opening tag.

---

## Step 5 — Credentials setup (runtime)

The `<securityIdentity>` value in the policy file must correspond to an `mqsisetdbparms` resource set on the integration server. The command format is:

```
mqsisetdbparms <integration-server> -n mq::<securityIdentity> -u <app-username> -p <api-key-or-password>
```

For IBM MQ on Cloud the username is typically the app user name from the MQ on Cloud service credentials, and the password is the corresponding API key.

On IWHI, credentials are supplied via a `setdbparms` **Configuration object** instead of running `mqsisetdbparms` directly.

---

## Step 6 — Deploying to IWHI (if applicable)

If the target is IBM App Connect as a Service (IWHI):

1. **Do not** package the policy project into the BAR file.
2. Build a separate policy zip with the project folder at the root:
   ```bash
   cd <parent-of-PolicyProjectDir>
   zip -r policy.zip <PolicyProjectName>/
   ```
   Verify: entries must look like `<PolicyProjectName>/` and `<PolicyProjectName>/<PolicyName>.policyxml` — **not** flat.
3. Base64-encode the zip:
   ```bash
   base64 < policy.zip
   ```
4. Create a `policyproject` Configuration object using `configurations` (action: replace) (not `configurations` (action: create)). The body must use `spec.data` for the base64 payload:
   ```json
   {
     "metadata": { "name": "my-policyproject" },
     "spec": { "type": "policyproject", "data": "<base64>" }
   }
   ```
5. Create a `setdbparms` Configuration object for MQ credentials:
   ```json
   {
     "metadata": { "name": "my-setdbparms" },
     "spec": {
       "type": "setdbparms",
       "data": "bXE6Om1xY3JlZCBhcHAyIHBhc3N3b3Jk"
     }
   }
   ```
   The `spec.data` value is base64 of the string `mq::<securityIdentity> <username> <password>` — e.g. `mq::mqcred app2 <apikey>`. Note the single space between each token; no newline.
6. Create an `mqclientini` Configuration object for `mqclient.ini` (required for almost all queue manager connections, enabling TLS 1.3 and Outbound SNI):
   ```json
   {
     "metadata": { "name": "my-mqclientini" },
     "spec": {
       "type": "mqclientini",
       "data": "<base64>"
     }
   }
   ```
   The `spec.data` value is base64 of the `mqclient.ini` file contents:
   ```ini
   SSL:
     AllowTLSV13=TRUE
     OutboundSNI=HOSTNAME
   ```
   > ⚠ **Troubleshooting Note:** A missing or unattached `mqclient.ini` configuration is the **most likely initial cause of SSL/TLS connection errors** when all other settings (cipher spec, credentials, policy) appear correct.
7. Reference all configuration names in `spec.configurations` on the IntegrationRuntime CR (e.g. `["my-policyproject", "my-setdbparms", "my-mqclientini"]`).
8. The flow BAR goes into `spec.barURL` as normal.

---

## Step 7 — Pre-output review checklist

Before returning output, verify every item:

- [ ] All four Application project files are present as actual file output: `.project`, `.settings/org.eclipse.core.resources.prefs`, `application.descriptor`, `.msgflow`
- [ ] `.project` contains exactly two natures — no extras
- [ ] `.settings/org.eclipse.core.resources.prefs` uses the actual project name in the encoding key — not the literal `<project>`
- [ ] `application.descriptor` uses element `ns2:appDescriptor` with `xmlns:ns2="http://com.ibm.etools.mft.descriptor.app"` — not `applicationDescriptor` or the `.base` namespace
- [ ] `application.descriptor` has `<references/>` with no child elements — no policy project reference
- [ ] All Policy project files are present: `.project`, `.settings/org.eclipse.core.resources.prefs`, `policy.descriptor`, `<PolicyName>.policyxml`
- [ ] `<policies>` wrapper wraps `<policy>` in the policyxml file
- [ ] `policyTemplate="MQEndpoint"` is set on `<policy>` — not `policyVersion`
- [ ] `<queueManagerHostname>` contains hostname only — no port
- [ ] `<listenerPortNumber>` contains a plain integer — no brackets
- [ ] `<channelName>` is used — not `<channel>`
- [ ] `<securityIdentity>` is lowercase — not `<SecurityIdentity>`
- [ ] If target is IBM MQ on Cloud: `<SSLPeerName>` is **empty**, `<SSLCipherSpec>` is `ANY_TLS13`
- [ ] If deploying to IWHI targeting MQ on Cloud: **no custom truststore** provided — IWHI has a built-in one
- [ ] If deploying to IWHI: `mqclientini` configuration attached to runtime containing `AllowTLSV13=TRUE` and `OutboundSNI=HOSTNAME`
- [ ] MQ Output node has `destinationQueueManagerName` set
- [ ] MQ Output node uses `policyUrl` — not `policyName`
- [ ] MQ Output node has `messageContextMode="none"` if target is IBM MQ on Cloud
- [ ] HTTP Input node type is `ComIbmWSInput.msgnode` — not `ComIbmWSInputResponse.msgnode`
- [ ] HTTP Reply node type is `ComIbmWSReply.msgnode`
- [ ] Only namespace prefixes for nodes actually used appear in the `.msgflow` `<ecore:EPackage>` header
- [ ] Version assumptions stated if ACE version was not confirmed by user

---

## Related shared files

- [`skills/shared/node-types.md`](../shared/node-types.md)
- [`skills/shared/policy-projects.md`](../shared/policy-projects.md)
- [`skills/shared/ace-projects.md`](../shared/ace-projects.md)
- [`skills/shared/ace-versions.md`](../shared/ace-versions.md)
- [`skills/shared/review-checklist.md`](../shared/review-checklist.md)
- [`skills/ace-iwhi/SKILL.md`](../ace-iwhi/SKILL.md) — for IWHI-specific deployment steps
