# Slack Recipe → Ingredient List (n8n workflow)

An n8n automation that lives entirely inside Slack. A user types the name of a recipe in a Slack channel, an AI model works out every product and ingredient needed to make it, and the workflow posts the list back in the same channel.

```
You type:        chicken biryani
Bot replies:     Ingredients for chicken biryani
                 Main ingredients
                 • Basmati rice - 2 cups
                 • Chicken - 800 g
                 ...
```

No external apps, forms or dashboards are involved. The question and the answer both happen in Slack.

---

## Table of contents

1. [How it works](#how-it-works)
2. [What is in this repository](#what-is-in-this-repository)
3. [Requirements](#requirements)
4. [Step 1: Create and configure the Slack app](#step-1-create-and-configure-the-slack-app)
5. [Step 2: Import the workflow into n8n](#step-2-import-the-workflow-into-n8n)
6. [Step 3: Create the n8n credentials](#step-3-create-the-n8n-credentials)
7. [Step 4: Configure the nodes](#step-4-configure-the-nodes)
8. [Step 5: Connect Slack to n8n (Request URL)](#step-5-connect-slack-to-n8n-request-url)
9. [Step 6: Publish and test](#step-6-publish-and-test)
10. [Node-by-node reference](#node-by-node-reference)
11. [Customisation](#customisation)
12. [Alternative trigger: slash command](#alternative-trigger-slash-command)
13. [Troubleshooting](#troubleshooting)
14. [Security notes](#security-notes)
15. [Limitations](#limitations)
16. [License](#license)

---

## How it works

```
Slack message
     │
     ▼
┌───────────────┐   ┌─────────────────────────────┐   ┌──────────────────────┐   ┌─────────────────────────────┐
│ Slack Trigger │──▶│ Ignore Bots and Empty       │──▶│ Work Out Ingredients │──▶│ Reply With Ingredient List  │
│ (message)     │   │ Messages (Filter)           │   │ (LLM Chain)          │   │ (Slack: post message)       │
└───────────────┘   └─────────────────────────────┘   └──────────┬───────────┘   └─────────────────────────────┘
                                                                 │
                                                       ┌─────────┴─────────┐
                                                       │ OpenAI Chat Model │
                                                       │ (gpt-4.1-mini)    │
                                                       └───────────────────┘
```

1. **Slack Trigger** fires whenever a new message is posted in the selected channel.
2. **Ignore Bots and Empty Messages** drops messages written by bots (including the workflow's own replies), messages with a subtype (edits, joins, file shares) and empty messages. This prevents an endless reply loop.
3. **Work Out Ingredients** sends the message text to the AI model together with a system prompt that tells it to list all ingredients and products for about 4 servings, grouped by category and formatted in Slack mrkdwn.
4. **Reply With Ingredient List** posts the AI output into the same channel the question came from.

If the message is clearly not a recipe name, the AI replies with one friendly sentence asking the user to type a recipe name instead.

> **Expect two executions per message.** The first (about 3 seconds) is your message, which runs through every node. The second (a few milliseconds) is the bot's own reply arriving back from Slack, which the filter stops. This is normal and intended.

---

## What is in this repository

| File | Purpose |
|------|---------|
| `slack-recipe-ingredients.json` | The exportable n8n workflow |
| `README.md` | This guide |

Your credentials are **not** included in the JSON. Anyone importing the workflow must attach their own Slack and OpenAI credentials.

---

## Requirements

| Requirement | Details |
|-------------|---------|
| n8n | A running instance reachable from the internet over **HTTPS** (n8n Cloud, or self-hosted behind a public HTTPS URL). Slack must be able to reach your webhook. |
| Slack | A workspace where you can create and install an app |
| AI provider | An OpenAI API key (the workflow uses `gpt-4.1-mini`). Any chat model supported by n8n can be swapped in. |
| Permissions | Ability to publish workflows in n8n and install apps in Slack |

---

## Step 1: Create and configure the Slack app

1. Go to <https://api.slack.com/apps> and click **Create New App**.
2. Choose **From scratch**, enter an app name (for example `recipeapp`) and pick your workspace.

### 1.1 Bot token scopes

Open **OAuth & Permissions → Scopes → Bot Token Scopes** and add:

| Scope | Why it is needed |
|-------|------------------|
| `chat:write` | Post the ingredient list |
| `channels:history` | Receive messages from public channels |
| `channels:read` | Let n8n load the channel list in the dropdown |
| `groups:history` | Only if you use a **private** channel |
| `groups:read` | Only if you use a **private** channel |
| `chat:write.public` | Optional. Lets the bot post in public channels it has not joined |

### 1.2 Install the app

1. On **OAuth & Permissions**, click **Install to Workspace** and approve.
2. Copy the **Bot User OAuth Token**. It starts with `xoxb-`.
3. Open **Basic Information → App Credentials** and copy the **Signing Secret**.

### 1.3 Make sure Socket Mode is off

Open **Settings → Socket Mode** and make sure it is **Off**.

> **Important:** When Socket Mode is on, Slack ignores the Request URL entirely and sends events over a websocket instead. The URL can still show "Verified", but n8n will never receive a single event. This is the most common reason the workflow never triggers.

### 1.4 Add the bot to your channel

In the Slack channel, run:

```
/invite @YourAppName
```

The bot must be a member of the channel to receive messages and to post there (unless you added `chat:write.public`).

> Events are configured in [Step 5](#step-5-connect-slack-to-n8n-request-url), after you have the n8n webhook URL.

---

## Step 2: Import the workflow into n8n

1. In n8n, open the workflow list and click **Create workflow**.
2. Click the **⋯** menu and choose **Import from File**.
3. Select `slack-recipe-ingredients.json`.

---

## Step 3: Create the n8n credentials

### Slack API credential

1. Open the **Slack Trigger** node and click the **Credential** dropdown, then **Create new credential**.
2. Choose **Access Token** (credential type *Slack API*).
3. Paste the **Bot User OAuth Token** (`xoxb-…`) into **Access Token**.
4. Paste the **Signing Secret** into **Signature Secret**. It is optional but strongly recommended.
5. Save.

> The Slack Trigger only supports the Access Token credential. Use **the same credential** on the **Reply With Ingredient List** node.
>
> The token, the app that owns the Request URL and the app that is a member of the channel must all be the **same Slack app**. A mismatch causes either no triggers at all or the error `not_in_channel`.

### OpenAI credential

1. Open the **OpenAI Chat Model** node and click **Create new credential**.
2. Paste your OpenAI API key and save.

---

## Step 4: Configure the nodes

| Node | What to set |
|------|-------------|
| **Slack Trigger** | **Trigger On:** `message`. **Channel to Watch:** select your channel from the list. If the list does not load, switch the field to **By ID** and paste the channel ID (see below). Leave **Watch Whole Workspace** off. |
| **Ignore Bots and Empty Messages** | No change needed |
| **Work Out Ingredients** | No change needed (optionally edit the system prompt, see [Customisation](#customisation)) |
| **OpenAI Chat Model** | Attach your OpenAI credential. Model: `gpt-4.1-mini` |
| **Reply With Ingredient List** | Attach the same Slack credential. The channel is set automatically from the incoming message by an expression, so you do not need to pick one. |

**Finding a channel ID:** open the channel in the Slack web app. The ID is the last part of the URL, for example `app.slack.com/client/T0XXXXXXX/C0XXXXXXXXX`, where `C0XXXXXXXXX` is the channel ID.

> **Do not use both `message` and `app_mention` triggers at once.** A message that @mentions the bot fires both events and you get two replies. Plain `message` already covers mentions, because the mention is just part of the text.

---

## Step 5: Connect Slack to n8n (Request URL)

1. In n8n, open the **Slack Trigger** node, then **Webhook URLs**.
2. Select the **Production URL** tab and copy the URL. It contains `/webhook/` and ends with `/webhook`, for example:
   `https://your-n8n-domain/webhook/<webhook-id>/webhook`
3. In your Slack app, open **Event Subscriptions** and turn **Enable Events** on.
4. Paste the URL into **Request URL**. Wait for the green **Verified** tick.
5. Under **Subscribe to bot events**, click **Add Bot User Event** and add:
   - `message.channels` for public channels
   - `message.groups` for private channels
6. Click **Save Changes**.
7. Open **Install App** and click **Reinstall to Workspace**. Slack does not start sending newly added events until you reinstall.

### Test URL vs Production URL

| | Test URL | Production URL |
|---|----------|----------------|
| Path contains | `/webhook-test/` | `/webhook/` |
| Works when | You clicked **Execute workflow** and the trigger shows "Waiting for trigger event" | The workflow is **published** |
| Use for | One-off debugging | Normal operation |

n8n cannot listen for test events and production events on a Slack Trigger at the same time. To use **Execute workflow**, unpublish first.

> **The webhook URL changes if the Slack Trigger node is deleted and recreated.** Each new node gets a new webhook ID. If you replace the node, paste the new Production URL into Slack's Request URL again.

---

## Step 6: Publish and test

1. Click **Publish** in n8n. **Publish again after every edit.** Saving only changes the draft, and the live workflow keeps running the last published version.
2. In the Slack channel, post a plain message such as `chicken biryani`. Do not post it as a thread reply.
3. Within roughly 5 to 10 seconds the bot should reply with the ingredient list.
4. Open the **Executions** tab in n8n to verify. A successful run shows every node green. Click **Copy to editor** to inspect the data node by node.

Live production runs never light up the editor canvas. Only manual test runs do. Use the **Executions** tab instead.

---

## Node-by-node reference

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | **Slack Trigger** | `n8n-nodes-base.slackTrigger` (v1) | Starts the workflow on every new message in the chosen channel |
| 2 | **Ignore Bots and Empty Messages** | `n8n-nodes-base.filter` (v2.2) | Lets a message through only if `bot_id` does not exist, `subtype` does not exist and `text` is not empty |
| 3 | **Work Out Ingredients** | `@n8n/n8n-nodes-langchain.chainLlm` (v1.9) | Builds the prompt from the Slack message and calls the model |
| 4 | **OpenAI Chat Model** | `@n8n/n8n-nodes-langchain.lmChatOpenAi` (v1.3) | The language model used by the chain (`gpt-4.1-mini`) |
| 5 | **Reply With Ingredient List** | `n8n-nodes-base.slack` (v2.7) | Posts the AI output to the originating channel, with mrkdwn on and the "sent from n8n" link off |

### The filter, in detail

All three conditions must be true (AND):

| Condition | Meaning |
|-----------|---------|
| `bot_id` does not exist | The message was written by a person, not a bot. This stops the bot answering its own replies. |
| `subtype` does not exist | The message is a normal message, not an edit, deletion, join or file share |
| `text` is not empty | There is something to send to the AI |

### The prompt

The user message sent to the model is:

```
Recipe requested in Slack: {{ $json.text }}
```

The system message instructs the model to:

- treat the message as a recipe name and ignore any Slack @mention
- list every product and ingredient for about 4 servings
- reply in Slack mrkdwn only, with a bold title (`*Ingredients for <recipe>*`) and bold section headings such as *Main ingredients*, *Spices and seasonings*, *Pantry staples* and *Optional extras*
- write one bullet per item as `• item - quantity`, using single asterisks for bold
- add no cooking steps, preamble or closing remarks
- if the message is clearly not a recipe name, reply with one friendly sentence asking for a recipe name

### Example

**Input (Slack message):**

```
chicken biryani
```

**Output (bot reply):**

```
Ingredients for chicken biryani

Main ingredients
• Basmati rice - 2 cups
• Chicken - 800 g to 1 kg
• Onions - 2 large, thinly sliced
• Tomatoes - 2 medium, chopped
• Yogurt - 1 cup
...

Spices and seasonings
• Ginger-garlic paste - 2 tablespoons
• Biryani masala - 2 tablespoons
• Turmeric - 1/2 teaspoon
...

Pantry staples
• Cooking oil or ghee - 4 tablespoons

Optional extras
• Saffron strands soaked in warm milk
• Fried onions for garnish
```

Exact quantities vary slightly from run to run.

---

## Customisation

| Goal | How |
|------|-----|
| Change servings | Edit "for about 4 servings" in the system message of **Work Out Ingredients** |
| Add cooking steps | Remove "Do not add cooking steps" and add "then give short numbered cooking steps" |
| Use metric only (or imperial only) | Add the instruction to the system message |
| Use a different AI model | Delete **OpenAI Chat Model** and connect another chat model node (for example Anthropic or Google) to the **Model** input of **Work Out Ingredients** |
| Reply inside a thread | In **Reply With Ingredient List**, add the option **Reply to a Message (Thread)** and set the timestamp to `{{ $("Slack Trigger").item.json.ts }}` |
| Reply in another language | Add "Reply in <language>" to the system message |
| Watch a different channel | Change **Channel to Watch** in **Slack Trigger**, then publish |
| React only when mentioned | Set the trigger to `app_mention` (and add the `app_mentions:read` scope and the `app_mention` bot event), and do not also keep `message` |

Remember to **publish** after any change.

---

## Alternative trigger: slash command

If message events are not an option in your environment, you can trigger the same logic with a slash command such as `/recipe chicken biryani`. This does not need Event Subscriptions or channel invites.

1. Replace the **Slack Trigger** with a **Webhook** node (HTTP method `POST`, response mode "Using Respond to Webhook node").
2. Add a **Respond to Webhook** node straight after it that returns a short text such as "Working out the ingredients...". Slack requires a reply within 3 seconds.
3. Continue with **Work Out Ingredients**, reading the recipe from `{{ $("Webhook").item.json.body.text }}`.
4. In **Reply With Ingredient List**, set the channel to `{{ $("Webhook").item.json.body.channel_id }}`. The filter node is no longer needed because slash commands do not echo bot messages.
5. In your Slack app, open **Slash Commands**, create `/recipe` and set the Request URL to the webhook's Production URL. Add the `chat:write` and `chat:write.public` scopes, then reinstall the app.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Nothing happens, no executions at all | **Socket Mode is on** in the Slack app | Turn it off under Settings → Socket Mode, then save and reinstall |
| Nothing happens, no executions at all | Workflow is not published | Click **Publish** |
| Nothing happens, no executions at all | Request URL in Slack is old, a test URL, or from a deleted trigger node | Copy the current **Production URL** from the Slack Trigger and paste it into Event Subscriptions |
| Nothing happens, no executions at all | `message.channels` (or `message.groups`) not subscribed, or the app was not reinstalled | Add the event, save, then **Reinstall to Workspace** |
| Nothing happens, no executions at all | Bot not in the channel, or you are posting in a different channel from the one selected | Run `/invite @YourApp`, and confirm the trigger's channel matches |
| Nothing happens, no executions at all | Message posted as a thread reply | Post a normal top-level message |
| Request URL will not verify | Workflow unpublished (production URL) or not listening (test URL), or n8n not reachable over public HTTPS | Publish the workflow, or click **Execute workflow** when using the test URL, and check your n8n domain |
| Events rejected silently | Wrong **Signature Secret** in the n8n credential | Use the Signing Secret of the same Slack app, or clear the field to test |
| Execution appears but ends after the filter | The message came from a bot, has a subtype, or is empty | Expected for the bot's own replies. For user messages, check the message data in the execution. |
| Error `not_in_channel` on the reply node | The token in the credential belongs to a bot that is not in the channel | Use the token from the app that is in the channel, or invite that app, or add `chat:write.public` and reinstall |
| Error `missing_scope` | A needed scope is missing | Add it under Bot Token Scopes and reinstall |
| Channel dropdown will not load | Missing `channels:read` / `groups:read` scope, or wrong token | Add the scopes and reinstall, or switch the field to **By ID** |
| OpenAI error on **Work Out Ingredients** | Missing or invalid OpenAI credential, or no API credit | Attach a valid credential and check billing |
| Two replies per message | Both `message` and `app_mention` are enabled | Keep only `message` |
| "Unpublish the workflow to execute" | n8n cannot test and listen in production at the same time on a Slack Trigger | Unpublish, run **Execute workflow** with the test URL, then switch back and publish |
| Changes have no effect | Edited but not published | Click **Publish** again |
| Bot replies endlessly to itself | The filter node was removed or changed | Restore **Ignore Bots and Empty Messages** |

### Debugging checklist

1. Is the workflow **Published**?
2. Does the Slack **Request URL** match the trigger's current **Production URL** and show **Verified**?
3. Is **Socket Mode Off**?
4. Is `message.channels` (or `message.groups`) listed under bot events, with the app **reinstalled** afterwards?
5. Is the **same Slack app** the owner of the Request URL, the token in n8n and the member of the channel?
6. Is the trigger's channel the one you are posting in?
7. Does the **Executions** tab show a run? If yes, open it and look for a red node.

---

## Security notes

- The exported JSON contains credential **names and IDs only**, never the actual token, signing secret or API key.
- Before committing, search the file for `xoxb-`, `xoxp-`, `sk-` and `signingSecret` to confirm nothing sensitive was added by hand.
- Never commit your Slack tokens, signing secret or OpenAI key. If you do, rotate them immediately.
- Set the **Signature Secret** so that n8n verifies that incoming requests really come from Slack.
- The workflow reads **every** message in the watched channel. Use a dedicated channel and tell your team what the bot does.

---

## Limitations

- **Every message is processed.** Each non-bot message in the channel triggers an AI call, which costs tokens. Use a dedicated channel.
- **AI output can be wrong.** Quantities are approximate and ingredients may be incomplete or inaccurate. Do not rely on it for allergy or dietary safety. Check the list yourself.
- **One recipe per message.** The whole message text is treated as a single recipe name.
- **Plain messages only.** Edits, file shares and bot messages are ignored by design.
- **Typical latency** is 3 to 6 seconds, mainly the model call.

---

## License

Choose and add a license of your own, for example MIT, before publishing the repository.
