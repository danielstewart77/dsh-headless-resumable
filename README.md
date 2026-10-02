# `@hive/dsh-headless-resumable`

A one-shot `dsh` surface that runs a turn in a conversation **it did not mint**.

The hive's one invariant is that hive-comms mints the conversation id when it
writes the session row, and hands that id to every spawn: `--session-id` for the
conversation's first process, `--resume` for every process after it. A mind
handed no id raises rather than inventing one. That is what makes a Telegram
turn and a browser-terminal turn the same conversation.

`@deepseek-ai/dsh-headless` mints `session-${randomUUID()}` and exposes no way
to continue anything, so a mind built on it could answer exactly one turn and
never the turn after — and every feature downstream of that (rotation,
cross-surface pickup, late turns, carry-forward) is a statement about a
conversation persisting across processes.

Both halves it needs already exist in `@deepseek-ai/dsh-agent`: `create` takes
the session id it is given, and `resume` loads a persisted one. What was missing
was a surface exposing them. This is that surface, and nothing else.

## What it adds beyond resuming

- **The turn comes back as data.** One line of JSON on stdout: the session it
  ran in, how it stopped, the final assistant text, the error code if there was
  one, and the tool traffic. A process exit status can carry none of that, and
  the tool traffic is the only measurement the training loop's first phase has.
- **Every refusal is a refusal.** No id, both ids, no task, no model, or a
  session that cannot be loaded — each reported as itself. Asked to continue a
  conversation that is not there, it refuses instead of creating one: creating
  one would answer in a conversation with no history and leave the one that was
  meant untouched and unfindable.
- **No compaction.** The bundle patch disables `compaction-basic` and
  `command-compact`. A summarizer whose priorities we cannot inspect must not
  rewrite the history of a run we are measuring. The model-free tool-result
  pruner stays on.

## Why it lives in this checkout

It is meant to be out-of-tree, and it is — its own git repository, its own
package, no patch to any upstream file but one line in `pnpm-workspace.yaml`.
It sits *inside* the upstream checkout only because the npm-published
`@deepseek-ai` set cannot be installed on its own: `dsh-agent@0.1.0-rc.6` needs
`dsh-invariants@^0.1.0-rc.6` and the published invariants is `0.0.1-rc.1`. The
workspace is the only place those specifiers resolve.

## Tests

```sh
../../node_modules/.bin/vitest run
```

Fourteen, over the real `SessionStore` and `AgentRegistry` with a scripted agent
factory, as upstream's own bundle tests do. `corepack pnpm vitest` does not work
here — pnpm's pre-run dependency check shells out to a `pnpm` on `PATH`, which
corepack does not install.
