# Open Brain Quick Start

Your Open Brain is set up and connected. Now let's make sure it works — and get you your first real wins in the next five minutes.

> **Not set up yet?** This guide assumes you have a working Open Brain (database + MCP server + AI client connected). If you have not done that yet, start with the [Setup Guide](01-getting-started.md).

> **Using the CLI-Direct approach?** If you set up Open Brain with the [`ob` CLI tool](../resources/ob-cli/) instead of MCP, this guide still applies — just use `ob capture`, `ob search`, `ob recent`, and `ob stats` commands instead of asking your AI to use MCP tools. See [CLI-Direct Approach](CLI_DIRECT_APPROACH.md) for details.

---

## Prerequisites

Before you start, confirm you have:

- A Supabase project with the `thoughts` table created
- Your MCP server deployed and running
- An AI client (Claude Desktop, ChatGPT, or similar) connected to your Open Brain via MCP

If any of those are not in place, the [Setup Guide](01-getting-started.md) covers each one step by step.

---

## Your First 5 Minutes

Work through these ten steps in order. Each one takes 30-60 seconds. By the end, you will have a working Open Brain with real data, and you will have experienced the core features that make it useful.

---

### Step 1: Capture a Test Thought

Open your connected AI and type exactly this:

```
Remember this: Sarah mentioned she's thinking about leaving her job to start a consulting business
```

**What to expect:** Your AI confirms the save and shows extracted metadata — something like: type "person_note," people ["Sarah"], topics ["career," "consulting"], action items [].

**If nothing happens:** Try being more explicit: "Use the Open Brain capture_thought tool to remember this: Sarah mentioned she's thinking about leaving her job to start a consulting business." If that still does not work, check that your MCP connector is enabled for this conversation (look for a connectors or tools panel in your AI client).

---

### Step 2: Verify in Supabase

Open your Supabase dashboard in a browser. Go to Table Editor in the left sidebar, then click on the `thoughts` table.

**What to check:** You should see one row. The `content` column holds your text. The `embedding` column holds a long array of numbers (this is the AI's understanding of meaning — you never need to touch it). The `metadata` column holds the structured JSON with the type, people, topics, and action items your AI extracted.

**Why this matters:** Seeing the row in Supabase confirms end-to-end: your AI captured the thought, the MCP server processed it, the embedding was generated, and everything landed in the database. The system is working.

---

### Step 3: Search for Your Thought

Back in your AI, type:

```
What did I capture about career changes?
```

**What to expect:** Your AI retrieves the Sarah thought — even though the words "career changes" never appeared in what you typed. You searched by meaning, not keywords.

**Why this is the key feature:** This is semantic search. It is what makes Open Brain different from a notes app. You will not always remember the exact words you used when you captured something. Semantic search finds it by meaning anyway.

---

### Step 4: Capture a Meeting Note

Type this (replace the specifics if you prefer something from your own work):

```
Meeting with design team about the dashboard redesign. Decided to cut the sidebar panels, keep the revenue chart, and add a trend line. Action: I send them the API spec by Thursday. Action: they send revised mockups by Monday.
```

**What to expect:** Your AI confirms the save. The metadata should extract: multiple action items (send API spec, send mockups), people involved, and topics (dashboard, redesign).

**Why this format works:** Meeting notes with decisions and action items are the highest-value things you can save. They are dense with searchable meaning and they capture things that would otherwise fade within a day.

---

### Step 5: Capture a Person Note

Type this (again, feel free to use someone from your own work):

```
Marcus — mentioned he's overwhelmed since the reorg. Wants to move to the platform team. His wife just had a baby.
```

**What to expect:** Your AI saves this and tags it as a person note with Marcus extracted as the key person.

**Why leading with a name helps:** Starting with someone's name gives the metadata extraction a strong signal about what kind of thought this is. The next time you have a meeting with Marcus, you can ask "what do I know about Marcus?" and this surfaces immediately.

---

### Step 6: Search Across Your Captures

You now have at least three thoughts saved. Try a broader search:

```
What have I captured about people and their career situations?
```

**What to expect:** Your AI should retrieve both the Sarah thought (consulting business) and possibly the Marcus thought (reorg, platform team). These are thematically related even though the word "career" only appeared in your search query, not in either captured thought.

Now try a more specific search:

```
What action items do I have outstanding?
```

**What to expect:** Your AI should surface the action items from the meeting note — send the API spec, send mockups — with context about when they were captured.

---

### Step 7: Check Your Stats

Type this:

```
How many thoughts do I have in my Open Brain? Give me an overview.
```

**What to expect:** Your AI calls the stats tool and returns a count of your thoughts, when the first one was captured, and possibly a breakdown by type (person notes, meeting notes, insights, etc.).

**What this tells you:** Stats are useful as your brain grows. Checking them periodically gives you a sense of what kinds of things you capture most and where gaps might be.

---

### Step 8: Try the Memory Migration Prompt

This step is the highest-return thing you can do after getting set up. If you have been using Claude or ChatGPT for a while, those AIs have built up a picture of who you are — your role, projects, colleagues, preferences, past decisions. That context is locked inside their memory. The Memory Migration extracts it and saves it to your Open Brain so it is accessible from every AI you use.

Copy the Memory Migration prompt from [docs/02-companion-prompts.md](02-companion-prompts.md) — it is Prompt 1 in that file. Paste it into whichever AI has the most memory of you. Follow the steps it describes.

**What to expect:** The AI will show you a list of everything it knows about you, organized by category (people, projects, preferences, decisions). You approve the list and it saves each item to your Open Brain. At the end you will have dozens of thoughts already in your brain — a real foundation instead of a mostly-empty database.

**If you have memory in multiple AI tools:** Run it once per tool. Each platform has accumulated different context.

---

### Step 9: Explore Extension 1

When you are ready to go further, try the first extension: [Household Knowledge Base](../extensions/household-knowledge/).

It adds a separate table for household facts — paint colors, appliance model numbers, vendor contacts, measurements, warranty information. The setup takes about 20 minutes and follows the same pattern as the core Open Brain setup.

After building it, try:

```
Add a household item: living room paint is Sherwin Williams Sea Salt SW 6204
```

Then later:

```
What paint color is in the living room?
```

Your AI retrieves it instantly. No more standing in the paint store trying to remember.

Extension 1 is designed as a learning exercise as much as a useful tool. Building it teaches you the patterns used across all six extensions — if you understand Extension 1, you can understand and extend any of them.

---

### Step 10: Join the Community

If you run into something unexpected, want to share something you built, or just want to see what other people are doing with their Open Brains — join the [Open Brain Discord](https://discord.gg/Cgh9WJEkeG).

The `#help` channel is active and the questions you are likely to hit in the first week are probably already answered there. The `#show-and-tell` channel is where people share extensions, recipes, and creative uses of the system.

You also have access to dedicated AI assistants that know this system in depth — links in [docs/02-companion-prompts.md](02-companion-prompts.md). If you prefer to debug in natural conversation rather than read documentation, those assistants are faster than searching through files.

---

## You Are Up and Running

At this point you have:

- Captured your first thoughts (a personal note, a meeting note, a person note)
- Verified the database is storing them correctly
- Searched by meaning and seen semantic search working
- Checked your stats
- Run (or read about) the Memory Migration
- Looked at Extension 1

The next step is to build the capture habit. The system gets more useful the more you put in it — but it compounds. Twenty good thoughts is genuinely useful. Two hundred is transformative.

The [User Guide](USER_GUIDE.md) covers the full picture of what Open Brain is, how it works, and what you can do with it. The [companion prompts](02-companion-prompts.md) have the templates and rituals that make the daily habit click. The [FAQ](03-faq.md) has answers to the questions you are most likely to hit in the first few weeks.

Good luck. The system is yours now.

---

*Built by Nate B. Jones — companion to [Build Your Open Brain](01-getting-started.md).*
