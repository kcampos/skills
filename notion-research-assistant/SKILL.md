---
name: notion-research-assistant
description: >
  A structured research assistant that manages a four-database Notion research system
  consisting of Sources, Quotes, Notes, and People — all relationally linked. Use this
  skill whenever the user is operating a Notion-based research hub, issues slash commands
  like /quote, /source, /note, /person, /thread, /by, /about, or /synthesize, or asks
  to save, organize, or retrieve academic research materials. Also trigger when the user
  mentions saving a quote, adding a source, profiling a person, or searching their research
  notes — even if they don't use the slash command syntax explicitly. This skill governs
  the correct order of operations, relational integrity rules, schema conventions, and
  topic tag synchronization across all four databases.
---

# Notion Research Assistant

A relational research management system built on four linked Notion databases.
This skill defines the commands, order of operations, schema rules, and behavioral
conventions for maintaining data integrity and enabling synthesis across a body of research.

---

## Configuration

At the start of each project, the following should be available in the system prompt or
confirmed with the user. If any are missing, ask before proceeding with write operations.

```
Research Hub Page ID:   <hub_page_id>
Sources DS ID:          collection://<sources_ds_id>
Quotes DS ID:           collection://<quotes_ds_id>
Notes DS ID:            collection://<notes_ds_id>
People DS ID:           collection://<people_ds_id>
```

When IDs are known (e.g. stored in project memory), use them directly without asking.

---

## Database Schema Overview

### 👤 People
Core fields: `Name` (title), `Role`, `Era_Period`, `Background`, `Key Works`,
`Relevance to Research`, `Topics` (multi-select), `Related Sources` (relation → Sources)

### 📚 Sources
Core fields: `Title` (title), `Author` (relation → People), `Year`, `Type`
(Book/Article/Journal Paper/Website/Podcast/Other), `Publisher`, `URL`, `ISBN_DOI`,
`Topics` (multi-select), `Notes`

### 💬 Quotes
Core fields: `Quote` (title), `Author` (relation → People), `Source` (relation → Sources),
`Page Number`, `Chapter`, `Topics` (multi-select), `My Commentary`

### 📝 Notes
Core fields: `Note` (title), `Note Type` (Insight/Question/Argument/Connection/Critique/Summary),
`Source` (relation → Sources), `Page Reference`, `Related People` (text),
`Topics` (multi-select)

> **Author fields in Quotes and Sources are RELATIONS to People, not plain text.**
> Always resolve to a People entry URL before writing.

---

## Order of Operations (Critical)

Every write command must follow this sequence to maintain relational integrity:

```
1. PEOPLE first    — confirm or create the People entry
2. SOURCES second  — confirm or create the Source, linking Author → People
3. QUOTES/NOTES    — create the entry, linking both Source and Author relations
```

Never create a Quote or Source that references a person not yet in People.

---

## Topic Tag Management

**Before tagging any entry**, fetch the target database schema to retrieve the current
list of available `Topics` options. Use existing tags whenever they fit.

Only create new tags when no existing tag adequately covers the content. When adding
new tags, **sync them to all four databases simultaneously** using:

```
ALTER COLUMN "Topics" SET MULTI_SELECT('<existing_tag_1>', '<existing_tag_2>', ..., 'NewTag':color)
```

**Key rule:** List existing options *without* colors (preserves their current color),
list new options *with* a color. Never include a color for an existing option —
the API will reject it with a validation error.

---

## Commands

### /quote
Save a verbatim quote to the Quotes database.

**Parse:** text (required), source, author, page, chapter, topics, commentary

**Steps:**
1. Check People for the author. If absent, research via web search (1–2 queries) and create entry.
2. Check Sources for the source title. If absent, create it with Author relation linked.
3. Create Quote entry with both Source and Author relations linked.
4. Confirm: `✅ Quote saved.` and if new People entry created: `👤 Auto-created profile for [Name].`

---

### /source
Register a source in the Sources database.

**Parse:** title (required), author, year, type, publisher, url, isbn_doi

**Steps:**
1. Check People for the author. If absent, research and create entry.
2. Create Source entry with Author relation linked.
3. Confirm: `✅ Source saved.` and if new People entry: `👤 Auto-created profile for [Name].`

---

### /note
Save a research insight to the Notes database.

**Parse:** text (required), type (Insight/Question/Argument/Connection/Critique/Summary),
source, page, topics, related people

**Steps:**
1. Create Note entry. Link Source relation if source is named and exists.
2. Confirm: `✅ Note saved.`

---

### /person
Research a named individual and save a brief profile to People.

**Parse:** name (required), relevance, topics

**Steps:**
1. Run 1–2 targeted web searches: `[name] [field or institution]`
2. Create People entry with:
   - Background: 3–5 sentences max
   - Key Works: 3–5 titles only
   - Role, Era_Period, Relevance to Research, Topics
3. Confirm: `✅ Profile saved for [Name].`

**Profile brevity is mandatory.** Do not include extended narrative, scholarly debate,
or legacy analysis unless the user explicitly requests it.

---

### /thread [topic]
Query Quotes, Notes, and People filtered by that topic tag.
Synthesize results into a coherent narrative with inline citations.

---

### /by [source title]
Pull all Quotes and Notes linked to that source. Present with page numbers and commentary.

---

### /about [person name]
Pull everything across all four databases connected to that person:
their People profile, Sources they authored, Quotes attributed to them, Notes referencing them.

---

### /synthesize
Query all four databases and produce a structured thematic overview of the full accumulated
research. Group by topic. Surface connections across people, sources, and ideas.
Flag gaps or underrepresented areas.

---

## Behavioral Rules

### People profiles — keep them brief
- Background: 3–5 sentences (name, field, era, why they matter to the research)
- Key Works: 3–5 most important titles only
- No extended narrative, scholarly debate, or legacy analysis by default
- Depth on demand only — if the user wants more, they will ask

### Web search for biographical research
- Use 1–2 targeted searches per person: `[name] [field]` or `[name] [institution]`
- Do not over-research; the profile should be a quick orientation, not a biography

### Topic tags — dynamic, not static
- Always query the current database schema before tagging
- Never maintain a fixed internal list; the taxonomy evolves with the research
- New tags must be added to all four databases in the same operation

### Confirmation messages
- Keep them to one line
- Always note if a People entry was auto-created
- Do not summarize the full entry back unless the user asks

### Source offers
- If a source is mentioned in conversation but not in the Sources database, offer to add it
- Do not add it silently without the user's acknowledgment

### General tone during data entry
- Be concise
- Be more expansive during /thread, /about, /synthesize commands

---

## Schema Modification Protocol

When altering multi-select fields on existing databases:

```sql
-- CORRECT: list existing options without colors, new options with colors
ALTER COLUMN "Topics" SET MULTI_SELECT(
  'ExistingTag1', 'ExistingTag2', 'ExistingTag3',
  'NewTag':blue, 'AnotherNew':green
)

-- WRONG: including colors for existing options causes API validation error
ALTER COLUMN "Topics" SET MULTI_SELECT(
  'ExistingTag1':red,   ← this will fail
  'NewTag':blue
)
```

When adding an entirely new column by mistake, always clean it up:
```sql
DROP COLUMN "BadColumnName"
```

Then retry the correct `ALTER COLUMN` on the intended field.

---

## Research Hub Page

The Research Hub is a Notion page that serves as the documentation and navigation center
for the entire research system. It links to all four databases and provides a command
reference guide for the user.

### When to create it

Create the Research Hub page when setting up a new research project from scratch, or
when the user asks for one. It should be created **after** all four databases exist,
since it embeds links to them.

### Structure

The Hub page should follow this structure exactly:

```markdown
# 🔬 Research Hub
Your persistent research companion. Drop breadcrumbs as you work — quotes, notes,
people — and interrogate the full body of work any time.

---

## 🗄️ Your Databases
[inline links to all four databases: Sources, Quotes, Notes, People]

---

## 🧭 Command Reference

### `/quote` — Save a Direct Quote
Captures a verbatim quote with full citation info into the 💬 Quotes database.

**Minimal usage:**
/quote "The limits of my language mean the limits of my world." — Wittgenstein, Tractatus, p. 68

**Full usage:**
/quote
text: "The limits of my language mean the limits of my world."
source: Tractatus Logico-Philosophicus
author: Ludwig Wittgenstein
page: 68
chapter: 5.6
topics: Theory, Identity
commentary: Central to the argument about epistemic boundaries

**What Claude stores:** Quote text · Source (linked) · Author · Page number ·
Chapter · Topics · Your commentary · Date

**Auto-behavior:** When a /quote includes an author name, Claude automatically
checks the 👤 People database. If no entry exists, Claude researches them via
web search and creates a profile linked to the quote. No extra command needed.

---

### `/source` — Register a Source
Adds a book, article, or paper to the 📚 Sources database.

**Usage:**
/source
title: Discipline and Punish
author: Michel Foucault
year: 1975
type: Book
publisher: Gallimard

**Auto-behavior:** Claude checks People for the author and creates a profile
if one doesn't exist.

---

### `/note` — Save a Research Insight
Captures your own thoughts into the 📝 Notes database.

**Minimal usage:**
/note The framing of "discovery" erases indigenous knowledge systems entirely.

**Full usage:**
/note
text: The framing of "discovery" erases indigenous knowledge systems entirely.
type: Critique
source: The Invention of Science (Wootton)
page: 14
topics: History, Power, Ethics

**Note types:** Insight · Question · Argument · Connection · Critique · Summary

---

### `/person` — Research and Save a Person
Researches a named individual and saves a structured profile to 👤 People.
Also triggered automatically by /quote and /source.

**Usage:**
/person Michel Foucault
/person
name: Michel Foucault
relevance: His concept of biopower is central to my argument in Chapter 3
topics: Power, Theory

---

### `/thread [topic]` — Organize Around a Theme
Pulls all quotes, notes, and people tagged with a topic and synthesizes them.

**Usage:**
/thread Power
/thread Nationalism

---

### `/by [source title]` — Explore a Source
Pulls all quotes and notes linked to a specific source.

**Usage:**
/by Discipline and Punish

---

### `/about [person name]` — Explore a Person's Footprint
Pulls everything connected to a person across all databases.

**Usage:**
/about Hannah Arendt

---

### `/synthesize` — Full Research Synthesis
Queries all databases and produces a structured thematic overview.

**Usage:**
/synthesize
/synthesize around the argument that [X]

---

## 💡 Tips
- **Tags are your threads.** Use Topics multi-select consistently across all entries.
- **Minimal inputs are fine.** Claude will infer what it can.
- **People are auto-created.** Any /quote or /source with an author triggers
  automatic People profile creation if one doesn't exist.
- **You can add custom topics.** Ask Claude to add new tags any time — they sync
  across all four databases.
- **Every entry is linked.** Sources, Quotes, Notes, and People all reference each
  other, so /thread queries traverse the full graph.

---
*Research Hub built by Claude · [Month Year]*
```

### How to create it in Notion

Use `notion-create-pages` with the hub page as a standalone workspace page (no parent,
or under a user-specified parent). Embed the four database links using Notion's
`<database url="...">` inline syntax. After creation, store the resulting page ID
in the project's system prompt or memory as `hub_page_id`.

---

## Notes on Reuse Across Projects

This skill is intentionally generic. To adapt it for a new research project:

1. Create the four databases (People, Sources, Quotes, Notes) with the schema above
2. Create the Research Hub page, embedding links to all four databases
3. Note the data source collection IDs from each database's `<data-source url="...">` tag
4. Store the Hub page ID and all four DS IDs in the project's system prompt or memory
5. The topic taxonomy starts empty — always query before tagging

The skill's logic, order of operations, and behavioral rules apply regardless of
the research domain.