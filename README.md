# AgentTeams worktree notifier for herdr

A [herdr](https://herdr.dev) plugin that tells AgentTeams as soon as a worktree is created or removed, so the RunnerBox list reflects the change immediately instead of waiting for the next Runner Git reconciliation.

Verified against **herdr 0.8.0**.

## Install

```bash
herdr plugin install AgentTeamsRun/herdr
```

herdr asks for confirmation before installing. In a non-interactive shell, pass `--yes`:

```bash
herdr plugin install AgentTeamsRun/herdr --yes
```

The install prints `manifest does not declare platforms; platform support unknown`. That warning appears because the manifest omits the optional `platforms` field; it does not mean the install failed or that your platform is unsupported.

Check that it is registered and enabled:

```bash
herdr plugin list
```

## Requirements

- **herdr 0.8.0 or newer.** The manifest declares `min_herdr_version = "0.8.0"`. Only 0.8.0 was verified; which command older versions reject, and with what message, was not checked.
- **The AgentTeams CLI must be on the PATH of the herdr server process**, not just your interactive shell. herdr runs event hooks from the server, so a CLI that is only visible to your login shell will not be found.

  ```bash
  npm install -g @agentteams/cli
  ```

- **The CLI must support `agentteams worktree notify-created --from-herdr-event`.** Older releases accept the `notify-created` / `notify-deleted` commands but not this flag, and the hook then exits with an unknown-option error. Confirm with:

  ```bash
  agentteams worktree notify-created --help
  ```

  If `--from-herdr-event` is not listed, the hook exits 1 and the notification is lost, but the worktree operation itself still completes.

  > **This flag is not published to npm yet.** The latest published `@agentteams/cli` at the time of writing is 0.1.104, which does not have it, so upgrading today will not make it appear in `--help`. Once a release including the flag is out, run `npm install -g @agentteams/cli@latest` and check again.

- **A daemon token must be configured on the same host.** Run this once before using the plugin:

  ```bash
  agentrunner init --token <token>
  ```

## How it works

The manifest registers two event hooks:

| Event              | Command                                                             |
| ------------------ | ------------------------------------------------------------------- |
| `worktree.created` | `agentteams worktree notify-created --from-herdr-event --quiet`     |
| `worktree.removed` | `agentteams worktree notify-deleted --from-herdr-event --quiet`     |

Only the dotted event names are valid. An underscore form such as `worktree_created` still links, but herdr reports `unknown event` and the hook never fires.

herdr runs hooks with the **plugin root** as the working directory, not the worktree. That is why both commands take `--from-herdr-event`: the flag makes the CLI read the worktree path, branch, and repository root from the `HERDR_PLUGIN_EVENT_JSON` payload herdr injects instead of from the current directory.

herdr fires `worktree.removed` **after** the Git worktree has already been removed. The CLI restores the same worktree identity from the parent directory, so the deleted event still matches the registry entry and no "notify before removal" flag is needed.

## Failure behaviour

Hook execution is best-effort. herdr's documentation does not state a retry or timeout policy, and the removal hook runs after the checkout is already gone, so a failing hook cannot reverse a local worktree operation. If an event is lost — the plugin is disabled, the CLI is missing, the host is offline, or `git worktree add`/`remove` was run outside herdr — the next AgentTeams Runner Git reconciliation still restores the correct `AVAILABLE` / `MISSING` state. Immediate notification is an optimisation, not the correctness path.

Inspect what actually ran:

```bash
herdr plugin log list --plugin agentteams
```

The log shows the exit code plus stdout/stderr for each hook invocation.

## Update

herdr 0.8.0 has no `plugin update` command (it is absent from `herdr plugin --help`). An installed plugin is pinned to the `resolved_commit` captured at install time, so it does not follow later changes in this repository on its own — but **re-running the install command replaces the existing installation.** There is no need to uninstall first.

```bash
herdr plugin install AgentTeamsRun/herdr
```

On a reinstall the preview gains a `replaces: agentteams from github:AgentTeamsRun/herdr@<commit>` line. herdr reuses the managed directory and rewrites the install record, so `resolved_commit` is resolved again.

## Uninstall

```bash
herdr plugin uninstall agentteams
```

This removes the managed checkout but leaves the per-plugin config directory (`<herdr config dir>/plugins/config/agentteams`) in place. Delete it yourself if you want the plugin fully gone.

## About this repository

This repository is a **generated mirror**. The source lives in the AgentTeams repository under `integrations/herdr/` and is published here automatically; commits pushed directly to this repository are overwritten on the next sync. Please report issues and suggest changes through AgentTeams rather than as pull requests here.

Integration documentation: <https://docs.agentteams.run/en/setup/herdr>
