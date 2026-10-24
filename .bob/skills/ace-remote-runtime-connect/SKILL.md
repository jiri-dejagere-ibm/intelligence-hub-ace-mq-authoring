---
name: ace-remote-runtime-connect
description: Use when the user wants to connect something they run themselves to App Connect — phrased any way at all, e.g. "connect my on-prem instance", "I have App Connect running on a server here", "connect my local runtime", "my runtime is not showing up", "hook up my ACEcc environment", or any mention of Secure Agents, private network agents or private network connections. Covers finding or creating the private network agent configuration, the Secure Agent installer and its licence, the switchclient configuration, where it goes on the remote side, and verifying the connection.
metadata:
  version: 1.0.0
---

# Connect a Remote ACE Runtime Skill

## Purpose

Guides a user through connecting a remote or on-premises App Connect Enterprise environment to their App Connect instance, so that its runtimes, BAR files and configurations become visible and manageable from the instance.

Two things are involved, and they are not the same:

- A **Secure Agent** is the executable the user runs in their own system. It dials out to the switch server in App Connect, so no inbound firewall rule is needed.
- A **private network agent** is a _configuration_ — endpoints, listeners and switch server details — that is handed to a Secure Agent, or to an App Connect Enterprise server, to configure it. It arrives as `switchclient.json`.

**Private network connection** is the umbrella term for connecting this way. The two are not 1:1, so keep them apart when talking to the user: you create and fetch _configurations_; the user installs and runs the _Secure Agent_.

## When to use

- The user asks to connect a remote ACEcc instance, an on-prem runtime, or an integration node.
- The user asks about Secure Agents, private network agents or private network connections.
- The user asks why their remote runtimes are not showing up in App Connect.
- The user asks to download the Secure Agent installer or a private network agent configuration.

## When not to use

- For deploying a BAR to a runtime that is already connected, use [`skills/ace-iwhi/SKILL.md`](../ace-iwhi/SKILL.md).
- For building the BAR itself, use [`skills/ace-bar-package/SKILL.md`](../ace-bar-package/SKILL.md).

## Tool priority rule

`deployment_location.*` tools are **only used in this skill** (Steps 6–7) to verify a switch/private-network connection and show what it exposes. They are **never** the right tool for managing IWHI runtimes, BAR files, or configurations — use `integration_runtimes.*`, `bar_files.*`, and `configurations.*` directly for that. Do not reach for `deployment_location.*` unless the user is explicitly asking about a remote or on-premises environment connected through a private network agent.

---

## Before you start

Assume the user does not know the product vocabulary. Someone who already knows what an integration node is, and how it differs from a dashboard, can do this faster in the UI than by talking to you. The people this helps are the ones who just want their own environment connected and should not have to learn the jargon first.

So **ask about things the user can see, never about what something is called.**

**Ask exactly one question to begin**, with selectable answers so the user can click rather than type:

Question: "Is App Connect Enterprise already installed and running on that machine, or is it a machine with nothing set up on it yet?" Suggested answers: `It is already installed and running` / `Nothing is set up there yet`

Ask this **before** Step 1, and do not call any tool until it is answered.

**Do not ask which operating system it is, and do not ask anything else in this opening message.** The operating system only decides which installer to download, so it belongs in Step 3 and only on the path that gets there. Asking someone who already runs App Connect Enterprise for their OS is a question whose answer changes nothing.

Their answer splits the flow:

| Their answer | What they need |
| --- | --- |
| **Already installed and running** | The configuration only, so their existing server can be managed remotely. **Skip Step 3 entirely.** |
| **Nothing set up yet** | The Secure Agent installer _and_ a configuration for it. |

Never ask "is this an ACEcc dashboard, an integration node, or a private network agent?" — that is the question they came to you to avoid. If you later need to know how they manage that environment, ask what they actually do:

> "When you work with that environment today, do you use a web page in a browser, or do you run commands on the machine itself?"

---

## Explain before you act

The user is being walked through something most people do rarely, and every step has a consequence on a machine they own. **A tool approval prompt is not an explanation.** If the first thing the user sees about a step is a request to run a tool, the explanation has been skipped.

**Open with the plan.** Before the first tool call, set out the whole thing in a few plain sentences, in their language rather than tool names:

> "Here's how this will go. First I'll look at the private network agent configurations you already have and create one for this environment. Then I'll get you the installer — you'll need to accept the IBM licence before I can download it. After that I'll give you the configuration files and tell you exactly where they go on your remote machine. Once you've done that and let me know, I'll check the connection and show you what we can now see from it."

**Say what each step is for, before you take it.** One sentence is enough, and it should describe the outcome, not the mechanism:

- Good — "Let me check which private network agents you already have, so we reuse one rather than creating a duplicate."
- Good — "Now I'll fetch the configuration files that let your agent prove who it is."
- Bad — "I'll call `private_network_agent` (action: list)."
- Bad — silence, followed by an approval prompt.

**Say what came back, and what it means.** One line after each step. "You have one agent already, but it belongs to the switch server, so we'll create your own" is useful. Pasting the raw tool output is not.

**Keep narrating even when tool calls are approved automatically.** The explanation is for the user's understanding, not for permission, so it must not depend on whether a prompt is shown.

**Name files and locations in full** whenever the user has to act on them.

**Explain each piece of vocabulary once, in a clause, the first time you use it** — "a Secure Agent, which is a small program you run in your own environment that dials out to App Connect". Never make the user look a term up, and never ask a question that only someone who already knows the product could answer.

---

## How to ask the user anything

Every question in this skill is a click, not a sentence. **Ask through whatever mechanism your agent harness provides for a question with selectable answers, and pass the choices to it** — MCP elicitation, `ask_followup_question`, or the equivalent. Where this skill says "ask with selectable answers", that is what it means.

- **Never write the choices into the message** — not as bullets, not as `A.` / `B.`, not with emoji. If the user has to type an answer back, the choices went in the wrong place.
- **One question per call.** Two questions cannot be answered with one click.
- If your harness genuinely cannot offer selectable answers, ask the question in plain words and accept a typed reply — but do not print a numbered menu and call it a choice.
- Do this for every question here: the opening question, reuse-or-create, the licence, the agent version, how they manage the environment, the confirmation that setup is done, and whether to list what is now visible.
- Saying "just click when you're ready" and then listing options as text is the exact failure this rule exists to prevent.

---

## Procedure

### Step 1 — Look at what already exists

Call `private_network_agent` (action: list) **once**, after the two questions above have been answered. Do not call it again later in the flow.

Each entry has:

| Field | Meaning |
| --- | --- |
| `name` | The identifier. **Always pass this to other tools.** Usually `pnagent-<uuid>`. |
| `displayName` | The label the user sees in the App Connect UI. Never pass this to a tool. |
| `isDefault` | `true` for the switch server's own agent. It cannot be renamed or deleted, and is not for connecting a remote environment. |

If a non-default configuration already exists that matches what the user is describing, offer to reuse it rather than creating a duplicate — creating a second one for the same environment is a common and confusing mistake.

**Raise this once and settle it.** Report what you found and ask reuse-or-create in the same breath, as one question with selectable answers, then act on the answer. Do not mention the existing agent, move on to something else, and then bring it up again — from the user's side that reads as the assistant having forgotten it already asked.

Question: "You already have an agent called <displayName>. Reuse it for this environment, or create a new one?" Suggested answers: `Reuse <displayName>` / `Create a new one`

### Step 2 — Create the configuration

Ask the user what to call it, then call `private_network_agent` (action: create) with that `displayName`. This creates a private network agent _configuration_ — it does not install anything.

Do **not** invent or ask for an id — the tool generates one. The result gives you the `name` (`pnagent-<uuid>`). **Remember this value**; every later step needs it, and the display name will not work.

### Step 3 — Secure Agent installer, with licence acceptance

Skip this step if the remote environment already runs App Connect Enterprise and only needs registering.

**Step 3a.** Ask which operating system that machine runs — this is the first point it matters.

Question: "Which operating system is that machine?" Suggested answers: `Linux` / `Windows` / `Mac`

Then call `secure_agent.get_installer_url` with the matching `selectedOS` (`linux_x64`, `win_x64`, `macos_x64`) and nothing else. It returns the IBM licence, the notices, the version, and `licenseAccepted: false`. It deliberately returns **no download link**.

**Step 3b.** Show the user a fair summary of the licence — program name, licence type, warranty and liability position — and tell them where to read it in full. Then ask:

> "Do you accept the IBM licence terms for the Secure Agent installer?"

**Never assume acceptance. Never call the tool again with `licenseAccepted: true` until the user has explicitly said yes.** If they decline, stop here and report that nothing was downloaded.

**Step 3c.** Once they accept, call `secure_agent.get_installer_url` again with `licenseAccepted: true`. It returns:

```json
{
  "file": "secureagent-macos_x64_13.0.547.tgz",
  "downloadUrl": "https://<host>/download/secure-agent-installer/<uuid>?token=<token>",
  "command": "curl -fL '<downloadUrl>' -o '<file>'",
  "expiresInSeconds": 300
}
```

**Run the `command` immediately** — the link expires after a few minutes. If it has expired you will get a `404`; call the tool again (with `licenseAccepted: true`) for a fresh link rather than retrying the old one.

The archive is around 75 MB. Extract it where the user wants the agent to live, and read the extracted contents to find the start script rather than guessing its name.

### Step 4 — Ask which version, then get the configuration

The configuration comes in two shapes and they are not interchangeable. **Never assume which one the user needs** — the wrong shape leaves the agent unable to authenticate.

| Agent version | What `get_config` returns |
| --- | --- |
| **13.0.478 or later** | `switchclient.json` plus `clientCert.pem` and `clientKey.pem`, to be saved side by side |
| **13.0.477 or earlier** | A single `switchclient.json` with the certificates embedded inside it |

**If the user downloaded the installer in Step 3**, you already know the version — it is in the file name, e.g. `secureagent-macos_x64_13.0.547.tgz` is 13.0.547. State it and confirm rather than asking blind:

> "You have just downloaded Secure Agent 13.0.547, so I will fetch the newer configuration format — a switchclient.json plus two PEM files. Is that right?"

**In every other case you cannot see the version, so you must ask.** This includes the common case where App Connect Enterprise was already installed on that machine — its ACE version decides the format, and you have no way to read it. Ask, with selectable answers:

Question: "Which version of App Connect Enterprise is on that machine?" Suggested answers: `13.0.478 or later` / `13.0.477 or earlier` / `I'm not sure`

If they are not sure, offer to work it out from the name of the file or folder they installed from — it usually looks like `secureagent-linux_x64_13.0.547.tgz`. Failing that, say plainly that newer agents take three files and older ones take a single file, and offer to try the newer format first since it is far more common — and that if the agent does not connect, swapping to the other format is a quick fix.

**Do not call `private_network_agent` (action: get_config) until you have either asked this question or confirmed the version from an installer you downloaded yourself.**

Then call `private_network_agent` (action: get_config) with the `name` from Step 1 or 2, passing `agentVersion: "13.0.477_or_earlier"` for older agents and leaving it unset for current ones.

**Write the result to files. Do not paste the contents into the conversation** — they contain the agent's private key. Tell the user where you wrote them, and how many files there are, so they know what to move to the remote machine.

### Step 5 — Tell the user what to do on the remote side, then wait

If you installed the Secure Agent yourself in Step 3, skip straight to its instructions below. Otherwise read on.

**If App Connect Enterprise is already running there**, there are two ways to apply the configuration and they have nothing in common. **Ask which — never assume, and never give both sets of steps at once.**

Question: "How do you manage that environment?" Suggested answers: `Through a web dashboard in my browser` / `By running commands on the machine`

#### Through a dashboard in a browser (ACEcc)

The file is not copied onto disk by hand here — it becomes a configuration object in that dashboard:

1. In the remote App Connect Dashboard, go to the **Configuration** page and create a new configuration.
2. Choose the type **Private Network Agent**.
3. Upload the `switchclient.json` you were given.
4. **Attach that configuration to the integration server** they want visible from App Connect — it is selected when creating or editing the server, not when creating the configuration.
5. Restart or redeploy that integration server so it picks the configuration up.

Step 4 is the one people miss: a configuration that exists but is not attached to a server does nothing at all.

If instead they run the container themselves with `kubectl`, the equivalent is a secret:

```
kubectl create secret generic cloud-switchclient-json \
  --from-file=switchclient.json=/path/to/switchclient.json
```

#### By running commands on the machine

1. Move `switchclient.json` into the `config/switch` folder of the server's work directory, creating `switch` if it does not exist. On a container that path is `/home/aceuser/ace-server/config/switch/switchclient.json`.
2. Restart the integration server:
   ```
   IntegrationServer --work-dir <workDirLocation> --name <uniqueName>
   ```
   The `--name` becomes the runtime's name in App Connect, so it is worth choosing something recognisable.
3. Watch the server console for confirmation it has dialled out:
   ```
   BIP6450I: The connection agent for remote callable flows has established a
   connection to the Switch server with URL 'wss://...'
   ```

For an integration **node** rather than a server, the file goes to `<workDirectory>/components/<integrationNodeName>/config/switch/` and the node is restarted instead.

Whichever route they take, tell them about `BIP6450I` — it lets them confirm success on their own side before you check from yours, which turns "it isn't working" into a question with an answer.

**If they are installing the Secure Agent on a fresh machine:**

1. Extract the archive you downloaded in Step 3.
2. Put `switchclient.json` in the directory they will pass as the config location.
3. Start the agent from that directory, accepting the licence by typing `yes` when prompted:
   ```
   ./secureagent -c /secureAgent -l /secureAgent      # Linux and Mac
   secureagent -c C:\secureAgent -l C:\secureAgent    # Windows
   ```
   `-c` is where the configuration lives, `-l` is where logs go.
4. A successful start prints `SwitchClient - All agents started`.

If they also want to reach a specific application inside that private network, the `endpoints` array in `switchclient.json` is where its hostname and port go. That is only needed for reaching private applications, not for managing the environment, so do not raise it unless they ask.

Now hand back to the user. **Ask with selectable answers** — this is a click, not a sentence for them to answer in prose.

Question: "Let me know once the configuration is in place and the server has restarted, and I will check the connection." Suggested answers: `Done — check it now` / `Not yet` / `I need help`

**Stop and wait.** Do not verify until the user confirms.

### Step 6 — Verify the connection

When the user confirms, check whether the environment is reachable. On the remote-administration path the decisive check is `deployment_location_runtimes` (action: list); `deployment_location` (action: list) shows the connected agents under `children.switchConnections.children[].children.agents.children` and is useful context.

**Verify by outcome, not by identity.** IBM's own success criterion is whether the remote servers become visible and manageable — not whether a particular id appears. Matching ids is unreliable here, for a reason worth understanding:

A `switchclient.json` with `"admin": "enabled"` connects as **AgentA**, the remote-administration client. An environment connecting that way identifies itself with its _own_ agent identity — for a container, one that comes from its instance configuration — not with the `pnagent-<uuid>` of the agent you created. So on the remote-administration path, **do not expect the uuid from `private_network_agent` (action: create) to appear**, and do not report its absence as a failure.

Check in this order:

**1. Do their runtimes appear?** Call `deployment_location_runtimes` (action: list) against the switch deployment location. If their servers are listed, the environment is connected and manageable — that is the answer, and you can stop. This is the check that matches what IBM documents.

**2. Does the deployment location show an agent at all?** `deployment_location` (action: list) returns each connected agent as `name` (its display name), `type` (e.g. `container`), and `agentId` (that environment's own agent id). Use this for context — "your environment is showing as _agni_, a container agent" — rather than as a pass/fail test.

**3. Where the user installed a Secure Agent themselves in Step 3** — it registers under the id from the `switchclient.json` you generated, so an entry whose `agentId` matches the uuid in `pnagent-<uuid>` is direct confirmation.

If the runtimes do not appear but an agent is connected, the server has dialled in and has not been **registered** yet: **Integration runtime management → Register runtime → App Connect runtimes**, where it is listed to select and add. Once added it appears as a tile. Say this rather than treating it as a failure.

**3. Nothing is connected at all.** Only now is it a troubleshooting problem:

- The agent may still be starting — wait a few seconds and call the tool again.
- Confirm the configuration landed in the right place on the remote side.
- Confirm the runtime was restarted after the file was added.
- Confirm they used the configuration for the agent you created, not an older one.
- Check the remote environment can reach the `switch.url` from the configuration on port 443.
- Ask whether they saw `BIP6450I` (existing server) or `SwitchClient - All agents started` (Secure Agent). If they did, the problem is on the App Connect side rather than theirs.

**Do not call `deployment_location` (action: list) repeatedly.** Two calls without the user having changed something on their side tells you nothing new — stop, and either offer option 2 above or ask them what they see. Retrying in a loop is the least useful thing you can do with the user's time.

If the agent is connected but **its runtimes do not appear**, the server has dialled in but has not been registered yet. In App Connect that is **Integration runtime management → Register runtime → App Connect runtimes**, where the connected server is listed for them to select and add. Say so rather than treating it as a failure.

There is no dedicated status tool — `deployment_location` (action: list) is the check. Say "your agent is showing at the deployment location", not "your agent is healthy", which you cannot know.

**When it is connected, say so and stop there.** The connection is the thing the user came for, and it deserves its own message rather than being buried under a wall of inventory. Confirm it, name the deployment location the agent came in on, then ask with suggested answers:

Question: "Your on-premises environment is connected. Want me to show you what is now visible from it?" Suggested answers: `Yes, show me everything` / `Just the runtimes` / `Not now`

**Do not call the listing tools until they answer.**

### Step 7 — Show what the connection gives them, when they ask

Only after the user asks. Call whichever they wanted, against the deployment location the agent connected on:

- `deployment_location_runtimes` (action: list) — the remote integration runtimes
- `deployment_location_bar_files` (action: list) — BAR files deployed there
- `deployment_location_configurations` (action: list) — configurations defined there

**List what is actually there, by name.** A count on its own is not an answer — the point of the demo is that the user can now see their own assets.

- **20 or fewer** — name every one of them.
- **More than 20** — give the exact number, name the first ten, and offer to show the rest or filter them.
- Never write "45+", "including …", or "etc." in place of names. Give the real count and real names.

Group them under a heading per category so the user can see at a glance what came from their environment.

Close by explaining what has changed: the Manage experience in App Connect now shows the runtimes, deployed BAR files and integrations, and configurations from the connected environment, alongside everything already running in the instance — all from one place, without opening the remote dashboard.

---

## Common mistakes

- **Passing the display name to a tool.** `private_network_agent` (action: get_config) and friends take the `name` (`pnagent-<uuid>`). A display name gives a `404`.
- **Skipping the licence gate.** The download link does not exist until the user accepts. Do not fabricate acceptance on their behalf.
- **Letting the download link go stale.** Run the returned `command` straight away; re-run the tool for a fresh link instead of retrying an expired one.
- **Giving command-line steps to someone who uses a dashboard.** They are different procedures, not variants. Ask how they manage the environment, then give only the matching set.
- **Asking for the operating system when App Connect Enterprise is already installed.** There is no installer to download on that path, so the answer changes nothing. Ask it in Step 3 or not at all.
- **Raising the same point twice.** Call `private_network_agent` (action: list) once, and settle reuse-or-create in one exchange. Repeating it reads as having forgotten.
- **Putting the options in the message instead of the question.** Options written as `A.` / `B.` bullets cannot be clicked. Pass them to your harness's question mechanism as selectable answers, one question per call.
- **Calling it a failure because a different agent name is showing.** Check whether their runtimes are listed first — that is what "connected" means in practice.
- **Burying the result.** When the agent connects, say so on its own, then ask before listing inventory.
- **Summarising instead of listing.** "45+ BAR files including…" is not a list. Give the real count and the real names.
- **Insisting on seeing your agent's uuid.** An environment connecting for remote administration registers as **AgentA** under its own identity, not as the `pnagent-<uuid>` you created. Verify that their runtimes appear instead.
- **Asking the user to classify their own environment.** Questions like "is this an integration node or a dashboard?" push the jargon back onto the person who came here to avoid it. Ask what they can see instead.
- **Letting an approval prompt do the explaining.** Say what a step is for before you take it, whether or not a prompt is shown.
- **Assuming the agent version.** `get_config` defaults to the 13.0.478+ format. Confirm the version with the user before calling it — an older agent given three files, or a current agent given one, will not connect.
- **Creating a duplicate agent.** Check `private_network_agent` (action: list) first and offer to reuse.
- **Using the default agent.** The entry with `isDefault: true` belongs to the switch server; never hand its configuration to a user for a new environment.
- **Printing certificates into the chat.** Write the configuration to files instead.

## Related skills

- [`skills/ace-iwhi/SKILL.md`](../ace-iwhi/SKILL.md) — deploy and manage runtimes once the environment is connected.
- [`skills/ace-bar-package/SKILL.md`](../ace-bar-package/SKILL.md) — build a BAR to deploy.
