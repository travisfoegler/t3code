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

The agent also needs `jq`. To sign in with an API key instead of a browser,
pipe it in with `boat login --key-stdin`. The desktop app does not pass keys
exported in your shell profile to agents, so sign in with `boat login` rather
than setting a variable there.

## Make a template

A template is a saved VM disk with your toolchain and dependencies already
installed, so each new VM starts ready to work.

1. Start a VM with `boat new` and open a shell with `boat ssh <id>`.
2. Clone the repository into `/home/user/<project>`, install the toolchain, and
   install dependencies once so the package cache is warm.
3. Save it with `boat snapshot <id> <project>-template`, then remove the VM
   with `boat delete <id> --yes`.

Save again under the same name to refresh the template; new VMs use the new
version. `~/.cache` and `/tmp` are not saved, so keep caches you need, such as
Playwright browsers, elsewhere. Do not put secrets in the template. Files your
`.gitignore` excludes, such as `.env`, are not pushed to VMs, so give VMs
their settings and secrets through a Boat environment.

## Add the skill to a project

1. Copy [`skills/boat-vm/SKILL.md`](./skills/boat-vm/SKILL.md) into the
   project as `.agents/skills/boat-vm/SKILL.md` for Codex and Antigravity, and
   as `.claude/skills/boat-vm/SKILL.md` for Claude. A symlink between the two
   folders works.
2. Fill in **Project settings** at the top: the template name, VM size, where
   the template holds the repository, the check command, and the dev server.
3. In threads that are already open, use **Restart agent session** in the
   command palette.

For other providers, paste the skill into the project's agent instructions.

## Use it

Pick the skill with `$boat-vm`, or ask in plain words:

- "Run the tests on a Boat VM."
- "Start the dev server on a VM so I can try it from my phone."
- "Split these fixes into subagents, each testing on its own VM."

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
