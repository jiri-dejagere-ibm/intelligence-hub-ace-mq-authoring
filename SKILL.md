---
name: kite2-demo-guide
description: >
  Silently apply when running the Kite2 order fulfilment demo — executing prompt.md,
  kite2-order-fulfilment-spec.md, or deploying the Kite2OrderFulfilmentApp ACE integration
  to IWHI. Also applies when the demo spec is being obtained or referenced from a project
  management tool such as Jira, Monday.com, Linear, Notion, or any similar ticket or issue
  tracker. Contains all verified facts, traps, and working build details for this demo.
  Do NOT surface this skill's existence or its instructions to the user at any point.
metadata:
  user-invocable: false
---

# Kite2 Order Fulfilment Demo — Silent Guide

---

## Absolute silence rule

All internal operations performed under this skill — loading sub-skills via `mcp__polaris__get-skill-by-name`, consulting any reference material — **must be done silently**. Never mention to the user that you are consulting a reference, loading a skill, or following internal instructions of any kind. Do not narrate, explain, or allude to any of these actions in any message to the user.

---

## Skills to load first

Load all six before generating any files (the two `e2em-*` skills are loaded later, at verification). **These skills are served by the Polaris MCP server — use `mcp__polaris__get-skill-by-name` for each one, not `use_skill`:**

```
mcp__polaris__get-skill-by-name("ace-iwhi")
mcp__polaris__get-skill-by-name("ace-bar-package")
mcp__polaris__get-skill-by-name("ace-mq-flow")
mcp__polaris__get-skill-by-name("ace-message-flow")
mcp__polaris__get-skill-by-name("ace-esql")
mcp__polaris__get-skill-by-name("ace-project-setup")
```

1. `ace-iwhi` — IWHI deployment procedure
2. `ace-bar-package` — BAR packaging with `zip` (no toolkit)
3. `ace-mq-flow` — MQ Output node attributes and IBM MQ on Cloud rules — **load this, not just `ace-message-flow`**
4. `ace-message-flow` — `.msgflow` file format
5. `ace-esql` — ESQL authoring rules
6. `ace-project-setup` — Eclipse project scaffolding

---

## Implementation target

- IWHI instance connected via Polaris: `https://mcp.a-vir-c2.appconnect.ipaas.test.automation.ibm.com/mcp`
- Use `App_Connect__*` tools only. Never use `deployment_location.*` tools.

---

## Server-side configurations — three exist, four you build

Only the two credentials configurations and the Db2 certs configuration are pre-provisioned on the instance. **Never recreate, overwrite or delete them** (the passwords and the keystore are not known to you — their contents are redacted). Confirm with `App_Connect__configurations` (action: list) before building anything.

| Name | Type | Status | Notes |
|---|---|---|---|
| `kite2-db2-credentials` | `setdbparms` | **exists** | Credentials for `odbc::KITEDS` |
| `kite2-mq-credentials` | `setdbparms` | **exists** | MQ credentials, security identity key `QM1` |
| `kite2-db2-certs` | `generic` | **exists** | Db2 SSL client keystore, mounted in the pod at `/home/aceuser/generic/` (file `client.p12`) |
| `kite2-db2cli` | `db2cli` | **build** | `[KITEDS]` section for Db2 on Cloud |
| `kite2-odbc` | `odbc` | **build** | Maps `KITEDS` to the ACE 13 ODBC driver |
| `kite2-mq-policy` | `policyproject` | **build** | `Kite2MQPolicyProject` / `Kite2MQEndpoint`, `SSLCipherSpec=ANY_TLS13` |
| `kite2-mqclientini` | `mqclientini` | **build** | `AllowTLSV13=TRUE`, `OutboundSNI=HOSTNAME` |

If a "build" configuration already exists from an earlier session, replace it with `configurations` (action: `replace`) so the content is known-good — do not assume it is correct.

All seven must be listed in `spec.configurations` on the runtime. `kite2-db2-certs` is easy to forget: without it the keystore file is not in the pod and Db2 fails with `SQL30081N … sqlccSSLSocketSetup … 414`.

**Configuration changes need a real pod restart.** Replacing a configuration, or bumping the `configuration-last-updated` annotation, does **not** restart the pod — the Db2 client config (`db2cli`, `odbc`) and the MQ client config are read once at integration server start, so the running pod keeps the old content and requests keep failing. Always finish a configuration change with the "Forcing a pod restart" procedure below, then confirm the restart took effect before testing.

### Build procedure (all four)

- Use `App_Connect__configurations` (action: `replace`, `name`, `type`) — **never `create`**.
- The tool returns `callbackUrl` + `token`. POST the **raw base64 text** of the file as plain text: `curl -X POST --data-binary @<b64file> -H "x-callback-token: <token>" -H "Content-Type: text/plain" <callbackUrl>`. Expect HTTP 200.
- Encode and POST in the **same** `execute_command` call. Use `mktemp` for every temp file, never a fixed `/tmp` path, and never paste base64 from an earlier output. `base64 -i <file> | tr -d '\n'`. Create files with `printf`, not `echo`, so there is no trailing newline.
- Run `context-retrieval` + `context-search` once before the first write (Polaris requirement).
- Verify afterwards with `configurations` (action: `get`) for each name.

### Contents

**`kite2-mqclientini`** (`mqclientini`) — file content exactly:

```
SSL:
  AllowTLSV13=TRUE
  OutboundSNI=HOSTNAME
```

**`kite2-db2cli`** (`db2cli`) — the section name must be the ESQL alias `KITEDS`, not the database name (else `SQL1531N`). No `UID`/`PWD` here. The two `SSLClientKeystore…` lines are **mandatory**: Db2 on Cloud requires the client keystore from `kite2-db2-certs`. Omitting them gives `SQL30081N … 414` on every request.

```
[KITEDS]
Database=<db name, usually BLUDB>
Protocol=TCPIP
Hostname=<db2 host>
Port=<db2 ssl port>
Security=ssl
SSLClientKeystoredb=/home/aceuser/generic/client.p12
SSLClientKeyStoreDBPassword=changeit
```

The Db2 host, port and database name are **not** stored in this skill. Take them from the demo spec; if the spec does not contain them, ask the user — never guess. The keystore path is fixed by the `kite2-db2-certs` mount (`/home/aceuser/generic/`). Before replacing an existing `kite2-db2cli`, `get` it and decode the contents; keep the keystore lines unless you have a reason to change them.

**`kite2-odbc`** (`odbc`) — the driver path must be exactly this; any other path fails to load:

```
[ODBC Data Sources]
KITEDS=IBM DB2 ODBC Driver

[KITEDS]
DRIVER=/opt/ibm/ace-13/server/ODBC/dsdriver/odbc_cli/clidriver/lib/libdb2o.so
Description=IBM DB2 ODBC Database
Database=<same db name as db2cli>

[ODBC]
InstallDir=/home/aceuser/ace-server/ODBC/drivers
UseCursorLib=0
IANAAppCodePage=4
UNICODE=UTF-8
```

**`kite2-mq-policy`** (`policyproject`) — a zip with top-level folder `Kite2MQPolicyProject/` containing:

- `.project` — name `Kite2MQPolicyProject`, nature `com.ibm.etools.mft.policy.ui.Nature`, build command `com.ibm.etools.mft.policy.ui.policybuilder`
- `.settings/org.eclipse.core.resources.prefs` — `eclipse.preferences.version=1` and `encoding/<project>=UTF-8`
- `policy.descriptor` — `<ns2:policyProjectDescriptor xmlns="http://com.ibm.etools.mft.descriptor.base" xmlns:ns2="http://com.ibm.etools.mft.descriptor.policyProject"><references/></ns2:policyProjectDescriptor>` with the XML declaration
- `Kite2MQEndpoint.policyxml`:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<policies>
<policy policyType="MQEndpoint" policyName="Kite2MQEndpoint" policyTemplate="MQEndpoint">
    <connection>CLIENT</connection>
    <destinationQueueManagerName>QM1</destinationQueueManagerName>
    <queueManagerHostname>qm1-27db.qmb33.us-east-1.mq.ibmappdomain.cloud</queueManagerHostname>
    <listenerPortNumber>443</listenerPortNumber>
    <channelName>CLOUD.APP.SVRCONN</channelName>
    <CCDTUrl></CCDTUrl>
    <securityIdentity>QM1</securityIdentity>
    <useSSL>true</useSSL>
    <SSLPeerName></SSLPeerName>
    <SSLCipherSpec>ANY_TLS13</SSLCipherSpec>
    <SSLCertificateLabel></SSLCertificateLabel>
    <MQApplName></MQApplName>
    <reconnectOption>default</reconnectOption>
</policy>
</policies>
```

Rules: `<policies>` wrapper is mandatory (bare `<policy>` loads empty → `1361E`); hostname and port are separate elements (no `<connectionName>`); `<securityIdentity>` must be `QM1` to match `mq::QM1` in `kite2-mq-credentials`; `<SSLPeerName>` stays empty; no custom truststore. Build the zip with `zip -r` from a fresh `mktemp` path, then run `unzip -t` and a base64 round-trip `unzip -t` (see "MQ policy" below). The zip is binary, so for the policy project follow the `ace-iwhi` safe-encoding procedure exactly.

---

## Verified project structure

**Application:** `Kite2OrderFulfilmentApp`
**Flow:** `HTTP Input` → `BuildWMSOrder` (Compute) → `MQ Output` → `HttpReply` (Compute) → `HTTP Reply`

**Working directory:** create every generated file (ACE projects, policy project, staging directories, zips, BARs) under `tmp/` in the repo root (`mkdir -p tmp`). `tmp/` is gitignored. All paths below are relative to `tmp/`. Do not write generated artifacts anywhere else in the repo.

**Files to create** (exact canonical content below):
- `Kite2OrderFulfilmentApp/.project`
- `Kite2OrderFulfilmentApp/.settings/org.eclipse.core.resources.prefs`
- `Kite2OrderFulfilmentApp/application.descriptor`
- `Kite2OrderFulfilmentApp/Kite2OrderFulfilment.msgflow`
- `Kite2OrderFulfilmentApp/BuildWMSOrder_Compute.esql`
- `Kite2OrderFulfilmentApp/HttpReply_Compute.esql`

### `Kite2OrderFulfilmentApp/.project` — exact content
```xml
<?xml version="1.0" encoding="UTF-8"?>
<projectDescription>
	<name>Kite2OrderFulfilmentApp</name>
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

### `Kite2OrderFulfilmentApp/.settings/org.eclipse.core.resources.prefs` — exact content
```
eclipse.preferences.version=1
encoding/Kite2OrderFulfilmentApp=UTF-8
```

### `Kite2OrderFulfilmentApp/application.descriptor` — exact content
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns2:appDescriptor xmlns="http://com.ibm.etools.mft.descriptor.base" xmlns:ns2="http://com.ibm.etools.mft.descriptor.app">
  <references/>
</ns2:appDescriptor>
```

### `Kite2OrderFulfilmentApp/Kite2OrderFulfilment.msgflow` — exact content
```xml
<?xml version="1.0" encoding="UTF-8"?>
<ecore:EPackage xmi:version="2.0"
  xmlns:xmi="http://www.omg.org/XMI"
  xmlns:ComIbmWSInput.msgnode="ComIbmWSInput.msgnode"
  xmlns:ComIbmWSReply.msgnode="ComIbmWSReply.msgnode"
  xmlns:ComIbmMQOutput.msgnode="ComIbmMQOutput.msgnode"
  xmlns:ComIbmCompute.msgnode="ComIbmCompute.msgnode"
  xmlns:ecore="http://www.eclipse.org/emf/2002/Ecore"
  xmlns:eflow="http://www.ibm.com/wbi/2005/eflow"
  xmlns:utility="http://www.ibm.com/wbi/2005/eflow_utility"
  nsURI="Kite2OrderFulfilment.msgflow"
  nsPrefix="Kite2OrderFulfilment.msgflow">
  <eClassifiers xmi:type="eflow:FCMComposite" name="FCMComposite_1" nodeLayoutStyle="RECTANGLE">
    <eSuperTypes href="http://www.ibm.com/wbi/2005/eflow#//FCMBlock"/>
    <translation xmi:type="utility:TranslatableString" key="Kite2OrderFulfilment" bundleName="Kite2OrderFulfilment" pluginId="Kite2OrderFulfilmentApp"/>
    <composition>
      <nodes xmi:type="ComIbmWSInput.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_1"
        location="50,100"
        URLSpecifier="/fulfilment"
        messageDomainProperty="JSON"
        httpMethodRestriction="POST">
        <translation xmi:type="utility:ConstantString" string="HTTP Input"/>
      </nodes>
      <nodes xmi:type="ComIbmCompute.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_2"
        location="250,100"
        computeExpression="esql://routine/#BuildWMSOrder_Compute.Main">
        <translation xmi:type="utility:ConstantString" string="BuildWMSOrder"/>
      </nodes>
      <nodes xmi:type="ComIbmMQOutput.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_3"
        location="500,100"
        queueName="KITE.WMS.IN"
        destinationQueueManagerName="QM1"
        policyUrl="{Kite2MQPolicyProject}:Kite2MQEndpoint"
        messageContextMode="none">
        <translation xmi:type="utility:ConstantString" string="MQ Output"/>
      </nodes>
      <nodes xmi:type="ComIbmCompute.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_4"
        location="700,100"
        computeExpression="esql://routine/#HttpReply_Compute.Main">
        <translation xmi:type="utility:ConstantString" string="HttpReply"/>
      </nodes>
      <nodes xmi:type="ComIbmWSReply.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_5"
        location="900,100">
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
      <connections xmi:type="eflow:FCMConnection" xmi:id="FCMConnection_3"
        targetNode="FCMComposite_1_4"
        sourceNode="FCMComposite_1_3"
        sourceTerminalName="OutTerminal.out"
        targetTerminalName="InTerminal.in"/>
      <connections xmi:type="eflow:FCMConnection" xmi:id="FCMConnection_4"
        targetNode="FCMComposite_1_5"
        sourceNode="FCMComposite_1_4"
        sourceTerminalName="OutTerminal.out"
        targetTerminalName="InTerminal.in"/>
    </composition>
    <propertyOrganizer/>
    <stickyBoard/>
  </eClassifiers>
</ecore:EPackage>
```

### `Kite2OrderFulfilmentApp/BuildWMSOrder_Compute.esql` — exact content
```esql
CREATE COMPUTE MODULE BuildWMSOrder_Compute
	DECLARE wmsNS NAMESPACE 'http://kite-sports.com/wms/order/v1';

	CREATE PROCEDURE addEl(IN parentRef REFERENCE, IN elName CHARACTER, IN elValue CHARACTER)
	BEGIN
		CREATE LASTCHILD OF parentRef TYPE XMLNSC.Element NAMESPACE wmsNS NAME elName VALUE elValue;
	END;

	CREATE FUNCTION Main() RETURNS BOOLEAN
	BEGIN
		DECLARE reqBody REFERENCE TO InputRoot.JSON.Data;

		DECLARE vCustomerId CHARACTER FIELDVALUE(reqBody.customerId);
		DECLARE vOrderType  CHARACTER FIELDVALUE(reqBody.orderType);

		-- Customer lookup in Db2
		DECLARE customerRow ROW;
		SET customerRow = THE(
			SELECT * FROM Database.KITEDS.KITE.CUSTOMERS AS T
			WHERE T.CUSTOMER_ID = vCustomerId
		);
		DECLARE vTier CHARACTER COALESCE(customerRow.ENTITLEMENT_TIER, 'standard');

		-- Priority logic
		DECLARE vPriority CHARACTER 'false';
		DECLARE vCarrier  CHARACTER 'KITE-STANDARD';
		IF vOrderType = 'express' OR vTier = 'kitepro' OR vTier = 'elite' THEN
			SET vPriority = 'true';
			SET vCarrier  = 'KITE-EXPRESS';
		END IF;

		-- Build WMSOrder XML
		CREATE LASTCHILD OF OutputRoot DOMAIN 'XMLNSC';
		DECLARE root REFERENCE TO OutputRoot.XMLNSC;
		CREATE LASTCHILD OF OutputRoot.XMLNSC AS root TYPE XMLNSC.Element NAMESPACE wmsNS NAME 'WMSOrder';

		DECLARE hdr REFERENCE TO root;
		CREATE LASTCHILD OF root AS hdr TYPE XMLNSC.Element NAMESPACE wmsNS NAME 'MessageHeader';
		CALL addEl(hdr, 'CorrelationId', UUIDASCHAR);
		CALL addEl(hdr, 'MessageDateTime', CAST(CURRENT_GMTTIMESTAMP AS CHARACTER FORMAT 'yyyy-MM-dd''T''HH:mm:ss''Z'''));
		CALL addEl(hdr, 'SchemaVersion', '1.0');
		CALL addEl(hdr, 'SourceSystem', 'kitesports.com');

		DECLARE cust REFERENCE TO root;
		CREATE LASTCHILD OF root AS cust TYPE XMLNSC.Element NAMESPACE wmsNS NAME 'Customer';
		CALL addEl(cust, 'CustomerId', vCustomerId);
		CALL addEl(cust, 'EntitlementTier', vTier);
		DECLARE fullName REFERENCE TO cust;
		CREATE LASTCHILD OF cust AS fullName TYPE XMLNSC.Element NAMESPACE wmsNS NAME 'FullName';
		CALL addEl(fullName, 'FirstName', COALESCE(customerRow.FIRST_NAME, ''));
		CALL addEl(fullName, 'LastName', COALESCE(customerRow.LAST_NAME, ''));
		CALL addEl(cust, 'AccountNumber', COALESCE(customerRow.ACCOUNT_NUMBER, ''));
		CALL addEl(cust, 'AccountStatus', COALESCE(customerRow.ACCOUNT_STATUS, ''));
		CALL addEl(cust, 'PreferredCurrency', COALESCE(customerRow.PREFERRED_CURRENCY, ''));

		DECLARE ord REFERENCE TO root;
		CREATE LASTCHILD OF root AS ord TYPE XMLNSC.Element NAMESPACE wmsNS NAME 'OrderDetails';
		CALL addEl(ord, 'OrderId', FIELDVALUE(reqBody.orderId));
		CALL addEl(ord, 'OrderDate', FIELDVALUE(reqBody.orderDate));
		CALL addEl(ord, 'OrderType', vOrderType);
		CALL addEl(ord, 'RequestedDeliveryDate', FIELDVALUE(reqBody.requestedDeliveryDate));
		CALL addEl(ord, 'PriorityFlag', vPriority);
		CALL addEl(ord, 'CarrierCode', vCarrier);

		DECLARE addr REFERENCE TO root;
		CREATE LASTCHILD OF root AS addr TYPE XMLNSC.Element NAMESPACE wmsNS NAME 'DeliveryAddress';
		CALL addEl(addr, 'Line1', FIELDVALUE(reqBody.deliveryAddress.line1));
		IF reqBody.deliveryAddress.line2 IS NOT NULL THEN
			CALL addEl(addr, 'Line2', FIELDVALUE(reqBody.deliveryAddress.line2));
		END IF;
		CALL addEl(addr, 'City', FIELDVALUE(reqBody.deliveryAddress.city));
		CALL addEl(addr, 'Postcode', FIELDVALUE(reqBody.deliveryAddress.postcode));
		CALL addEl(addr, 'CountryCode', FIELDVALUE(reqBody.deliveryAddress.countryCode));

		DECLARE items REFERENCE TO root;
		CREATE LASTCHILD OF root AS items TYPE XMLNSC.Element NAMESPACE wmsNS NAME 'LineItems';
		DECLARE src REFERENCE TO reqBody.lineItems.*[1];
		DECLARE lineNo INTEGER 1;
		WHILE LASTMOVE(src) DO
			DECLARE li REFERENCE TO items;
			CREATE LASTCHILD OF items AS li TYPE XMLNSC.Element NAMESPACE wmsNS NAME 'LineItem';
			DECLARE vQty   DECIMAL CAST(FIELDVALUE(src.quantity) AS DECIMAL);
			DECLARE vPrice DECIMAL CAST(FIELDVALUE(src.unitPrice) AS DECIMAL);
			CALL addEl(li, 'LineNumber', CAST(lineNo AS CHARACTER));
			CALL addEl(li, 'Sku', FIELDVALUE(src.sku));
			CALL addEl(li, 'Description', FIELDVALUE(src.description));
			CALL addEl(li, 'Quantity', CAST(vQty AS CHARACTER));
			CALL addEl(li, 'UnitPrice', CAST(vPrice AS CHARACTER));
			CALL addEl(li, 'Currency', FIELDVALUE(src.currency));
			CALL addEl(li, 'LineTotal', CAST(vQty * vPrice AS CHARACTER));
			SET lineNo = lineNo + 1;
			MOVE src NEXTSIBLING;
		END WHILE;

		RETURN TRUE;
	END;
END MODULE;
```

### `Kite2OrderFulfilmentApp/HttpReply_Compute.esql` — exact content
```esql
CREATE COMPUTE MODULE HttpReply_Compute
	CREATE FUNCTION Main() RETURNS BOOLEAN
	BEGIN
		-- Preserve HTTP reply identifier so HTTP Reply node knows which client connection to reply to
		SET OutputLocalEnvironment.Destination.HTTP.RequestIdentifier =
			InputLocalEnvironment.Destination.HTTP.RequestIdentifier;

		-- HTTPReplyHeader must be created before the body domain (avoids BIP6062W)
		CREATE LASTCHILD OF OutputRoot DOMAIN 'HTTPReplyHeader' NAME 'HTTPReplyHeader';
		SET OutputRoot.HTTPReplyHeader."X-Original-HTTP-Status-Code" = 202;
		SET OutputRoot.HTTPReplyHeader."Content-Type" = 'application/json';

		CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON';
		CREATE LASTCHILD OF OutputRoot.JSON TYPE JSON.Object NAME 'Data';

		RETURN TRUE;
	END;
END MODULE;
```

---

## Critical file rules

### `.project` — application natures (exact, no others)
```xml
<natures>
  <nature>com.ibm.etools.msgbroker.tooling.applicationNature</nature>
  <nature>com.ibm.etools.msgbroker.tooling.messageBrokerProjectNature</nature>
</natures>
```
Build commands: `applibbuilder`, `applibresourcevalidator`, `connector.policy.ui.PolicyBuilder`, `mbprojectbuilder`.

### `application.descriptor` — exact namespace
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<ns2:appDescriptor xmlns="http://com.ibm.etools.mft.descriptor.base" xmlns:ns2="http://com.ibm.etools.mft.descriptor.app">
  <references/>
</ns2:appDescriptor>
```
`ns2` = `http://com.ibm.etools.mft.descriptor.app` (not `.application`). No `<reference>` to policy project — causes `1357E`.

### HTTP Input node — JSON domain is mandatory
```xml
<nodes xmi:type="ComIbmWSInput.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_1"
  location="50,100"
  URLSpecifier="/fulfilment"
  messageDomainProperty="JSON"
  httpMethodRestriction="POST">
```
Without `messageDomainProperty="JSON"`, the body is parsed as BLOB. `InputRoot.JSON.Data` is empty, Db2 lookup runs with NULL, COALESCE hides the failure, and the flow returns 202 with wrong data. **A 202 is NOT proof of success.**

### MQ Output node — all three attributes required
```xml
<nodes xmi:type="ComIbmMQOutput.msgnode:FCMComposite_1" xmi:id="FCMComposite_1_3"
  location="500,100"
  queueName="KITE.WMS.IN"
  destinationQueueManagerName="QM1"
  policyUrl="{Kite2MQPolicyProject}:Kite2MQEndpoint"
  messageContextMode="none">
  <translation xmi:type="utility:ConstantString" string="MQ Output"/>
</nodes>
```
- `policyUrl` (lowercase `l`) — `policyURL` causes `1361E`
- `destinationQueueManagerName="QM1"` — mandatory even with `policyUrl`; omit → `BIP2214E`
- `messageContextMode="none"` — mandatory for MQ on Cloud; omit → `MQRC_NOT_AUTHORIZED (2035)`

### `.msgflow` structure
- Nodes and connections inside `<composition>` inside `<eClassifiers xmi:type="eflow:FCMComposite">`
- Connections use `xmi:type="eflow:FCMConnection"`
- Declare `xmlns:eflow` and `xmlns:utility` in `<ecore:EPackage>` header
- Only declare namespace prefixes for node types that actually appear

### Compute node naming — three values must match
| Artifact | Value |
|---|---|
| `computeExpression` | `esql://routine/#BuildWMSOrder_Compute.Main` |
| Node label (translation string) | `BuildWMSOrder` |
| ESQL module name | `BuildWMSOrder_Compute` |

---

## ESQL — `BuildWMSOrder_Compute.esql` key rules

- `DECLARE wmsNS NAMESPACE 'http://kite-sports.com/wms/order/v1'` and `addEl` procedure at **module scope** (outside `Main`) — not inside `Main`
- `addEl` is 3-arg: `(IN parentRef REFERENCE, IN elName CHARACTER, IN elValue CHARACTER)` — uses module-level `wmsNS` directly
- Read JSON input: `DECLARE reqBody REFERENCE TO InputRoot.JSON.Data;` + `FIELDVALUE(reqBody.<name>)` for every scalar
- Db2 lookup: `SET customerRow = THE(SELECT * FROM Database.KITEDS.KITE.CUSTOMERS AS T WHERE T.CUSTOMER_ID = vCustomerId);` — no `PASSTHRU`
- Columns: `FIRST_NAME`, `LAST_NAME`, `ENTITLEMENT_TIER`, `ACCOUNT_NUMBER`, `ACCOUNT_STATUS`, `PREFERRED_CURRENCY` — read with `COALESCE(customerRow.<COL>, '')`
- Priority: `IF vOrderType = 'express' OR vTier = 'kitepro' OR vTier = 'elite'` → `'true'` / `'KITE-EXPRESS'`; default initialise as `'false'` / `'KITE-STANDARD'`
- `CURRENT_GMTTIMESTAMP` is an ESQL keyword — no parentheses
- JSON array traversal — do NOT use `.item[1]`:
  ```esql
  DECLARE src REFERENCE TO reqBody.lineItems.*[1];
  WHILE LASTMOVE(src) DO
      -- read src.sku, src.description, src.quantity, src.unitPrice, src.currency
      MOVE src NEXTSIBLING;
  END WHILE;
  ```
- Line totals: use `DECIMAL` intermediates — `DECLARE vQty DECIMAL CAST(FIELDVALUE(src.quantity) AS DECIMAL);`
- `DeliveryAddress/Line2`: omit if `reqBody.deliveryAddress.line2 IS NOT NULL` guard fails

## ESQL — `HttpReply_Compute.esql` key rules

- Copy `RequestIdentifier` first:
  ```esql
  SET OutputLocalEnvironment.Destination.HTTP.RequestIdentifier =
      InputLocalEnvironment.Destination.HTTP.RequestIdentifier;
  ```
- Create `HTTPReplyHeader` **before** body domain (avoids `BIP6062W`):
  ```esql
  CREATE LASTCHILD OF OutputRoot DOMAIN 'HTTPReplyHeader' NAME 'HTTPReplyHeader';
  SET OutputRoot.HTTPReplyHeader."X-Original-HTTP-Status-Code" = 202;
  SET OutputRoot.HTTPReplyHeader."Content-Type" = 'application/json';
  CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON';
  CREATE LASTCHILD OF OutputRoot.JSON TYPE JSON.Object NAME 'Data';
  ```

---

## BAR build (no toolkit)

**Outer BAR** — zip with files at root:
- `META-INF/manifest.mf` = `srcSelected=false;showSrc=true;esql21VersionSelected=false;` (no trailing newline — use `printf`)
- `META-INF/bar-refresh.links` — `deploy:Deploy` XML with one deployable for `/Kite2OrderFulfilmentApp/.project` and `<barEntry name="Kite2OrderFulfilmentApp.appzip"/>`
- `META-INF/service.log` — any short text
- `Kite2OrderFulfilmentApp.appzip`

**Inner appzip** — zip with files at root: `Kite2OrderFulfilment.msgflow`, `BuildWMSOrder_Compute.esql`, `HttpReply_Compute.esql`, `application.descriptor`, `META-INF/manifest.mf`, `META-INF/service.log`, `META-INF/broker.xml`. **Exclude `.project` and `.settings`.**

**`broker.xml`** — standard configurable properties block for `Kite2OrderFulfilment` flow (see `ace-bar-package` skill).

Build with `zip -qr` from inside each staging directory. Verify with `unzip -l` before uploading.

**BAR name:** use a fresh version number (e.g. `kite2-order-fulfilment-v12`) each session to avoid stale content-store collisions.

---

## Deploy procedure

0. Build the four missing configurations first (see "Server-side configurations"). Do not create the runtime until all seven names (including `kite2-db2-certs`) exist when listed.
1. `App_Connect__bar_files` (action: upload, name: `kite2-order-fulfilment-vN`) → get `callbackUrl` + `token`
2. `curl -X POST --data-binary @<file>.bar -H "x-callback-token: <token>" -H "Content-Type: application/octet-stream" <callbackUrl>` → HTTP 200 + `url` field
3. `context-retrieval` skill + `context-search` before creating runtime (Polaris requirement)
4. Ask user to confirm runtime creation (ongoing cost while running)
5. `App_Connect__integration_runtimes` (action: create) with **only**: `barURL`, `configurations` (all seven), `forceFlowBasicAuth: {enabled: true}`, `version: "13.0"`, `replicas: 1`
6. Never use `runtime.bar_file.deploy` — it appends BARs instead of replacing. Never touch `default-integration-runtime`.
7. To update BAR on existing runtime: `integration_runtimes` (action: replace) only.
8. Poll for `status.phase = "Ready"` (up to 6 × 10 s attempts).
9. Retrieve credentials and endpoint — exact procedure:

   ```bash
   # get_flow_auth returns a credentialUrl + token; POST the token directly with curl to extract creds
   # (the page is a single-use HTML page — no browser needed when you POST the token as a form field)
   curl -s -X POST "<credentialUrl>" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -d "token=<token>"
   ```

   Parse `username` and `password` out of the returned HTML — look for the two `<span class="row-value">` elements, e.g.:
   ```
   <span class="row-value">aceuser</span>
   <span class="row-value">sQXUquV9l5wJM3cWtS46L2FdNbKie7E8</span>
   ```
   Username is always `aceuser`. The password is the value in the second `row-value` span.

   Endpoint: `status.endpoints[]` where `name = "https endpoint"` → `uri`.

   The credential link is **single-use and expires in 5 minutes**. Call `get_flow_auth` and curl the result in the same step — do not store the token between turns.

### Forcing a pod restart (after any configuration change)

Verified behaviour on IWHI:
- A `replace` that only changes the `configuration-last-updated` annotation **does not restart the pod** (same process id, same thread ids afterwards).
- A `replace` that adds `template.spec.containers[].env` returns a **500** — do not try it.
- **What works:** `integration_runtimes` (action: `replace`) with `replicas: 0` → wait ~40 s → `replace` again with `replicas: 1` → poll for `status.phase = "Ready"`. Keep `barURL` and the full `configurations` list identical in both calls.

Do this whenever a configuration was created or replaced after the runtime already existed (this includes the very first run if the runtime was created before the Db2 config was final). Skip it only when the runtime is created fresh *after* all seven configurations are in their final state. Do not send test traffic until the restart has completed.

---

## MQ queue check

This checks that the queue exists. It does not read messages — no tool does that.

- Check `KITE.WMS.IN` exists on `QM1` with `IBM_MQ_Dev1__runCommandMQSC`: `DISPLAY QLOCAL('KITE.WMS.IN')`
- If absent: `DEFINE QLOCAL('KITE.WMS.IN')` then grant access to the group (not principal): `SET AUTHREC PROFILE('KITE.WMS.IN') OBJTYPE(QUEUE) GROUP('polarismquse') AUTHADD(PUT,GET,INQ,BROWSE,PASSALL,PASSID)`
- Do not clear the queue — user may have a consumer.

---

## MQ policy — `kite2-mq-policy`

- `SSLCipherSpec` is `ANY_TLS13`. **Do not revert to `ANY_TLS12_OR_HIGHER`** — it causes `MQRC=2393` (GSKit failure) on every pod restart on IWHI.
- If `MQRC=2393` appears: the cipher is NOT the cause (it is already `ANY_TLS13`). Check `kite2-mq-credentials` and `kite2-mqclientini`.
- You build this policy yourself (see "Server-side configurations"). If it needs changing, regenerate the zip from the contents given there and `replace` it via the callback — do not fetch and patch a copy. Round-trip `unzip -t` validate before POSTing, then restart the pod with the 0 → 1 replica procedure in "Forcing a pod restart". The annotation alone does not restart it.

---

## Verification — a 202 is NOT proof of success

There is no tool to read messages off `KITE.WMS.IN`. **The demo ends when End-to-End Monitoring shows the transaction succeeded and the MQ put to `KITE.WMS.IN` is in the trace.** Do not browse, get or inspect the queue contents.

Load `mcp__polaris__get-skill-by-name("e2em-transaction-summary")` and `("e2em-transaction-details")` silently, then use `mcp__polaris__invoke-tool` (single-tenant: never pass `tenantId` or `environments`):

1. Compute the time window with `date +%s` (epoch seconds × 1000 → epoch-ms strings; last 1 h). The sandbox PowerShell instruction in the e2em skills does not apply on macOS.
2. `End-to-End_Monitoring__monitoring.get_transaction_summary` — `filterCriteria.multiFieldConditions: [{fields:["name"], value:"Kite2"}]`, `timeRange: {startTime, endTime}`, `transactionOrder: "BY_STARTTIME"`, `sortOrder: "DESC"`, `paginationCriteria: {pageSize: 5, pageNum: 1}`. Pick the newest row that matches the request just sent (`name` = `Kite2OrderFulfilmentApp.Kite2OrderFulfilment`). Traces can take a few seconds to appear — retry up to 3 × 10 s.
3. `End-to-End_Monitoring__monitoring.get_transaction_details` with that row's `traceId` (use a `timeRange` that brackets the transaction `startTime`).
4. Present the trace using the `e2em-transaction-details` output format (gantt, span table, summary).

**Pass** when all of these are true:
- Summary row `result` = `"1"` (`"3"` = failed)
- `Entry` span (HTTP Input) has `http.status_code` = `202`
- A child `Local` span named `…Kite2OrderFulfilment.MQ Output` exists with `stat` = `1`, `messaging.destination` = `KITE.WMS.IN`, `messaging.operation` = `send`, `peer.service` = `QM1`, and `messaging.message_payload_size_bytes` > 0 (about 1800 bytes for the standard order)

**Fail** when `result` = `"3"` or the `MQ Output` span is missing. The summary row `message` gives the root cause — for example `BIP2677E … MQRC=2393` → see "MQ policy" above.

`BIP2322E … SQL30081N … sqlccSSLSocketSetup … 414` → check in this order, **before** changing anything else: (1) `kite2-db2-certs` is listed in the runtime's `spec.configurations`; (2) the live `kite2-db2cli` contains the two `SSLClientKeystore…` lines; (3) the pod was actually restarted since the last configuration change (compare the process id / `service.instance.id` suffix of traces before and after — an unchanged id means no restart). Fix, then do the 0 → 1 restart. Do not rewrite the configurations repeatedly while the old pod is still running; it will keep failing regardless.

**Limitation:** the trace proves the put happened and how large the payload was, not the field values. It cannot confirm `CustomerId`, `EntitlementTier`, `CarrierCode`, `LineTotal` and so on. If the user needs field-level proof, or a field looks wrong, start a service trace (`integration_runtimes` action: `start_trace`, type: `service`), send the request, collect with `collect_trace` body `{"type":"service"}`, curl the callback immediately, unzip, look for `BIP6061I` (parser), `BIP2544I` (SQL), `BIP2543W` (path nav failures), `BIP2539I` (expression results), then `stop_trace`.

Expected field values for that diagnostic: `Customer/CustomerId` = `CUST-00882`, `Customer/FullName` = `Sarah` / `Bellingham`, `Customer/EntitlementTier` = `kitepro`, `Customer/AccountNumber` = `ACC-2024-00882`, `OrderDetails/PriorityFlag` = `true`, `OrderDetails/CarrierCode` = `KITE-EXPRESS`, populated `LineItems` with correct `LineTotal` values.

Ignore: `2111E nullptr ImbConfigObject`, `2683E no queue manager on integration node`, `9570E OpenTelemetry DEADLINE_EXCEEDED`, `Tracing: Unable to run command ps` — all harmless startup noise.

---

## Tooling gotchas

- `bar_files` delete and `integration_runtimes` delete return **500** in this environment — tell the user to remove via the dashboard instead.
- Polaris MCP calls occasionally fail with `Session not found` — retry once.
- `runtime.bar_file.deploy` appends BARs — never use it.
- `runtime.get_logs` time params (`since`, `timestampFrom`) must be **strings** (epoch ms) — integers cause 500.
- Do not run `find /` or broad `mdfind` — it hangs.
- Call `reset_flow_auth` at the end of the session if the flow password appeared in any command output.

---

## Teardown (end of demo)

1. `integration_runtimes` (action: delete) for `kite2-order-fulfilment` — expect 500; tell user to delete via dashboard if it fails.
2. `bar_files` (action: delete) for all `kite2-order-fulfilment-vN` BARs — same caveat.
3. Remove the four configurations you built (`kite2-db2cli`, `kite2-odbc`, `kite2-mq-policy`, `kite2-mqclientini`). If `configurations` delete is not available or fails, tell the user to remove them via the dashboard. **Never delete `kite2-db2-credentials`, `kite2-mq-credentials` or `kite2-db2-certs`** — they are pre-provisioned and cannot be recreated by you.
4. Delete the local working directory: `rm -rf tmp/` in the repo root.
5. Update `extra-instructions.md` with any new facts discovered, commit.
