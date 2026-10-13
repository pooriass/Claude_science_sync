# project-handoff: user guide

Oct 12, 2026 · @Salehi

## 1. What the skill does

You and your colleague use different Claude accounts, so neither Claude can see the other's chats. This skill makes **the project folder on the server** the shared memory instead. Every session reads what happened before, writes down what it does, and leaves an exact next step, so the other person can continue — even if a session stops suddenly because of a token limit.

It does three jobs:

1. **Records** every step, run, decision and failure in a few small text files inside the project.
2. **Runs long jobs safely**: they keep running after the chat ends, and they record their own result (finished, failed, or crashed).
3. **Hands over**: at the start of a session, Claude reads the notes, checks running jobs, and tells you where things stand and what to do next.

&#91;embedded content: how the four places connect · shared folder + imports\]

The two Claude Science accounts never talk to each other directly; they meet in the `.handoff/` notes. Claude web chats are outside the server, so their work enters only when you copy it into `imports/`.

### Words used in this guide

| Word | Meaning |
| --- | --- |
| Server | The machine where the data and code live. Both of you log in to it with your own Linux user. |
| Claude Science | Claude working directly on the server. This is where the skill runs. |
| Claude web | claude.ai in the browser. It runs in its own temporary sandbox and **cannot see the server**. |
| Project folder | One folder on the server that holds the project, e.g. `/srv/shared/visium_pdac` |
| `.handoff/` | The folder the skill adds inside the project. It holds all notes. Nothing else in your project is changed. |
| Helper script | `.handoff/bin/handoff.sh`. Claude uses it for logging, locking and running jobs. You can also use it in a terminal. |
| Lock | A note saying "Ali is working now". It stops two sessions editing the same thing at once. |
| Checkpoint | `CHECKPOINT.md`: what is being done right now and the exact next action. Updated *before* each step. |
| Run | A job started through the helper. It gets an id like `20261013-101200_rctd_s2_ali` and its own log files. |
| Adopt | The one-time step that adds `.handoff/` to a project, including one that is already half done. |
| Resume | What Claude does at the start of each session: read the notes, check jobs, brief you. |

## 2. One-time setup (about 15 minutes)

Do this once. After it, you never repeat it.

1. **Install the skill in both Claude Science accounts.** Add the folder from `project-handoff.zip` as a skill in your account and in your colleague's account. The folder must contain `SKILL.md`, `scripts/` and `references/`. The web accounts do not need it, because Claude web cannot reach the server.
2. **Put both Linux users in one group.** Ask the server admin, or run as admin: `sudo usermod -aG lab ali` and `sudo usermod -aG lab sara` (replace `lab`, `ali`, `sara` with your real names). Check with `id ali`.
3. **Make the project folder shared.** Run this as the owner of the files (or with `sudo`):

```bash
chgrp -R lab /path/to/project
chmod -R g+rwX /path/to/project
find /path/to/project -type d -exec chmod g+s {} +
```

4. **Make new files shared too.** Each of you adds this line to your own `~/.bashrc` once: `umask 002`
5. **Check git exists** on the server: `git --version`. If it is missing, everything still works, but you lose history and rollback of code.

Why steps 2–4 matter: without them, files one person creates can be read-only for the other, and your colleague's Claude will fail with "Permission denied". After adoption, `.handoff/inventory/perms.txt` tells you if anything is still wrong.

## 3. Day 1: adopt your existing project

Your project is already half done, and the work is spread over four places: your Claude web, your Claude Science, your colleague's Claude web and his Claude Science. The Claude Science work is already on the server, so the skill finds it by scanning. The Claude web work is **not** on the server, so each of you must bring it in first.

### Step A — Each person collects their Claude web work (in claude.ai)

Each of you does this in your **own** claude.ai account. Paste the prompt below into a new chat. If your project chats were inside a claude.ai Project, paste it inside that Project too (chat search only looks inside the current Project).

```text
Search my past chats about this research project and write ONE report as a downloadable
Markdown file named report.md.
Keywords to search (use each one): <project name>, <dataset names>, <methods e.g. Visium,
RCTD, CellCharter>, <sample names>, <key genes>.

Rules: include only what is actually in the chats. Mark anything uncertain [unsure].
Do not invent results. Separate "Claude suggested" from "I decided".

Format:
# Web-work report — <my name> — <date>
## Chat list
| date | chat title | link | topic |
## For each chat (oldest first)
### <date> — <title> — <link>
- Goal:
- Data used: which files/samples; full data or a subset; where from?
- Where code ran: (a) in the chat sandbox, (b) copied to server/laptop, (c) never run
- Code produced: file names + purpose (attach FINAL version of each)
- Parameters:
- Results: key numbers, figures, tables (file names downloaded)
- Decisions I made and why:
- Failed attempts / errors:
- Open next steps:
## All code files to download (final versions)
## Contradictions between chats
```

Download the report and every code file it lists.

### Step B — Put the reports on the server

Copy them into the project, one folder per person (create the folders if needed):

```text
/path/to/project/.handoff/imports/ali_web/report.md
/path/to/project/.handoff/imports/ali_web/code/...
/path/to/project/.handoff/imports/sara_web/report.md
/path/to/project/.handoff/imports/sara_web/code/...
```

The `.handoff` folder does not exist yet — that is fine, create the path; adoption keeps what is there. If the reports are not ready, skip this step; you can add them later.

### Step C — Adopt (one person, in Claude Science)

Open Claude Science on the server and type:

> Use project-handoff to adopt /path/to/project. Work was also done in both our Claude web accounts; the reports are in .handoff/imports/.

Claude will then:

1. Ask a few questions (your names, other folders used, whether it may read your own command history).
2. Scan the folder **without moving, renaming or deleting anything**: files, scripts in date order, which file each script reads or writes, software versions, running jobs, git history.
3. Rebuild the history of the project in `BASELINE.md`. Every line gets a tag: `[verified]`, `[inferred]`, `[stated by NAME]` or `[unknown]`.
4. Read the web reports and sort each item: already on the server · run only in a web chat (must be **rerun on the server**) · drafted but never run · result with no code (unknown origin) · decision (goes to `DECISIONS.md`) · conflict between sources (asked to both of you).
5. Ask you only what it could not work out, e.g. *which of `02_rctd.R` and `02_rctd_old.R` is the real one?*
6. Write `STATE.md` (where the project stands) and `TODO.md` (what is next).

### Step D — Your colleague confirms his part

Your colleague opens his Claude Science and types:

> Resume /path/to/project. Then go through BASELINE.md with me and correct the parts about my work.

His corrections are added next to the old text, never over it. After this, both of you are in sync and use the daily routine in section 4.

If you each worked in **different folders** on the server, tell Claude in Step C. It will help you choose one folder as the real one and compare the other against it, without copying anything until you agree.

## 4. Every session: what to type

Every working session has the same shape: **resume → work → hand off**. You only type short messages; Claude does the bookkeeping.

| Moment | What you type in Claude Science | What Claude does |
| --- | --- | --- |
| Start of session | `Resume /path/to/project` | Reads the notes, checks running jobs and unsaved code edits, takes the lock, and gives you a short briefing with the next step |
| During work | Discuss, then `Apply this pipeline` (section 5); small tasks: just ask, e.g. "fix the QC script" | Updates the checkpoint before each step, starts long jobs safely, logs results and decisions after each step |
| You want something recorded | `Log this` or `Add this decision: …` | Writes it to the log or to `DECISIONS.md` |
| End of session | `Handoff` | Rewrites `STATE.md`, updates `TODO.md`, releases the lock, gives you a short message to forward to your colleague |
| Tokens almost gone | `Emergency handoff` | Saves the checkpoint and releases the lock in 2 quick steps |
| Colleague stopped mid-task | `Resume /path/to/project — Ali ran out of tokens, continue his task` | Takes over the lock (and logs why), checks his jobs and half-edited files, continues from his exact next action |
| Check a job | `How are the runs going?` | Shows each job as RUNNING, DONE, FAILED, KILLED or DIED |

### What the start-of-session briefing looks like

```text
Project: visium_pdac — last activity: Sara, 2 h ago (lock now yours).
Interrupted task: RCTD for samples 2–6 (Sara's session ended mid-task).
- Run rctd_s2to6: DONE 40 min ago (exit 0). 5/5 output files present.
- Half-edited: scripts/03_merge_weights.R (column harmonisation missing).
- Unlogged changes since last scan: none.
Next action: finish 03_merge_weights.R, then run it. Shall I continue?
```

### Why a token limit does not lose work

You cannot predict when a session stops, so the skill never depends on a final note. Four things protect you:

1. **Checkpoint first.** Claude writes the next action *before* doing it, so the note is never more than one step old.
2. **Jobs outlive the chat.** Long jobs run detached; when they finish, they write their own result and log entry.
3. **Unsaved code shows up.** Any half-finished edit appears as an uncommitted change in git, and the next Claude shows it to you.
4. **Unlogged changes show up.** A rescan lists every file added, changed or deleted since the last scan.

### When both of you work at the same time

The lock means "one person owns the main task". If you both need to work in parallel, agree on separate parts (different scripts, different output folders). The second person tells Claude: `Work on figure 2 in parallel; use a personal checkpoint`. Both sessions still log everything, and writes to the shared files are safe at the same moment.

## 5. Pipelines: discuss → plan → apply

This is how you already work: you discuss a problem, reach a pipeline, then ask Claude Science to apply it. The skill adds one thing in the middle: **the agreed pipeline is saved as a plan file before anything runs.** Your colleague never sees your chat, but he sees the plan, with its reasons, steps and parameters. So he can review it, apply it, or continue it.

### What you type

| You want to… | Type in Claude Science | What happens |
| --- | --- | --- |
| Run the pipeline you just agreed | `Apply this pipeline` | Claude saves the plan, shows you a short summary (steps, parameters, outputs), waits for your **yes**, then runs it step by step |
| Keep it for later or for your colleague | `Save this pipeline as a plan, don't run it yet` | Saved as `agreed`; nothing runs |
| The discussion is long and not finished | `Save what we have so far as a draft plan` | Saved as `draft`, with the open questions — protects you if tokens run out mid-discussion |
| Run a plan your colleague saved | `Resume <path>. Show me plan 03, then apply it.` | Claude shows it, then applies after your yes |
| Continue a plan someone stopped in the middle | `Resume <path> — continue plan 03` | Claude checks finished steps and jobs, and continues from the first open step |
| Bring a pipeline from a Claude web chat | `Import this plan from my web chat: …` (paste it) | Saved the same way, marked as coming from a web chat |

You only trigger the skill once per session, with `Resume <path>`. After that, the prompts above are enough.

### What a plan looks like

Plans live in `.handoff/plans/`, one file each, numbered: `03_cellcharter.md`, `04_enrichment.md`.

```text
# Plan 03_cellcharter — CellCharter niches
Status: running
Owner: Ali | Created: 2026-10-14
Goal: find spatial niches and test CAF subtype enrichment per niche
Why: niches capture multicellular context; CellCharter handles several samples
     and picks the number of clusters by stability
Rejected: BANKSY — single-sample focus; Leiden on spots — ignores neighbours
Inputs: results/rctd/P0X_weights.csv
Outputs: results/niches/P0X_labels.csv, results/niches/enrichment.csv
Parameters: layers=3, n_clusters=5-12 (auto), seed=42
Environment: conda env cellcharter
## Steps
- [x] 1. Build spatial graphs (squidpy) → results/niches/graphs/ — 6 graphs
- [x] 2. CellCharter clustering per sample → labels — 6 files (run cc_resume)
- [ ] 3. CAF enrichment per niche → results/niches/enrichment.csv
## Success checks
- niches stable across 3 seeds
## Amendments
- A1 2026-10-14 Ali: n_clusters 5-12, because 15 gave empty niches in P04
```

### What Claude does while applying

1. Before each step, it notes in the checkpoint which step it is on and the exact next action.
2. It writes or edits the script and saves it to git.
3. Long steps run as detached jobs, so they keep going if the chat ends.
4. After each step, it ticks the step in the plan with a one-line result.
5. If something has to change (an error, a parameter), it **asks you first**, then adds an amendment like A1 above. It never changes the method silently, so plan, code and results always match.
6. At the end, it checks the success checks and marks the plan `done` or `failed`.

Statuses: `draft` (still being discussed) → `agreed` → `running` → `done` / `failed` / `abandoned`.

Bonus: when you write the Methods section of your paper, each plan already holds the method, parameters, reasons and changes for one analysis.

## 6. Keeping token use low

Version 2 of the skill uses about **60% fewer tokens** than version 1 for the same work. This matters more than it looks: everything Claude reads at the start of a session stays in the conversation and is carried along with every later message.

| Part of a session | Version 1 | Version 2 | Saved |
| --- | --- | --- | --- |
| Skill instructions loaded | \~3,500 | \~1,600 | 53% |
| Start of session (reading the notes) | \~1,700 | \~460 | 72% |
| Each pipeline step (notes + result) | \~380 | \~120 | 69% |
| Emergency handoff | \~390 | \~100 | 75% |
| **Whole session start** | **\~5,100** | **\~2,100** | **60%** |
| **Takeover session with 10 steps** | **\~10,400** | **\~4,100** | **60%** |

Tokens estimated as characters ÷ 4, measured on the test server with a realistic project history (12 runs, 17 log entries).

### What the skill does to save tokens

- **One call to start.** `brief` replaces reading five files. It shows only what changed since *your* last session, so a second session the same day costs even less.
- **Rare instructions stay closed.** Adoption, web-chat imports and takeover rules sit in separate files that Claude opens only in that situation.
- **Short notes.** Checkpoints change only the fields that changed; a pipeline step's result, tick and git save is one short command; every helper command answers in one line.
- **No dumping.** Claude reads job errors and the last lines of output, and only the read/write lines of a script — never whole logs or scripts.
- **No waiting in the chat.** Long jobs run detached; Claude checks them later instead of polling.

### Habits that help you

- **One session per task.** A conversation gets more expensive with every message, because the whole history is carried along. With the handoff notes, a fresh session costs only \~2,100 tokens to catch up, so start a new one for each new task instead of one endless chat.
- **Save plans early.** In a long discussion, say `Save what we have so far as a draft plan` from time to time. If tokens run out, the next session starts from the draft, not from zero.
- **Hand off before the limit.** When you see a usage warning, say `Emergency handoff` right away; it needs only two short commands.
- **Ask for summaries, not dumps.** "Did the run work?" is cheap; "show me the whole log" is not.
- **Very large folder?** Tell Claude `Resume <path> without scanning`. The file scan costs little in tokens but can take minutes on millions of files.

## 7. What is inside `.handoff/`

You can open any of these files yourself. If you only read two, read `STATE.md` and `CHECKPOINT.md`.

| File | What it holds | How it changes |
| --- | --- | --- |
| `STATE.md` | One-page snapshot: goal, current stage, jobs running, next command, known problems | Rewritten at every handoff |
| `CHECKPOINT.md` | The task in progress: sub-steps with ticks, half-edited files, exact next action | Rewritten before and after each step |
| `TODO.md` | Now / Next / Blocked / Done | Edited |
| `LOG.md` | Everything that happened, oldest at top, newest at bottom | Only added to, never edited |
| `BASELINE.md` | The rebuilt history of work done before you adopted the skill | Written once, corrected with tags |
| `PIPELINES.md` | Every script: purpose, inputs, outputs, status (done, partial, running, failed, planned, obsolete) | Edited |
| `DECISIONS.md` | Why each choice was made (parameters, methods, rejected options) | Added to |
| `DATA_MANIFEST.tsv` | Key data files: role (raw, reference, final…), size, checksum, who, when | Added to |
| `RUNS.tsv` and `runs/` | Every job: command, logs, exit code, duration | Automatic |
| `LOCK` | Who is working now | Automatic |
| `imports/` | Reports and code from Claude web chats or laptops | You add them; Claude marks each one done |
| `inventory/` | Automatic scans: file list, scripts, software, running jobs, permissions, changes since last scan | Automatic |
| `config.env` | Settings, e.g. hours before a lock counts as abandoned (default 3) | Rarely edited |
| `bin/handoff.sh` | The helper script both of you use | Copied in at adoption |

Also: `plans/` holds one file per agreed pipeline (section 5).

Your data is **not** copied into git. Git only keeps code, notes and `.handoff/`, so the history stays small. Data stays where it is, and `DATA_MANIFEST.tsv` records it.

## 8. Helper commands (optional)

You never *have* to type these — Claude runs them for you. They are useful when you want to check things yourself in a terminal. First `cd /path/to/project`, then:

| Command | What it does |
| --- | --- |
| `.handoff/bin/handoff.sh brief` | Start of session: lock, checkpoint, open plans, jobs, unsaved code, and what changed since your last brief |
| `.handoff/bin/handoff.sh runs` | List jobs and their state |
| `.handoff/bin/handoff.sh show <run id>` | A job's details plus the last lines of its output and errors |
| `.handoff/bin/handoff.sh rescan` | Find files changed since the last scan (result in `inventory/drift_latest.txt`) |
| `.handoff/bin/handoff.sh run --name qc -- Rscript scripts/01_qc.R` | Start a job that survives logout, with logs and status |
| `echo "what I did" \| .handoff/bin/handoff.sh log NOTE "short title"` | Add a log entry by hand |
| `.handoff/bin/handoff.sh lock --task "what I will do"` | Claim the project |
| `.handoff/bin/handoff.sh takeover --reason "Ali out of tokens"` | Take the lock from your colleague (logged) |
| `.handoff/bin/handoff.sh stop <run id> --reason "…"` | Stop one of **your** jobs (logged) |
| `.handoff/bin/handoff.sh manifest <file> final "description"` | Record an important data file with its checksum |
| `echo "summary" \| .handoff/bin/handoff.sh end` | Log the handoff, save to git, release the lock |
| `.handoff/bin/handoff.sh perms` | Check whether both of you can write all files |
| `.handoff/bin/handoff.sh help` | Full list |

Plan commands (Claude normally runs these for you):

```bash
.handoff/bin/handoff.sh plan list                     # all plans: status, steps done, next step
.handoff/bin/handoff.sh plan show 03                  # steps + parameters (add --full for the reasons)
.handoff/bin/handoff.sh plan step 03 2 done -m "6 label files" --run cc_resume
.handoff/bin/handoff.sh plan amend 03 -m "n_clusters 5-12: 15 gave empty niches"
.handoff/bin/handoff.sh plan set 03 done -m "success checks passed"
```

Job states: **RUNNING** (still going) · **DONE** (finished, exit 0) · **FAILED** (finished with an error) · **KILLED** (stopped on purpose) · **DIED** (vanished with no result — usually out of memory or a server restart; its output may be half-written) · ORPHANED (the job's wrapper was killed but its work is still running — do not start it again).

## 9. Rules for both of you

**Do**

- Start every Claude Science session with `Resume /path/to/project` and end it with `Handoff`.
- Run anything that produces a result **on the server**, through Claude Science, so it is logged.
- Use Claude web for thinking, reading papers and drafting. If a web chat produces code or a decision you will use, bring it to the server the same day (paste it to Claude Science and say "import this from my web chat").
- Before discussing the project in a Claude web chat, paste the relevant part of `STATE.md` into it, so the web chat starts from the real state.
- Give new versions new names (`_v2`) instead of overwriting.
- Make long scripts resumable: one output file per sample, and skip samples that are already done. Claude will suggest this. For multi-step pipelines, Snakemake or Nextflow do this for you.

**Don't**

- Don't rename or reorganise folders in the middle of the project — scripts contain paths and will break.
- Don't take over your colleague's lock unless you know he stopped.
- Don't restart a job that shows RUNNING.
- Don't trust the output of a DIED or KILLED job before checking it.
- Don't treat a result from a web chat as final until it has been regenerated on the server.
- Don't put passwords, tokens or API keys in any note or command.

## 10. Troubleshooting

| Problem | What it means | What to do |
| --- | --- | --- |
| "Project is LOCKED by Ali" | Ali's session holds the lock, and was active recently | Ask Ali. If he stopped or ran out of tokens, tell Claude so; it takes over and logs the reason |
| "Lock is STALE" | No activity from the lock holder for 3 hours | Tell Claude to take over |
| A job shows **DIED** | It vanished without a result (out of memory, server restart, killed hard) | Ask Claude to check its output and rerun it; resumable scripts skip finished samples |
| A job shows **FAILED** | The script ended with an error | Ask Claude to show the errors (`show <run id>`) and fix the cause |
| "Could not signal pid … owned by another Unix user" | You tried to stop your colleague's job | Only he (or an admin with `sudo`) can stop it |
| "Permission denied" on a file | The one-time setup in section 2 is missing for that file | Run the commands from `.handoff/inventory/perms.txt` |
| "No .handoff/ found" | Claude is not in the project folder, or the project was never adopted | Give the full path: `Resume /path/to/project`, or adopt first |
| "Already adopted" | Someone already adopted this project | Use `Resume` instead |
| "IMPORT NOT YET RECONCILED" in status | A web-chat report in `imports/` has not been merged yet | Tell Claude: `Reconcile the imports` |
| Claude did not log something | Claude forgot, or the session stopped before logging | Say `Log this`; at the next resume, a rescan also catches unlogged file changes |
| Claude did not use the skill at all | The skill did not trigger | Start your message with `Use project-handoff to …` |
| Your colleague's notes are wrong | Claude guessed from file dates (`[inferred]`) | Correct it; Claude adds a `[stated by NAME]` correction beside the old text |

## 11. Example: two days, two people

Ali and Sara share `/srv/shared/visium_pdac`. (These steps were tested on a test server with two real users.)

**Day 1, morning — Ali**

1. Types `Resume /srv/shared/visium_pdac`. Claude: lock is free, last step was QC; next is RCTD.
2. Says "run RCTD on samples 1–3". Claude writes the plan into the checkpoint, then starts the job as a detached run (`rctd_ali`). It writes one file per sample.
3. While it runs, Ali starts editing `03_merge.R`. Then his tokens run out. No handoff happened.

**What is on the server at that moment:** the checkpoint says "waiting for `rctd_ali`; then finish `03_merge.R`". The job is still running. The half-edited script is visible to git. The lock still says Ali.

**Day 1, evening — Sara**

4. Types `Resume /srv/shared/visium_pdac — Ali ran out of tokens`. Claude's briefing: Ali's lock (takes it over, logged); `rctd_ali` **DIED** after sample 1 (server ran out of memory); `03_merge.R` has an unfinished edit; next action from the checkpoint.
5. Claude checks sample 1's file is complete, then reruns RCTD. The script skips sample 1 and does 2 and 3. Status: DONE.
6. Claude shows Sara Ali's half-edit. She says "finish it". Claude completes it and saves it to git as *"finish 03\_merge.R (started by Ali)"*.
7. Sara types `Handoff`. `STATE.md` now says: RCTD done for 1–3; merge script ready; next command written out.

**Day 2 — Ali**

8. Types `Resume …`. The briefing tells him everything Sara did, in about 6 lines. He continues from the next command. He never had to ask Sara anything.

## 12. Full walkthrough: a real project, start to finish

This is a complete run-through of a realistic case, with every prompt to copy, every terminal command, and what Claude typically answers. Replace names, paths and keywords with your own. Sample replies are shortened; your Claude's wording will differ.

### The situation

You and Ali analyse 6 Visium PDAC samples (P01–P06) to map CAF subtypes in tumour niches. The work so far is scattered:

| Where | Who | What happened there |
| --- | --- | --- |
| Your Claude Science | You | QC of all 6 samples in `/srv/shared/visium_pdac`; outputs in `data/processed/` |
| Your Claude web | You | Drafted the RCTD reference script; tested it on a P01 subset you uploaded |
| Ali's Claude web | Ali | Discussed CellCharter settings; decided on 3 neighbourhood layers |
| Ali's Claude Science | Ali | Ran RCTD on P01–P02, but in his own copy `/home/ali/visium`, not the shared folder |

Goal: one shared folder that both Claude Science accounts read, with nothing lost.

### Step 1 — One-time server setup (you, terminal, 5 minutes)

Log in to the server and run (ask an admin for the `sudo` lines if needed):

```bash
sudo groupadd -f lab
sudo usermod -aG lab you
sudo usermod -aG lab ali
sudo chgrp -R lab /srv/shared/visium_pdac
sudo chmod -R g+rwX /srv/shared/visium_pdac
sudo find /srv/shared/visium_pdac -type d -exec chmod g+s {} +
echo 'umask 002' >> ~/.bashrc        # Ali runs this line too, in his own login
```

Then both of you log out and in again, so the new group takes effect. Both of you install `project-handoff.zip` as a skill in your Claude Science accounts.

### Step 2 — Collect the web-chat work (each person, in claude.ai)

You open claude.ai and paste this into a **new chat** (keywords filled in for this project):

```text
Search my past chats about this research project and write ONE report as a downloadable
Markdown file named report.md.
Keywords to search (use each one): Visium, PDAC, RCTD, spacexr, CellCharter, DestVI,
CAF, myCAF, iCAF, basal, classical, P01, P02, P03, P04, P05, P06, deconvolution, niche.

Rules: include only what is actually in the chats. Mark anything uncertain [unsure].
Do not invent results. Separate "Claude suggested" from "I decided".

Format:
# Web-work report — <my name> — <date>
## Chat list
| date | chat title | link | topic |
## For each chat (oldest first)
### <date> — <title> — <link>
- Goal:
- Data used: which files/samples; full data or a subset; where from?
- Where code ran: (a) in the chat sandbox, (b) copied to server/laptop, (c) never run
- Code produced: file names + purpose (attach FINAL version of each)
- Parameters:
- Results: key numbers, figures, tables (file names downloaded)
- Decisions I made and why:
- Failed attempts / errors:
- Open next steps:
## All code files to download (final versions)
## Contradictions between chats
```

A typical result (shortened):

```text
# Web-work report — You — 2026-10-12
## Chat list
| 2026-09-14 | RCTD reference from scRNA atlas | <link> | RCTD |
| 2026-09-20 | Testing RCTD on P01            | <link> | RCTD |
### 2026-09-20 — Testing RCTD on P01
- Data used: P01 subset, 500 spots, uploaded as CSV
- Where code ran: (a) in the chat sandbox
- Code produced: 02_rctd_ref.R (final version attached)
- Parameters: UMI_min = 100, doublet_mode = "full"
- Results: myCAF and iCAF separated; 8 cell types detected
- Decisions I made: use k = 8 clusters in the reference
## Contradictions between chats
- UMI_min was 50 on 09-14, changed to 100 on 09-20 [unsure which is final]
```

Download `report.md` and `02_rctd_ref.R`. Ali does the same in his account and gets his own report.

### Step 3 — Copy the reports to the server (each person, from your laptop)

```bash
ssh you@server "mkdir -p /srv/shared/visium_pdac/.handoff/imports/you_web/code"
scp report.md you@server:/srv/shared/visium_pdac/.handoff/imports/you_web/
scp 02_rctd_ref.R you@server:/srv/shared/visium_pdac/.handoff/imports/you_web/code/
```

Ali does the same into `imports/ali_web/`. (WinSCP or FileZilla work too, if you prefer dragging files.)

### Step 4 — Adopt the project (you, in Claude Science)

Open Claude Science on the server and type:

```text
Use project-handoff to adopt /srv/shared/visium_pdac.
I am <your name>, my colleague is Ali.
Work was also done in both our Claude web accounts; the reports are in .handoff/imports/.
Ali also ran RCTD in his own folder /home/ali/visium.
You may scan my own shell history.
```

Claude scans, reads the reports and the scripts, then asks only what it could not work out. Typical questions and short answers:

```text
Claude: 1. scripts/02_rctd_ref.R (server) and imports/you_web/code/02_rctd_ref.R differ:
           UMI_min = 50 vs 100. Which is final?
        2. Which outputs are final, and which are exploratory?
        3. Are Ali's RCTD results in /home/ali/visium the ones to keep?
        4. What is the next step?
You:    1. 100.  2. QC outputs in data/processed are final; results/test_* are exploratory.
        3. Ali should answer that.  4. RCTD for all 6 samples, then CellCharter.
```

Then Claude reports what it built (shortened):

```text
Adopted /srv/shared/visium_pdac. BASELINE.md and PIPELINES.md written.
Imports reconciled:
- 02_rctd_ref.R: also on server; your web version (UMI_min=100) is canonical [stated by you]
- RCTD test on P01 subset: run only in a web chat → status "web-only, rerun needed"
- k = 8 reference clusters → DECISIONS D1 [web-chat: you, 2026-09-20]
- CellCharter 3 layers → DECISIONS D2 [web-chat: Ali]
- /home/ali/visium: RCTD for P01–P02 → open question for Ali (keep or rerun?)
Next (TODO): RCTD for P01–P06, then CellCharter.
```

Type `Handoff` to finish the session.

### Step 5 — Ali confirms his part (Ali, in his Claude Science)

```text
Resume /srv/shared/visium_pdac.
Then go through BASELINE.md with me and correct the parts about my work.
My RCTD results in /home/ali/visium used the old reference with UMI_min = 50.
```

Claude records his answer as `[stated by Ali]` and marks his P01–P02 results `obsolete` (they used the old setting). Now both accounts share one picture of the project.

### Step 6 — Normal work (you, Claude Science)

```text
Resume /srv/shared/visium_pdac
```

Then:

```text
We agreed: RCTD on all six samples with the canonical reference (k = 8, UMI_min = 100),
one output file per sample, then merge. Apply this pipeline.
```

Claude saves the pipeline as plan `04_rctd_all` and shows a short summary. You answer `yes`. It then makes the script skip finished samples, saves it to git, and starts step 1 as a detached job:

```text
Plan 04_rctd_all (agreed):
  1. RCTD per sample → results/rctd/P0X_weights.csv
  2. Merge → results/rctd_all.csv
  Parameters: k=8, UMI_min=100. Apply now?
You: yes
Step 1 started as run 20261014-101200_rctd_all_you (detached; ~3 h).
Checkpoint updated. While it runs, shall I draft the CellCharter script?
```

You say yes. Halfway through the CellCharter draft, **your tokens run out**. No handoff happened.

### Step 7 — Ali takes over (Ali, Claude Science, same evening)

```text
Resume /srv/shared/visium_pdac — my colleague ran out of tokens, continue his task.
```

Briefing from Claude:

```text
Lock: held by you, heartbeat 2 h ago → taken over by Ali, reason logged.
Checkpoint: plan 04_rctd_all step 1/2 · half-edited: scripts/04_cellcharter.R (neighbourhood step missing)
Open plans: 04_rctd_all running 0/2 · next: 1. RCTD per sample
Runs: rctd_all_you DIED (no exit recorded; likely out of memory). P01–P04 present, P05–P06 missing.
Next (from checkpoint): rerun step 1 (skips P01–P04), then finish 04_cellcharter.R.
```

Ali: `Continue. Use fewer cores so it fits in memory.` Fewer cores changes the plan, so Claude adds amendment A1 to plan 04 (what changed and why). It reruns the job, which skips P01–P04 and finishes P05–P06, and ticks step 1. Claude logs the memory problem as a `FAIL` entry, so nobody repeats it. Then Ali types:

```text
Finish 04_cellcharter.R using our CellCharter decision, then Handoff.
```

### Step 8 — Next morning (you)

```text
Resume /srv/shared/visium_pdac
```

You get a short briefing: RCTD complete for P01–P06, the memory fix, the finished CellCharter script, and the exact command to run it. You continue from there without messaging Ali.

### Step 9 — Using a web chat later, the safe way

You want to brainstorm figure layouts in claude.ai. First copy the state from the server:

```bash
cat /srv/shared/visium_pdac/.handoff/STATE.md
```

Paste it into the web chat with your question. If the chat produces a decision you will use, bring it back the same day. In Claude Science:

```text
Import this decision from my web chat: Figure 2 shows niche maps for P02 and P05 only,
because they have the clearest basal/classical split. Add it to DECISIONS.
```

### All prompts from this example

| When | Where | Prompt |
| --- | --- | --- |
| Collect web work | claude.ai | The report prompt in Step 2 |
| First setup | Claude Science | `Use project-handoff to adopt <path>. I am …, my colleague is …. Reports are in .handoff/imports/.` |
| Colleague confirms | Claude Science | `Resume <path>. Then go through BASELINE.md with me and correct the parts about my work.` |
| Start of any session | Claude Science | `Resume <path>` |
| You agreed on a pipeline | Claude Science | `Apply this pipeline` |
| Save a pipeline for later | Claude Science | `Save this pipeline as a plan, don't run it yet` |
| Long discussion, not finished | Claude Science | `Save what we have so far as a draft plan` |
| Run or continue a saved plan | Claude Science | `Resume <path> — continue plan 04` |
| Colleague stopped mid-task | Claude Science | `Resume <path> — my colleague ran out of tokens, continue his task.` |
| Record something | Claude Science | `Log this: …` or `Add this decision: …` |
| Bring in web-chat work | Claude Science | `Import this from my web chat: …` |
| End of session | Claude Science | `Handoff` |
| Almost out of tokens | Claude Science | `Emergency handoff` |
