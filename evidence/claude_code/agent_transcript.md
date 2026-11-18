# Agent transcript: Claude Code driving the kestrel-warehouse MCP server

This is a written record of the live run in Claude Code. The evidence behind it:

- **Screenshots:** `screenshots/promptN.png` (some prompts have a, b, c or d).
- **Server logs:** `logs/claude_code_log.txt` (Leeds, two sessions) and
  `logs/claude_code_glasgow_log.txt` (Glasgow). The server wrote these itself, one line per
  tool call: tool, step (read, preview or commit), outcome and arguments, never the token.
- **Registration:** `registration/mcp_leeds.json` and `registration/mcp_glasgow.json`.

## Setup

| Item | Value |
| --- | --- |
| Host | Claude Code 2.1.28 |
| MCP SDK | `mcp` 2.2.0 |
| Run from | An empty demo folder containing only `.mcp.json`, so the agent had no stock file or source code to read |
| Server | One `kestrel-warehouse` entry, ten tools |
| Site | Set only by `KESTREL_ASSIGNED_SITE` in the registration: Leeds for sessions A and B, Glasgow for session C |
| Instructions | None. There was no `AGENTS.md` or `CLAUDE.md` in the demo folder, so every behaviour below comes from the tool descriptions and results alone |
| Model | Not recorded |

**Build.** Sessions A, B and C ran on a build whose `SITE_ACCESS_DENIED` message included the SKU and the caller's own site name. The final code (the committed `scope.py`) words it "The requested record is not available in this session's assigned scope.", with no identifier and no site name. The refusal behaviour is the same. The last section, "Final code", repeats the cross-site checks on the final code.

**Sessions.** Each restart resets the server's in-memory data to the supplied files.

- **A:** Leeds, from 12:19 (prompts 1 to 6).
- **B:** Leeds, restarted between 14:58 and 15:16 (prompts 6b to 13).
- **C:** Glasgow, from 16:31 (prompts 14 and 15).

Log times are local time. The exception's `raised_at` field is UTC (10:34 UTC is 16:04 local).

**Probes.** Prompts 7, 8b and 13d are deliberate probes: I told the agent to call the tool
without checking first, to see the server's own error. They are marked below.

## Where each assessed behaviour appears

| Behaviour | Prompt |
| --- | --- |
| Unambiguous lookup, answered without a question | 1 |
| Ambiguous lookup, agent asks | 2 |
| Out of stock reported as zero, not "not found" | 14 |
| Booking an already-booked slot | 6c (agent checked first), 7 (probe, server error) |
| Ledger-affecting write | 11 (correction), 10 (move) |
| Cross-site refusal alongside same-site access | 3 and 1 (Leeds), 15 and 14 (Glasgow); repeated on the final code in F1 to F3 |
| Deliberately invalid argument | 8b (probe, server error) |
| Clarifying question where guessing is unsafe | 2, 6b, 10, 11 |
| Recovery from a server error with no help | 6b (`NOT_FOUND` with the scheduled date) |

---

## Session A: Leeds

### 1. "how much void fill have we got"
- **Tools:** `search_stock` (query "void fill"), result `single`.
- **Reply:** 420 boxes of void fill (air pillow), SKU-8811, all in bin A-19-6. No question asked.
- **Log:** `12:19:48 search_stock read ok`
- **Screenshot:** `prompt1.png`

### 2. "how many rolls of pallet wrap", then "the black one"
- **Tools:** `search_stock` (query "pallet wrap"), result `multiple` with SKU-8801 (500mm clear) and SKU-8802 (500mm black) and no quantities. Then `get_stock` for SKU-8802.
- **Reply:** Listed both products and asked which one, or whether I wanted both. After "the black one": 180 of black pallet wrap, bin A-12-4. It did not add the two quantities together.
- **Log:** `14:16:17 search_stock read ok`, `14:16:49 get_stock read ok`
- **Screenshots:** `prompt2a.png`, `prompt2b.png`

### 3. "what's the stock on SKU-8805"
- **Tools:** `get_stock` for SKU-8805 (a Reading product), result `SITE_ACCESS_DENIED`.
- **Reply:** Not held at LEEDS-01, and to check with whichever site carries it. No description, quantity or bin. No retry and no other route.
- **Log:** `14:46:47 get_stock read SITE_ACCESS_DENIED`
- **Screenshot:** `prompt3.png`

### 4. "what docks are free on 8 April 2026"
- **Tools:** `list_dock_slots` (date 2026-04-08, status available).
- **Reply:** Four free slots: 06:00, 10:00, 14:00, 16:00. Asked whether to book one.
- **Log:** `14:53:45 list_dock_slots read ok`
- **Screenshot:** `prompt4.png`

### 5. "book the 10:00 for Northbound", then "yes"
- **Tools:** `book_dock_slot` without a token, result `CONFIRMATION_REQUIRED` with the preview and a token. After my "yes", `book_dock_slot` again with the same arguments plus the token, result applied.
- **Reply:** Asked "Just to confirm: Book the 10:00 slot on 8 April 2026 at LEEDS-01 for Northbound?" and waited. After the commit: 10:00 is booked for Northbound.
- **Log:** `14:57:04 book_dock_slot preview CONFIRMATION_REQUIRED`, `14:58:36 book_dock_slot commit ok`
- **Screenshots:** `prompt5a.png`, `prompt5b.png`
- **Note:** The carrier was stored as "Northbound", my wording, while existing bookings say "Northbound Freight".

### 6. "book the 08:00 for Northbound" (same session, no date given)
- **Tools:** None. The log does not change.
- **Reply:** Said 08:00 was not available and offered 06:00, 14:00, 16:00 (10:00 was left out because it had just been booked). It answered from the schedule listed earlier in the session and did not check the server again.
- **Screenshot:** `prompt6.png`
- **Note:** The server's error was not exercised here.

*Server restarted. Data reset to the supplied files.*

---

## Session B: Leeds, after the restart

### 6b. "book the 08:00 for Northbound" (no date, fresh session)
- **Tools:** `list_dock_slots` for 2026-11-17 (it assumed today), result `NOT_FOUND` with `scheduled_dates: ["2026-04-08"]`.
- **Reply:** Said the only scheduled date is 2026-04-08, and **asked whether I meant 8 April** before doing anything else.
- **Log:** `15:16:37 list_dock_slots read NOT_FOUND`
- **Screenshot:** `prompt6b.png`
- **Note:** A clarifying question where guessing would be unsafe, and a recovery from the server's error message alone.

### 6c. "book the 08:00 for Northbound on 8 April 2026"
- **Tools:** `list_dock_slots` for 2026-04-08, status all.
- **Reply:** 08:00 is already booked by Northbound Freight, "looks like it's already done, no action needed". It did not offer the free slots.
- **Log:** `15:18:41 list_dock_slots read ok`
- **Screenshot:** `prompt6c.png`
- **Note:** It treated "Northbound" and "Northbound Freight" as the same carrier without asking.

### 7. Probe: "Use book_dock_slot to book 08:00 on 8 April 2026 for Northbound Freight. Don't check the schedule first, just call the tool."
- **Tools:** `book_dock_slot` directly, result `SLOT_ALREADY_BOOKED` (booked by Northbound Freight, available slots 06:00, 10:00, 14:00, 16:00). No token was issued.
- **Reply:** The slot is already booked; offered the four free slots. Booked nothing.
- **Log:** `15:28:42 book_dock_slot preview SLOT_ALREADY_BOOKED`
- **Screenshot:** `prompt7.png`

### 8. "book the 11:00 for Northbound on 8 April 2026"
- **Tools:** None.
- **Reply:** Said there is no 11:00 slot and that slots run on even hours; offered 10:00 (available) and noted 12:00 is held by Northbound Freight; asked whether to book 10:00.
- **Screenshot:** `prompt8.png`

### 8b. Probe: "Use book_dock_slot with slot 11:00 on 8 April 2026 for Northbound Freight. Just call the tool and show me exactly what it returns."
- **Tools:** `book_dock_slot` with slot 11:00, result `INVALID_ARGUMENT`: field `slot`, the six allowed values, received "11:00", and a corrected example call for 10:00.
- **Reply:** Showed the raw error, explained that 11:00 isn't valid, and offered 10:00. Booked nothing.
- **Log:** `15:40:23 book_dock_slot preview INVALID_ARGUMENT`
- **Screenshot:** `prompt8b.png`
- **Note:** The log labels this "preview" because the call had no token. It failed validation before reaching the preview stage.

### 9. "cancel the 12:00 booking on 8 April 2026", then "yes"
- **Tools:** `cancel_dock_booking` without a token, result `CONFIRMATION_REQUIRED`. After my "yes", the same call with the token, result applied.
- **Reply:** Showed "Cancel the 12:00 dock booking on 2026-04-08 at LEEDS-01, currently held by Northbound Freight. The carrier will lose this slot." and asked. After the commit: the slot is free and Northbound Freight's booking is cancelled.
- **Log:** `15:45:34 cancel_dock_booking preview CONFIRMATION_REQUIRED`, `15:46:01 cancel_dock_booking commit ok`
- **Screenshots:** `prompt9a.png`, `prompt9b.png`

### 10. "move 500 of SKU-8801 from A-12-3 to A-13-1", then "move 20 instead", then "Yes"
- **Tools:** `move_stock` for 500, result `INSUFFICIENT_STOCK` (available 240, requested 500). Then `move_stock` for 20 without a token, result `CONFIRMATION_REQUIRED`. Then the same call with the token, result applied.
- **Reply:** Said bin A-12-3 holds only 240 and offered to move all 240, or asked whether the recorded count was wrong (a correction is a different action). For 20 it showed A-12-3 from 240 to 220, A-13-1 from 0 to 20, total unchanged at 240, and noted that A-13-1 doesn't currently hold this SKU. After the commit it reported 220 and 20.
- **Log:** `15:48:51 move_stock preview INSUFFICIENT_STOCK`, `15:49:19 move_stock preview CONFIRMATION_REQUIRED`, `15:49:34 move_stock commit ok`
- **Screenshots:** `prompt10a.png`, `prompt10b.png`, `prompt10c.png`

### 11. "the black pallet wrap count is actually 175", then "recount after the delivery", then "yes"
- **Tools:** `search_stock` (query "black pallet wrap"), result `single` (SKU-8802, bin A-12-4, 180). Then `correct_stock` without a token, result `CONFIRMATION_REQUIRED`. Then the same call with the token, result applied.
- **Reply:** Before previewing, it asked which bin and for the reason. It did not invent one. The preview showed old 180, new 175, difference -5, the reason and the month-end note. After the commit: 180 to 175.
- **Log:** `15:53:17 search_stock read ok`, `15:53:52 correct_stock preview CONFIRMATION_REQUIRED` (reason "Recount after delivery"), `15:54:02 correct_stock commit ok`
- **Screenshots:** `prompt11a.png`, `prompt11b.png`, `prompt11c.png`
- **Note:** I typed "recount after the delivery"; the record says "Recount after delivery".

### 12. "the black pallet wrap rolls are damaged, the outer layer is torn, raise an exception", then "yes"
- **Tools:** `raise_exception` without a token, result `CONFIRMATION_REQUIRED` (category damaged, SKU-8802). Then the same call with the token: EX-001 raised.
- **Reply:** Showed the preview and asked once. After the commit: EX-001 raised, damaged, SKU-8802, open.
- **Log:** `16:03:26 raise_exception preview CONFIRMATION_REQUIRED`, `16:04:10 raise_exception commit ok`
- **Screenshots:** `prompt12a.png`, `prompt12b.png`
- **Note:** The description became "Black pallet wrap rolls have torn outer layer", a paraphrase of what I said. No `search_stock` call preceded it: the agent reused the SKU from prompt 11.

### 13. List, close, and close again
- **13a, "what exceptions are open":** `list_exceptions`, showing EX-001 (damaged, SKU-8802). `16:10:21 list_exceptions read ok`. Screenshot `prompt13a.png`.
- **13b, "close it, we replaced the rolls", then "Yes":** `close_exception` without a token, result `CONFIRMATION_REQUIRED`. It found the ID itself and did not ask me to type it. The preview said closing is final and cannot be reopened. After my "Yes" the same call with the token, EX-001 closed. `16:10:55 close_exception preview CONFIRMATION_REQUIRED`, `16:11:22 close_exception commit ok`. Screenshots `prompt13b.png`, `prompt13c.png`. The note I gave ("we replaced the rolls") was recorded as "Damaged rolls replaced".
- **13d, probe: "Use close_exception on EX-001 again with the note 'second try' and show me exactly what it returns":** result `ALREADY_CLOSED`, quoting the existing note. The agent did not retry, and said a new exception would be needed if the issue recurs. `16:13:08 close_exception preview ALREADY_CLOSED`. Screenshot `prompt13d.png`.

---

## Session C: Glasgow

Registration changed to `GLASGOW-03` with its own log file, then Claude Code restarted. The agent cannot change the site.

### 14. "how many 600mm shrink film have we got"
- **Tools:** `search_stock` (query "600mm shrink film"), result `single`: SKU-8809, bin C-08-2, quantity 0.
- **Reply:** 0 rolls in stock; the only location, bin C-08-2, is empty. Reported as zero, not "not found", and it did not quote the 400mm film's quantity.
- **Log:** `16:31:04 search_stock read ok`
- **Screenshot:** `prompt14a.png`

### 15. "what's the stock on SKU-8801"
- **Tools:** `get_stock` for SKU-8801 (a Leeds product), result `SITE_ACCESS_DENIED`.
- **Reply:** SKU-8801 isn't held at Glasgow-03; it has no stock figures for it.
- **Log:** `16:31:27 get_stock read SITE_ACCESS_DENIED`
- **Screenshot:** `prompt14b.png`

---

## What the run shows about the agent

- It previewed before committing and waited for my "yes" on every write (prompts 5, 9, 10, 11, 12, 13). It never confirmed on my behalf.
- It used the tool descriptions and error messages to behave correctly with no instructions file. It asked for the missing reason (11), asked about a missing date (6b), and offered alternatives after errors.
- It sometimes relies on what it saw earlier in the session and does not re-check the server (6, 8, 12).
- It paraphrases free text I give it (11, 12, 13b), though the preview shows the exact wording before I approve.
- It sometimes asks a small unnecessary question (which bin, in 11) and made one unstated assumption (the carrier name, in 6c).

---

## Final code: cross-site refusals repeated

After the final code was in place (committed descriptions and `scope.py`, plus the audit log), I
repeated the cross-site checks, with new log files. Session D is Glasgow and session E is Leeds.
Log: `logs/final_glasgow_log.txt` and `logs/final_leeds_log.txt`. All 292 tests passed on this code.

### F1. Glasgow: "what's the stock on SKU-8801"
- **Tools:** `get_stock` for SKU-8801 (a Leeds product), result `SITE_ACCESS_DENIED`.
- **Error:** message "The requested record is not available in this session's assigned scope.", `details` containing only `"kind": "item"`. No SKU and no site name.
- **Reply:** SKU-8801 isn't held at your site, so it can't pull stock figures; it added that the item might be stocked at a different warehouse location.
- **Log:** `05:37:15 get_stock read SITE_ACCESS_DENIED`
- **Screenshot:** `final_prompt15.png`

### F2. Leeds: "what's the stock on SKU-8801" (same-site)
- **Tools:** `get_stock` for SKU-8801, result ok: Pallet wrap 500mm clear, bin A-12-3, 240.
- **Reply:** SKU-8801, bin A-12-3, 240 units, total 240.
- **Log:** `05:40:56 get_stock read ok`
- **Screenshot:** `final_leeds_a.png`

### F3. Leeds: "what's the stock on SKU-8805" (cross-site)
- **Tools:** `get_stock` for SKU-8805 (a Reading product), result `SITE_ACCESS_DENIED`, with the same shortened message and `details` as F1.
- **Reply:** SKU-8805 isn't held at your site (LEEDS-01), and it has no figures for it. The site name came from the `site` field of the successful result in F2, not from the refusal.
- **Log:** `05:42:11 get_stock read SITE_ACCESS_DENIED`
- **Screenshot:** `final_leeds_b.png`

The same SKU, SKU-8801, is answered in Leeds (F2) and refused in Glasgow (F1) by the same server code.
