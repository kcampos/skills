<!-- Human documentation only. Not intended for LLM consumption. -->

# Requirements
AI agent + Notion MCP active

# Description
This skill allows AI agents to act as a research assitant that takes care of organizing and follow up research tasks on the breadcrumbs you leave. Historically I've been frustrated with note taking systems and forms of knowledge graphs in the sense that it introduces a lot of work and overhead to engage and maintain those systems and a whole lot more work at the moment you try and synthesize your work later on.

My intent here is to create a breadcrumb systems that allows you(the human) to capture thoughts, people, quotes and their sources quickly and let the AI system do the heavy lifting of persisting and connectings those breadcrumbs topically while you continue on. I find thids particularly helpful in long forms of research where you are going deep and you are developing a knowledge graph over time.

It's leveraging Notion as its persistence and organization layer, I'm sure it could work just as well on other MCP enabled note systems that support a form of knowledge graph. (future work)

Below is the Research Hub page that this skill will create on initialization that acts as your reference guide

# 🔬 Research Hub

Your persistent research companion. Drop breadcrumbs as you work — quotes, notes, people — and interrogate the full body of work any time.

---

## 🗄️ Your Databases

<database url="[📚 Sources](https://www.notion.so/4ad337de49424605a2db8001ad662e66?pvs=21)">

<database url="[💬 Quotes](https://www.notion.so/f4319c6db5084dd0afe89eecfcbc0a28?pvs=21)">

<database url="[📝 Notes](https://www.notion.so/cfd35315a15840d7886b3ad1cd0494c2?pvs=21)">

<database url="[👤 People](https://www.notion.so/9546ffb7a4554d9b881495e8c93fedc2?pvs=21)">

---

## 🧭 Command Reference

Use these commands in your conversation with Claude to capture breadcrumbs:

### `/quote` — Save a Direct Quote

Captures a verbatim quote with full citation info into the 💬 Quotes database.

**Minimal usage:**

```
/quote "The limits of my language mean the limits of my world." — Wittgenstein, Tractatus, p. 68
```

**Full usage:**

```
/quote
text: "The limits of my language mean the limits of my world."
source: Tractatus Logico-Philosophicus
author: Ludwig Wittgenstein
page: 68
chapter: 5.6
topics: Theory, Identity
commentary: Central to the argument about epistemic boundaries
```

**What Claude stores:** Quote text · Source (linked) · Author · Page number · Chapter · Topics · Your commentary · Date

**Auto-behavior:** When a `/quote` includes an author name, Claude automatically checks the 👤 People database. If no entry exists for that person, Claude researches them via web search and creates a full People profile, linked to the quote and its sources. No extra command needed — this happens automatically every time.

---

### `/source` — Register a Source

Adds a book, article, or paper to the 📚 Sources database so it can be linked from quotes and notes.

**Usage:**

```
/source
title: Discipline and Punish
author: Michel Foucault
year: 1975
type: Book
publisher: Gallimard
```

**Auto-behavior:** When a `/source` is registered with an author, Claude automatically checks the 👤 People database. If no entry exists for that author, Claude researches them and creates a People profile linked to the source.

---

### `/note` — Save a Research Insight

Captures your own thoughts, observations, questions, or arguments into the 📝 Notes database.

**Minimal usage:**

```
/note The framing of "discovery" erases indigenous knowledge systems entirely.
```

**Full usage:**

```
/note
text: The framing of "discovery" erases indigenous knowledge systems entirely.
type: Critique
source: The Invention of Science (Wootton)
page: 14
topics: History, Power, Ethics
```

**Note types:** Insight · Question · Argument · Connection · Critique · Summary

**What Claude stores:** Note text · Note type · Source (linked) · Page reference · Topics · Related people · Date

---

### `/person` — Research and Save a Person

Triggers a background research task on a named individual and saves a structured profile to the 👤 People database. Use this to explicitly flag someone for research. Also triggered automatically by `/quote` and `/source`.

**Minimal usage:**

```
/person Michel Foucault
```

**Full usage:**

```
/person
name: Michel Foucault
relevance: His concept of biopower is central to my argument in Chapter 3
topics: Power, Theory
```

**What Claude researches and stores:** Role/field · Era · Biographical summary · Key works · Relevance to your research · Topic tags · Linked sources

---

### `/thread [topic]` — Organize Around a Theme

Ask Claude to pull together all quotes, notes, and people tagged with a given topic and synthesize them into a coherent narrative.

**Usage:**

```
/thread Power
/thread Nationalism
/thread [any topic or free-form question]
```

Claude will query all four databases and return a structured synthesis.

---

### `/by [source title]` — Explore a Source

Pull all quotes and notes linked to a specific source.

**Usage:**

```
/by A Church Undone
/by Discipline and Punish
```

---

### `/about [person name]` — Explore a Person's Footprint

Pull everything connected to a specific person across all databases.

**Usage:**

```
/about Paul Althaus
/about Hannah Arendt
```

---

### `/synthesize` — Full Research Synthesis

Claude queries all databases and produces a structured overview of your entire body of accumulated research, organized by theme. Use when you want to see the big picture.

**Usage:**

```jsx
/synthesize
/synthesize around the argument that [X]
```

---

### `/upgrade-hub` — Update This Page

Checks whether this hub page matches the current skill version and rewrites it if the template has changed. Run after updating the skill.

**Usage:**

```jsx
/upgrade-hub
```

---

## 💡 Tips

- **Tags are your threads.** The Topics multi-select is your primary way to weave connections across quotes, notes, and people. Use them consistently.
- **Minimal inputs are fine.** Claude will infer what it can and ask only if truly necessary.
- **People are auto-created.** Any time you use `/quote` or `/source` with an author, Claude will automatically research and save a People entry if one doesn't already exist.
- **`/person` triggers research.** Unlike other commands which just store, `/person` will actively research the individual before saving — so expect a brief response before the Notion entry appears.
- **You can add custom topics.** The topic tags in each database are just starting points — ask Claude to add new tags any time.
- **Every entry is linked.** Sources, Quotes, Notes, and People all reference each other — so `/thread` queries can traverse the full graph.