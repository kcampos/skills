# Research Hub — Setup & Upgrade Guide

The Research Hub is a Notion page that serves as the documentation and navigation center
for the entire research system. It links to all four databases and provides a command
reference guide for the user.

**Current template version: v1.0**

The hub footer includes a version tag: `*Research Hub · v1.0 · Built by Claude*`.
This is used to detect whether an existing hub is out of date.

---

## When to Create

Create the Research Hub page when setting up a new research project from scratch, or
when the user asks for one. Always create it **after** all four databases exist,
since it embeds links to them.

---

## How to Create

Use `notion-create-pages` with no parent (or a user-specified parent). Embed the four
database links using Notion's `<database url="...">` inline syntax.

After creation, store the resulting page ID in project memory as `hub_page_id`.

---

## Hub Page Template

Use the structure below exactly when creating or rewriting the hub.

````markdown
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

### `/upgrade-hub` — Update This Page
Checks whether this hub page matches the current skill version and rewrites it
if the template has changed. Run after updating the skill.

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
*Research Hub · v1.0 · Built by Claude · [Month Year]*
````

---

## Skill Upgrade Protocol

Skill upgrades can affect two distinct layers: the **hub page content** (command
reference, tips) and the **database schemas** (fields, types, relations). Both
require different remediation steps.

### Detecting that an upgrade is needed

When `hub_page_id` is known:
1. Fetch the hub page
2. Look for the version tag in the footer (e.g., `v1.0`)
3. Compare to the current template version at the top of this file
4. If they differ, or if no version tag is found, an upgrade is needed

### Hub content upgrade (command reference changed)

Triggered when: new commands were added, command syntax changed, or tips were updated.

**Steps:**
1. Fetch the current hub page with `notion-fetch`
2. Confirm with the user: "The hub page is on v[X], current template is v[Y].
   Shall I rewrite it with the updated command reference?"
3. On confirmation, use `notion-update-page` with `command: replace_content` and
   the full hub template above
4. Confirm: `✅ Research Hub updated to v[Y].`

### Schema upgrade (database fields changed)

Triggered when: new fields were added to one or more of the four databases, field
types changed, or relations were added.

**Steps:**
1. Identify which databases are affected (People, Sources, Quotes, Notes)
2. For each affected database, describe the change to the user:
   - Added fields and their types
   - Removed or renamed fields
   - New relation targets
3. Ask for confirmation before running any DDL
4. Apply changes using `notion-update-data-source` with the appropriate
   `ADD COLUMN`, `DROP COLUMN`, or `ALTER COLUMN` statements
5. If new tags are needed in `Topics`, sync them across all four databases
   per the topic tag rules in `SKILL.md`
6. After all schema changes are applied, check whether the hub content also
   needs updating (new fields may warrant a tips update)
7. Confirm each change individually: `✅ Sources schema updated.`, etc.

### Combined upgrade (both hub and schema)

Apply schema changes first, then rewrite the hub. This ensures the hub reflects
the final database structure.
