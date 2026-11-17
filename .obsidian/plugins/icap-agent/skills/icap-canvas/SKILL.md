---
name: icap-canvas
description: How to make an Obsidian canvas (.canvas, JSON Canvas 1.0): a board of cards, notes and links joined by arrows, for a concept map the learner arranges. Read this before creating or changing one.
---

# Canvases

A canvas is a board the learner arranges: text cards, notes and links placed in space and joined by arrows. Use one for a concept map the learner will keep working on, or to lay several notes side by side. A picture that only needs reading is simpler as a Mermaid diagram.

## Create

create_note with type "canvas"; the content is JSON: {"nodes": [...], "edges": [...]}.

Every node has id, type, x, y, width, height, and may have color ("1" red to "6" purple). By type:
- text: "text", Markdown; a new line is \n in the JSON string (never an escaped \\n).
- file: "file", a vault path ("Physics/Momentum.md"), and may have "subpath": "#Heading". For a note that exists, prefer a file node: the canvas shows the note itself.
- link: "url".
- group: "label"; place its members inside its box.

An edge has id, fromNode, toNode, and may have fromSide / toSide (top, right, bottom, left), toEnd ("arrow" by default, or "none") and label.

## Layout

x grows to the right, y downwards; a node's x and y are its top-left corner. A short text card is about 260 × 120, a note 400 × 300. Leave 60 to 100 between cards and align to a grid of 20. Lay out by meaning: a centre with spokes for a concept map, left to right for a process, columns for a comparison.

## Before you send it

Ids are unique (16 hex characters), every edge's fromNode and toNode is a node of this canvas, and the JSON parses. create_note refuses a canvas that breaks these, and says why.

## Change one

propose_edit on the .canvas file: find an exact piece of its JSON and replace it so that the whole stays valid.
