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

### Upstream synchronization record: 2026-08-02

Upstream `f86cfc736` was merged into the guarded fork as merge commit
`cfcc48cd7`. The merge brought the fork current with Block's `main` without
rewriting fork history. The ACP refactor merged without textual conflicts, and
all four fork-specific guardrail configuration and enforcement paths remained
present in the merged source.

This synchronization changes source only. It does not by itself rebuild
`buzz-acp`, restart the systemd fleet, or move the digest-pinned relay
container image. Treat build, rollout, and image upgrades as separate,
verified operations.

The merge also added the open-source remote-agent provider protocol and its
first implementation:

- `docs/remote-agents.md` formally specifies discovery, trust boundaries,
  lifecycle, presence-based status, shutdown, and conformance.
- `buzz-backend-kubernetes` lets Buzz Desktop deploy a managed agent as a
  Kubernetes Pod while retaining the managed-agent record in Desktop.
- The provider uses ambient kubeconfig, a per-attempt Kubernetes Secret, and
  an immutable digest-qualified Sprig image.
- `BUZZ_ACP_EXIT_AFTER_INACTIVITY` adds an optional, pool-independent
  self-stop bound for remote workloads.

This is a new deployment path, not an adoption path for the existing systemd
fleet. The current Kubernetes v1 path has no undeploy operation and no baked
public default Sprig image; operators must provide an image digest and manage
any residual Kubernetes objects. See `docs/remote-agents.md`.

## ACP context and spend guardrails

The guarded fork's `main` branch uses these `buzz-acp` defaults:

| Guardrail | Environment variable | Default | Purpose |
|---|---|---:|---|
| Absolute turn duration | `BUZZ_ACP_MAX_TURN_DURATION` | 1,800 seconds | Stops a turn after 30 minutes even if output keeps it active. |
| Session rotation | `BUZZ_ACP_MAX_TURNS_PER_SESSION` | 25 turns | Prevents indefinite transcript replay by rotating channel sessions. |
| Tool-call circuit breaker | `BUZZ_ACP_MAX_TOOL_CALLS_PER_TURN` | 100 calls | Stops continuously active tool loops. |
| Compaction fuse | `BUZZ_ACP_MAX_COMPACTIONS_PER_TURN` | 1 compaction | Stops the turn on its second compaction so compact-and-continue loops cannot run indefinitely. |

The tool-call limit counts ACP `tool_call` updates, not ordinary text or status
chunks. Exceeding it invalidates the adapter sessions and replaces that adapter
process. The interrupted batch uses the existing bounded retry queue and
exponential backoff.

The compaction fuse recognizes the structured Codex ACP context-compaction
update and Claude Agent ACP's compaction status update. One compaction is
allowed. When the next compaction starts, Buzz invalidates every session on the
adapter, replaces its process, dead-letters the batch immediately, and posts a
failure notice. It deliberately does not retry the task: retrying the same
runaway request would defeat the fuse.

Setting any count limit to `0` disables that guardrail and is not recommended
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

For a systemd fleet whose unit names follow the Buzz Codex and Claude
conventions, first inspect the matched units and then restart that exact set:

```bash
systemctl list-units --all --type=service \
  "buzz-agent-*-codex.service" buzz-agent-codex.service \
  "buzz-agent-*-claude.service" buzz-agent-claude.service
sudo systemctl restart \
  "buzz-agent-*-codex.service" buzz-agent-codex.service \
  "buzz-agent-*-claude.service" buzz-agent-claude.service
systemctl is-active \
  "buzz-agent-*-codex.service" buzz-agent-codex.service \
  "buzz-agent-*-claude.service" buzz-agent-claude.service
```

The quoted wildcard is expanded by systemd against loaded units, not by the
shell against files. Review the list before restarting so unrelated services
are not included. A service may run more than one adapter subprocess; service
count and adapter-process count are therefore not necessarily equal.

Verify the startup log for each service. It should report values equivalent to:

```text
max_turn=1800s max_turns_per_session=25 max_tool_calls_per_turn=100 max_compactions_per_turn=1
```

Building the binary alone does not update an already-running process.

### Deployment record: 2026-08-01

The guarded `main` binary was deployed to 30 systemd services: 15 Codex and
15 Claude harnesses. Each service runs two adapters, for 60 adapter subprocesses.
After restart, every service was active, all 60 adapter children were present,
and every startup summary reported:

```text
idle_timeout=900s max_turn=1800s max_turns_per_session=25 max_tool_calls_per_turn=100 max_compactions_per_turn=1
```

No startup errors were present in the post-restart service logs. The restart
cleared all sessions that predated the guardrail deployment.
