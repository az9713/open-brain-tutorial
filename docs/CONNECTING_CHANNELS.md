# Connecting Channels to Open Brain

A complete guide to capturing thoughts from Slack, Telegram, Discord, Email, Obsidian, WhatsApp, and SMS — with full, copy-paste-ready code for every integration.

**Audience:** This guide assumes you have a working Open Brain setup (the `thoughts` table exists in Supabase, your MCP server is deployed, and you have an OpenRouter API key). It assumes you may not know what a webhook is, what a bot token is, or how to write a Deno function. All of that is explained here from scratch.

**Reading time:** 20 minutes to read the whole thing. 15–45 minutes per channel to actually build it.

---

## Table of Contents

1. [How Channel Integration Works](#1-how-channel-integration-works)
2. [What You Need Before Starting](#2-what-you-need-before-starting)
3. [Slack](#3-slack-fully-implemented)
4. [Telegram](#4-telegram-build-guide)
5. [Discord](#5-discord-completion-guide)
6. [Email](#6-email-via-forwarding)
7. [Obsidian](#7-obsidian-local-vault)
8. [WhatsApp](#8-whatsapp-business-api)
9. [SMS via Twilio](#9-sms-via-twilio)
10. [Channel Comparison](#10-channel-comparison)
11. [Building Your Own Integration](#11-building-your-own-integration)

---

## 1. How Channel Integration Works

Every integration in this guide follows the same five-step pattern. The specific details differ per channel, but the structure is always identical.

```
+------------------+     webhook      +------------------+
|                  |  (HTTP POST)     |                  |
|  External        +---------------->|  Edge Function   |
|  Channel         |                 |  (your code on   |
|  (Slack,         |                 |  Supabase)       |
|  Telegram,       |                 |                  |
|  Discord, etc.)  |                 +--------+---------+
|                  |                          |
+------------------+                          | parallel calls
                                              |
                              +---------------+---------------+
                              |                               |
                   +----------v---------+       +------------v---------+
                   |                    |       |                      |
                   |  OpenRouter        |       |  OpenRouter          |
                   |  Embeddings API    |       |  Chat API            |
                   |  (text-embedding   |       |  (gpt-4o-mini)       |
                   |   -3-small)        |       |  Extracts metadata:  |
                   |  Turns text into   |       |  people, topics,     |
                   |  1536 numbers      |       |  type, action items  |
                   |                    |       |                      |
                   +----------+---------+       +------------+---------+
                              |                               |
                              +---------------+---------------+
                                              |
                                             both
                                             results
                                              |
                                   +----------v-----------+
                                   |                      |
                                   |  Supabase            |
                                   |  thoughts table      |
                                   |                      |
                                   |  INSERT {            |
                                   |    content,          |
                                   |    embedding,        |
                                   |    metadata          |
                                   |  }                   |
                                   |                      |
                                   +----------+-----------+
                                              |
                                   +----------v-----------+
                                   |                      |
                                   |  Confirmation reply  |
                                   |  sent back to        |
                                   |  channel user        |
                                   |                      |
                                   +----------------------+
```

### What each step does

**Step 1: Something happens in the channel**

You type a message in Slack. You send a text to your Telegram bot. You post in a Discord channel. This is the trigger.

**Step 2: The channel sends a webhook to your Edge Function**

This is the key concept. A webhook is not magic — it is just an HTTP POST request.

Here is what normally happens when you visit a website: your browser sends an HTTP GET request to a server, the server sends back HTML, your browser shows the page. That is the web.

A webhook reverses this. Instead of you asking a server for data, the server asks YOU. When a message is posted in Slack, Slack's servers send an HTTP POST request to a URL you provided during setup. That URL is your Edge Function. The body of the POST request contains the message content as JSON.

In plain terms: "Your Edge Function has a URL. You give that URL to Slack/Discord/Telegram during setup. When a message is posted, the platform sends an HTTP POST to your URL with the message content as JSON. Your function processes it and responds with 200 OK (a standard HTTP success code)."

The channel platform does not care what you do with the data. It sends the message, waits up to a few seconds for a 200 OK response, and moves on.

**Step 3: The Edge Function receives the message text**

An Edge Function is a small piece of code that runs on Supabase's servers whenever it receives an HTTP request. It runs on demand — you are not paying for a server to sit idle. When a webhook arrives, Supabase wakes up your function, runs it, and shuts it down. Each function invocation typically takes 2–5 seconds.

Edge Functions in this project are written in TypeScript and run on Deno (a JavaScript/TypeScript runtime, similar to Node.js but newer). You do not need to know Deno deeply. The complete code is provided for every channel.

**Step 4: Generate embedding and extract metadata in parallel**

Two calls happen simultaneously:

- The text is sent to OpenRouter's embeddings endpoint, which returns a list of 1536 numbers (a vector). This vector captures the meaning of the text in a mathematical form that enables semantic search later.
- The text is also sent to gpt-4o-mini with a prompt that extracts structured metadata: what type of thought is this, who is mentioned, what topics does it cover, are there action items.

Running them in parallel (using `Promise.all`) cuts the wait time roughly in half.

**Step 5: Insert into the thoughts table**

Both results come back. The Edge Function calls `supabase.from("thoughts").insert(...)` with the content, the embedding, and the metadata. The row is stored. The channel receives a 200 OK response.

**Step 6 (optional): Send a confirmation reply**

Most integrations send a brief reply back to the channel: "Captured as person_note — career, consulting. People: Sarah." This closes the loop so you know the capture worked.

---

## 2. What You Need Before Starting

Before building any integration, confirm you have:

**1. A working Open Brain setup**

The `thoughts` table must exist in your Supabase project with the `content`, `embedding`, and `metadata` columns. The MCP server must be deployed. If you have not done this, start with `docs/01-getting-started.md`.

**2. Supabase CLI installed and linked**

The CLI is the command-line tool that lets you deploy Edge Functions from your terminal. Verify it is installed:

```bash
supabase --version
```

If you see a version number, you are good. If the command is not found, go back to `docs/01-getting-started.md` Step 7.

You also need to be logged in and linked to your project:

```bash
supabase login
supabase link --project-ref YOUR_PROJECT_REF
```

Replace `YOUR_PROJECT_REF` with the string from your Supabase dashboard URL: `supabase.com/dashboard/project/THIS_PART`.

**3. Your OpenRouter API key**

This was set during the main setup as a Supabase secret. Verify it is set:

```bash
supabase secrets list
```

You should see `OPENROUTER_API_KEY` in the list. If not, set it:

```bash
supabase secrets set OPENROUTER_API_KEY=your-key-here
```

**4. Your Supabase project URL and service role key**

These are automatically available inside Edge Functions as `SUPABASE_URL` and `SUPABASE_SERVICE_ROLE_KEY`. You do not need to set them manually.

---

## 3. Slack (Fully Implemented)

The complete Slack integration lives in `integrations/slack-capture/README.md`. It is 338 lines with full troubleshooting. This section is a summary of the key steps and a data flow diagram.

### What the full implementation does

A Slack bot monitors a designated channel. Every message triggers an Edge Function (`ingest-thought`) that embeds the text, extracts metadata, stores the thought, and replies in the thread with what it captured.

```
+-------------------+
|  You type in      |
|  #capture channel |
+--------+----------+
         |
         | Slack sends webhook to Edge Function URL
         | HTTP POST with event JSON
         v
+--------+----------+
|  ingest-thought   |
|  Edge Function    |
|                   |
|  1. Verify this   |
|     is the right  |
|     channel       |
|  2. Skip bots,    |
|     edits, etc.   |
+--------+----------+
         |
         | Promise.all([embedding, metadata])
         |
    +----+------+
    |           |
    v           v
+-------+  +----------+
| Embed |  | Extract  |
| text  |  | metadata |
+---+---+  +----+-----+
    |            |
    +----+-------+
         |
         v
+--------+----------+
|  supabase         |
|  .from("thoughts")|
|  .insert(...)     |
+--------+----------+
         |
         v
+--------+----------+
|  replyInSlack()   |
|  "Captured as     |
|   person_note —   |
|   career,         |
|   consulting"     |
+-------------------+
```

### Key setup steps (summary)

1. Create a private Slack channel (#capture). Get its Channel ID (right-click the channel in Slack, View channel details, scroll to bottom — starts with C).
2. Go to api.slack.com/apps, create a new app "From scratch", select your workspace.
3. Under OAuth & Permissions, add Bot Token Scopes: `channels:history`, `groups:history`, `chat:write`.
4. Install to workspace. Copy the Bot User OAuth Token (starts with `xoxb-`).
5. Deploy the Edge Function (code is already in `integrations/slack-capture/`):
   ```bash
   supabase functions new ingest-thought
   # Copy the code from integrations/slack-capture/README.md into the new function
   supabase functions deploy ingest-thought --no-verify-jwt
   ```
6. Set secrets:
   ```bash
   supabase secrets set SLACK_BOT_TOKEN=xoxb-your-token-here
   supabase secrets set SLACK_CAPTURE_CHANNEL=C0your-channel-id-here
   ```
7. Copy the Edge Function URL from the deploy output.
8. In your Slack app, go to Event Subscriptions, enable events, paste the URL, wait for the green "Verified" checkmark.
9. Add bot events: `message.channels` AND `message.groups`. You need both — public channels fire the first, private channels fire the second.
10. Invite the bot to your channel: `/invite @Open Brain`.
11. Test by posting a message. You should see a threaded reply within 10 seconds.

For the complete code, full troubleshooting, and step-by-step screenshots, see `integrations/slack-capture/README.md`.

**Time to build:** ~30 minutes. **Difficulty:** Easy.

---

## 4. Telegram (Build Guide)

Telegram is the easiest integration to build. The Bot API requires no app review, no OAuth flow, no scope configuration, no business account. You message a bot called BotFather, get a token, set a webhook URL, and you are done in 15 minutes.

### What a bot token is

When you create a Telegram bot, BotFather gives you a token that looks like this:

```
7123456789:AAHdqTcvOH-vKATkQ2U9_abcXYZ123example
```

This token is both the bot's identity and its password. Anyone with this token can send messages as your bot and read messages sent to it. Keep it secret — treat it like a password.

The token goes into your Edge Function via a Supabase secret (`TELEGRAM_BOT_TOKEN`). Your Edge Function uses it to call the Telegram API to send replies.

### Step 1: Create a bot

1. Open Telegram on your phone or desktop.
2. In the search bar, search for `@BotFather` — it has a blue verified checkmark.
3. Start a conversation. Send the command `/newbot`.
4. BotFather asks for a display name. Type: `Open Brain Capture`
5. BotFather asks for a username. It must end in "bot". Try: `ob1_capture_bot` (if taken, try `openbrain_capture_bot` or any variant).
6. BotFather replies with your token. It looks like the example above. Copy it immediately.

That is the entire setup on Telegram's side. No dashboard, no app review, no waiting.

### Step 2: Create the Edge Function

In your terminal, create the function:

```bash
supabase functions new telegram-capture
```

This creates the file `supabase/functions/telegram-capture/index.ts`. Open it and replace the entire contents with the following code:

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

// --- Configuration ---
// These values come from Supabase secrets (set via `supabase secrets set`)
// SUPABASE_URL and SUPABASE_SERVICE_ROLE_KEY are set automatically
const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const OPENROUTER_API_KEY = Deno.env.get("OPENROUTER_API_KEY")!;
const TELEGRAM_BOT_TOKEN = Deno.env.get("TELEGRAM_BOT_TOKEN")!;

const OPENROUTER_BASE = "https://openrouter.ai/api/v1";
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

// --- Embedding ---
// Converts text to a 1536-dimensional vector for semantic search.
// This is what makes "what was I thinking about leadership?" work even
// if you never used the word "leadership".
async function getEmbedding(text: string): Promise<number[]> {
  const r = await fetch(`${OPENROUTER_BASE}/embeddings`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/text-embedding-3-small",
      input: text,
    }),
  });
  const d = await r.json();
  return d.data[0].embedding;
}

// --- Metadata Extraction ---
// Uses gpt-4o-mini to extract structured metadata from the raw text.
// Returns an object like:
// { type: "person_note", people: ["Sarah"], topics: ["career"], action_items: [] }
async function extractMetadata(text: string): Promise<Record<string, unknown>> {
  const r = await fetch(`${OPENROUTER_BASE}/chat/completions`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/gpt-4o-mini",
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Extract metadata from the user's captured thought. Return JSON with:
- "people": array of people mentioned (empty array if none)
- "action_items": array of implied to-dos (empty array if none)
- "topics": array of 1-3 short topic tags (always at least one)
- "type": one of "observation", "task", "idea", "reference", "person_note"
Only extract what is explicitly present. Do not infer or add things not in the text.`,
        },
        { role: "user", content: text },
      ],
    }),
  });
  const d = await r.json();
  try {
    return JSON.parse(d.choices[0].message.content);
  } catch {
    // If the LLM returns something we can't parse, use safe defaults
    return { topics: ["uncategorized"], type: "observation" };
  }
}

// --- Telegram Reply ---
// Sends a message back to the user in the same chat.
// parse_mode: "Markdown" lets us use *bold* in the reply.
async function sendTelegramMessage(chatId: number, text: string): Promise<void> {
  await fetch(
    `https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        chat_id: chatId,
        text,
        parse_mode: "Markdown",
      }),
    },
  );
}

// --- Main Handler ---
// This function runs every time Telegram sends a webhook to our Edge Function URL.
// Telegram sends updates as JSON objects. We look for message.text.
Deno.serve(async (req: Request): Promise<Response> => {
  try {
    const body = await req.json();
    const message = body.message;

    // Guard: ignore non-text messages (photos, stickers, etc.), edits, and bot messages
    if (!message || !message.text || message.from?.is_bot) {
      return new Response("ok", { status: 200 });
    }

    const text: string = message.text;
    const chatId: number = message.chat.id;
    const fromName: string = message.from?.first_name || "Unknown";

    // Handle /start command — sent when someone first opens the bot
    if (text === "/start") {
      await sendTelegramMessage(
        chatId,
        "Open Brain Capture is ready. Send me any thought and I will save it to your brain.",
      );
      return new Response("ok", { status: 200 });
    }

    // Handle /help command
    if (text === "/help") {
      await sendTelegramMessage(
        chatId,
        "Just send me text and I save it. Examples:\n\n" +
          "• *Sarah mentioned she wants to start consulting*\n" +
          "• *Meeting with design: ship dashboard v2 by March 15*\n" +
          "• *Idea: weekly async standups instead of daily sync*",
      );
      return new Response("ok", { status: 200 });
    }

    // Process the thought: embed and extract metadata in parallel
    const [embedding, metadata] = await Promise.all([
      getEmbedding(text),
      extractMetadata(text),
    ]);

    // Insert into the thoughts table
    const { error } = await supabase.from("thoughts").insert({
      content: text,
      embedding,
      metadata: {
        ...metadata,
        source: "telegram",
        from: fromName,
      },
    });

    if (error) {
      console.error("Supabase insert error:", error);
      await sendTelegramMessage(chatId, `Failed to capture: ${error.message}`);
      return new Response("error", { status: 500 });
    }

    // Build a confirmation reply showing what was captured and how it was classified
    const meta = metadata as Record<string, unknown>;
    let confirmation = `Captured as *${meta.type || "thought"}*`;
    if (Array.isArray(meta.topics) && meta.topics.length > 0) {
      confirmation += ` — ${meta.topics.join(", ")}`;
    }
    if (Array.isArray(meta.people) && meta.people.length > 0) {
      confirmation += `\nPeople: ${meta.people.join(", ")}`;
    }
    if (Array.isArray(meta.action_items) && meta.action_items.length > 0) {
      confirmation += `\nAction items: ${meta.action_items.join("; ")}`;
    }

    await sendTelegramMessage(chatId, confirmation);
    return new Response("ok", { status: 200 });
  } catch (err) {
    console.error("Function error:", err);
    return new Response("error", { status: 500 });
  }
});
```

### Step 3: Deploy

Set your bot token as a secret, then deploy:

```bash
supabase secrets set TELEGRAM_BOT_TOKEN=7123456789:AAHdqTcvOH-your-actual-token-here
supabase functions deploy telegram-capture --no-verify-jwt
```

The `--no-verify-jwt` flag is important. Telegram does not send a JWT token with its webhooks (JWT is a Supabase-specific auth mechanism). Without this flag, Supabase would reject all Telegram requests.

After deployment, the output shows your Edge Function URL. It looks like:

```
https://abcdefghijklmnop.supabase.co/functions/v1/telegram-capture
```

Copy this URL. You need it in the next step.

### Step 4: Register the webhook with Telegram

You now have a bot (with a token) and a URL (your Edge Function). You need to tell Telegram: "whenever someone sends a message to my bot, POST it to this URL."

Run this curl command, replacing both placeholders:

```bash
curl -X POST "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook" \
  -H "Content-Type: application/json" \
  -d '{"url": "https://YOUR_PROJECT_REF.supabase.co/functions/v1/telegram-capture"}'
```

Telegram responds with:

```json
{"ok": true, "result": true, "description": "Webhook was set"}
```

If you see `{"ok": true}`, the webhook is registered. Telegram will now POST every incoming message to your Edge Function.

To verify the webhook is set correctly at any time:

```bash
curl "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getWebhookInfo"
```

### Step 5: Test

1. Open Telegram and find your bot (search for the username you gave it, like `@ob1_capture_bot`).
2. Send: `Sarah mentioned she wants to start consulting`
3. Within 5–10 seconds you should see a reply: `Captured as *person_note* — career, consulting. People: Sarah`
4. Check your Supabase dashboard → Table Editor → thoughts. There should be a new row.

### Complete data flow

```
+------------------+
|  You on Telegram |
|  "Sarah mentioned|
|   she wants to   |
|   start          |
|   consulting"    |
+--------+---------+
         |
         | Telegram sends HTTP POST to your
         | Edge Function URL containing:
         | {
         |   "message": {
         |     "text": "Sarah mentioned...",
         |     "chat": { "id": 123456789 },
         |     "from": { "first_name": "You" }
         |   }
         | }
         v
+--------+---------+
| telegram-capture |
| Edge Function    |
+--------+---------+
         |
    +----+----+
    |         |
    v         v
+-------+  +----------+
| Embed |  | Extract  |
| text  |  | metadata |
+---+---+  +----+-----+
    |            |
    +----+-------+
         |
         v
+--------+---------+
| thoughts table   |
| INSERT row       |
+--------+---------+
         |
         v
+--------+---------+
| Telegram API     |
| sendMessage      |
| "Captured as     |
|  person_note..."  |
+------------------+
```

**Time to build:** ~15 minutes. **Difficulty:** Easiest of all channels.

---

## 5. Discord (Completion Guide)

The repo has a stub at `integrations/discord-capture/README.md` with the setup steps outlined but the Edge Function code missing. This section provides the complete implementation.

Discord is more complex than Telegram for one reason: Discord requires Ed25519 signature verification on all webhooks. Before your function processes any message, it must verify that the request actually came from Discord and was not forged. This is a security requirement Discord enforces — if your function does not verify the signature, Discord will not send real message data.

### What Ed25519 signature verification is

Ed25519 is a cryptographic signature scheme. When Discord sends a webhook to your URL, it includes two HTTP headers:

- `X-Signature-Ed25519`: a cryptographic signature of the request body
- `X-Signature-Timestamp`: the timestamp of the request

Your function uses Discord's public key (a string from the Discord Developer Portal) to verify that the signature is valid. If the signature does not match, you return a 401 and ignore the request. This prevents anyone from sending fake "Discord messages" to your function.

This is why the Discord setup is Medium difficulty rather than Easy — there is crypto code involved. The code is provided in full below.

### Step 1: Create a Discord Application

1. Go to discord.com/developers/applications.
2. Click "New Application". Name it "Open Brain".
3. On the General Information page, find the "Public Key" field. Copy it and save it — this is your `DISCORD_PUBLIC_KEY`.
4. In the left sidebar, click "Bot".
5. Click "Add Bot" then "Yes, do it!" to confirm.
6. Under the bot settings, find "Token". Click "Reset Token" → "Yes, do it!" → copy the token. Save it as your `DISCORD_BOT_TOKEN`.
7. Scroll down to "Privileged Gateway Intents". Enable: **Message Content Intent**. Without this, the bot can see that messages exist but cannot read their text.
8. Click Save Changes.

### Step 2: Invite the bot to your server

1. In the left sidebar, click "OAuth2" → "URL Generator".
2. Under Scopes, check: `bot`
3. Under Bot Permissions, check: `Read Messages/View Channels`, `Send Messages`
4. Copy the generated URL at the bottom.
5. Paste it in your browser, select your server, click Authorize.

The bot is now a member of your server, but it cannot see channels until you invite it to specific ones. In Discord, right-click the channel you want to monitor → Edit Channel → Permissions → add your bot.

### Step 3: Create the Edge Function

```bash
supabase functions new discord-capture
```

Open `supabase/functions/discord-capture/index.ts` and replace the contents with:

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

// --- Configuration ---
const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const OPENROUTER_API_KEY = Deno.env.get("OPENROUTER_API_KEY")!;
const DISCORD_PUBLIC_KEY = Deno.env.get("DISCORD_PUBLIC_KEY")!;
const DISCORD_BOT_TOKEN = Deno.env.get("DISCORD_BOT_TOKEN")!;

// Channel IDs to monitor — comma-separated list of Discord channel IDs
// Example: "1234567890,9876543210"
// Get channel IDs by right-clicking a channel in Discord with Developer Mode on
const DISCORD_CHANNEL_IDS_RAW = Deno.env.get("DISCORD_CHANNEL_IDS") || "";
const DISCORD_CHANNEL_IDS = DISCORD_CHANNEL_IDS_RAW.split(",").map((s) =>
  s.trim()
).filter(Boolean);

const OPENROUTER_BASE = "https://openrouter.ai/api/v1";
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

// --- Ed25519 Signature Verification ---
// Discord requires this. If the signature check fails, Discord will stop sending
// real events to your endpoint. This is security, not optional.
async function verifyDiscordRequest(
  body: string,
  signature: string,
  timestamp: string,
  publicKey: string,
): Promise<boolean> {
  try {
    // Convert the hex public key to a CryptoKey object
    const keyBytes = hexToUint8Array(publicKey);
    const cryptoKey = await crypto.subtle.importKey(
      "raw",
      keyBytes,
      { name: "Ed25519", namedCurve: "Ed25519" },
      false,
      ["verify"],
    );

    // The message that was signed is: timestamp + body (concatenated as bytes)
    const message = new TextEncoder().encode(timestamp + body);
    const signatureBytes = hexToUint8Array(signature);

    return await crypto.subtle.verify("Ed25519", cryptoKey, signatureBytes, message);
  } catch {
    return false;
  }
}

// Helper: convert a hex string ("deadbeef") to a Uint8Array
function hexToUint8Array(hex: string): Uint8Array {
  const bytes = new Uint8Array(hex.length / 2);
  for (let i = 0; i < hex.length; i += 2) {
    bytes[i / 2] = parseInt(hex.substring(i, i + 2), 16);
  }
  return bytes;
}

// --- Embedding ---
async function getEmbedding(text: string): Promise<number[]> {
  const r = await fetch(`${OPENROUTER_BASE}/embeddings`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/text-embedding-3-small",
      input: text,
    }),
  });
  const d = await r.json();
  return d.data[0].embedding;
}

// --- Metadata Extraction ---
async function extractMetadata(text: string): Promise<Record<string, unknown>> {
  const r = await fetch(`${OPENROUTER_BASE}/chat/completions`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/gpt-4o-mini",
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Extract metadata from the user's captured thought. Return JSON with:
- "people": array of people mentioned (empty array if none)
- "action_items": array of implied to-dos (empty array if none)
- "topics": array of 1-3 short topic tags (always at least one)
- "type": one of "observation", "task", "idea", "reference", "person_note"
Only extract what is explicitly present.`,
        },
        { role: "user", content: text },
      ],
    }),
  });
  const d = await r.json();
  try {
    return JSON.parse(d.choices[0].message.content);
  } catch {
    return { topics: ["uncategorized"], type: "observation" };
  }
}

// --- Discord Reply ---
// Sends a message to a Discord channel using the bot token
async function sendDiscordMessage(channelId: string, content: string): Promise<void> {
  await fetch(`https://discord.com/api/v10/channels/${channelId}/messages`, {
    method: "POST",
    headers: {
      "Authorization": `Bot ${DISCORD_BOT_TOKEN}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ content }),
  });
}

// --- Main Handler ---
Deno.serve(async (req: Request): Promise<Response> => {
  try {
    // Step 1: Read the raw body as text (needed for signature verification)
    const rawBody = await req.text();
    const signature = req.headers.get("x-signature-ed25519") || "";
    const timestamp = req.headers.get("x-signature-timestamp") || "";

    // Step 2: Verify the Discord signature
    // Discord will send a PING first to verify your endpoint before sending real events.
    // This verification must pass for Discord to trust your endpoint.
    const isValid = await verifyDiscordRequest(
      rawBody,
      signature,
      timestamp,
      DISCORD_PUBLIC_KEY,
    );

    if (!isValid) {
      return new Response("Invalid signature", { status: 401 });
    }

    // Step 3: Parse the verified body
    const body = JSON.parse(rawBody);

    // Step 4: Handle PING (Discord sends this when you first register the endpoint)
    // Must respond with type: 1 to pass verification
    if (body.type === 1) {
      return new Response(JSON.stringify({ type: 1 }), {
        headers: { "Content-Type": "application/json" },
      });
    }

    // Step 5: Handle incoming messages
    // Discord sends MESSAGE_CREATE events (type not present in webhook body —
    // this is a direct webhook, not a Gateway event)
    // For channel webhooks, the body IS the message object
    const messageContent: string = body.content || "";
    const channelId: string = body.channel_id || "";
    const authorUsername: string = body.author?.username || "Unknown";
    const isBot: boolean = body.author?.bot === true;

    // Guard: skip bots, empty messages, and unmonitored channels
    if (isBot || !messageContent.trim()) {
      return new Response("ok", { status: 200 });
    }

    // If we have a channel filter, only process messages from monitored channels
    if (DISCORD_CHANNEL_IDS.length > 0 && !DISCORD_CHANNEL_IDS.includes(channelId)) {
      return new Response("ok", { status: 200 });
    }

    // Step 6: Process the thought
    const [embedding, metadata] = await Promise.all([
      getEmbedding(messageContent),
      extractMetadata(messageContent),
    ]);

    const { error } = await supabase.from("thoughts").insert({
      content: messageContent,
      embedding,
      metadata: {
        ...metadata,
        source: "discord",
        channel_id: channelId,
        author: authorUsername,
      },
    });

    if (error) {
      console.error("Supabase insert error:", error);
      await sendDiscordMessage(channelId, `Failed to capture: ${error.message}`);
      return new Response("error", { status: 500 });
    }

    // Build confirmation reply
    const meta = metadata as Record<string, unknown>;
    let confirmation = `Captured as **${meta.type || "thought"}**`;
    if (Array.isArray(meta.topics) && meta.topics.length > 0) {
      confirmation += ` — ${meta.topics.join(", ")}`;
    }
    if (Array.isArray(meta.people) && meta.people.length > 0) {
      confirmation += `\nPeople: ${meta.people.join(", ")}`;
    }

    await sendDiscordMessage(channelId, confirmation);
    return new Response("ok", { status: 200 });
  } catch (err) {
    console.error("Function error:", err);
    return new Response("error", { status: 500 });
  }
});
```

### Step 4: Deploy and configure

**Enable Developer Mode in Discord** (needed to copy channel IDs):
Discord → User Settings → Advanced → Developer Mode: ON.

Now right-click any channel you want to monitor and click "Copy Channel ID".

**Set secrets:**

```bash
supabase secrets set DISCORD_PUBLIC_KEY=your-public-key-from-developer-portal
supabase secrets set DISCORD_BOT_TOKEN=your-bot-token-here
supabase secrets set DISCORD_CHANNEL_IDS=123456789012345678,987654321098765432
```

**Deploy:**

```bash
supabase functions deploy discord-capture --no-verify-jwt
```

**Register the webhook in Discord:**

1. Go to your Discord application in the Developer Portal.
2. In the left sidebar, click "General Information".
3. Find "Interactions Endpoint URL".
4. Paste your Edge Function URL: `https://YOUR_PROJECT_REF.supabase.co/functions/v1/discord-capture`
5. Click Save. Discord will send a PING immediately to verify. Your function handles this with the `type === 1` check. If Discord shows "All your changes were saved", verification passed.

### Data flow

```
+--------------------+
| You in Discord     |
| #capture channel   |
| "Team agreed to    |
|  delay launch"     |
+--------+-----------+
         |
         | Discord sends webhook HTTP POST
         | Headers include:
         |   X-Signature-Ed25519: abc123...
         |   X-Signature-Timestamp: 1234567890
         | Body: JSON message object
         v
+--------+-----------+
| discord-capture    |
| Edge Function      |
|                    |
| 1. Read raw body   |
| 2. Verify Ed25519  |
|    signature using |
|    Discord public  |
|    key             |
| 3. If invalid:     |
|    return 401      |
| 4. If PING:        |
|    return {type:1} |
| 5. Parse message   |
+--------+-----------+
         |
    +----+----+
    |         |
    v         v
+-------+  +----------+
| Embed |  | Extract  |
| text  |  | metadata |
+---+---+  +----+-----+
    |            |
    +----+-------+
         |
         v
+--------+-----------+
| thoughts table     |
| INSERT row         |
+--------+-----------+
         |
         v
+--------+-----------+
| Discord API        |
| POST /channels/    |
| {id}/messages      |
| "Captured as       |
|  observation —     |
|  planning, launch" |
+--------------------+
```

**Time to build:** ~45 minutes. **Difficulty:** Medium (signature verification adds complexity).

---

## 6. Email (via Forwarding)

Email capture has two fundamentally different approaches. Choose based on your situation.

### Approach A: Inbound email webhook (recommended)

Services like Mailgun, SendGrid, and Postmark can receive email sent to an address you control (like `capture@yourdomain.com`) and POST the parsed email content to a webhook URL — which can be your Edge Function.

```
+---------------------+
|  You send an email  |
|  To:                |
|  capture@           |
|  yourdomain.com     |
|                     |
|  Subject: Meeting   |
|  notes              |
|                     |
|  Body: We decided   |
|  to hire a PM...    |
+----------+----------+
           |
           | Email delivered to Mailgun/SendGrid/Postmark
           | These services receive email FOR your domain
           v
+----------+----------+
|  Inbound Email      |
|  Service            |
|  (Mailgun, etc.)    |
|                     |
|  Parses the email:  |
|  - From header      |
|  - Subject line     |
|  - Body text        |
|  - Attachments      |
+----------+----------+
           |
           | HTTP POST to your Edge Function URL
           | (form-encoded or JSON, depending on service)
           v
+----------+----------+
|  email-capture      |
|  Edge Function      |
|                     |
|  Combines subject + |
|  body into content  |
+----------+----------+
           |
      same pipeline
           |
           v
+----------+----------+
|  thoughts table     |
|  content: "Subject: |
|  Meeting notes\n    |
|  We decided to      |
|  hire a PM..."      |
+---------------------+
```

**Setup with Mailgun (free tier, 1000 emails/month):**

1. Create a free account at mailgun.com.
2. Add your domain under "Sending" → "Domains". Follow the DNS instructions (you add a few TXT and MX records in your domain registrar).
3. Under "Receiving" → "Routes", create a new route:
   - Expression type: Match recipient
   - Recipient: `capture@yourdomain.com`
   - Actions: Forward to `https://YOUR_PROJECT_REF.supabase.co/functions/v1/email-capture`
4. Deploy the Edge Function below.

**Edge Function code for email capture (Mailgun format):**

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const OPENROUTER_API_KEY = Deno.env.get("OPENROUTER_API_KEY")!;

const OPENROUTER_BASE = "https://openrouter.ai/api/v1";
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

async function getEmbedding(text: string): Promise<number[]> {
  const r = await fetch(`${OPENROUTER_BASE}/embeddings`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ model: "openai/text-embedding-3-small", input: text }),
  });
  const d = await r.json();
  return d.data[0].embedding;
}

async function extractMetadata(text: string): Promise<Record<string, unknown>> {
  const r = await fetch(`${OPENROUTER_BASE}/chat/completions`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/gpt-4o-mini",
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Extract metadata from the user's captured thought. Return JSON with:
- "people": array of people mentioned (empty array if none)
- "action_items": array of implied to-dos (empty array if none)
- "topics": array of 1-3 short topic tags (always at least one)
- "type": one of "observation", "task", "idea", "reference", "person_note"
Only extract what is explicitly present.`,
        },
        { role: "user", content: text },
      ],
    }),
  });
  const d = await r.json();
  try {
    return JSON.parse(d.choices[0].message.content);
  } catch {
    return { topics: ["uncategorized"], type: "observation" };
  }
}

Deno.serve(async (req: Request): Promise<Response> => {
  try {
    // Mailgun sends email data as multipart form data
    // Other services (SendGrid, Postmark) use slightly different formats
    // but all provide subject and body in some form
    const formData = await req.formData();

    const from = formData.get("from")?.toString() || "";
    const subject = formData.get("subject")?.toString() || "";
    // "body-plain" is the plain text version of the email body (no HTML)
    const bodyPlain = formData.get("body-plain")?.toString() || "";

    // Combine subject + body into the thought content
    // The subject often contains the core idea; the body has details
    const content = subject
      ? `Subject: ${subject}\n\n${bodyPlain}`
      : bodyPlain;

    if (!content.trim()) {
      return new Response("no content", { status: 200 });
    }

    const [embedding, metadata] = await Promise.all([
      getEmbedding(content),
      extractMetadata(content),
    ]);

    const { error } = await supabase.from("thoughts").insert({
      content,
      embedding,
      metadata: {
        ...metadata,
        source: "email",
        from,
        subject,
      },
    });

    if (error) {
      console.error("Supabase insert error:", error);
      return new Response("error", { status: 500 });
    }

    // Email capture does not send a reply — the delivery itself is confirmation
    return new Response("ok", { status: 200 });
  } catch (err) {
    console.error("Function error:", err);
    return new Response("error", { status: 500 });
  }
});
```

Deploy:

```bash
supabase functions new email-capture
# Replace index.ts with the code above
supabase functions deploy email-capture --no-verify-jwt
```

### Approach B: Gmail API polling

If you want to capture from an existing Gmail inbox rather than a dedicated forwarding address, the `recipes/email-history-import/` recipe uses the Gmail API. That recipe is designed for bulk historical import, but the same approach can be run on a schedule to pull new emails periodically.

```
+------------------+
|  Gmail Inbox     |
|  (existing email)|
+--------+---------+
         |
         | Gmail API (OAuth, polling)
         | Scheduled Edge Function runs
         | every N minutes
         v
+--------+---------+
|  email-poll      |
|  Edge Function   |
|                  |
|  GET /gmail/v1/  |
|  users/me/       |
|  messages        |
|  (filter by      |
|  label or query) |
+--------+---------+
         |
         v
    same pipeline
         |
         v
+--------+---------+
|  thoughts table  |
+------------------+
```

**Recommendation:** Use Approach A for ongoing capture. Set up a dedicated email address (`capture@yourdomain.com`) that you BCC on important emails or forward things to. It is simpler, has no OAuth complexity, and works from any email client.

Use Approach B (the `recipes/email-history-import/` recipe) if you want to import your historical email archive.

---

## 7. Obsidian (Local Vault)

Obsidian is fundamentally different from every other channel in this guide. Slack and Telegram are cloud services — when you send a message, it goes through their servers, which can then call your webhook. Obsidian is a local application. Your notes live on your hard drive. There is no Obsidian server to send a webhook.

This means the integration has to reach out from your machine rather than receive data from a server. Three approaches cover the full range of needs.

### Option A: Obsidian Community Plugin (recommended)

Obsidian has a plugin system. Plugins are JavaScript that runs inside Obsidian itself, with full access to your vault (files) and network. A custom plugin can watch for file saves and call Supabase directly — no Edge Function needed.

```
+-------------------+
|  Obsidian         |
|  (desktop app)    |
|                   |
|  You save a note  |
|  (Cmd+S or auto)  |
+--------+----------+
         |
         | Plugin's vault.on('modify') event fires
         | Plugin reads the file contents
         v
+--------+----------+
|  OpenBrainSync    |
|  Plugin           |
|  (runs inside     |
|  Obsidian)        |
|                   |
|  - Read markdown  |
|  - Call OpenRouter|
|    for embedding  |
|  - Insert into    |
|    Supabase REST  |
+--------+----------+
         |
         | Direct HTTP calls from your machine
         | (no Edge Function needed)
         v
+--------+----------+
|  thoughts table   |
|  via Supabase     |
|  REST API         |
+-------------------+
```

**Plugin skeleton (main.ts):**

This is a starting point, not a fully deployed plugin. Building a full Obsidian plugin requires Node.js, npm, and an esbuild step. The Obsidian developer documentation at docs.obsidian.md explains the build process.

```typescript
import { Plugin, TFile } from "obsidian";

// Simple debounce: wait 2 seconds after the last save before syncing.
// This prevents a flood of syncs if Obsidian auto-saves frequently.
function debounce<T extends (...args: unknown[]) => void>(fn: T, ms: number): T {
  let timer: ReturnType<typeof setTimeout>;
  return ((...args: unknown[]) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), ms);
  }) as T;
}

export default class OpenBrainSync extends Plugin {
  // These come from the plugin settings UI (not hardcoded)
  supabaseUrl = "";
  supabaseKey = ""; // Use the anon key here, not the service role key
  openrouterKey = "";

  async onload() {
    // Load settings (URL, keys) from Obsidian's data store
    await this.loadData().then((data) => {
      if (data) {
        this.supabaseUrl = data.supabaseUrl || "";
        this.supabaseKey = data.supabaseKey || "";
        this.openrouterKey = data.openrouterKey || "";
      }
    });

    // Watch for file modifications
    // debounce prevents syncing on every keystroke when auto-save is on
    const syncOnModify = debounce(async (file: TFile) => {
      if (!file.path.endsWith(".md")) return; // only markdown files
      if (file.path.startsWith("_templates/")) return; // skip template folder
      await this.syncFile(file);
    }, 2000);

    this.registerEvent(
      this.app.vault.on("modify", syncOnModify),
    );

    console.log("Open Brain Sync: loaded");
  }

  async syncFile(file: TFile) {
    if (!this.supabaseUrl || !this.supabaseKey || !this.openrouterKey) {
      console.warn("Open Brain Sync: missing credentials, skipping sync");
      return;
    }

    const content = await this.app.vault.read(file);
    if (!content.trim()) return;

    // Truncate very long notes to avoid token limits
    const truncated = content.slice(0, 8000);

    try {
      // Generate embedding
      const embeddingRes = await fetch(
        "https://openrouter.ai/api/v1/embeddings",
        {
          method: "POST",
          headers: {
            "Authorization": `Bearer ${this.openrouterKey}`,
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            model: "openai/text-embedding-3-small",
            input: truncated,
          }),
        },
      );
      const embeddingData = await embeddingRes.json();
      const embedding = embeddingData.data[0].embedding;

      // Upsert into thoughts table using file path as a stable identifier
      // This updates the row if the file was already synced, inserts if new
      const response = await fetch(
        `${this.supabaseUrl}/rest/v1/thoughts`,
        {
          method: "POST",
          headers: {
            "apikey": this.supabaseKey,
            "Authorization": `Bearer ${this.supabaseKey}`,
            "Content-Type": "application/json",
            "Prefer": "resolution=merge-duplicates",
          },
          body: JSON.stringify({
            content: truncated,
            embedding,
            metadata: {
              source: "obsidian",
              file_path: file.path,
              file_name: file.basename,
              synced_at: new Date().toISOString(),
            },
          }),
        },
      );

      if (response.ok) {
        console.log(`Open Brain Sync: synced ${file.path}`);
      } else {
        console.error(`Open Brain Sync: failed to sync ${file.path}`, await response.text());
      }
    } catch (err) {
      console.error("Open Brain Sync: error", err);
    }
  }
}
```

Note: The upsert approach using `Prefer: resolution=merge-duplicates` requires your thoughts table to have a unique constraint on a column that identifies the note. The simplest approach is to add a `source_id` column:

```sql
ALTER TABLE thoughts ADD COLUMN IF NOT EXISTS source_id TEXT;
CREATE UNIQUE INDEX IF NOT EXISTS thoughts_source_id_idx ON thoughts(source_id) WHERE source_id IS NOT NULL;
```

Then set `source_id: file.path` in the metadata, and adjust the insert to use `source_id` as the upsert key. The current skeleton stores things but may create duplicates on re-saves — acceptable for a prototype.

### Option B: File watcher script (no Obsidian dependency)

This approach does not require building an Obsidian plugin. It is a standalone Node.js script that you run in a terminal. It watches your vault folder for file changes using the built-in `fs.watch` API and syncs changed files to Supabase.

This works even if you do not use Obsidian — point it at any folder of markdown files.

```
+-------------------+
|  Your vault       |
|  ~/Documents/     |
|  ObsidianVault/   |
|                   |
|  note.md saved    |
+--------+----------+
         |
         | fs.watch() event fires in Node.js script
         | (running in a terminal on your machine)
         v
+--------+----------+
|  watch-vault.ts   |
|  (Node.js script) |
|                   |
|  - Read changed   |
|    file           |
|  - Embed via      |
|    OpenRouter     |
|  - Insert into    |
|    Supabase       |
+--------+----------+
         |
         v
+--------+----------+
|  thoughts table   |
+-------------------+
```

**Complete file watcher script (`watch-vault.ts`):**

```typescript
import { watch } from "fs";
import { readFile } from "fs/promises";
import { join, extname, basename } from "path";
import { createClient } from "@supabase/supabase-js";

// --- Configuration ---
// Set these as environment variables before running:
//   export SUPABASE_URL=https://your-ref.supabase.co
//   export SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
//   export OPENROUTER_API_KEY=your-openrouter-key
//   export VAULT_PATH=/Users/yourname/Documents/ObsidianVault
const SUPABASE_URL = process.env.SUPABASE_URL!;
const SUPABASE_SERVICE_ROLE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY!;
const OPENROUTER_API_KEY = process.env.OPENROUTER_API_KEY!;
const VAULT_PATH = process.env.VAULT_PATH!;

if (!SUPABASE_URL || !SUPABASE_SERVICE_ROLE_KEY || !OPENROUTER_API_KEY || !VAULT_PATH) {
  console.error("Missing required environment variables. Set SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, OPENROUTER_API_KEY, and VAULT_PATH.");
  process.exit(1);
}

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

// Debounce: wait 2 seconds after the last change before syncing
// File saves often trigger multiple fs events in quick succession
const pending = new Map<string, ReturnType<typeof setTimeout>>();
function debounceSync(filePath: string) {
  if (pending.has(filePath)) {
    clearTimeout(pending.get(filePath)!);
  }
  pending.set(
    filePath,
    setTimeout(async () => {
      pending.delete(filePath);
      await syncFile(filePath);
    }, 2000),
  );
}

async function getEmbedding(text: string): Promise<number[]> {
  const r = await fetch("https://openrouter.ai/api/v1/embeddings", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/text-embedding-3-small",
      input: text.slice(0, 8000), // stay within token limits
    }),
  });
  const d = await r.json();
  return d.data[0].embedding;
}

async function syncFile(filePath: string) {
  // Only process markdown files
  if (extname(filePath) !== ".md") return;

  let content: string;
  try {
    content = await readFile(filePath, "utf-8");
  } catch {
    // File was deleted — skip
    return;
  }

  if (!content.trim()) return;

  console.log(`Syncing: ${filePath}`);

  try {
    const embedding = await getEmbedding(content);

    const { error } = await supabase.from("thoughts").insert({
      content: content.slice(0, 8000),
      embedding,
      metadata: {
        source: "obsidian",
        file_path: filePath,
        file_name: basename(filePath, ".md"),
        synced_at: new Date().toISOString(),
      },
    });

    if (error) {
      console.error(`Failed to sync ${filePath}:`, error.message);
    } else {
      console.log(`Synced: ${basename(filePath)}`);
    }
  } catch (err) {
    console.error(`Error syncing ${filePath}:`, err);
  }
}

// Watch the vault directory recursively
console.log(`Watching vault at: ${VAULT_PATH}`);
watch(VAULT_PATH, { recursive: true }, (_event, filename) => {
  if (!filename) return;
  debounceSync(join(VAULT_PATH, filename));
});

console.log("Watcher running. Press Ctrl+C to stop.");
```

**Run it:**

```bash
# Install dependencies
npm install @supabase/supabase-js

# Set environment variables (Mac/Linux)
export SUPABASE_URL=https://your-ref.supabase.co
export SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
export OPENROUTER_API_KEY=your-openrouter-key
export VAULT_PATH=/Users/yourname/Documents/ObsidianVault

# Run with ts-node (install: npm install -g ts-node typescript)
ts-node watch-vault.ts
```

The script runs as long as your terminal is open. To run it continuously in the background, use a tool like `pm2` (`npm install -g pm2`) or add it to your system's startup items.

### Option C: One-time import via companion prompts

If you do not want to set up a watcher and just want to migrate existing notes, use Prompt 2 (Second Brain Migration) from `docs/02-companion-prompts.md`.

The workflow:
1. Open Obsidian. Open a note or set of notes you want to migrate.
2. Select all content (Cmd+A), copy.
3. Open an MCP-connected AI (Claude Desktop, ChatGPT with Open Brain MCP, etc.).
4. Paste Prompt 2 from `docs/02-companion-prompts.md`, then paste your note content.
5. The AI transforms your notes into standalone thoughts and saves them using `capture_thought`.

```
+-------------------+
|  Your Obsidian    |
|  vault            |
|                   |
|  Select notes,    |
|  copy contents    |
+--------+----------+
         |
         | Paste into MCP-connected AI
         v
+--------+----------+
|  Claude / ChatGPT |
|  (with Open Brain |
|   MCP connected)  |
|                   |
|  Prompt 2:        |
|  "Second Brain    |
|   Migration"      |
|                   |
|  AI parses your   |
|  markdown, breaks |
|  it into thoughts,|
|  saves each one   |
+--------+----------+
         |
         | capture_thought MCP tool
         v
+--------+----------+
|  thoughts table   |
+-------------------+
```

This approach has no setup cost. The downside is it is manual and does not sync future changes. Use it for a one-time migration, then switch to Option A or B for ongoing capture.

---

## 8. WhatsApp (Business API)

WhatsApp is included here for completeness, but it is the hardest integration to build and not recommended for personal use unless you already have Meta Business infrastructure.

The fundamental constraint: WhatsApp does not offer a simple personal bot API. The only officially supported way to receive webhook messages is through the WhatsApp Business Platform, which requires:

1. A Meta Business account (verified)
2. A dedicated phone number (cannot be a number already using WhatsApp)
3. A Meta app with WhatsApp API access
4. App review for production use

The technical pattern is the same as every other channel once you have access:

```
+-------------------+
|  Someone sends    |
|  you a WhatsApp   |
|  message          |
+--------+----------+
         |
         | Meta Cloud API sends webhook
         | (requires Meta Business account,
         |  phone number, and app review)
         v
+--------+----------+
|  whatsapp-capture |
|  Edge Function    |
|                   |
|  Verifies Meta    |
|  webhook token    |
|  Parses message   |
|  object format    |
+--------+----------+
         |
         v
    same pipeline
         |
         v
+--------+----------+
|  thoughts table   |
+-------------------+
```

**If you already have Meta Business access**, the technical setup is:

1. Create a Meta App at developers.facebook.com. Add the "WhatsApp" product.
2. Configure a phone number (or use the test number Meta provides).
3. Under Webhooks, subscribe to the `messages` event. Point it at your Edge Function URL.
4. Set a verify token (any string you choose) — Meta sends this in the initial verification request. Your function must respond with the `hub.challenge` value if the token matches.
5. The message format is deeply nested JSON. The actual text is at `body.entry[0].changes[0].value.messages[0].text.body`.

For personal use, consider Telegram instead. It has no restrictions, works with a personal account, takes 15 minutes to set up, and the user experience is essentially identical.

---

## 9. SMS via Twilio

Twilio is the simplest way to give Open Brain a phone number. You text it, it captures the thought. No smartphone app required — any phone can use it.

```
+-------------------+
|  You send an SMS  |
|  to your Twilio   |
|  number           |
|  +1 555-123-4567  |
+--------+----------+
         |
         | Twilio receives the SMS
         | Sends HTTP POST to your webhook URL
         | Body is form-encoded (not JSON):
         | Body=Sarah+mentioned+she+wants...
         | &From=%2B15551234567
         | &To=%2B15557654321
         v
+--------+----------+
|  sms-capture      |
|  Edge Function    |
|                   |
|  Parses           |
|  URL-encoded form |
|  data             |
+--------+----------+
         |
         v
    same pipeline
         |
         v
+--------+----------+
|  thoughts table   |
+--------+----------+
         |
         v
+--------+----------+
|  Twilio API       |
|  Reply via TwiML  |
|  "Captured as     |
|   person_note"    |
+-------------------+
```

### Setup

1. Create a Twilio account at twilio.com. The free trial gives you $15 credit — enough for hundreds of SMS messages.
2. Under Phone Numbers → Manage → Buy a number, get a number with SMS capability. Cost: ~$1/month.
3. Deploy the Edge Function below.
4. In your Twilio phone number settings, under "Messaging" → "A message comes in", paste your Edge Function URL. Set the method to HTTP POST.

**Edge Function code:**

```typescript
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const OPENROUTER_API_KEY = Deno.env.get("OPENROUTER_API_KEY")!;

const OPENROUTER_BASE = "https://openrouter.ai/api/v1";
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

async function getEmbedding(text: string): Promise<number[]> {
  const r = await fetch(`${OPENROUTER_BASE}/embeddings`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ model: "openai/text-embedding-3-small", input: text }),
  });
  const d = await r.json();
  return d.data[0].embedding;
}

async function extractMetadata(text: string): Promise<Record<string, unknown>> {
  const r = await fetch(`${OPENROUTER_BASE}/chat/completions`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/gpt-4o-mini",
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Extract metadata from the user's captured thought. Return JSON with:
- "people": array of people mentioned (empty array if none)
- "action_items": array of implied to-dos (empty array if none)
- "topics": array of 1-3 short topic tags (always at least one)
- "type": one of "observation", "task", "idea", "reference", "person_note"
Only extract what is explicitly present.`,
        },
        { role: "user", content: text },
      ],
    }),
  });
  const d = await r.json();
  try {
    return JSON.parse(d.choices[0].message.content);
  } catch {
    return { topics: ["uncategorized"], type: "observation" };
  }
}

Deno.serve(async (req: Request): Promise<Response> => {
  try {
    // Twilio sends data as URL-encoded form (not JSON)
    // "Body=Hello+World&From=%2B15551234567&To=%2B15557654321"
    const formText = await req.text();
    const params = new URLSearchParams(formText);

    const smsBody = params.get("Body") || "";
    const fromNumber = params.get("From") || "";

    if (!smsBody.trim()) {
      // Respond with empty TwiML (Twilio requires a valid XML response)
      return new Response("<Response></Response>", {
        headers: { "Content-Type": "text/xml" },
      });
    }

    const [embedding, metadata] = await Promise.all([
      getEmbedding(smsBody),
      extractMetadata(smsBody),
    ]);

    const { error } = await supabase.from("thoughts").insert({
      content: smsBody,
      embedding,
      metadata: {
        ...metadata,
        source: "sms",
        from_number: fromNumber,
      },
    });

    if (error) {
      console.error("Supabase insert error:", error);
      // Respond with TwiML that sends an error SMS back
      return new Response(
        `<Response><Message>Failed to capture: ${error.message}</Message></Response>`,
        { headers: { "Content-Type": "text/xml" } },
      );
    }

    // Build confirmation
    const meta = metadata as Record<string, unknown>;
    let replyText = `Captured as ${meta.type || "thought"}`;
    if (Array.isArray(meta.topics) && meta.topics.length > 0) {
      replyText += ` (${meta.topics.join(", ")})`;
    }
    if (Array.isArray(meta.people) && meta.people.length > 0) {
      replyText += `. People: ${meta.people.join(", ")}`;
    }

    // TwiML response: Twilio reads this XML and sends the SMS reply
    return new Response(
      `<Response><Message>${replyText}</Message></Response>`,
      { headers: { "Content-Type": "text/xml" } },
    );
  } catch (err) {
    console.error("Function error:", err);
    return new Response("<Response></Response>", {
      headers: { "Content-Type": "text/xml" },
      status: 500,
    });
  }
});
```

Note the key difference from other channels: Twilio sends form-encoded data (`Body=text&From=number`), not JSON. The response must be XML in a format called TwiML (Twilio Markup Language) — Twilio reads this XML to decide what to do next, including sending the reply SMS.

**Deploy:**

```bash
supabase functions new sms-capture
# Replace index.ts with the code above
supabase functions deploy sms-capture --no-verify-jwt
```

**Cost:** ~$1/month for the phone number. Incoming SMS on Twilio trial are free until your trial credit runs out; paid plans are ~$0.0079/SMS.

---

## 10. Channel Comparison

| Channel | Difficulty | Time to Build | Monthly Cost | Uses Webhook | Code Status |
|---------|------------|---------------|--------------|--------------|-------------|
| Telegram | Easiest | 15 min | Free | Yes | Complete (this guide) |
| Slack | Easy | 30 min | Free | Yes | Complete (`integrations/slack-capture/`) |
| SMS (Twilio) | Easy | 30 min | ~$1 + usage | Yes | Complete (this guide) |
| Discord | Medium | 45 min | Free | Yes (Ed25519) | Complete (this guide) |
| Obsidian (watcher) | Easy | 30 min | Free | No (file watch) | Complete (this guide) |
| Email (forwarding) | Medium | 60 min | Free–$5/mo | Via service | Complete (this guide) |
| Obsidian (plugin) | Medium | 2+ hours | Free | No (local) | Skeleton (this guide) |
| WhatsApp | Hard | Hours + review | Free (limits) | Yes | No (overview only) |

**Recommendation for most people:** Start with Telegram (15 minutes, works from your phone, zero cost) or Slack (30 minutes if you already use Slack). Both give you the same core experience. Add other channels as your workflow demands them.

---

## 11. Building Your Own Integration (The Template)

Every channel integration in this guide is a variation of the same template. The core pipeline — embed, extract metadata, insert — is identical. The only things that differ:

1. How you parse the incoming message (different JSON/form structure per channel)
2. How you send the confirmation reply (different API per channel)
3. Whether you need any special verification (only Discord requires Ed25519)

Here is the complete reusable template. Copy it, change `parseIncomingMessage()` and `sendConfirmation()`, and you have a new integration in 15 minutes.

```typescript
// =============================================================================
// OPEN BRAIN CHANNEL INTEGRATION TEMPLATE
// =============================================================================
// Copy this file to create a new channel integration.
// You only need to change two things:
//   1. parseIncomingMessage() — extract text + sender from your channel's JSON
//   2. sendConfirmation()    — send a reply using your channel's API
//
// Everything else (embedding, metadata, Supabase insert) stays the same.
// =============================================================================

import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

// --- Standard configuration (same for every integration) ---
const SUPABASE_URL = Deno.env.get("SUPABASE_URL")!;
const SUPABASE_SERVICE_ROLE_KEY = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;
const OPENROUTER_API_KEY = Deno.env.get("OPENROUTER_API_KEY")!;

// --- Channel-specific secrets (add your own here) ---
// Example: const MY_CHANNEL_TOKEN = Deno.env.get("MY_CHANNEL_TOKEN")!;

const OPENROUTER_BASE = "https://openrouter.ai/api/v1";
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

// =============================================================================
// CHANGE THIS FUNCTION
// =============================================================================
// Input:  The raw request from your channel's webhook
// Output: { text: string, sender: string, replyTarget: string }
//         text        — the message content to store as a thought
//         sender      — who sent the message (for metadata)
//         replyTarget — channel ID, chat ID, phone number, etc. needed to reply
//                       (can be empty string if your channel does not support replies)
// Return null to ignore the message (bots, edits, empty messages, etc.)
async function parseIncomingMessage(
  req: Request,
): Promise<{ text: string; sender: string; replyTarget: string } | null> {
  // EXAMPLE: Generic JSON webhook
  // Replace this with your channel's actual format
  const body = await req.json();

  const text = body.message?.text || "";
  const sender = body.message?.from?.name || "Unknown";
  const replyTarget = body.message?.chat_id?.toString() || "";

  if (!text.trim()) return null;

  return { text, sender, replyTarget };
}

// =============================================================================
// CHANGE THIS FUNCTION
// =============================================================================
// Sends a confirmation reply to the user in your channel.
// If your channel does not support replies, make this a no-op:
//   async function sendConfirmation(_target: string, _message: string) {}
async function sendConfirmation(
  replyTarget: string,
  message: string,
): Promise<void> {
  // EXAMPLE: Generic HTTP reply
  // Replace with your channel's API call

  // const MY_CHANNEL_TOKEN = Deno.env.get("MY_CHANNEL_TOKEN")!;
  // await fetch("https://api.mychannel.com/send", {
  //   method: "POST",
  //   headers: {
  //     "Authorization": `Bearer ${MY_CHANNEL_TOKEN}`,
  //     "Content-Type": "application/json",
  //   },
  //   body: JSON.stringify({ target: replyTarget, text: message }),
  // });

  console.log(`Reply to ${replyTarget}: ${message}`);
}

// =============================================================================
// DO NOT CHANGE BELOW THIS LINE
// This is the standard pipeline: embed → extract → insert → confirm
// =============================================================================

async function getEmbedding(text: string): Promise<number[]> {
  const r = await fetch(`${OPENROUTER_BASE}/embeddings`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/text-embedding-3-small",
      input: text,
    }),
  });
  const d = await r.json();
  return d.data[0].embedding;
}

async function extractMetadata(text: string): Promise<Record<string, unknown>> {
  const r = await fetch(`${OPENROUTER_BASE}/chat/completions`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/gpt-4o-mini",
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Extract metadata from the user's captured thought. Return JSON with:
- "people": array of people mentioned (empty array if none)
- "action_items": array of implied to-dos (empty array if none)
- "topics": array of 1-3 short topic tags (always at least one)
- "type": one of "observation", "task", "idea", "reference", "person_note"
Only extract what is explicitly present.`,
        },
        { role: "user", content: text },
      ],
    }),
  });
  const d = await r.json();
  try {
    return JSON.parse(d.choices[0].message.content);
  } catch {
    return { topics: ["uncategorized"], type: "observation" };
  }
}

function buildConfirmationText(
  metadata: Record<string, unknown>,
): string {
  let msg = `Captured as ${metadata.type || "thought"}`;
  if (Array.isArray(metadata.topics) && metadata.topics.length > 0) {
    msg += ` — ${metadata.topics.join(", ")}`;
  }
  if (Array.isArray(metadata.people) && metadata.people.length > 0) {
    msg += `. People: ${metadata.people.join(", ")}`;
  }
  if (Array.isArray(metadata.action_items) && metadata.action_items.length > 0) {
    msg += `. Actions: ${metadata.action_items.join("; ")}`;
  }
  return msg;
}

Deno.serve(async (req: Request): Promise<Response> => {
  try {
    // Step 1: Parse the incoming message (channel-specific)
    const parsed = await parseIncomingMessage(req);
    if (!parsed) {
      return new Response("ok", { status: 200 });
    }
    const { text, sender, replyTarget } = parsed;

    // Step 2: Embed and extract metadata in parallel (standard pipeline)
    const [embedding, metadata] = await Promise.all([
      getEmbedding(text),
      extractMetadata(text),
    ]);

    // Step 3: Insert into thoughts table (standard pipeline)
    const { error } = await supabase.from("thoughts").insert({
      content: text,
      embedding,
      metadata: {
        ...metadata,
        // CHANGE: update "my_channel" to your channel name
        source: "my_channel",
        sender,
      },
    });

    if (error) {
      console.error("Supabase insert error:", error);
      if (replyTarget) {
        await sendConfirmation(replyTarget, `Failed to capture: ${error.message}`);
      }
      return new Response("error", { status: 500 });
    }

    // Step 4: Send confirmation reply (channel-specific)
    if (replyTarget) {
      await sendConfirmation(replyTarget, buildConfirmationText(metadata));
    }

    return new Response("ok", { status: 200 });
  } catch (err) {
    console.error("Function error:", err);
    return new Response("error", { status: 500 });
  }
});
```

### How to use this template

1. Copy this file to `supabase/functions/YOUR-CHANNEL-capture/index.ts`
2. Implement `parseIncomingMessage()` for your channel's JSON format. Find this by looking at your channel's webhook documentation — they all show you example payloads.
3. Implement `sendConfirmation()` using your channel's API. Usually this is one `fetch()` call with a bot token.
4. Add any channel-specific secrets to the secrets block at the top.
5. Deploy:
   ```bash
   supabase functions new YOUR-CHANNEL-capture
   supabase secrets set YOUR_CHANNEL_TOKEN=...
   supabase functions deploy YOUR-CHANNEL-capture --no-verify-jwt
   ```
6. Register the webhook URL with your channel's developer console.

That is the entire process. The embedding, metadata extraction, and Supabase insert are already written and tested. You are only filling in the two channel-specific functions.

### Example: Adding a new channel in 15 minutes

Suppose you want to capture from a hypothetical service called "Lark" that sends webhooks like this:

```json
{
  "event": {
    "message": {
      "content": "{\"text\": \"Sarah mentioned consulting\"}",
      "sender": { "name": "Nate" },
      "chat_id": "oc_abc123"
    }
  }
}
```

Your `parseIncomingMessage()` would look like:

```typescript
async function parseIncomingMessage(req: Request) {
  const body = await req.json();
  const msg = body.event?.message;
  if (!msg) return null;

  // Lark wraps the content as a JSON string inside the JSON
  let text = "";
  try {
    text = JSON.parse(msg.content).text || "";
  } catch {
    return null;
  }

  return {
    text,
    sender: msg.sender?.name || "Unknown",
    replyTarget: msg.chat_id || "",
  };
}
```

Everything else is inherited from the template. Lark integration: done.

---

*Part of the [Open Brain project](https://github.com/NateBJones-Projects/OB1). Built by the community. See `CONTRIBUTING.md` to submit a new integration.*
