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

Molecule automates exactly that in a disposable container — its default
sequence includes an `idempotence` step, then runs the assertions in
`molecule/default/verify.yml`:

```bash
molecule test        # create, converge, idempotence, verify, destroy
molecule converge    # leave the container up while iterating
yamllint . && ansible-lint
```

The scenario skips `docker,apps,security`: Docker Engine, snapd and
`unattended-upgrades` all need systemd, which a plain container lacks.
`.github/workflows/ci.yml` covers those in a
`full-run` job on a real VM runner, which also re-runs the playbook and greps
the recap for `changed=0`.

`molecule/default/verify.yml` is a normal playbook and runs against any host,
including this machine:
`ansible-playbook -i inventory.ini molecule/default/verify.yml`.

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

Six roles in `playbook.yml`, in order: `apt-packages`, `dev-tools`, `docker`,
`dotfiles`, `snap-flatpak-apps`, `security-updates`. `docker`, `dotfiles` and
`security-updates` are gated on `docker_enabled` / `dotfiles_enabled` /
`security_updates_enabled`.

User-facing docs live in `docs/en/` and `docs/pt-BR/`, with a short index at
the root `README.md`. **Both languages must be updated together** — a change to
`docs/en/README.md` needs the matching edit in `docs/pt-BR/README.md`. This
file stays at the root and in English.

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
Ubuntu 25.10+ ships `sudo-rs`, which wraps Ansible's `-p` prompt in its own
template, so Ansible never matches it and every privileged task times out. The
`pre_tasks` in `playbook.yml` stat `/usr/bin/sudo.ws` **on the target** and
`set_fact` `ansible_become_exe` to it when present, then assert that the active
sudo is drivable. Architecture is asserted there too.

That stat is deliberately target-side. It was once a `group_vars` expression
using the `exists` test, which evaluates on the **controller** — correct only
because this is normally a local connection, and wrong the moment the playbook
runs against a container or any other host.

## Conventions

**The config files carry no comments by design.** Task names describe what a
task does; the reasoning for a non-obvious choice lives in the commit message.
Before "simplifying" a line that looks arbitrary, run `git log -p` on it — most
odd-looking constructs here are fixes for bugs that reached a working machine.
New commits must carry that reasoning, since nothing else records it.

Prefer modules over `command`/`shell`. Where a shell call is unavoidable, make
it idempotent with `creates:`, a `when:` on a prior check, or an accurate
`changed_when:`. Every role gets a tag.

**Commits follow [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/)**
— `<type>[scope][!]: <description>`, subject under 72 characters, blank line
before the body. The `commit-msg` hook in `.githooks/` enforces the subject and
`pre-commit` runs the lint CI runs; both are active once someone has run
`git config core.hooksPath .githooks`, so a fresh clone may not have them.
Messages are written in English regardless of doc language.

**Open questions are settled before the work is published, never inside it.** A
commit body or pull request description states decisions and their rationale.
It is not the place to ask "should I also…?" or "say the word if you'd rather…".
Both are read asynchronously and merged past, so a question parked there either
blocks review on the author or lets the decision default silently. Raise the
doubt while the work is still in progress, get an answer, then write that answer
in as a decision.

Vendor apt keys carry a pinned `key_fingerprint` verified after download, and
the uv/nvm installers are checksum-pinned in `group_vars/all.yml`. Changing a
pinned version means updating its recorded `sha256` in the same commit.

## Pitfalls

These have each produced a run that reported success while installing nothing,
or a commit that recorded a reason which was not true.

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

**A vendor's prose is not the state of the machine.** Install docs describe the
happy path on a clean system, not what a real one looks like after someone
followed them by hand. The Claude desktop docs describe an apt entry the package
writes and leaves commented out; on this machine that file is live and owned by
no package at all. Both facts went into a commit message as justification before
either was checked, and `dpkg-query -W -f='${Conffiles}'` plus `dpkg -S`
disproved them in seconds. Read the machine before writing the rationale — and
when a claim about packaging behaviour is load-bearing for a design decision,
it belongs in the commit message only once a command has confirmed it.
