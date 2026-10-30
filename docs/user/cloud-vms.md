# Build and test on cloud VMs

Agents can run builds, tests, and dev servers on disposable
[Boat](https://boat.dev) cloud VMs instead of your machine. The agent pushes
its checkout to a VM, runs the command there, and deletes the VM when it is
done. Your code, diffs, and checkpoints stay where the thread runs, and it
works with every provider. Subagents can each get their own VM, so heavy checks
run side by side without slowing down your computer.

Boat bills by the second while a VM runs. You need a Boat account.

## Set up Boat on the T3 Code host

Agents run on the machine that hosts T3 Code, so install and sign in there. For
a remote, SSH, or T3 Connect environment that is the remote machine, not the
device you control it from.

```sh
curl -fsSL https://boat.dev/install | sh
boat login
boat status --check
```

The host also needs `jq` and OpenSSH (`ssh` and `ssh-keygen`). Agents copy
files to VMs over SSH. On a host whose network allows only HTTPS, agents can
still run commands on VMs and fetch work already pushed to your repository,
but cannot copy uncommitted edits. If Claude's sandbox is on, it blocks that
SSH connection too, so Claude reruns `boat push` outside the sandbox.
Permission modes that ask for approval will prompt for it.

Instead of `boat login`, you can give agents an API key: the CLI reads
`BOAT_API_KEY`. The desktop app does not pass variables exported in your shell
profile to agents, so set it as a sensitive environment variable on the
provider instance in **Settings → Providers**, or sign in with `boat login`.

Optionally, give agents Boat's documentation as MCP tools so they can look up
flags the skill does not cover:

```sh
claude mcp add --transport http boat-docs --scope user https://docs.boat.dev/mcp
codex mcp add boat-docs --url https://docs.boat.dev/mcp
```

## Make a template

A template is a VM disk with your toolchain and dependencies already
installed, so each new VM starts ready to work. VMs start in a couple of
seconds from either kind below.

1. Start a VM with `boat new --no-env --ttl 7200` and open a shell with
   `boat ssh <id>`.
2. Clone the repository into `/home/user/<project>`, install the toolchain, and
   install dependencies once so the package cache is warm.
3. Save it with `boat snapshot <id> <project>-template`, then remove the VM
   with `boat delete <id> --yes`. Agents start VMs from the name.

Saving again under the same name refreshes the template; new VMs use the new
version. `~/.cache` and `/tmp` are not saved, so keep caches you need, such as
Playwright browsers, elsewhere. Do not put secrets in the template. Files your
`.gitignore` excludes, such as `.env`, are not pushed to VMs.

Without `--no-env`, a VM gets your default Boat environment, which can hold
your GitHub and AI logins, and files it writes stay in a template made from
it. A VM copied from a saved template gets the environment you pass when you
create it, so the skill passes `--no-env` every time. A fork keeps the
environment of the VM it copies, so forks of a `--no-env` VM stay that way.
If checks need secrets, put only those in a separate environment
(`boat env new`), use `--environment <name>` instead of `--no-env`, and name
it in the skill's settings.

The Boat free trial cannot save snapshots. During the trial, stop the prepared
VM with `boat stop <id>` instead of saving it, and put its id in the skill as
the template: agents copy it with `boat fork`. Boat deletes a stopped VM after
14 days, and resuming it changes what later copies start from. The trial also
runs at most two VMs at once; a stopped template does not count.

## Add the skill to a project

1. Copy [`skills/boat-vm/SKILL.md`](./skills/boat-vm/SKILL.md) into the
   project as `.agents/skills/boat-vm/SKILL.md` for Codex and Antigravity, and
   as `.claude/skills/boat-vm/SKILL.md` for Claude. A symlink between the two
   folders works.
2. Fill in **Project settings** at the top: the template name (or the stopped
   VM's id during the trial), VM size, where the template holds the
   repository, the check command, and the dev server.
3. In threads that are already open, use **Restart agent session** in the
   command palette.

For other providers, paste the skill into the project's agent instructions.

## Use it

Pick the skill with `$boat-vm`, or ask in plain words:

- "Run the tests on a Boat VM."
- "Start the dev server on a VM so I can try it from my phone."
- "Split these fixes into subagents, each testing on its own VM."
- "Run this script I don't trust, or the compose stack, on a VM."

A single check uses a VM that is deleted as soon as the command ends. For
repeated checks or a dev server, the agent keeps one VM for the thread and
names it after the branch. A dev server's URL opens in the
[browser panel](./remote-access.md#browser-on-a-remote-environment) and works
from any device; it carries an access token, so share it only with people who
should reach the server.

In **Supervised** mode, each `boat` command asks first. See
[Permission modes](./permission-modes.md).

## Costs and cleanup

Agents delete their VMs when they finish. As a safety net, each VM stops one
hour after it starts unless the agent extends it, and Boat keeps a stopped
VM's disk for 14 days before deleting it. Run `boat list --all` to find
leftovers and `boat delete <id> --yes` to remove one. Your Boat plan limits
how many VMs run at once and how many start per minute; `boat limits` shows
what remains.
