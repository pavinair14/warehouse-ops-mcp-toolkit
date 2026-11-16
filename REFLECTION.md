# Reflection

## Validation spike (task 4.1)

With MCP SDK `2.3.0`, I sent `book_dock_slot` an invalid `slot` value (`"11:00"`) through an in-memory MCP client. The SDK passed the arguments through to the tool handler instead of rejecting them based on the published enum. In the spike, validating the arguments with `BookDockSlotInput` raised a Pydantic `ValidationError`; translating that into the project's `ToolError` envelope returned `INVALID_ARGUMENT`, `field: "slot"`, the allowed slot values, and an `example_call` to the client. The client reported `is_error: true`.

I chose to catch and translate server-side validation failures rather than disable SDK input validation. The MCP pass-through check is reproducible with
`scratch/spike_client.py` and `scratch/spike_server.py`. The temporary handler used to verify the translated envelope was not retained.

## `correct_stock` no-op handling

The `correct_stock` tool rejects a correction when the physical count equals the recorded quantity, using `INVALID_ARGUMENT`, and does not issue a confirmation token or add an audit entry. This behavior is an explicit addition to spec 4.7: a no-op would otherwise
create a meaningless ledger entry. The existing error code is reused, so the tool interface and error-code set remain unchanged.

## `move_stock` design decisions

The move requirement says total stock is unchanged and a move can be undone by a reverse
move. I made these additional implementation choices:

- A full move removes the empty source row; a partial move leaves the remainder there. `get_stock` reports the bins holding a SKU, so omitting a zero-quantity source avoids presenting an empty bin as a stock location while preserving the SKU's total.

- A destination need not already contain the SKU. The supplied stock snapshot lists current stock locations, not a closed inventory of every usable bin, so I accept any bin with the caller site's recognized prefix. A prefix assigned to another site is denied; an unrecognized prefix such as `Z-99` is invalid.

- The preview states the source and destination quantities before and after the move, the unchanged total, and whether the destination currently has no row for the SKU. That makes a possibly mistyped new destination visible before confirmation. The confirmation is bound to the preview text; the server recomputes it at commit, so a change to the stock state invalidates a stale token.

## Exception ID sequencing trade-off

Exception IDs use one counter shared across all sites. This keeps IDs unique and lets `close_exception` distinguish an unknown ID from an ID belonging to another site. The trade-off is that a user may infer how many exceptions were raised elsewhere from gaps in the sequence (for example, seeing `EX-001` followed by `EX-004` implies two intervening IDs). The IDs reveal no exception contents, only a count. A per-site counter would avoid that inference but would change the `EX-001` format specified for exception IDs.

## Business validation before confirmation-token validation

Three test expectations assumed that a missing, closed, or cross-site exception target would be reported as a token error when a token was supplied. The implementation is right to return the more specific business error instead: `close_exception` checks the
target's existence, site ownership, and open status before checking the token. The shared write flow runs these business checks on previews and again at commit, before consuming a token. For example, an exception closed by a colleague after preview returns `ALREADY_CLOSED`, not `CONFIRMATION_INVALID`; unknown and cross-site IDs similarly return `NOT_FOUND` and `SITE_ACCESS_DENIED`. This gives the agent the actionable reason the operation cannot proceed.
