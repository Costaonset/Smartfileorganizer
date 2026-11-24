# Smart File Organizer and Report Generator

A small local web application that sorts the files in a folder into category folders (Documents, Images, Videos, ...), always **previews first**, moves files only **after explicit confirmation**, and saves a **JSON report** of every operation.

Built for **BSE2201 – Emerging AI Technologies in Software Engineering** (Zambia University College of Technology) to demonstrate **Agentic Engineering**: the application itself contains no AI model; the *AI coding agent* helped build it. See [docs/AI_AGENT_WORKFLOW.md](docs/AI_AGENT_WORKFLOW.md).

## Problem statement

Download and project folders fill up with mixed files. Sorting them by hand is slow, and naive scripts are dangerous: they can overwrite files, follow links into the wrong place, or fail half-way without saying so. This tool automates the sorting **safely** and tells the user exactly what happened.

## Objectives

- Classify files by extension into 8 categories (configurable).
- Show a read-only preview before anything changes.
- Move files only after explicit confirmation; never overwrite or delete.
- Report successes, failures and skipped files honestly (including partial failures).
- Keep the code modular and simple enough to explain and defend.

## Technologies

Node.js (ES modules, built-in `fs`/`crypto`/`node:test`), Express 4 (the only dependency), HTML5, CSS3, vanilla JavaScript (ES6+), Git, npm.

## Requirements

- Node.js **18.18 or newer** (developed and tested on Node 22) and npm
- Git (optional, to browse the history)

## Installation

```bash
cd smart-file-organizer
npm install
```

`npm install` also creates `package-lock.json` (it could not be generated while the project was built, because the build environment had no internet access). **Commit that file after your first install:** `git add package-lock.json && git commit -m "Add package-lock.json"`.

## Run the application

```bash
npm start
```

Open **http://localhost:3000**. The server listens on `127.0.0.1` only (not reachable from other computers). To use another port:

- macOS / Linux: `PORT=3001 npm start`
- Windows PowerShell: `$env:PORT=3001; npm start`

## How to use it

1. **Choose a folder** – type its full path (e.g. `C:\Users\You\Downloads` or `/home/you/Downloads`) and click **Scan folder**. *Read-only.*
2. Review the files found. Click **Preview organization**. *Read-only.*
3. Check the preview: category, destination, category totals, and files that will be renamed (`report (1).pdf`) because the name already exists.
4. Tick the confirmation box and click **Confirm and organize files**. *This is the only step that moves files.*
5. Read the result (success / partially completed / failed). Failed or skipped files are listed with reasons and stay where they were.
6. Open the **Reports** section to read any report. Raw JSON files are saved in `reports/`.

### Try it safely with synthetic files

```bash
npm run demo:create
```

This creates a **new temporary folder** of fake sample files and prints its path. Paste that path into the app. Never test with your real documents.

## Run the tests

```bash
npm test
```

See [docs/TESTING.md](docs/TESTING.md) for the strategy and the real results from the build.

## Project structure

```
smart-file-organizer/
├── src/
│   ├── server.js              starts the server
│   ├── app.js                 builds the Express app (security headers, host guard, routes)
│   ├── config/                settings.js, categories.js (add file types here)
│   ├── routes/                routeTable.js, apiHandlers.js, expressRouter.js
│   ├── services/              scanner, classifier, organizer, report
│   └── utils/                 pathValidator, fileOperations, errors, hostGuard
├── public/                    index.html, css/styles.css, js/{api,render,app}.js, favicon.svg
├── tests/                     node:test suites + helpers (temp folders only)
├── scripts/create-demo-folder.js
├── reports/                   generated JSON reports (git-ignored)
└── docs/                      ARCHITECTURE, TESTING, AI_AGENT_WORKFLOW, DEMO_SCRIPT, PRESENTATION_OUTLINE
```

## Safety design (summary)

| Rule | How it is enforced |
|---|---|
| Never move without confirmation | `/api/organize` needs `confirm: true` **and** the `planId` of a preview; otherwise nothing happens |
| Preview is read-only | Scan/preview code never writes; tests compare folder snapshots before/after |
| Folder changed after preview | The `planId` fingerprint no longer matches → request refused (HTTP 409) |
| Never overwrite | Files are moved with hard-link + unlink (or exclusive copy), never `rename`; duplicates become `name (1).ext` |
| Never delete user data | Only the original *name* is removed after the new name exists; on any error the original is kept |
| Path safety | Absolute paths only, `..` rejected, real path resolved, roots/system folders/the app's own folder blocked, symlinks rejected |
| Re-validation | Every file and destination is re-checked immediately before it is moved |
| Own output | Reports are saved in the app's `reports/` folder (never in the organized folder); report-like files are skipped |
| Honest results | Every file ends as *moved*, *failed* or *skipped*; counts add up; status is `success`, `partial`, `failed` or `nothing_to_do` |
| No file-browsing API | Only typed paths; no endpoint lists or reads arbitrary files |
| Local only | Binds to 127.0.0.1, host-header check, strict Content-Security-Policy, JSON-only bodies |

## Known limitations

- **Express was not run during the build.** The build environment had no internet access, so Express could not be installed there. All logic and the browser UI were tested through a Node `http` test harness that shares the same route table and handlers; the thin Express layer (`src/app.js`, `src/routes/expressRouter.js`, `src/server.js`) and its 5 integration tests in `tests/express.integration.test.js` **must be verified on your machine** with `npm install && npm test`.
- Tested only on Linux with Node 22. Windows and macOS behaviour (hard links on NTFS, case-insensitive file systems, symlink tests needing permission) is designed for but **not yet verified**.
- One permission test is skipped when tests run as the root user.
- Top-level files only; subfolders are never entered.
- Hidden files are detected by a leading dot only; Windows "hidden" attributes are not checked.
- There is **no undo button**. Each report lists every moved file with its new location, so moves can be reversed manually.
- The folder is entered as typed text, because browsers cannot reveal real folder paths to a web page.
- Files are moved one at a time; very large folders (tens of thousands of files) will be slow.
- Network (UNC) paths, the filesystem root, system folders and your home folder itself are intentionally refused.
