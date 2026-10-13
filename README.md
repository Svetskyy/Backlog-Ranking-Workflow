# Backlog Ranking Workflow (n8n + Claude)

An n8n workflow that turns a messy job backlog into a ranked work order for the next fortnight, with a one-line reason for every job. Jobs it is not sure about go to a person instead of being guessed.

I built this as a paid technical project for a field services company in Australia. This repo is the public version: **the client's name and data are removed, and the sample data here is made up.**

---

## The idea in two lines

- **The AI reads, the code decides.** Claude only pulls dates and conditions out of free-text notes. Plain JavaScript checks its answers, ranks the jobs and writes the reasons.
- **Gate first, then queue.** A job is ranked only if nothing about it is uncertain. Everything else goes to a **Needs Review** list with the reason and the text that caused it.

---

## What it does

1. **Reads `backlog.csv`**, an export from the CRM.
2. **Checks the Rules tab** in a Google Sheet. If a rule has a mistake, the run stops.
3. **Removes closed jobs** (for example Completed).
4. **Flags problems** it can see without AI: a state that doesn't match the address, paid inspections, bad dates, unknown statuses, duplicate job numbers.
5. **Uses Claude only to read the notes** and return structured fields: date windows, weekday limits, time of day, notice periods, and things that must happen before booking.
6. **Checks every AI answer.** Each quote must really appear in the notes, each date must be a real date, and a written weekday must match its date. An answer that fails goes to Needs Review.
7. **Code ranks the jobs** and writes three tabs: **Ranked**, **Needs Review** and a **Run Log** row.

**Ranking order** (editable in the Rules tab): Director Priority → Education jobs inside that state's school holidays → In Progress → everything else. Inside each group: oldest Created Date first, then Job Number.

**Needs Review has two categories:**

- **Not this fortnight:** the job can't happen in this window but has a clear later date.
- **Needs human decision:** a person must look at it (data problem, a precondition such as a purchase order, a date window that has passed, unclear timing).

**Result on the original private dataset:** 115 rows read, 53 closed, 62 open → **40 Ranked**, **22 Needs Review**. The same result came back on a second run.

---

## How it's built

**Main workflow**

| #     | Node                             | What it does                                                                                 |
| ----- | -------------------------------- | -------------------------------------------------------------------------------------------- |
| 1     | Schedule Trigger                 | Starts the run                                                                               |
| 2     | Read Rules Tab                   | Reads the Rules tab                                                                          |
| 3     | Validate Rules                   | Checks the rules and turns them into one settings item. Stops if something is wrong          |
| 4     | Get and Reads backlog.csv        | Reads the file                                                                               |
| 5     | Checks backlog.csv               | Turns the CSV into one item per row                                                          |
| 6     | Filter and Flag Rows             | Stops on an empty CSV or missing columns. Removes closed jobs, flags problems, picks AI jobs |
| 7     | Needs AI?                        | Sends jobs with real notes to the AI                                                         |
| 8     | AI: Read Notes                   | Claude reads the notes and returns structured fields                                         |
| 9     | Merge LLM Results                | Joins the AI answers back to the jobs and checks they can be trusted                         |
| 10    | Categorize and Rank              | Decides Ranked or Needs Review, ranks, writes reasons                                        |
| 11    | Counts Check                     | Final safety check. Nothing is written if it fails                                           |
| 12–13 | Clear Ranked, Clear Needs Review | Empties the two tabs (keeps the header row)                                                  |
| 14–15 | Ranked Rows, Append Ranked       | Writes the Ranked tab                                                                        |
| 16–17 | Review Rows, Append Needs Review | Writes the Needs Review tab                                                                  |
| 18–19 | Run Log Row, Append Run Log      | Adds a "Success" row to the Run Log                                                          |

**Error workflow (separate):** Error Trigger → Build Alert → Send an alert message (Gmail) → Log Failure (adds a "Failed" row to the Run Log).

Two things to know before editing:

- **The "false" output of _Needs AI?_ has no wire on purpose.** Merge LLM Results reads all jobs from Filter and Flag Rows, so jobs that skip the AI are never lost.
- **The code refers to other nodes by name.** Renaming a node breaks the code that uses it.

---

## Safety layers

| Layer                                | What it catches                                                                                                            |
| ------------------------------------ | -------------------------------------------------------------------------------------------------------------------------- |
| Rules check (Validate Rules)         | Mistakes in the Rules tab. Stops before anything else runs                                                                 |
| CSV check (Filter and Flag Rows)     | Empty export, missing or renamed columns                                                                                   |
| Job flags                            | Bad dates, state/address mismatch, unknown status, duplicate or missing Job Number, paid inspections                       |
| AI answer checks (Merge LLM Results) | Made-up quotes, dates that don't exist, weekdays that don't match the date                                                 |
| AI failure guard (Counts Check)      | AI down, out of balance or giving bad answers. If more than 20% of calls fail (and at least 3), stop and keep the old list |
| Counts Check                         | Every job lands in exactly one tab: none dropped, none doubled, ranks 1..N. Nothing is written unless this passes          |
| Error workflow                       | A plain-English email to staff and a "Failed" row in the Run Log                                                           |

### The bug that shaped the design

In testing, the AI account ran out of balance. Every AI call failed, every job went to Needs Review, the good list was overwritten, and **the run still reported "Success"**. The fix was the AI failure guard above: when too many calls fail, the run stops before it writes anything, and the previous list stays.

### Error emails say whether the list is still safe

| Where it failed                              | Is the list safe?                                                 |
| -------------------------------------------- | ----------------------------------------------------------------- |
| Any step **before** Clear Ranked             | **Yes.** The tabs still show the last successful run              |
| While clearing or writing the two list tabs  | **No.** A tab may be empty or half-filled. Use Version history    |
| Run Log Row, Append Run Log                  | **Yes.** The tabs were updated; only the log entry is missing     |

---

## Changing the rules (no workflow editing)

All rules live in the **Rules tab**: `Rule | State | Value | From | To | Notes`. Capitals and extra spaces don't matter. Dates are `dd/mm/yyyy`.

| To…                            | Do this                                                                                      |
| ------------------------------ | -------------------------------------------------------------------------------------------- |
| **Add a public holiday**       | Add a row: Rule `Public holiday`, State, Value = holiday name, From and To = the date        |
| **Change the planning window** | Change From and To on the `Planning window` row                                              |
| **Swap the rank order**        | Swap the Values of the `Rank step 1`, `Rank step 2` and `Rank step 3` rows                   |
| **Add a status**               | Add a row: Rule `Open status` (rank it) or `Closed status` (skip it), Value = the stage name |

If a rule is wrong (unknown rule name, missing date, To before From, no planning window, a status listed as both Open and Closed), the run stops, the email names the row to fix, and the old list stays.

---

## Try it yourself

You need Docker, a Google account and an Anthropic API key.

1. **Start n8n:** run `docker compose up -d` in this folder, then open **http://localhost:5679**. The sample backlog is mounted read-only at `/files/backlog.csv`.
2. **Make a Google Sheet** with four tabs: `Rules`, `Ranked`, `Needs Review`, `Run Log`.
   - **Rules:** import `sample/rules-tab.csv`.
   - **Ranked** header row: `Rank | Job Number | Organization | State | Sector | Status | Created Date | Rank Step | Allowed Dates | Booking Notes | Reason | Run ID`
   - **Needs Review** header row: `Job Number | Organization | State | Sector | Status | Created Date | Issue Category | Issue | Source Text | Run ID`
   - **Run Log** header row: `Run ID | Started | Finished | Result | Rows Read | Closed Skipped | Open Jobs | Ranked | Needs Review | LLM Calls | LLM Failures | Error`
3. **Import both workflows** from `workflows/`.
4. **Connect credentials:** Google Sheets, Gmail and Anthropic. In every Google Sheets node, replace `YOUR_GOOGLE_SHEET_ID` with your Sheet's ID. In the Gmail node, replace `you@example.com`.
5. **Link the error workflow:** in the main workflow's settings, choose *Backlog Ranking: Error Alert* as the Error Workflow.
6. **Run it:** open the main workflow and click **Execute workflow**.

**About the sample data:** `sample/backlog.csv` has 24 invented jobs: 3 closed and 21 open. Of the open jobs, 12 have notes for the AI to read and 4 are flagged by code before the AI is called (state mismatch, paid inspection, unknown status, bad date). The Rules tab uses real 2026 school and public holiday dates for VIC, NSW and QLD, and a planning window of 5–16 October 2026.

---

## Known limits

- **No fallback AI model.** If Claude is down or the balance runs out, the run stops and keeps the old list.
- **If 0 jobs need the AI, that branch doesn't run**, and the run ends without writing anything.
- **The planning window is updated by hand** before each run.
- **Runs can overlap** if the schedule is shorter than a run. Scheduled runs ignore pinned data and call the AI for real.
- **Writing to the Sheet isn't all-or-nothing.** A failure partway through can leave a tab half-filled. The error email says so.
- **Google logins expire after about 7 days** while the Google Cloud app is in "Testing" mode.
- **An empty Rules tab** ends the run without writing anything and without an email.
- **The State column isn't checked against a list**, so a typo such as `QDL` is only caught for education jobs.

## What I'd add next

- Dropdowns on the Rules tab, so rule names can't be misspelled
- A fallback AI model
- A planning window that updates itself from the run date
- A snapshot of the Rules in each Run Log row
- A "Review Decisions" tab, so people can mark "rank anyway" on Needs Review jobs

---

## Stack

n8n 2.38 (self-hosted in Docker) · JavaScript (Code nodes) · Claude via the Anthropic API · Google Sheets · Gmail

Built with help from Claude Code and Claude chat for planning, code and explanations.
