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
`Page Number`, `Chapter`, `Topics` (multi-select), `My Commentary`, `Claude Commentary`

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
4. After saving the entry, generate a brief Claude Commentary (1-2 paragraphs):
   - Interpret the quote's significance within the source's argument and the research's broader themes
   - Note connections to other logged sources, people, or topics where relevant
   - Write in a voice that is analytically engaged, not merely descriptive
   
   Then:
   - Save the commentary to the `Claude Commentary` field via update_properties on the created entry
   - Output the same commentary text in chat so the user sees it immediately
5. Confirm: `✅ Quote saved.` and if new People entry created: `👤 Auto-created profile for [Name].`

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

### /upgrade-hub
Check whether the Research Hub page is current and update it if the skill has changed.

**Steps:**
1. Fetch the hub page (`hub_page_id`) and read the version tag from the footer.
2. Compare to the template version in [`references/hub-setup.md`](references/hub-setup.md).
3. Describe what changed and ask the user to confirm before rewriting.
4. Apply hub content updates and/or database schema changes per the upgrade protocol
   in `references/hub-setup.md`.
5. Confirm: `✅ Research Hub updated to v[X].`

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

### Claude Commentary — automatic on /quote
- Generated for every /quote entry without prompting, even if the user has not specified their own commentary
- Aims for interpretive depth: what does the quote *do* — in its argument, its framing, or its context?
- May surface connections to other logged sources, figures, or topics
- Kept to 1-2 paragraphs — substantive but not exhaustive
- Written in chat AND saved to `Claude Commentary` in the Quotes database

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

See [`references/hub-setup.md`](references/hub-setup.md) for the hub page template,
creation instructions, and the full upgrade protocol (both hub content and database
schema changes).

---

## Notes on Reuse Across Projects

This skill is intentionally generic. To adapt it for a new research project:

1. Create the four databases (People, Sources, Quotes, Notes) with the schema above
2. Create the Research Hub page using the template in `references/hub-setup.md`
3. Note the data source collection IDs from each database's `<data-source url="...">` tag
4. Store the Hub page ID and all four DS IDs in the project's system prompt or memory
5. The topic taxonomy starts empty — always query before tagging

When the skill is updated and `hub_page_id` is known, run `/upgrade-hub` to bring
the hub page and database schemas in line with the new version.

The skill's logic, order of operations, and behavioral rules apply regardless of
the research domain.