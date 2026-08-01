# Fork maintenance and ACP safety

This fork keeps local ACP context and spend guardrails while continuing to
receive updates from the upstream Buzz project.

## Repositories and remotes

- Fork: <https://github.com/ashik112/buzz>
- Upstream: <https://github.com/block/buzz>
- Guardrail branch: `safety/context-guardrails`

The local clone uses `origin` for the fork and `upstream` for the Block project.
Confirm the mapping before synchronizing:

```bash
git remote -v
```

## Pull updates from upstream

The fork `main` branch is the production branch and contains the local
guardrails. Merge upstream into it without rewriting published history:

```bash
git fetch upstream
git switch main
git merge upstream/main
. ./bin/activate-hermit
cargo test -p buzz-acp
git push origin main
```

If the merge reports conflicts, resolve them and rerun the tests before pushing
or deploying. Never force-push `main`. GitHub **Sync fork** is designed for a
fork with no local divergence, so use the explicit merge workflow above for
this customized fork.

Develop additional changes on a short-lived branch, review and test them, and
then merge them into `main`. The merged `safety/context-guardrails` branch may
remain as a historical reference; running code should be built from `main`.

## ACP context and spend guardrails

The guarded branch changes these `buzz-acp` defaults:

| Guardrail | Environment variable | Default | Purpose |
|---|---|---:|---|
| Absolute turn duration | `BUZZ_ACP_MAX_TURN_DURATION` | 1,800 seconds | Stops a turn after 30 minutes even if output keeps it active. |
| Session rotation | `BUZZ_ACP_MAX_TURNS_PER_SESSION` | 25 turns | Prevents indefinite transcript replay by rotating channel sessions. |
| Tool-call circuit breaker | `BUZZ_ACP_MAX_TOOL_CALLS_PER_TURN` | 100 calls | Stops continuously active tool loops. |

The tool-call limit counts ACP `tool_call` updates, not ordinary text or status
chunks. Exceeding it invalidates the adapter sessions and replaces that adapter
process. The interrupted batch uses the existing bounded retry queue and
exponential backoff.

Setting either count limit to `0` disables that guardrail and is not recommended
for unattended agents. The owner can use `!cancel` for the current turn or
`!rotate` to force a fresh channel session.

## Build and roll out

After synchronizing or changing the guardrails:

```bash
. ./bin/activate-hermit
cargo fmt --all --check
cargo test -p buzz-acp
cargo build --release -p buzz-acp
```

Restart every service that runs `buzz-acp` so it loads the new binary. A restart
briefly interrupts agents and clears their in-memory ACP sessions; obtain
operational approval first when agents may have work in flight.

Verify the startup log for each service. It should report values equivalent to:

```text
max_turn=1800s max_turns_per_session=25 max_tool_calls_per_turn=100
```

Building the binary alone does not update an already-running process.
