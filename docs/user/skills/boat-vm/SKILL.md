---
name: boat-vm
description: Run builds, tests, and dev servers on disposable Boat cloud VMs while the code stays in this checkout. Use when a check is too heavy for this machine, needs a clean Linux box, or several subagents each need their own machine. Not for moving the agent itself to a VM.
---

# Boat cloud VMs

Code stays in this checkout. A VM only receives pushed files and runs
commands, so edit locally and push before every run. For any flag not covered
here, read the Boat docs at https://docs.boat.dev/llms.txt.

## Project settings

Fill these in when adding the skill to a project. The commands below use these
example values; substitute the project's own.

- Template: `my-project-template` (a saved snapshot with the toolchain and a warm package cache), or the `bx_…` id of a stopped VM during the Boat free trial
- Size: `default`
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

Prefer this for a single check. It creates a VM, pushes this folder, runs the
command, prints the cost, and deletes the VM:

```sh
boat run --from my-project-template --type default --push . -- sh -c 'npm ci && npm test'
```

The files land in `/home/user/<this folder's name>`, so the command must
install dependencies itself. Report the exit code and the cost line. With a
stopped-VM template, use a VM for the thread instead.

## Keep a VM for this thread

Use one VM when you will run several cycles or a dev server:

```sh
branch="$(git rev-parse --abbrev-ref HEAD | tr -c 'A-Za-z0-9-\n' '-' | cut -c1-40)"
id="$(boat new --from my-project-template --type default --name "t3-$branch-$RANDOM" --ttl 3600 | jq -r 'select(.event == "ready") | .id')"
```

With a stopped-VM template, copy it instead, then name the copy:

```sh
id="$(boat fork bx_template --ttl 3600 --wait | jq -r 'select(.event == "ready") | .id')"
boat rename "$id" "t3-$branch-$RANDOM"
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
- `boat push` copies over SSH and never deletes. Remove files you deleted or
  renamed locally (`git status` lists them) with
  `boat exec "$id" --cwd my-project -- rm <paths>`. If the push times out
  connecting, this network blocks SSH: tell the user instead of retrying.
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

## Several machines

To split work, delegate with `delegate_task` and `isolation: "worktree"`, and
tell each child to use this skill. Each child creates its own VM from the
template and pushes its own worktree. Never fork a running VM for this: a fork
can miss the last minute of writes.

Boat limits how many VMs run at once and how many start per minute. On
`limit_reached`, wait for another VM to be deleted or tell the user; the free
trial runs two at once. On `rate_limited`, wait for the window the error
names, then retry; the CLI does not retry for you. Run `boat limits` instead of
quoting limits or prices from memory.

## Clean up

Before your final answer, run `boat delete "$id" --yes`. If the user may want
to inspect the VM, run `boat stop "$id"` instead and tell them; a stopped VM is
kept for 14 days. Never create, change, resume, or delete the template unless
the user asks, and never pass secrets with `--env`.
