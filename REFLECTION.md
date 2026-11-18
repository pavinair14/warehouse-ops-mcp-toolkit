# Reflection: Warehouse Ops MCP Toolkit

## 1. What I built

A Python MCP server with **10 tools** (4 read-only, 6 state-changing) over the supplied stock
and dock data. Ops asked for 5. The site comes from server configuration and no tool takes a
site. Every write is a two-step preview and token. The repository virtual environment
(Python 3.13.14) run passed all 292 automated tests, including tests that drive the server
through the protocol with a real client. The checked-in `evidence/pytest_output.txt` also
records 292 passing tests.

**Evidence.** The main evidence is a run in **Claude Code**, the host named in `plan.md`
(`evidence/claude_code/`: a written transcript, screenshots, the server's own logs and the
registration files). I ran the demos in VS Code with GitHub Copilot first, then repeated
everything in Claude Code. The Copilot material is archived in `evidence/archive_copilot/`,
and I use it only for the specific findings that came from it, which I say where I do.

`CHANGES.md` has the full list of changes, each with its evidence. This document covers what
I learned and where the design is weak.

## 2. What the data forced

I read `stock_snapshot.csv` before designing the lookup, and it decided the interface.

- **12 of 16 SKUs have a same-site twin** that differs by one word or number (SKU-8801/8802
  clear vs black wrap, and five other pairs). A lookup that returns a best match would return
  the wrong product for 75% of the catalogue. So `lookup` became `get_stock` (exact SKU) and
  `search_stock` (returns every candidate).
- **Ambiguity depends on the site.** "Void fill" matches two products across the company but
  exactly one at Leeds (SKU-8811) and one at Glasgow (SKU-8810). "Pallet" matches four
  products at Leeds. A design that always asks fails the first case, and one that never asks
  fails the second, so the search decides after filtering by site.
- **Zero stock is real data.** SKU-8809 has quantity 0 while its sibling SKU-8808 has 150.
  Treating 0 as "not found" would make an agent quote 150 for the 600mm film.
- **Every booking is Northbound Freight, and Leeds and Reading have two each.** So bookings
  are identified by date and slot, never by carrier.
- **A SKU lives at exactly one site, and bin names carry a site prefix.** That made a
  cross-site refusal decidable, and showed that `to_bin` on a move is also a cross-site route.

## 3. Design decisions and why

- **Site scoping without a parameter.** A `site` argument has to be trusted or verified.
  A parameter that does not exist cannot be misused. The server reads the site from its
  environment and refuses to start without a valid one. A `site` argument is rejected.
  Tests run all 16 SKUs against each of the three sites in one session, so same-site access
  and cross-site refusal are proven together.
- **`SITE_ACCESS_DENIED`, not `NOT_FOUND`, for another site's identifier.** "Not found"
  would be untrue and would send the agent to check a correct code for typos. The denial
  reveals one fact, that the identifier exists elsewhere, and nothing else.
- **Confirmation on all six writes.** I considered a lighter tier for booking and raising,
  and chose one uniform rule because it is harder for an agent to get wrong. The cost is one
  extra step on a forklift terminal, which I accepted knowingly.
- **A preview that issues a token, not a `confirmation: true` flag.** A required boolean
  would make the preview unreachable and would let the agent type `true` without asking.
- **No reopen for closed exceptions.** Ops asked for close but not its reverse. I took the
  position that closing is final, say so in the preview, and answer a mistake by raising a
  new exception.

## 4. What I got wrong, and what testing taught me

- **The token was not tied to what the user saw.** While writing the cancel tests I saw that
  the token covered the arguments but not the situation. If another booking took the slot
  between preview and commit, the same arguments would have cancelled a booking nobody
  confirmed. A correction could have applied to a quantity that had since changed. I bound
  the token to the preview text, and both cases now have tests.
- **My own test expectations were wrong twice, and the code was right both times.** I
  expected `CONFIRMATION_INVALID` when a used correction token was replayed, and when a bogus
  token was sent for an already-closed exception. The server returned the real reason
  instead (nothing to correct, already closed), because business rules run before the token
  check. That is the better behaviour, since the agent gets the most specific explanation.
- **I checked the SDK instead of assuming.** During the initial SDK probe, `mcp` 2.2.0 was
  installed. I read its signatures and sent a throwaway server four bad calls (an enum
  violation, an extra `site`, a missing field, and a valid one). All four reached my handler
  unchecked, so I used Pydantic models for input enforcement. The repository virtual
  environment currently has `mcp` 2.3.0; protocol tests still assert that a `site` argument
  is rejected and that the toolkit returns its own error shape.

## 5. What the live demos showed

### 5.1 The Claude Code run (main evidence)

I ran it from an empty folder containing only the registration, so the agent had no data file
or source code to read, and with no instructions file, so every behaviour below came from the
tool descriptions and results alone. The server logged each call (tool, preview or commit,
outcome). `evidence/claude_code/agent_transcript.md` has every prompt, the tools called, the
reply, the log lines and the screenshot names.

| Prompt (as Ops might say it) | What happened |
| --- | --- |
| "how much void fill have we got" | One `search_stock` call, single match, answered directly with no question. |
| "how many rolls of pallet wrap" | Two candidates, no quantity; the agent asked which one; then `get_stock` for the black one. |
| "what's the stock on SKU-8805" (Leeds) | `SITE_ACCESS_DENIED`; the agent said it is not held at the site and quoted nothing. |
| Book the 10:00 slot | Preview, my "yes", then the booking. |
| "book the 08:00" with no date, fresh session | The agent assumed today, got `NOT_FOUND` with the scheduled date, and **asked me whether I meant 8 April** before doing anything. |
| Book 08:00 when it is taken (probe) | `SLOT_ALREADY_BOOKED` from the server, with the free slots; nothing booked. |
| Book 11:00 (probe) | `INVALID_ARGUMENT` with the six allowed times and a corrected example. |
| Cancel the 12:00 booking | The preview named the carrier who would lose the slot; committed only after my "yes". |
| "move 500 of SKU-8801" | `INSUFFICIENT_STOCK` (240 available); the agent offered to move 240, or asked whether the count was wrong; then "move 20" gave a preview and a commit. |
| "the black pallet wrap count is actually 175" | The agent **asked for the reason** before previewing, then preview, "yes", commit. |
| Raise, list and close an exception | The agent found the ID itself and warned that closing is final. Closing again (probe) gave `ALREADY_CLOSED`. |
| 600mm shrink film (Glasgow) | Quantity 0 reported as zero, not "not found". |
| SKU-8801 (Glasgow) | `SITE_ACCESS_DENIED`. |

On every write the agent previewed first, asked once, and waited for my "yes". It never
confirmed on my behalf, and the log shows the preview before each commit.

### 5.2 What the Claude Code run showed about the agent

- **It leans on earlier context and does not always re-check the server.** After listing the
  schedule it answered later booking questions from memory (no tool call), and it reused a SKU
  from an earlier turn when raising an exception. If another user had changed the schedule in
  between, it would have given stale information. The server would still have stopped a bad
  write at the preview.
- **It paraphrases free text.** I typed "recount after the delivery" and the record says
  "Recount after delivery". I said "the outer layer is torn" and the description says "Black
  pallet wrap rolls have torn outer layer". I said "we replaced the rolls" and the note says
  "Damaged rolls replaced". The preview showed the exact wording before I approved it, so the
  check works, but "use the user's own words" is followed loosely.
- **Small unstated assumptions.** It treated "Northbound" and "Northbound Freight" as the same
  carrier, and stored my wording, so the schedule now holds two spellings. Carrier is free
  text in my design.
- **A small unnecessary question.** It asked which bin to correct when only one bin holds the
  product.
- **Careful agents rarely send the server a mistake.** Booking a taken slot and an invalid
  time reached the server only when I told the agent to skip its checks. The protocol tests
  cover both errors.
- **The log has two quirks.** A call that fails validation is labelled "preview" because it
  carried no token, and the exception's `raised_at` is in UTC while the log is in local time.

### 5.3 The earlier Copilot runs

The Copilot runs (`evidence/archive_copilot/`) gave three findings that shaped the final design,
and one that still stands as a limit.

**The workspace can undermine the toolkit.** With the repo open, the agent read
`data/stock_snapshot.csv` and answered "240" for SKU-8801 in a **Glasgow** session, which is
the leak Ops described. With an empty folder containing only `.vscode/mcp.json`, the same
prompt and the same model called `get_stock` and was refused. The server enforced the
boundary on every call it received, but it cannot stop a host that reads the data file
directly. In a real deployment the stock data would not sit in the agent's workspace. I did
not run Claude Code with the repo open, so I cannot say whether it would also read the file.

| Problem seen in Copilot | Fix | Evidence |
| --- | --- | --- |
| The agent invented the correction reason ("Physical count correction") | `correct_stock` and `close_exception` descriptions say to use the user's words and ask if none were given | Before: Copilot screenshots. After: Copilot `log_1_correction.txt`, and the same behaviour in Claude Code (prompt 11). |
| Two confirmation questions per raise or close | Descriptions say to preview first and ask once | `log_2_exceptions_before_fix.txt` vs `log_3_exceptions_after_fix.txt`. In Claude Code every write asked once. |
| After a failed search the agent dropped "black" and asked which product | The `NOT_FOUND` suggestion says to drop generic words and keep colour and size | `log_2` vs `log_3` |

All three are changes to wording the agent reads, driven by what the agent actually did and
not by how I expected it to behave. The Claude Code run then ran on the fixed descriptions.
The search-retry behaviour was only exercised in Copilot, because Claude Code reused a SKU
instead of searching again.

### 5.4 Limits of this evidence

One run per prompt, one person approving. The Claude Code model is not recorded. The
Claude Code run was done only from the empty folder. Copilot picked its own model, and it
changed between chats.

## 6. Where the design is weak

- **The site is only as trustworthy as whatever launches the server.** The environment
  variable stands in for an authenticated terminal session. Anyone who can edit the
  registration can change the site.
- **The token proves the commit matches the preview, not that a human agreed.** The human
  decision rests on the agent showing the preview and on the host's own approval prompt.
  In the demos I answered every confirmation myself.
- **Free text is stored and shown back to agents.** Exception descriptions and notes written
  by one user are returned by `list_exceptions` to a later agent session. I did not test
  whether such text could carry instructions to the agent. I would treat that as the first
  thing to test next.
- **`SITE_ACCESS_DENIED` confirms that an identifier exists elsewhere**, and the shared
  exception counter lets a user infer a count of other sites' exceptions. No content leaks.
- **State is in memory.** Restarting the server resets it, and there is no real concurrency
  control beyond re-checking rules at commit time.
- **The agent paraphrases and can act on stale context** (Section 5.2).
- **Carrier names are free text**, so the same carrier can appear under two spellings.
- **Only one date has a schedule**, and bins are validated by prefix, not against a real
  list of bins.

## 7. Answers to my own challenge questions

- **Read the stock file before designing lookup?** Yes (Section 2).
- **Does lookup ever return one item when the data supports more?** No. `search_stock`
  returns every match, and a test confirms "pallet" at Leeds returns four.
- **Does it demand disambiguation where there is one match?** No. A test types the full
  description of each of the 16 products at its own site and expects exactly one match each.
  Live, "void fill" at Leeds was answered with no question (Claude Code, prompt 1).
- **Irreversible or hard-to-unwind operations?** `correct_stock` (feeds month-end),
  `close_exception` (no reopen), and `cancel_dock_booking` (the carrier may be en route).
- **Tools that were really two or three?** `lookup` was two, `check_dock` three, and
  `stock_change` two.
- **Closed sets in the schema?** Slot, status and category are enums, and SKU and exception
  IDs have patterns. The server rejects what the schema forbids and returns our error shape
  (live: slot 11:00, Claude Code prompt 8b).
- **Is site scoping enforceable?** Yes at the toolkit (no parameter, scoping in one place,
  refusal checked for all 16 SKUs from all three sites), with the limits in Sections 5.3 and 6.
- **Can the agent fix a failed call from the error alone?** Yes. All nine codes are
  reachable through the protocol, each has `suggested_action`, and a test checks that every
  `example_call` is itself a valid call. Live, the agent recovered from `NOT_FOUND` for a
  date with no schedule and asked me which date I meant (prompt 6b).
- **An operation whose reverse was not requested?** `close_exception`. I chose no reopen.

## 8. Process and effort

**Process note.** The project began in VS Code with GitHub Copilot and was later moved to
Claude Code for the final demonstrations. The repository history does not preserve the
required four-commit sequence: the commit named `initial spec` added `spec.md`, `plan.md`
and `tasks.md` together, and there are no distinct `01-spec`, `02-plan`, `03-tasks` and
`04-implement` commits. The implementation and tests appear in a later commit, but that
does not satisfy the required history. The commit log records commit times, not when the
documents were first drafted or the exact sequence of my work, so it cannot verify the
required process. The current spec includes design amendments made during implementation
and demos; their reasons and evidence are in `CHANGES.md` Part B, and earlier behavior is
in `evidence/archive_copilot/`.

**Declared effort.**

- **Time:** roughly 5 to 7 days, covering reading the brief and data, spec, plan, tasks,
  implementation, tests, live demos and documents.
- **AI assistance:** I used Claude as an assistant throughout, working one step at a time in a
  chat. It helped with tests and documents. I used VS Code with GitHub Copilot
  for the first round of demos and Claude Code for the final run.
- **What I did myself:** made the design decisions I was asked to make (confirmation on all
  six writes, `SITE_ACCESS_DENIED` for another site's identifier, repeating the demos in
  Claude Code from a clean folder); wrote the code based on the tasks one by one. ran every test and every live demo on my own machine, answered the confirmation prompts myself, and saved the logs and screenshots; and fixed environment problems in my setup, including a duplicate server registration, a stale test-output file and moving the project to a new folder.
- **What I did not do alone:** few code, the tests and the wording of the documents were
  produced with AI help and then run and checked by me.
