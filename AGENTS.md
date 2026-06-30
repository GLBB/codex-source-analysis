# Codex Source Analysis Writing Instructions

These instructions apply to all files in this repository. They are meant to preserve the style of the Chinese Codex source-reading handbook and to make future agent edits more consistent.

## Source-Reading Documents

- Write source-reading navigation as prose for a reader with the source open. Do not make `rg`, `find`, or other shell commands the main path through the article.
- Start each hands-on source-reading article by defining the reading boundary: the one main path to follow, the branches to skip on the first pass, and what the reader should be able to explain afterward.
- Prefer stable file, type, function, and module names over line numbers. Line numbers drift; symbols and ownership boundaries are better long-term anchors.
- Separate current behavior, historical behavior, and design inference. Do not present an inference as a source fact.
- Keep the first pass focused. Mention related systems only as landmarks unless the article is specifically about them.

## Async And Event-Driven Systems

- For channel-, queue-, callback-, or event-loop-based code, explain endpoint ownership before describing the call chain.
- Explicitly name who owns each sender and receiver, where the receiver is awaited, and how the result returns to the external caller.
- When an API returns an id, ack, handle, or subscription rather than the final result, say so directly.
- Do not describe asynchronous control flow as if it were a synchronous function return.
- Include a short mental replay that crosses async boundaries: external entry, background receive, dispatch, task or loop execution, event emission, and external consumption.
- When one stop hands off to another through a framework method, scheduler, trait object, or lifecycle wrapper, add an explicit bridge stop instead of jumping from the call site to the eventual implementation.

## Terminology And Mental Models

- If a field name can mislead a first-time reader, call that out directly. For example, `tx_*` is a sender, and the corresponding `rx_*` may live in a different task.
- Keep terms consistent within an article. Avoid switching between names like `session_loop` and `submission_loop` unless the source uses both and the distinction is explained.
- Distinguish runtime events, model history, rollout records, logs, metrics, and UI projections. They may be produced by the same action, but they are not the same data.
- Distinguish manager, handle, session, task, turn, and loop responsibilities. Managers often create and register; they do not necessarily execute the core loop.

## Article Structure

- Give each major stop a concrete stopping standard, such as "can explain submit and next_event directions" or "can explain why a tool call causes another sampling request."
- Prefer "why not this simpler path?" explanations when they clarify architecture. Examples: why `submit` does not return the final answer, why `Op::UserInput` does not directly call `run_turn`, or why `TurnComplete` is emitted outside `run_turn`.
- End long walkthroughs with a complete mental replay of the path in numbered steps.
- Add "do not dwell on" notes when a nearby branch is real but would distract from the current reading goal.

## Review Checklist

When reviewing or revising a source-reading article, check:

- Are the main path and first-pass boundaries clear?
- Are async endpoints and ownership stated correctly?
- Are return values described accurately?
- Are event, history, rollout, metrics, and UI output kept separate?
- Are terms consistent across the article?
- Does each section have a useful stopping standard?
- Does the final mental replay include both input and output paths?
- Does the path include bridge layers such as task frameworks, lifecycle wrappers, trait dispatch, spawned tasks, and scheduler handoffs between adjacent sections?
- Are commands used for verification rather than as the primary reading navigation?
