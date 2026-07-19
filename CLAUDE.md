# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

An Ansible playbook that configures an Ubuntu laptop, run against `localhost`
over a local connection. There is no build and no test suite — correctness is
established by running the playbook and checking idempotency.

## Commands

```bash
ansible-galaxy install -r requirements.yml    # community.general, ansible.posix

ansible-playbook -i inventory.ini playbook.yml --syntax-check
ansible-playbook -i inventory.ini playbook.yml -K --check --diff
ansible-playbook -i inventory.ini playbook.yml -K
```

Run a subset by tag (`apt`, `dev-tools`, `docker`, `dotfiles`, `apps`, plus
per-language `python`, `node`, `go`, `rust`):

```bash
ansible-playbook -i inventory.ini playbook.yml -K --tags docker
```

Behaviour that is off by default, enabled per-run:

```bash
-e apt_perform_upgrade=true    # full apt upgrade
-e uv_self_update=true         # uv self update
-e rust_update=true            # rustup update
```

### Verifying a change

**The test is running the playbook twice; the second run must report
`changed=0`.** A task that reports `changed` every run is a bug even when the
end state is correct. `--check` mode is not sufficient: tasks that inspect a
tool an earlier task would have installed report failures on an unconfigured
machine, and several bugs here only appeared on a real run.

User-space tasks (uv, nvm, rustup, npm globals — anything under `$HOME`) can be
exercised without sudo or touching the real home directory, because every
derived path descends from `target_home`:

```yaml
- hosts: local
  become: false
  vars:
    target_home: /tmp/fakehome
  tasks:
    - ansible.builtin.include_tasks: roles/dev-tools/tasks/node.yml
```

## Architecture

Five roles in `playbook.yml`, in order: `apt-packages`, `dev-tools`, `docker`,
`dotfiles`, `snap-flatpak-apps`. `docker` and `dotfiles` are gated on
`docker_enabled` / `dotfiles_enabled`.

**Variables.** `group_vars/all.yml` is the single file a user edits. Each role
mirrors the variables it reads into its own `defaults/main.yml` so it still
works if lifted out; `group_vars` outranks role defaults, so the mirror never
shadows a real value. Adding a variable means editing both.

**Vendor apt repositories are one shared implementation.**
`roles/apt-packages/tasks/external_repos.yml` handles keyring directory, key
fetch, debconf preseeding, repo registration and the conditional cache refresh.
It takes a list in `external_repos` and has two callers:

- `apt-packages/tasks/main.yml` passes `apt_external_repos` via `include_tasks`
- `docker/tasks/main.yml` passes `docker_external_repos` via `include_role` +
  `tasks_from: external_repos.yml`

Do not hand-roll keyring or repo setup in a new role; call this instead. Docker
previously duplicated it, and a key-rotation fix had to be written twice.

Per-repo keys: `key_armored: false` for a binary `.gpg` keyring instead of an
armored `.asc`; `debconf:` to preseed an answer before the package installs
(VS Code uses this to decline adding its own source); `self_managed_marker:` for
vendors that take over their own source file.

**`self_managed_marker` is a bootstrap-only pattern.** Chrome's postinst
migrates our `.list` to a deb822 `.sources` file and deletes ours. Once the
marker path exists, the repo entry is skipped and our stale `.list` is removed —
our entry only has to survive long enough for the first install.

**Language runtimes resolve versions at run time, not at scaffold time.**
`node.version: "lts"` resolves via `nvm version-remote --lts`; `go.version:
"latest"` resolves from `https://go.dev/VERSION?m=text`; Rust follows the
`stable` channel. Python installs **uv only** — no interpreter — because uv's
`python-downloads` defaults to `automatic`, so each project's `.python-version`
drives the download. `dev_tools_languages.python.versions` is a cache warmer,
not the normal path.

**Privilege escalation.** `become: false` at play level; `become: true` on
individual tasks only. Everything under `$HOME` runs unprivileged.
`ansible_become_exe` in `group_vars/all.yml` points at `/usr/bin/sudo.ws` when
present: Ubuntu 25.10+ ships `sudo-rs`, which wraps Ansible's `-p` prompt in its
own template so Ansible never matches it and every privileged task times out.
Preflight asserts in `playbook.yml` catch this, and an unsupported architecture,
before anything runs.

## Conventions

**The config files carry no comments by design.** Task names describe what a
task does; the reasoning for a non-obvious choice lives in the commit message.
Before "simplifying" a line that looks arbitrary, run `git log -p` on it — most
odd-looking constructs here are fixes for bugs that reached a working machine.
New commits must carry that reasoning, since nothing else records it.

Prefer modules over `command`/`shell`. Where a shell call is unavoidable, make
it idempotent with `creates:`, a `when:` on a prior check, or an accurate
`changed_when:`. Every role gets a tag.

## Pitfalls

These have each produced a run that reported success while installing nothing.

**Substring tests in `when:`.** `'foo' in output` matches an installed
`foobar`; `'v2'` matches `v22.18.0`. Anchor with `regex_escape` and `search`,
as `roles/dev-tools/tasks/node.yml` does for both the Node version and npm
globals.

**Command output wider than what you are testing.** `nvm ls` prints the LTS
alias table next to installed versions, so a match can hit a version that is
not installed — hence `--no-alias`. Narrow the command rather than filtering
after.

**Non-zero exits on the fresh-machine path.** `nvm ls` exits 3 when nothing is
installed; `npm ls` exits non-zero on any unmet dependency. Both still print
usable output, so they carry `failed_when: false`.

**Version strings must be fully qualified.** A partial Go version has no
download and redirects to an HTML page that would be saved as a `.tar.gz`;
`go.yml` asserts `x.y.z` with `\Z` (not `$`, which also matches before a
trailing newline).

**Verify a package provides what you assume before adding it.** Read `snap
info`'s channel map rather than its description — `aws-cli`'s `latest/stable`
tracks v1, so the entry pins `v2/stable`.
