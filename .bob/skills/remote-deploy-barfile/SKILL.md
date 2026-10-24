---
name: remote-deploy-barfile
description: Deploy a BAR file to a deployment location runtime. Use when the user wants to deploy a BAR file to a runtime, phrases like "deploy barfile", "deploy BAR to runtime", "deploy to deployment location", "push BAR file", or "deploy integration". Covers checking if the deployment location and runtime exist, creating a runtime if needed, deploying the BAR file, waiting for the runtime to be ready, and returning the endpoints.
metadata:
  version: 1.6.0
---

# Deploy BAR File to a Deployment Location

Follow these steps in order when the user wants to deploy a BAR file to a runtime at a deployment location.

## Step 1 — Gather Required Inputs

If the user has not already provided them, ask (using `ask_followup_question`) for:

- The **deployment location name** to deploy to
- The **BAR file name** to deploy (without `.bar` extension)

Do **not** ask for the runtime name here — it is always chosen interactively in Step 3.

## Step 2 — Check Deployment Location Exists

Call `deployment_location` (action: list) to retrieve all deployment locations.

- If the named deployment location is **not found**, tell the user it does not exist and stop. Offer to list available deployment locations.
- If it **exists**, note its **type** (e.g. `managedServiceConnection`, `switchConnection`, `nodeConnection`, `containerConnection`) — you will need it in later steps. Proceed to Step 3.

## Step 3 — Ask the User Which Runtime to Deploy To

Call `deployment_location_runtimes` (action: list) with the deployment location name to list all available runtimes.

**Always** present the list to the user and ask which runtime to deploy to — never assume or auto-select one, even if only one runtime is available or one appears to be in a ready state. Use `ask_followup_question` with the existing runtime names as suggestions, and always include **"Create a new runtime"** as an additional option.

- If the user selects an **existing runtime**, proceed to Step 5.
- If the user chooses to **create a new runtime**, proceed to Step 4.

## Step 4 — Create the Runtime (if it doesn't exist)

Before creating, confirm with the user:

- State the runtime name and deployment location
- Ask for explicit approval
- Optionally ask if they want to change the defaults or attach configurations. Defaults depend on the deployment location type:
  - **managedServiceConnection / containerConnection**: version `13.0`, 1 replica, 500m CPU, 1024Mi memory
  - **switchConnection**: version `12.0`, 1 replica, 300m CPU, 368Mi memory
  - **nodeConnection**: no resource settings apply

For **switchConnection** deployment locations, also pass the `agentId` (obtained from `deployment_location` list) when calling create.

Once approved, call `deployment_location_runtime` (action: create) with:

- `deploymentLocation`: the deployment location name
- `name`: the runtime name
- `agentId`: the agent ID (switchConnection only)
- Any user-specified overrides (version, replicas, CPU, memory, configurations)

After creation, proceed to Step 5.

## Step 5 — Ensure the BAR File is Available at the Deployment Location

**If the deployment location type is `nodeConnection`**, skip this step entirely — node connections have no BAR file store of their own. The BAR file bytes are sent directly in Step 6 via the callback mechanism. Proceed to Step 6.

For all other location types, call `deployment_location_bar_files` (action: list) with the deployment location name (and `agentId` for switchConnection locations).

- If the BAR file is **already present** at the deployment location, proceed to Step 6.
- If the BAR file is **not present**, check the instance-wide BAR file store by calling `bar_files` (action: list).
  - If found on the instance, copy it to the deployment location using `deployment_location_bar_files` (action: copy), then proceed to Step 6.
  - If not found anywhere, automatically initiate an upload by calling `deployment_location_bar_files` (action: upload) with the deployment location name and BAR file name (without `.bar` extension). Then use `execute_command` to run the upload yourself — do **not** ask the user to run it:
    ```
    curl -X POST "<callbackUrl>" \
      -H "x-callback-token: <token>" \
      -H "Content-Type: application/octet-stream" \
      --data-binary @<BarFileName>.bar
    ```
    Verify the HTTP response is 200, then proceed to Step 6.

## Step 6 — Deploy the BAR File

Call `runtime.bar_file.deploy` with:

- `deploymentLocation`: the deployment location name
- `runtimeName`: the runtime name
- `barFileName`: the BAR file name **including** the `.bar` extension (e.g. `MyAPI.bar`)
- `agentId`: the agent ID (switchConnection locations only)

For **nodeConnection** locations, this call returns a callback URL and token. Use `execute_command` to POST the raw BAR file bytes yourself — do **not** ask the user to run it:

```
curl -X POST "<callbackUrl>" \
  -H "x-callback-token: <token>" \
  -H "Content-Type: application/octet-stream" \
  --data-binary @<BarFileName>.bar
```

Verify the HTTP response is 200 before proceeding.

## Step 7 — Wait for the Runtime to Be Ready

After deployment, poll the runtime status by calling `deployment_location_runtimes` (action: describe) with the deployment location name and runtime name every few seconds.

- Check the `status` field of the runtime.
- Keep polling (up to ~2 minutes) until the status indicates the runtime is **ready/running**.
- If the runtime does not become ready within the timeout, report the last known status and suggest the user check the logs using `runtime.get_logs`.

Inform the user when the runtime is up and running.

## Step 8 — Return Endpoints

Call `deployment_location_runtimes` (action: describe) once more to retrieve the final runtime details.

Return to the user:

- The runtime **name** and **status**
- Any **endpoints** or **URLs** exposed by the runtime
- The **deployment location** it is running in

Present this information clearly so the user can immediately use the deployed integration.
