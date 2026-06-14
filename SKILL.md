---
name: knowledge
description: "Ark Vault brain operations — query, ingest, capture, process inbox, find connections, and vault health. Use when user wants to search their knowledge base, add new knowledge, process captures, or maintain their Obsidian vault."
trigger: /knowledge
---

# /knowledge

Interact with the user's Ark Vault Obsidian knowledge base. Detects intent from args and dispatches to the right workflow. All operations follow the vault's conventions exactly.

## Vault constants

```
VAULT="$HOME/Library/Mobile Documents/iCloud~md~obsidian/Documents/Ark Vault"
NOTES="$VAULT/50_Knowledge/notes"
MOCS="$VAULT/50_Knowledge/_moc"
INBOX="$VAULT/90_System/92_QuickCapture"
ARCHIVE="$VAULT/90_System/99_Archive"
TEMPLATES="$VAULT/90_System/91_Templates"
```

## Usage

```
/knowledge <question>          # query — search vault and synthesise answer from your notes
/knowledge ingest <url>        # fetch URL → QuickCapture → zettelkasten note
/knowledge ingest <text>       # capture raw text → process into zettelkasten note
/knowledge capture <thought>   # fast capture a fleeting thought to QuickCapture inbox
/knowledge process             # process ALL unprocessed inbox items one by one
/knowledge process <filename>  # process a specific QuickCapture file into zettelkasten
/knowledge connect             # find orphaned notes with no MOC and suggest links
/knowledge status              # vault health: unprocessed inbox count, orphaned notes, MOC gaps
/knowledge moc <topic>         # create or update a MOC for a topic
```

If no subcommand matches, treat the entire input as a query.

---

## Mode: query `<question>`

Answer from the vault, not from training data.

1. Extract 2-4 keywords from the question
2. Search MOCs first: `grep -rl "<keyword>" "$MOCS/"` — read any matching MOC to get the cluster map
3. Search notes: `grep -rl "<keyword>" "$NOTES/"` — read the top 5 most relevant hits
4. Also check `40_Study/`, `30_Business/`, `20_Work/` if notes don't surface enough
5. Synthesise an answer citing specific note filenames as `[[note-slug]]`
6. If vault has nothing on the topic, say so explicitly and offer to ingest a source

---

## Mode: ingest `<url or text>`

Full pipeline: source → QuickCapture → zettelkasten note.

### Step 1 — Fetch / receive content
- If URL: use WebFetch to retrieve the content
- If text: use as-is

### Step 2 — Create QuickCapture note
File: `$INBOX/YYYYMMDDHHMMSS-[slug].md`

```markdown
---
id: "YYYYMMDDHHMMSS"
title: "Web Fetch: [Source Title]"
created: YYYY-MM-DD
tags:
  - type/inbox
  - source/web-fetch
url: "https://..."
fetched: YYYY-MM-DD
processed: false
---

## Summary
[2-3 sentence summary of what was fetched and why]

## Key Points
- [Point 1]
- [Point 2]

## Action
- [ ] Process into zettelkasten note in 50_Knowledge/notes/
- [ ] Link to relevant MOCs
```

### Step 3 — Extract core insight
- Identify the single most valuable insight worth keeping as an atomic note
- If multiple distinct ideas exist, plan multiple notes (one per idea)

### Step 4 — Create zettelkasten note(s)
File: `$NOTES/YYYYMMDDHHMMSS-[slug].md`

```markdown
---
id: "YYYYMMDDHHMMSS"
title: "Descriptive Title of the Concept"
created: YYYY-MM-DD
modified: YYYY-MM-DD
tags:
  - topic/[area]
  - type/concept
  - source/web-fetch
status: seedling
aliases: []
---

[Insight written in user's own words — never paste raw content. One idea only.]

## Source
- [Title](url)

## Links
- [[related-note-slug]]
```

### Step 5 — Link to MOC
- Check `$MOCS/` for a matching MOC
- If found: add the new note as a wikilink under the correct section
- If not found and 5+ notes now exist on this topic: offer to create a new MOC
- Mark the QuickCapture file's `processed: false` → `processed: true`

---

## Mode: capture `<thought>`

Fast path for fleeting thoughts — no processing, just inbox it.

Create `$INBOX/YYYYMMDDHHMMSS-capture.md`:

```markdown
---
id: "YYYYMMDDHHMMSS"
title: "Capture: [first 6 words of thought]"
created: YYYY-MM-DD
tags:
  - type/inbox
  - source/personal
processed: false
---

[thought verbatim]
```

Confirm saved. Don't process it now.

---

## Mode: process `[filename]`

Turn inbox items into proper zettelkasten notes following the same Steps 3-5 from ingest.

- If no filename: `grep -rl "processed: false" "$INBOX/"` to list all pending items
- Show the user the list and ask which to process, OR process them all sequentially
- For each: read the QuickCapture note → extract insight → create zettelkasten note → link to MOC → mark processed
- Report: "Processed X items, created Y notes, updated Z MOCs"

---

## Mode: connect

Find knowledge gaps and orphaned notes.

1. List all notes in `$NOTES/` 
2. For each note, check if it appears as a `[[wikilink]]` in any MOC: `grep -rl "note-slug" "$MOCS/"`
3. Report notes with zero MOC appearances — these are orphaned
4. For each orphan, suggest which existing MOC it fits based on its tags and content
5. Offer to add the links automatically

---

## Mode: status

Vault health report.

```bash
# Unprocessed inbox
grep -rl "processed: false" "$INBOX/" | wc -l

# Total zettelkasten notes
ls "$NOTES/" | wc -l

# Total MOCs
ls "$MOCS/" | wc -l

# Notes modified in last 7 days
find "$NOTES/" -name "*.md" -mtime -7 | wc -l

# Notes with status: seedling (need development)
grep -rl "status: seedling" "$NOTES/" | wc -l
```

Report as:
- Inbox pending: N items
- Total notes: N (N seedling, N budding, N evergreen)
- MOCs: N
- Recent activity: N notes modified this week
- Suggested action if inbox > 5 or seedlings > 20% of total

---

## Mode: moc `<topic>`

Create a new MOC or update an existing one.

1. Check if `$MOCS/MOC-[topic].md` exists
2. If updating: search for all notes tagged `topic/[topic]` and add any not yet linked
3. If creating: search for all relevant notes, scaffold the MOC structure, populate with wikilinks

MOC format:
```markdown
---
title: "MOC: [Topic Name]"
created: YYYY-MM-DD
modified: YYYY-MM-DD
tags:
  - type/moc
---

# [Topic Name]

[Brief description of this knowledge cluster.]

## Core Concepts
- [[note-slug]]

## How-To / Practical
- [[note-slug]]

## Related MOCs
- [[MOC-related-topic]]

## Open Questions
- [What still needs exploring?]
```

---

## Conventions (always follow)

- **Never paste raw content** into zettelkasten notes — always summarise and reframe
- **One idea per note** — split if a source has multiple distinct insights
- **Always link** — every new note must link to at least one other note or MOC
- **Timestamp IDs** — `YYYYMMDDHHMMSS` using current datetime
- **Tag hierarchy** — `topic/`, `type/`, `source/`, `project/`
- **Status** — new notes start as `seedling`; only promote to `budding`/`evergreen` explicitly
- **Do NOT reorganise** existing folders or modify dashboard files
- **Wikilinks** — use `[[slug]]` not `[[full-path/slug]]`
