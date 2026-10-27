# Task Manager

A tiny personal task manager that you control with plain sentences in German or English.
One HTML file, no backend, no build step. Your tasks live as Markdown in a private GitHub repository, and Google Gemini turns your commands into edits.

> "Am 3.11. um 14 Uhr Zahnarzt" · "Friseurtermin ist erledigt" · "Add: Steuerunterlagen anfordern"

**Live app:** https://mikhail-poda.github.io/task-manager/

## How it works

```
 you type a command
        │
        ▼
 index.html (in your browser)
   1. reads todo.md and done.md from the private data repo (GitHub API)
   2. parses todo.md into a tree and sends the tree + your command to Gemini
   3. Gemini answers with a short list of operations (not whole files)
   4. the app applies the operations to the tree in code
   5. both files are written back as ONE commit
```

Gemini never rewrites your files. It only says what should change ("add this event on that date", "finish item L17"). The app makes the change itself, so unrelated lines cannot be altered, dropped or reordered by accident.

## Features

- **Natural-language commands** in German or English.
- **Tree instead of text editing.** `todo.md` is parsed into sections, optional groups and single items, and edited by item ID. No line numbers, no ambiguity with duplicate items.
- **Chronological section stays sorted.** New events are placed under the right day (created if needed, with the correct German weekday) and sorted by start time. Items without a time come first.
- **Empty parent items disappear.** When the last child of a group is finished or deleted, the group is removed too.
- **Flat history.** Finished items are appended as plain lines to the end of `done.md`.
- **Train rides clean themselves up.** Items containing "Zugfahrt" are removed automatically once their start time has passed (they are not archived).
- **Atomic commits.** Changes to both files go into a single Git commit, so a failure cannot leave half-applied state. Git history is your undo.
- **Safe by design.** Item text is preserved verbatim, a stale write is rejected instead of overwriting newer changes, and any invalid operation aborts the whole command.
- **Optional preview.** A setting shows a diff and asks before committing.

## Data format

The data repo contains two Markdown files.

`todo.md` uses numbered `#` sections separated by `---`. Items are bullets with at most one level of children:

```markdown
# 1. Termine & Reisen (Chronologisch)

* **Donnerstag, 29.10.2026**
  * **10:30 Uhr:** Wohnsitzanmeldung Bürgerbüro Pasing
  * **23:03 – 06:50 Uhr:** Zugfahrt München Hbf → Hannover Hbf

---

# 2. Dringend & Kurzfristig

* **Behörden & Steuern:**
  * Arbeitsagentur beobachten
* Perlator entkalken
```

Rules:

- The sections are fixed. Rename, add or remove them by hand in GitHub.
- A bullet with children is a **group**, a bullet without children is a **leaf**. No deeper levels. If the app finds any, it refuses to edit instead of guessing.
- The section whose title contains "Chronologisch" is the sorted calendar. Section "Mittelfristig" is the catch-all for items that fit nowhere else.

`done.md` is a flat list, newest at the bottom:

```markdown
# Erledigt / Historie

* Rechner von Duya in Betrieb nehmen
```

## Setup

You need two repositories, a GitHub token and a Gemini key.

1. **Data repo (private)**, for example `tasks-data`, with `todo.md` and `done.md` in the root of the `main` branch.
2. **This app repo** with `index.html` in the root. Enable **Settings → Pages → Deploy from a branch → `main` / root** (a public repo is required on the free GitHub plan).
3. **GitHub token:** Settings → Developer settings → Fine-grained tokens. Limit it to the data repo only, permission *Contents: Read and write*, nothing else.
4. **Gemini key:** create one in [Google AI Studio](https://aistudio.google.com).
5. Open the app, go to **Settings**, enter token, key, repository owner and name, branch, file paths and the Gemini model name, then save.

To run it locally instead, serve the folder with any static server, for example `npx serve .`. Do not open the file via `file://`.

On a phone, open the live address and choose "Add to Home Screen".

## Security and privacy

- `index.html` contains no secrets. Token and key are typed into the page and stored only in that browser's `localStorage`. Each browser or device needs them entered once.
- The GitHub token should be fine-grained and limited to the data repo. Rotate it when it expires.
- The Gemini key is sent as a request header, not in the URL.
- Rendered Markdown is sanitized with DOMPurify before it is shown.
- **Your task text is sent to Google** with every command. Check the data-use terms of your Gemini tier, since free tiers may use inputs to improve Google's products.
- The app repo is public, the data repo is not. Keep them separate.

## Limitations

- Only `todo.md` structured as described is supported. Free text between bullets, or a third list level, makes the app refuse to edit.
- Moving an existing event to a different date or time is done as delete + add, so the leaf text is re-entered.
- "Past" for train rides means the start time has passed, not the arrival.
- One user, no conflict merging: if the repo changes while a command is running, the write is rejected and you retry.
- Quality of interpretation depends on the Gemini model. The system prompt is in `index.html` (`SYSTEM_PROMPT`) and can be tuned.

## Tech

Plain HTML, CSS and JavaScript in one file. [marked](https://github.com/markedjs/marked) renders Markdown and [DOMPurify](https://github.com/cure53/DOMPurify) sanitizes it. It talks directly to the GitHub REST API (Git Data API for commits) and the Gemini `generateContent` API. No build tools, no dependencies to install.
