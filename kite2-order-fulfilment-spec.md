# Kite2 — Order Fulfilment Integration Specification
**Persona:** Isaac — integration developer at Kite2**Integration runtime name:** `kite2-order-fulfilment`**Application project name:** `Kite2OrderFulfilmentApp`**BAR file name:** `kite2-order-fulfilment-v3.bar`
---
## Business Context
When a customer places an order on kitesports.com, a fulfilment instruction must reach the on-premises Warehouse Management System (WMS) reliably and in the correct format. Currently orders are forwarded to the warehouse by email, causing delays and errors. This integration replaces that manual process.

---
## What the Integration Must Do
1. **Accept a REST request** at `POST /fulfilment`. The request body contains: - `orderId` (string) - `orderDate` (ISO 8601 datetime) - `customerId` (string, pattern `CUST-[0-9]{5}`) - `orderType` (`standard` or `express`) - `requestedDeliveryDate` (ISO 8601 date) - `deliveryAddress` object: `line1`, `line2` (optional), `city`, `postcode`, `countryCode` - `lineItems` array, each with: `sku`, `description`, `quantity`, `unitPrice`, `currency`
2. **Look up the customer** in the Kite2 Db2 database using `customerId` as the key. Retrieve from `KITE.CUSTOMERS`: - `FIRST_NAME`, `LAST_NAME` - `ENTITLEMENT_TIER` (`standard`, `kitepro`, or `elite`) - `ACCOUNT_NUMBER`, `ACCOUNT_STATUS` - `PREFERRED_CURRENCY`
3. **Apply fulfilment priority logic:** - Set `PriorityFlag = true` and `CarrierCode = KITE-EXPRESS` if either: - `orderType` is `express`, **or** - the customer's `ENTITLEMENT_TIER` is `kitepro` or `elite` - Otherwise set `PriorityFlag = false` and `CarrierCode = KITE-STANDARD`
4. **Construct an XML message** conforming to the WMS order schema (namespace `http://kite-sports.com/wms/order/v1`, root element `WMSOrder`) with the following structure:
| XML element | Source | |---|---| | `MessageHeader/CorrelationId` | Generated UUID | | `MessageHeader/MessageDateTime` | Current timestamp (ISO 8601) | | `MessageHeader/SchemaVersion` | Fixed value `1.0` | | `MessageHeader/SourceSystem` | Fixed value `kitesports.com` | | `Customer/CustomerId` | Request `customerId` | | `Customer/EntitlementTier` | Db2 `ENTITLEMENT_TIER` | | `Customer/FullName/FirstName` | Db2 `FIRST_NAME` | | `Customer/FullName/LastName` | Db2 `LAST_NAME` | | `Customer/AccountNumber` | Db2 `ACCOUNT_NUMBER` | | `Customer/AccountStatus` | Db2 `ACCOUNT_STATUS` | | `Customer/PreferredCurrency` | Db2 `PREFERRED_CURRENCY` | | `OrderDetails/OrderId` | Request `orderId` | | `OrderDetails/OrderDate` | Request `orderDate` | | `OrderDetails/OrderType` | Request `orderType` | | `OrderDetails/RequestedDeliveryDate` | Request `requestedDeliveryDate` | | `OrderDetails/PriorityFlag` | Derived (see priority logic above) | | `OrderDetails/CarrierCode` | Derived (see priority logic above) | | `DeliveryAddress/Line1` | Request `deliveryAddress.line1` | | `DeliveryAddress/Line2` | Request `deliveryAddress.line2` (omit if absent) | | `DeliveryAddress/City` | Request `deliveryAddress.city` | | `DeliveryAddress/Postcode` | Request `deliveryAddress.postcode` | | `DeliveryAddress/CountryCode` | Request `deliveryAddress.countryCode` | | `LineItems/LineItem[n]/LineNumber` | Sequence number (1-based) | | `LineItems/LineItem[n]/Sku` | Request `lineItems[n].sku` | | `LineItems/LineItem[n]/Description` | Request `lineItems[n].description` | | `LineItems/LineItem[n]/Quantity` | Request `lineItems[n].quantity` | | `LineItems/LineItem[n]/UnitPrice` | Request `lineItems[n].unitPrice` | | `LineItems/LineItem[n]/Currency` | Request `lineItems[n].currency` | | `LineItems/LineItem[n]/LineTotal` | Derived: `quantity × unitPrice` |
5. **Put the XML message** onto the IBM MQ queue `KITE.WMS.IN`.
6. **Return `202 Accepted`** to the caller once the message has been placed on the queue.
---
## Db2 Connection
| Property | Value ||---|---|| Datasource alias | `KITEDS` || Database | `BLUDB` || Hostname | `f115e671-7603-467f-bae6-b4f7c1606478-eugb.bt1ibm.db2.ibmappdomain.cloud` || Port | `32506` || SSL | Enabled || SSLClientKeystoredb | `/home/aceuser/generic/client.p12` || SSLClientKeyStoreDBPassword | `changeit` |
The Db2 connection must be registered as **three separate configurations** so credentials can be changed without rebuilding the integration:
| Configuration | Type | Security context (key) ||---|---|---|| `kite2-db2cli` | db2cli | `db2ini.cfg` — tells the Db2 CLI driver the hostname, port, and SSL settings (`SSLClientKeystoredb=/home/aceuser/generic/client.p12`, `SSLClientKeyStoreDBPassword=changeit`) for alias `KITEDS` || `kite2-odbc` | ODBC datasource | Maps alias `KITEDS` to the Db2 CLI driver at the standard path for the ACE 13 runtime || `kite2-db2-credentials` | setdbparms | `odbc::KITEDS` |
---
## MQ Connection
| Property | Value ||---|---|| Queue manager | `QM1` || Hostname | `qm1-27db.qmb33.us-east-1.mq.ibmappdomain.cloud` || Port | `443` || Channel | `CLOUD.APP.SVRCONN` || Queue | `KITE.WMS.IN` || Connection type | Client with TLS || SSL cipher spec | `ANY_TLS12_OR_HIGHER` || SSL peer name | Leave empty (IBM MQ on Cloud) |
The MQ connection must be registered as the following configurations so the endpoint and credentials can be changed without rebuilding the integration:
| Configuration | Type | Security context (key) ||---|---|---|| `kite2-mq-policy` | Policy project | MQ endpoint and TLS settings for queue manager `QM1` || `kite2-mq-credentials` | setdbparms | `mq::QM1` |
---
## Integration Runtime
- **Name:** `kite2-order-fulfilment`- **Configurations to attach:** `kite2-db2cli`, `kite2-odbc`, `kite2-db2-credentials`, `kite2-db2-certs`, `kite2-mq-policy`, `kite2-mq-credentials`, `kite2-mqclientini`- **Flow basic auth:** enabled (`forceFlowBasicAuth`)- The runtime should be created before the application is deployed so that connection health can be verified independently of the flow.
---
## Test Customer
Use `CUST-00882` (Sarah Bellingham, tier `kitepro`, account `ACC-2024-00882`) to validate the integration end-to-end. A successful test call to `POST /fulfilment` should return `202 Accepted` and result in a `WMSOrder` XML message on `KITE.WMS.IN` with `PriorityFlag` set to `true` and `CarrierCode` set to `KITE-EXPRESS` (because her tier is `kitepro`).
---
## Supporting Files
| File | Purpose ||---|---|| `demo-customers.ddl` | Full schema and sample data for `KITE.CUSTOMERS` || `demo-wms-order.xsd` | XML schema for the `WMSOrder` message placed on `KITE.WMS.IN` |