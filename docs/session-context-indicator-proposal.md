# Proposal: session context indicator

## Problem

Orchestra's Usage status item shows account-level limits shared across sessions.
It does not show how much of the active conversation's model context window is
occupied. These are different measurements and should remain separate.

## Initial scope

Add a **Context** item to the status bar for the selected agent session. When a
valid reading is available, show the percentage of the model's context window
used; a tooltip or accessible label can give the estimated token count, window
size, and when it was last observed. Make it independently hideable and
reorderable using the existing status bar preferences.

Start with Pi, whose extension context exposes `getContextUsage()`. Its result
includes `tokens`, `contextWindow`, and `percent`; tokens and percent can be
unknown, including immediately after compaction. Pi describes the token count
as an estimate, so the UI must not imply exact measurement. See the
[extension API](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/docs/extensions.md#ctxgetcontextusage)
and [type definitions](https://github.com/earendil-works/pi/blob/main/packages/coding-agent/src/core/extensions/types.ts).

Other agents should report **Context unavailable** until their own documented
APIs and semantics are verified. Do not infer context occupancy from account
quotas, cumulative token billing, terminal output, or model names. Plain
terminal sessions do not need a context item.

## Data flow

1. Extend the existing process-local Pi status extension to observe context
   usage at relevant lifecycle events, including completed turns, session or
   model changes, and compaction. Check the installed Pi version and event
   timing during implementation: before a usable response, or when the API
   returns unknown, publish an unavailable reading rather than zero. Invalidate
   the previous reading when switching conversations or beginning compaction.
2. Publish a small, versioned per-session observation file alongside the
   existing status and agent ID files. Include the active agent conversation
   ID, model identifier when available, token count, context window and
   observation time; do not write prompts, transcripts, credentials or other
   content. Use a private runtime directory and atomic writes. Surface and log
   errors following the existing session-monitoring conventions.
3. Have `Store` validate and consume observations in the existing terminal
   monitor, associate them with the current launch and conversation, and notify
   the frontend only when the reading changes. Keep measurements transient,
   not in persisted workspace state or analytics. Reject malformed, negative,
   incomplete or mismatched readings instead of presenting them as valid.
4. Reconnect to a live background session's observation file when Orchestra
   reopens. Clear it for a new launch or conversation, remove it when the
   session ends, and extend stale-file cleanup without deleting files belonging
   to live host sessions. The shared-file approach does not require changing
   the session-host protocol.

The status item should distinguish **unavailable** from a genuine zero.
An idle session's last observation should remain labeled as such rather than
expiring solely because no new event occurs. When the current value cannot be
trusted after an invalidation, show unavailable until a new reading arrives.
If a value exceeds the window size, preserve the reported numbers and avoid
misleading progress-bar semantics.

## Verification before implementation is considered complete

- Exercise a fake Pi extension context with known, zero and unknown values,
  completed turns, conversation and model changes, and compaction. Verify
  event timing against a compatible Pi version without sending real prompts.
- Test file validation, atomic replacement, concurrent sessions, old or
  mismatched conversation IDs, launch restart, cleanup, and background
  reattachment. Ensure no transcript or credential data enters the payload.
- Test status-bar visibility, accessible labels, unavailable and over-window
  readings, session switching, and independence from account Usage. Check
  narrow windows and the existing themes.
- Add other agents only after confirming a reliable, per-conversation source
  and its semantics; unsupported versions must remain unavailable.

## Open decisions

- Should the status item always be visible for unsupported agents, or only when
  an agent supports reporting? This proposal favors explicit unavailable
  status, but maintainer feedback is welcome.
- Which Pi versions expose the required API and lifecycle events, and which
  event provides the freshest post-compaction value? Confirm with a local,
  isolated smoke test before selecting a minimum supported version.
