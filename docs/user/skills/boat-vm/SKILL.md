---
name: boat-vm
description: Run builds, tests, dev servers, Docker or compose stacks, and untrusted code on disposable Boat cloud VMs while the code stays in this checkout. Use when a check is too heavy or too long for this machine, needs a clean Linux box or Docker, runs code you don't trust, or several subagents each need their own machine. Not for moving the agent itself to a VM.
---

# Boat cloud VMs

Code stays in this checkout. A VM only receives pushed files and runs
commands, so edit locally and push before every run. For any flag not covered
here, use the `boat-docs` MCP tools when you have them, otherwise
https://docs.boat.dev/llms.txt.

## Project settings

Fill these in when adding the skill to a project. The commands below use these
example values; substitute the project's own.

- Template: `my-project-template` (a saved snapshot with the toolchain and a warm package cache), or the `bx_…` id of a stopped VM during the Boat free trial
- Size: `default`
- Boat environment: none (`--no-env`), or `--environment <name>` when checks need its secrets
- Directory on the VM: `/home/user/my-project` (where the template holds the repository)
- Check: `npm ci && npm test`
- Dev server: `npm run dev -- --host 0.0.0.0` on port `5173`

## Before you start

Run `boat status --check`. If it fails, or `boat`, `jq`, or `ssh` is missing,
stop and tell the user to follow T3 Code's "Build and test on cloud VMs"
guide. Never ask for a Boat key in chat and never start an interactive login.

In a non-interactive shell, every `boat` command prints JSON. `boat exec`
exits with the command's exit code and puts its output in `.stdout` and
`.stderr`.

## Run a check once

If the template is a stopped VM's `bx_…` id, skip this section: `--from`
accepts only saved snapshots. Use a VM for the thread and delete it after the
check.

Prefer this for a single check. It creates a VM, pushes this folder, runs the
command, prints the cost, and deletes the VM:

```sh
boat run --from my-project-template --type default --no-env --push . -- sh -c 'npm ci && npm test'
```

The files land in `/home/user/<this folder's name>`, so the command must
install dependencies itself. Report the exit code and the cost line.

## Keep a VM for this thread

Use one VM when you will run several cycles or a dev server:

```sh
branch="$(git rev-parse --abbrev-ref HEAD | tr -c 'A-Za-z0-9-\n' '-' | cut -c1-40)"
id="$(boat new --from my-project-template --type default --no-env --name "t3-$branch-$(date +%s)" --ttl 3600 | jq -r 'select(.event == "ready") | .id')"
```

A copy of a saved snapshot gets the Boat environment you pass, so always pass
the one from the project settings. A fork keeps its template's.

With a stopped-VM template, copy it instead, then always name the copy:

```sh
id="$(boat fork bx_template --ttl 3600 --wait | jq -r 'select(.event == "ready") | .id')"
boat rename "$id" "t3-$branch-$(date +%s)"
```

VMs are ready in a few seconds. State the VM id in your reply so later turns
reuse it instead of creating another. `boat list` shows running VMs by name.

Each cycle:

```sh
boat push "$id" . --to /home/user/my-project
boat exec "$id" --cwd my-project --timeout 600 -- sh -c 'npm ci && npm test'
```

- Always pass `--timeout`. The default of 30 seconds kills builds. Use `0`
  for commands that may take longer than 10 minutes; output then streams as
  JSON lines.
- Run `boat push` on its own, not piped, and stop if it exits nonzero. The
  template already holds an older copy of the code, so a check after a failed
  push passes against stale files.
- `boat push` copies over SSH and never deletes. Remove files you deleted or
  renamed locally (`git status` lists them) with
  `boat exec "$id" --cwd my-project -- rm <paths>`.
- If `boat push` fails with `Operation not permitted`, your sandbox blocked
  its SSH connection. Rerun only that push outside the sandbox; other `boat`
  commands work inside it.
- If the push times out connecting, this network blocks SSH. When the work is
  committed and pushed to the remote, fetch it on the VM over HTTPS instead:
  `boat exec "$id" --cwd my-project --timeout 600 -- sh -c 'git fetch origin <branch> && git checkout --detach FETCH_HEAD'`.
  Uncommitted edits are not included, and a private repository needs a Boat
  environment with access to it. Otherwise tell the user instead of retrying.
- Ignored files such as `.env` are not pushed.

## Keep it alive, but not forever

A VM stops one hour after it starts. Commands do not move that deadline.
Before work that may run past it, run `boat extend "$id" --hours 1`; this sets
the deadline to one hour from now. Never use `--no-auto-stop`.

## Dev server

```sh
pid="$(boat exec "$id" --cwd my-project --detach 'npm run dev -- --host 0.0.0.0' | jq -r .processId)"
boat host "$id" 5173 | jq -r .url
```

The URL carries an access token. Open it with T3's browser tools when you
have them, otherwise give it to the user, and never paste it anywhere else.
If the page answers `502 upstream unavailable` because the app listens only on
localhost, run `boat host` again with `--localhost`. Vite needs
`server.allowedHosts: [".on.boat.dev"]`. Read the server's output with
`boat exec "$id" --status "$pid"`.

VMs have Docker. Start a compose stack the same way, with
`--detach 'docker compose up'`. Containers stop when the VM stops.

## Several machines

To split work, start one child per VM and tell each to use this skill and
delete its VM when done. Each child creates its own VM from the template.
Children that only run checks can share this checkout. Children that edit
need their own: delegate with `delegate_task` and `isolation: "worktree"`, and
each pushes its own worktree. Never fork a running VM for this: a fork can
miss the last minute of writes.

Boat limits how many VMs run at once and how many start per minute. On
`limit_reached`, wait for another VM to be deleted or tell the user; the free
trial runs two at once. On `rate_limited`, wait for the window the error
names, then retry; the CLI does not retry for you. Run `boat limits` instead of
quoting limits or prices from memory.

## Bring results back

Copy a text file, such as a test report or log, over HTTPS; `jq -j` keeps it
byte for byte, up to 8 MiB:

```sh
boat exec "$id" --cwd my-project -- cat reports/junit.xml | jq -j .stdout > junit.xml
```

Copy binaries and directories with `boat scp -r "$id:/home/user/my-project/<path>" <local path>`,
which uses SSH like `boat push`.

## Clean up

Before your final answer, run `boat delete "$id" --yes`. If the VM serves a
dev server the user asked for, leave it running and give them the id and the
delete command. If the user may want to inspect the VM, run `boat stop "$id"`
instead and tell them; a stopped VM is kept for 14 days. Never create, change,
resume, or delete the template unless the user asks, and never pass secrets
with `--env`.
