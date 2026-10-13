# project-handoff · Cheat Sheet

Two people, two Claude accounts, one project folder on one server.
The skill keeps shared notes in `<project>/.handoff/`, so either person can see what was done and continue the other's work — even work cut off by a token limit.

> Claude web (claude.ai) cannot see the server. Run everything that produces results in **Claude Science**.

---

## Setup (once)

**1. Install the skill** in both Claude Science accounts (folder with `SKILL.md`, `scripts/`, `references/`).

**2. Share the folder** between both Linux users:

```bash
sudo usermod -aG lab you && sudo usermod -aG lab colleague   # same group for both
sudo chgrp -R lab /path/to/project
sudo chmod -R g+rwX /path/to/project
sudo find /path/to/project -type d -exec chmod g+s {} +
echo 'umask 002' >> ~/.bashrc                                  # each user, once
```

**3. Adopt the project** (works on half-done projects; nothing is moved or deleted):

```text
Use project-handoff to adopt /path/to/project. I am <name>, my colleague is <name>.
```

Work done in Claude web chats? Collect it first: see [Web-chat work](#web-chat-work).

---

## Daily prompts

| When | Type in Claude Science |
|---|---|
| Start of every session | `Resume /path/to/project` |
| You agreed on a pipeline | `Apply this pipeline` → Claude saves a plan, shows it, runs after your **yes** |
| Keep a pipeline for later | `Save this pipeline as a plan, don't run it yet` |
| Long discussion, not finished | `Save what we have so far as a draft plan` |
| Run / continue a saved plan | `Resume /path/to/project — continue plan 03` |
| Colleague stopped mid-task | `Resume /path/to/project — my colleague ran out of tokens, continue his task` |
| Record something | `Log this: …` · `Add this decision: …` |
| Bring in web-chat work | `Import this from my web chat: …` |
| End of session | `Handoff` |
| Tokens almost gone | `Emergency handoff` |
| Large folder (millions of files) | `Resume /path/to/project without scanning` |

---

## Pipeline flow

```text
discuss  →  "Apply this pipeline"  →  plan saved + shown  →  "yes"  →  steps run, each ticked
                                                            ↳ change needed? Claude asks, records an amendment
```

A plan (`.handoff/plans/NN_name.md`) holds: goal · why · rejected options · inputs · outputs · parameters · environment · steps · success checks · amendments.
Statuses: `draft` → `agreed` → `running` → `done` / `failed` / `abandoned`.

---

## Web-chat work

Paste into a **new claude.ai chat** in each account (edit the keywords):

```text
Search my past chats about this research project and write ONE report as a downloadable
Markdown file named report.md. Keywords: <project>, <datasets>, <methods>, <samples>.
Include only what is in the chats; mark uncertain items [unsure]; separate "Claude suggested"
from "I decided". Per chat: date, link, goal, data used, where code ran (chat sandbox /
server / never), code files (attach final versions), parameters, results, decisions,
failed attempts, open next steps. End with: all code files to download; contradictions.
```

Copy the report + code to `/path/to/project/.handoff/imports/<name>_web/`. Claude merges it during adoption. Results computed only in a web chat get rerun on the server.

---

## What's in `.handoff/`

| File | Purpose |
|---|---|
| `STATE.md` | 1-page snapshot: goal, stage, next command, problems |
| `CHECKPOINT.md` | Task in progress + exact next action (written *before* each step) |
| `plans/` | One file per agreed pipeline |
| `LOG.md` | Everything that happened (append-only) |
| `DECISIONS.md` | Why choices were made |
| `TODO.md` | Now / Next / Blocked / Done |
| `BASELINE.md` | Reconstructed history from before adoption |
| `PIPELINES.md` | Every script: inputs, outputs, status |
| `DATA_MANIFEST.tsv` | Key data files with checksums |
| `runs/`, `RUNS.tsv` | Every job: command, logs, exit status |
| `imports/` | Reports from Claude web chats |

Code and notes are versioned in git; data is never committed.

---

## Helper commands (optional, terminal)

```bash
cd /path/to/project
HS=.handoff/bin/handoff.sh
$HS brief                         # what changed since your last session
$HS runs                          # jobs and their state
$HS show <run_id>                 # errors + last output lines of a job
$HS plan list                     # plans: status, progress, next step
$HS plan show 03                  # one plan's steps + parameters
$HS run --name qc -- Rscript scripts/01_qc.R    # job that survives logout
$HS log NOTE "title" -m "details" # add a log entry
$HS takeover --reason "colleague out of tokens"
$HS end -m "summary"              # handoff: log, git save, release lock
$HS help                          # everything else
```

**Job states:** `RUNNING` · `DONE` · `FAILED` (error) · `KILLED` (stopped on purpose) · `DIED` (gone, e.g. out of memory; output may be half-written) · `ORPHANED` (wrapper gone, work still running — don't restart)

---

## Rules

- ✅ `Resume` at the start, `Handoff` at the end — every session.
- ✅ One session per task; fresh sessions are cheap (~2k tokens to catch up).
- ✅ New versions get new names (`_v2`); raw data is read-only.
- ❌ Don't rename or reorganise folders mid-project.
- ❌ Don't restart a `RUNNING` job; don't trust output of a `DIED` job before checking it.
- ❌ Don't take over a lock unless the other person stopped.
- ❌ No passwords, tokens or API keys in notes or commands.

---

## Troubleshooting

| Message | Fix |
|---|---|
| `LOCKED by <name>` | Ask them. If they stopped, tell Claude; it takes over and logs why |
| `Lock is STALE` | No activity for 3 h: tell Claude to take over |
| `Permission denied` | Run setup step 2; see `.handoff/inventory/perms.txt` |
| `No .handoff/ found` | Give the full path, or adopt first |
| `IMPORT not reconciled` | Tell Claude: `Reconcile the imports` |
| Skill didn't trigger | Start with `Use project-handoff to …` |
| Can't stop colleague's job | Only he (or `sudo`) can stop processes he started |
