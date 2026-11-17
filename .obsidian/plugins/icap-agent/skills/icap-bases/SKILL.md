---
name: icap-bases
description: How to write an Obsidian Base: a live table, card or list view of notes by their properties (a base code block in a note, or a .base file). Read this before writing one.
---

# Bases

A base is a live view of notes by their properties: a table, cards or a list that updates as the notes change. Use one when the learner wants to see many notes side by side (concepts with their status, sources by author, what is still open), not for content a list would hold.

## Where

- In a note: a fenced `base` block holding the YAML below. Preferred: it stays next to the text it serves, and `this` is that note.
- As its own file: create_note with type "base" and the YAML as content (no fence). A note embeds it with ![[Name.base]] or ![[Name.base#View name]].

## Shape

```yaml
filters:                    # which notes: one expression, or and / or / not with a list
  and:
    - file.inFolder("Physics")
    - 'status != "done"'
formulas:
  age: '(now() - file.ctime).days.round(0)'
properties:
  formula.age:
    displayName: "Days"
views:
  - type: table             # table | cards | list
    name: "Open"
    order: [file.name, status, formula.age]
    groupBy: { property: status, direction: ASC }
    summaries: { formula.age: Average }
```

## Expressions

- Properties: `status` or `note.status` (from the note's properties), `file.name`, `file.folder`, `file.tags`, `file.links`, `file.backlinks`, `file.mtime`, `formula.name`.
- Functions: file.hasTag("t"), file.inFolder("F"), file.hasLink("Note"), file.hasProperty("p"), list.contains(x), if(test, a, b), date("2026-01-31"), today(), now(). Operators: == != > < >= <= && || !.
- A property name with a hyphen is note["my-prop"] in a filter or formula (a bare my-prop subtracts); in order, groupBy and properties it is note.my-prop.
- Date minus date is a duration: take .days (a fraction) before any number function, (date(due) - today()).days.round(0).
- A property may be missing on some notes: guard it, if(due, (date(due) - today()).days, "").
- Summaries: Sum, Average, Min, Max, Median, Earliest, Latest, Checked, Filled, Unique.

## Before you write it

- Use properties the notes really have: look at a few of them first.
- Quote every expression; when it contains double quotes, wrap it in single quotes: 'if(done, "yes", "no")'.
- Every formula.name you list must be defined under formulas.
