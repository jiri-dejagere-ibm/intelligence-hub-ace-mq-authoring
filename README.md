# intelligence-hub-ace-mq-authoring

Demo: real-time order fulfilment for Kite Sports, built with IBM Bob and IBM Intelligence Hub. A REST call is enriched from Db2, turned into a warehouse order, and put on an IBM MQ queue by an App Connect Enterprise flow.

- **Hill:** Hill 37, Productive Use Hill
- **Persona:** Isaac, Integration Engineer
- **Platform:** IBM Intelligence Hub + IBM Bob

## Before you start

- The App Connect instance has just **one runtime**.
- The Monday board is open, with the **"Order Fulfilment Integration"** item listed. The integration spec is embedded in that item.
- These configurations exist on the instance and must be kept: `kite2-db2-certs`, `kite2-db2-credentials`, `kite2-mq-credentials`.
- The spec is also available locally as [`kite2-order-fulfilment-spec.md`](kite2-order-fulfilment-spec.md).

## Run the demo

### 1. Prompt Bob

Copy the Monday board item URL and paste this prompt into Bob:

```
Build the integration described in @kite2-order-fulfilment-spec.md
```

### 2. What Bob does

1. Reads the Monday ticket through the Monday MCP server.
2. Reads the spec and identifies it as an integration use case.
3. Calls Polaris to retrieve the available MCP servers and skills.
4. Builds the integration: creates a project folder, generates the integration files and builds the BAR file.
5. Checks the MQ infrastructure. The queue manager already exists, so no action is needed.
6. Uploads the BAR file and asks for permission to create a runtime, because that has a resource unit cost. Grant it and the runtime is provisioned.
7. Waits for the runtime to become ready, defines the REST message shape and asks for approval, then runs functional tests.

### 3. Functional checks

| Check | Result |
|---|---|
| Messages arriving on the MQ queue from the REST API | Messages on the queue |
| Db2 lookup working | Verified through the ACE logs and End-to-End Monitoring |

### 4. Verify in the dashboard

Return to the App Connect dashboard. The new runtime is in place and the flow is running.

## Clean up

Once the demo is over, give Bob this prompt:

```
Okay, demo over, clean up time. Remove what you've created but make sure you keep kite2-db2-certs, kite2-db2-credentials and kite2-mq-credentials. Also clean out what you made in the local repo
```

Bob removes the local project folder and BAR file. The App Connect runtime, BAR file and the four configurations Bob built (`kite2-db2cli`, `kite2-odbc`, `kite2-mq-policy`, `kite2-mqclientini`) may not delete through the tools in this environment. If any return an error, remove them from the App Connect dashboard, deleting the runtime first.

## Summary

| Step | Tool used | Outcome |
|---|---|---|
| Read spec from Monday | Monday MCP server | Spec retrieved |
| Identify integration skills | Polaris | Skills loaded |
| Build integration files and BAR | App Connect / Bob | BAR built cleanly |
| Check MQ infrastructure | IBM MQ via Polaris | Queue manager existed |
| Deploy to runtime | App Connect | Runtime provisioned |
| Functional testing | Bob + ACE logs | All checks passed |

See [`Script.md`](Script.md) for the full presenter transcript.
