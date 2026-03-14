# Open Brain User Guide

A plain-language guide to understanding and using your Open Brain — written for people who want to get value out of the system, not people who want to understand how it was built.

---

## Getting Started

### What Is Open Brain?

Open Brain is your AI's memory — one shared memory that works across all your AI tools.

Here is the problem it solves: every time you start a new conversation with an AI, it knows nothing about you. You explain context from scratch. You re-describe your project, your team, your decisions. The AI gives you great answers, and then that conversation closes and everything is gone. The next AI you talk to starts at zero again.

Open Brain fixes that. It is a database that sits alongside all your AI tools. When you want to save something, you tell your AI "remember this" and it saves the thought to your database — with your AI's understanding of its meaning built in. When you ask a question later, any AI connected to Open Brain can search that database and pull back what is relevant. Switch tools, start a new conversation, come back a week later: your context is there.

Think of it as giving all your AI tools a shared brain that never forgets.

### What You Need

Three things, all free to start:

- **A Supabase account** — This is where your thoughts are stored. Supabase is a database service; think of it as the filing cabinet. Free tier covers normal use easily.
- **An OpenRouter account** — This is what helps your AI understand meaning (not just keywords) when it reads and searches your thoughts. Free to sign up; you add a small amount of credit that lasts months.
- **An AI tool** — Claude Desktop, ChatGPT (paid plan), Claude Code, Cursor, or any tool that supports MCP connections. You probably already have one.

If you have not set these up yet, start with the [Setup Guide](01-getting-started.md). It walks you through the whole process step by step with nothing assumed.

---

## Understanding Your Open Brain

### The Thoughts Table

Every thought you save becomes one row in a database table called `thoughts`. Each row holds three things:

- **The text** — exactly what you captured, word for word
- **An embedding** — a mathematical representation of the *meaning* of your thought (1,536 numbers that describe what the text is about, generated automatically when you save)
- **Metadata** — structured information automatically extracted from your text: what type of thought it is, who is mentioned, what topics appear, what dates are referenced, and what action items are present

You never touch any of this directly. You type something to your AI, your AI saves it, and all three pieces are created automatically.

### Semantic Search

When you search your Open Brain, you are not searching for keywords. You are searching by *meaning*.

This is the difference: if you saved "Sarah mentioned she is thinking about leaving her job to start a consulting business," a keyword search for "career changes" would find nothing because those words do not appear in your note. Semantic search finds it anyway, because it understands that consulting and career changes mean related things.

This is why capturing naturally worded thoughts works better than trying to tag or categorize everything yourself. The meaning is already in the text.

### MCP: How Your AI Connects

MCP (Model Context Protocol) is the standard that lets AI tools communicate with external systems like your database. When your AI is connected to Open Brain via MCP, it gains access to four tools: one to save thoughts, one to search by meaning, one to browse recent captures, and one to check stats.

You do not need to understand MCP to use Open Brain. Once it is set up, it works invisibly. Your AI talks to your database; you just talk to your AI.

### CLI-Direct: An Alternative to MCP

If you use terminal-based AI tools (Claude Code, OpenAI Codex, Gemini CLI), you can skip MCP entirely and use the [`ob` CLI tool](../resources/ob-cli/) instead. It talks directly to Supabase using `curl` and `jq` — no Edge Function, no MCP server, no deployment step. Commands like `ob capture`, `ob search`, `ob recent`, and `ob stats` do the same thing as the MCP tools. Both approaches use the same database and are fully interoperable. See the [CLI-Direct Approach](CLI_DIRECT_APPROACH.md) for the full guide.

---

## 10 Simple Use Cases (Quick Wins)

These ten captures will get you comfortable with the system and show you what it can do. Work through them in order the first time.

### 1. Save Your First Thought

Type this into any AI connected to your Open Brain:

```
Remember this: Sarah mentioned she's thinking about leaving her job to start a consulting business
```

Your AI should confirm the save and show you extracted metadata — the type of thought, people mentioned (Sarah), and topics. Then open your Supabase dashboard, go to Table Editor, and click on the `thoughts` table. You should see one row with your text, a long embedding value, and metadata. That is your first thought, stored and searchable.

### 2. Search by Meaning

Now try finding that thought without using the same words:

```
What did I capture about career changes?
```

Your AI should retrieve the Sarah thought even though "career changes" was not in what you typed. This is semantic search working. The system matched the meaning of your question to the meaning of your captured thought.

### 3. Save a Meeting Note

After any meeting, capture the key points while they are fresh:

```
Meeting with design team: decided to cut the sidebar, keep the dashboard charts, and add dark mode. Action: I send mockups by Friday.
```

Your AI will extract the participants, decisions made, and the action item (send mockups by Friday). Meeting notes like this are among the highest-value things you can save — dense with people, decisions, and follow-ups.

### 4. Log a Decision

Decisions fade fast. Capture them with context:

```
Decision: Moving the launch to March 15. Context: QA found blockers in payment flow. Owner: Rachel.
```

Three months from now, when someone asks why the launch date moved, your Open Brain has the answer — and who made the call.

### 5. Save a Person Note

Capture context about people you work with:

```
Marcus — mentioned he's overwhelmed since the reorg. Wants to move to the platform team. His wife just had a baby.
```

Leading with someone's name helps your system tag this as a person note and extract Marcus as a key person. The next time you talk to Marcus or prepare for a meeting with him, you can ask "what do I know about Marcus?" and this context surfaces immediately.

### 6. Save AI Output

Good AI output disappears into chat history. Save the things worth keeping:

```
Saving from Claude: Framework for evaluating vendors — score on integration effort (40%), maintenance burden (30%), switching cost (30%)
```

Starting with "Saving from [AI tool]" creates a natural reference classification. Now this framework is searchable from any AI you use, not locked inside one chat window.

### 7. Capture an Insight

Ideas triggered by something you observed or read are worth capturing with their context:

```
Insight: Our onboarding flow assumes users already understand permissions. Triggered by watching a new hire struggle for 20 minutes.
```

Including what triggered the insight gives the embedding richer meaning and helps you remember the original context months later.

### 8. Experience Cross-Tool Memory

This one demonstrates the core promise of Open Brain. Start a conversation in ChatGPT and capture a thought:

```
Remember this: decided to use Notion for project management and linear for engineering tickets, not Jira
```

Now open Claude Desktop and start a fresh conversation. Ask:

```
What did I note recently about project management tools?
```

Claude should find the thought you saved from ChatGPT. Same database, different tools, shared memory. This is what Open Brain is built for.

### 9. Run a Weekly Review

At the end of your week, paste the Weekly Review companion prompt (from [docs/02-companion-prompts.md](02-companion-prompts.md)) into any connected AI. It will retrieve everything you captured that week, surface themes you might have missed, flag action items that are still open, and suggest what to focus on next week. The review takes about five minutes and becomes more valuable every week as your brain grows.

### 10. Build the Household Extension

Once you are comfortable capturing and searching, try Extension 1 (Household Knowledge Base in [extensions/household-knowledge/](../extensions/household-knowledge/)). It adds a separate table to your database for storing household facts — paint colors, appliance model numbers, vendor contacts, measurements. Your AI can then answer questions like "What's the model number of our dishwasher?" or "Which plumber did we use last year?" in seconds. It is a gentle introduction to the extension system.

---

## Extensions Overview

Extensions are optional add-ons that give your Open Brain new capabilities beyond the core thought capture and search. There are six, designed to be built in order — each one teaches new ideas through something genuinely useful.

**Extension 1: Household Knowledge Base** — Stores facts about your home that matter at the wrong moment: paint colors, appliance details, vendor contacts, measurements, warranty information. Built for beginners. After this, your AI can answer "what shade of blue is in the living room?" instantly.

**Extension 2: Home Maintenance Tracker** — Tracks recurring maintenance tasks and their history. Add tasks with frequencies (HVAC filter every 90 days), log completed work, and ask your AI what is coming due this month. Never get surprised by an overdue service again.

**Extension 3: Family Calendar** — A multi-person scheduling system for households. Track each family member's recurring activities and one-time events, add important dates with reminders, and ask your AI to surface conflicts or upcoming commitments. Designed for the chaos of overlapping schedules.

**Extension 4: Meal Planning** — Stores your recipe collection, plans weekly meals, and generates shopping lists automatically from those plans. Includes a shared access mode so a partner can view the meal plan and check off grocery items without accessing your full Open Brain.

**Extension 5: Professional CRM** — Contact management with interaction logging and follow-up tracking. Log every meeting and call, track relationship context over time, and get reminders before contacts go cold. Connects directly to your core Open Brain thoughts, so thoughts mentioning people can be linked to their contact records.

**Extension 6: Job Hunt Pipeline** — A complete job search tracking system with companies, postings, applications, interviews, and contacts. Your AI can show you your pipeline at a glance, surface upcoming interviews, calculate conversion rates across stages, and bridge job search contacts into your long-term professional network via Extension 5. The most sophisticated build in the series.

Extensions compound. Your CRM knows about thoughts you have captured. Your meal planner can check who is home this week from the family calendar. Your job hunt contacts become professional network contacts automatically.

---

## Tips for Getting the Most Out of Open Brain

**Capture when context is fresh.** Right after a meeting, call, or decision is when the most useful details are in your head. Five minutes later is fine. Next week is not — you will have already forgotten the texture of what happened.

**Use the capture templates.** The [companion prompts](02-companion-prompts.md) include five capture templates (Decision, Person Note, Insight, Meeting Debrief, and AI Save) that are structured to help your system extract better metadata. You do not have to use them, but they help early on while you are building the habit.

**Run the Memory Migration first.** If you have been using Claude or ChatGPT for a while, those AIs have accumulated context about you — your role, projects, preferences, people you work with. The Memory Migration companion prompt extracts all of that and saves it to your Open Brain so you start with a foundation instead of zero. Run it once per AI platform you use.

**Use metadata-friendly language.** Name people ("Marcus mentioned") rather than using pronouns. Include dates when they matter. Phrase action items clearly ("Action: I send the report by Friday"). You are not writing for a human reader — you are giving your AI's metadata extraction the clearest possible signal.

**Quality beats quantity.** One dense thought — a meeting note with decisions, people, and action items — is worth more than five vague one-liners. Capture things that will actually matter when you search for them in three months.

**Search broadly.** Your semantic search is tolerant of imprecise queries. You do not need to remember exactly what you typed. "What did I note about the API project?" will find thoughts even if you wrote "redesigning the endpoints" at the time.

---

## Troubleshooting for Users

**"My AI doesn't use the Open Brain tools automatically."**

Some AI tools are less aggressive about using available tools without prompting. Be explicit the first few times: "Use the Open Brain capture_thought tool to save this" or "Use the Open Brain search_thoughts tool to find my notes about [topic]." After the AI gets the pattern once or twice in a conversation, it usually starts picking it up on its own. ChatGPT in particular needs more explicit direction than Claude.

**"I searched but nothing came back."**

Two likely causes. First, you may not have captured enough yet — semantic search needs data to work with. If you have fewer than 10-20 thoughts, add more and try again. Second, try asking your AI to "search with a lower threshold, around 0.3" to cast a wider net. The default threshold filters out lower-confidence matches; lowering it shows more results, including less certain ones.

**"I got a 401 error."**

This means an access key mismatch. The key embedded in your MCP Connection URL does not match what is stored in your Supabase secrets. Double-check that the `?key=` value at the end of your URL matches your MCP Access Key from the credential tracker exactly. For troubleshooting steps, see the [Setup Guide](01-getting-started.md) and the [FAQ](03-faq.md).

**"It's slow the first time."**

Edge Functions (the server component of Open Brain) sleep when they have not been used recently and take a few seconds to wake up on the first request. Subsequent requests in the same session are fast. This is normal behavior; it is not a sign that something is broken.

**"The AI saved the thought but the metadata looks wrong."**

Metadata extraction is best-effort — an AI making its best guess from your text. The embedding (which powers search) is generated separately and works regardless of how the metadata gets classified. If you consistently want specific classifications, use the capture templates from the companion prompts to give the system clearer signals.

**Where to get help:**

- [FAQ](03-faq.md) — covers the most common setup and search issues in detail
- [Open Brain Discord](https://discord.gg/Cgh9WJEkeG) — active community with a `#help` channel for troubleshooting
- The dedicated AI assistants (Claude Skill, ChatGPT Custom GPT, Gemini GEM) listed in the [companion prompts](02-companion-prompts.md) know this system well and can walk you through problems in natural conversation
- The Supabase AI assistant (chat icon in the bottom-right of your Supabase dashboard) is excellent for anything database-related

---

*Built by Nate B. Jones — companion to [Build Your Open Brain](01-getting-started.md).*
