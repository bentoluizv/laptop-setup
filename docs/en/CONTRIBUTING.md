# Contributing

*[English](CONTRIBUTING.md) · [Português](../pt-BR/CONTRIBUTING.md)*

How to adopt this repo for your own machine, extend it, and get changes in.

## Adopting it for yourself

This is personal machine configuration. The package lists reflect one person's
setup, so the normal way to use it is to fork and edit, not to send a pull
request changing what gets installed.

```bash
git clone https://github.com/bentoluizv/laptop-setup.git
cd laptop-setup
git remote set-url origin https://github.com/<your-user>/laptop-setup.git
```

Then edit `group_vars/all.yml` and push to your own fork. Point the
`ansible-pull` command in [README.md](README.md) at your fork.

Pull requests are welcome for anything that improves the machinery itself:
role logic, idempotency bugs, new vendor-repo capabilities, documentation.

## Setting up to work on it

```bash
sudo apt install -y ansible git ansible-lint yamllint
ansible-galaxy install -r requirements.yml
git config core.hooksPath .githooks
```

That last line enables two hooks kept in [`.githooks/`](../../.githooks):

- **pre-commit** runs yamllint, ansible-lint and a syntax check over staged
  YAML — the same checks CI runs, before the commit exists. It skips a linter
  that is not installed rather than failing.
- **commit-msg** rejects a subject that does not follow Conventional Commits.

Bypass either with `git commit --no-verify` when you need to.

## Adding a package

Almost every addition is a one-line edit to `group_vars/all.yml`. Pick the
right list — [README.md](README.md#add-packages) documents each one and the
optional keys for vendor repos.

| What | Where |
| --- | --- |
| CLI or system package from Ubuntu's repos | `apt_packages` |
| Package from a vendor's own apt repo | `apt_external_repos` + `apt_packages` |
| GUI app | `snap_packages` or `flatpak_packages` |
| Global npm CLI | `npm_global_packages` |
| Language runtime | `dev_tools_languages` |

**Check the package exists and provides what you expect before adding it.**
This is not a formality — a snap's default channel does not necessarily track
the version you assume:

```bash
apt-cache policy <name>        # in Ubuntu's repos, and which version
snap info <name>               # read the channel map, not the description
npm view <name> version
```

For a vendor apt repo, confirm the package is actually published for your
architecture before writing the repo line:

```bash
curl -sf https://example.com/apt/dists/stable/main/binary-amd64/Packages.gz \
  | gunzip -c | grep -A3 '^Package: <name>$'
```

Then add it, and test as below.

## Testing a change

Nothing is accepted without an idempotency check. Four levels, cheapest first.

[`.github/workflows/ci.yml`](../../.github/workflows/ci.yml) runs all of them on every
push and pull request, in three jobs:

| Job | Covers |
| --- | --- |
| `Lint` | yamllint + ansible-lint |
| `Converge and idempotence (container)` | `molecule test` — everything except the `docker`, `apps` and `security` tags |
| `Full playbook on a real VM` | the whole playbook twice on a runner, then the verify assertions |

You do not need all of this locally. Level 1 and a twice-run on your own machine
catch most things; let CI do the container and VM work.

**1. Syntax and lint**

```bash
sudo apt install ansible-lint yamllint    # or: pipx install ansible-lint yamllint
ansible-playbook -i inventory.ini playbook.yml --syntax-check
yamllint .
ansible-lint
```

Lint config lives in [`.yamllint`](../../.yamllint) and
[`.ansible-lint`](../../.ansible-lint). Four rules are skipped deliberately:
`role-name` (the role directories are hyphenated by project layout),
`var-naming[no-role-prefix]` (`target_user`, `deb_arch` and friends are shared
across roles on purpose), and `command-instead-of-module` /
`risky-shell-pipe` (kept for shell calls that pipe, though the uv, nvm and
rustup installers are now downloaded and checksum-verified rather than piped).

**2. Dry run**

```bash
ansible-playbook -i inventory.ini playbook.yml -K --check --diff
```

Check mode reports failures for tasks that inspect a tool an earlier task would
have installed. That is expected on a machine that is not set up yet.

**3. Run it twice — this is the real test**

```bash
ansible-playbook -i inventory.ini playbook.yml -K
ansible-playbook -i inventory.ini playbook.yml -K
```

The second run **must** report `changed=0`. A run that works once is not
idempotent; a task that reports `changed` on every run is a bug, even when the
end state is correct.

**4. Molecule — a disposable container, converged and checked automatically**

```bash
pip install molecule "molecule-plugins[docker]" docker
molecule test
```

This needs a working Docker. If you have just run the playbook for the first
time, log out and back in first — the `docker` group is not active in the
session that added you to it.

The default sequence creates an `ubuntu:26.04` container, converges the
playbook, runs it a **second time and fails if anything reports `changed`**,
then runs the assertions in `molecule/default/verify.yml`. Useful subcommands
while iterating:

```bash
molecule converge     # create + run, leave the container up
molecule idempotence  # re-run against the live container
molecule verify       # assertions only
molecule login        # shell into it
molecule destroy
```

The scenario passes `--skip-tags docker,apps,security`, because Docker Engine,
snapd and unattended-upgrades all need systemd and none work in a plain
container. Those roles are covered by the `full-run` CI job, which runs the
whole playbook on a real VM runner.

**Verifying the end state anywhere.** `molecule/default/verify.yml` is an
ordinary playbook, so it also runs against your own machine:

```bash
ansible-playbook -i inventory.ini molecule/default/verify.yml
```

It asserts the tools exist, that node reports a real version, that go reports a
fully-qualified one, that rust is on the configured channel, that the managed
`~/.profile` blocks appear exactly once each, and that every entry in
`npm_global_packages` is installed.

**Testing user-space tasks without containers.** Anything that installs under
`$HOME` (uv, nvm, rustup, npm globals) can run against a throwaway home
directory, because `target_home` drives every derived path:

```yaml
- hosts: local
  become: false
  vars:
    target_home: /tmp/fakehome
  tasks:
    - ansible.builtin.include_tasks: roles/dev-tools/tasks/node.yml
```

Run that twice, confirm `changed=0`, then `rm -rf /tmp/fakehome`. This catches
install bugs without needing sudo or dirtying your real `$HOME`.

## Conventions

**No comments in the config files.** Task names describe what a task does; the
reasoning behind a non-obvious choice goes in the commit message, where
`git blame` and `git log` will surface it. If a line looks arbitrary, its
justification is one `git log -p` away — check there before "simplifying" it.

**`become: true` on individual tasks, never on a play or role.** Anything that
writes under `$HOME` runs unprivileged.

**Prefer modules over `command`/`shell`.** Where a shell call is unavoidable,
make it idempotent with `creates:`, a `when:` on a prior check, or an accurate
`changed_when:`.

**Every role gets a tag**, so it can run standalone via `--tags`.

**Variables live in `group_vars/all.yml`**, mirrored into the role's
`defaults/main.yml` so the role still works if lifted out of this repo.

## Adding a role

```bash
mkdir -p roles/my-role/{tasks,defaults}
```

Add `tasks/main.yml` and `defaults/main.yml`, then register it in
`playbook.yml`:

```yaml
- role: my-role
  tags: [my-role]
```

If the role needs a vendor apt repo, do not hand-roll the keyring and repo
setup. Call the shared implementation:

```yaml
- name: Register my vendor repository
  ansible.builtin.include_role:
    name: apt-packages
    tasks_from: external_repos.yml
  vars:
    external_repos: "{{ my_role_external_repos }}"
```

## Pitfalls

These have all caused real bugs in this repo.

**Substring tests in `when:`.** A bare `x not in output` check matches
partial names. `foo` matches an installed `foobar`; `v2` matches `v22.18.0`.
The task then skips and the play reports success while nothing was installed.
Anchor the match:

```yaml
when: not (output is search('(?m)/' ~ (name | regex_escape) ~ '$'))
```

**Command output that is a superset of what you want.** `nvm ls` prints the
LTS alias table alongside installed versions, so a naive match sees versions
that are not installed. Narrow the command (`--no-alias`) rather than
post-filtering.

**Commands that exit non-zero on the fresh-machine path.** `nvm ls` exits 3
with nothing installed; `npm ls` exits non-zero on any unmet dependency. Both
still print usable output. Add `failed_when: false` where a non-zero exit is
information rather than failure.

**Vendor packages that manage their own apt source.** Some `.deb` postinst
scripts add, migrate, or delete their own file in `/etc/apt/sources.list.d/`,
fighting the one Ansible writes and producing a permanent `changed`. Use
`self_managed_marker` so our entry only bootstraps the first install.

**Architecture and version assumptions.** `deb_arch` maps only x86_64 and
aarch64. Version strings must be fully qualified where upstream has no
redirect for a partial one.

**Filesystem tests that run on the wrong machine.** Jinja tests like
`'/some/path' is exists` evaluate on the **controller**, not the target. For a
local-connection run they are the same host, so a mistake here stays invisible
until the playbook runs against a container or a remote machine. Inspect the
target with `ansible.builtin.stat` and a `set_fact` instead. This is exactly
how `ansible_become_exe` was wrong until molecule exposed it.

## Git flow

`main` is the only long-lived branch and is expected to be working. Work on a
branch, one logical change per commit.

```bash
git switch -c fix-node-version-match
# edit, then test as above
git commit
git push -u origin fix-node-version-match
```

Open a pull request against `main`. Rebase rather than merge to keep history
linear:

```bash
git fetch origin
git rebase origin/main
```

### Commit messages

This repo follows [Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/).
The `commit-msg` hook enforces the subject; the body is on you.

```
<type>[optional scope][!]: <description>

[optional body]

[optional footer]
```

| Type | Use for |
| --- | --- |
| `feat` | A new capability — a role, an installed tool, a new option |
| `fix` | A defect: wrong behaviour, a broken run, a non-idempotent task |
| `docs` | Documentation only |
| `test` | Molecule scenarios, verify assertions |
| `ci` | Workflow and pipeline changes |
| `refactor` | Restructuring with no behaviour change |
| `style` | Formatting, line length, whitespace |
| `chore` | Tooling, lint config, housekeeping |
| `build` | Dependencies, `requirements.yml` |
| `perf` | Making something measurably faster |
| `revert` | Reverting an earlier commit |

Scope is optional and names the affected area: `fix(molecule):`,
`feat(docker):`, `docs(pt-br):`. A `!` before the colon marks a breaking
change — for this repo, anything that alters an existing machine's state on
rerun or removes a variable people set.

Because the config carries no comments, the body is the only place the
reasoning is recorded. Write it for someone debugging this in a year:

- Explain **why**, and what breaks without the change. Name the concrete
  failure, not the edit — the diff already shows the edit.
- State how it was verified: `changed=0` on a second run, versions confirmed,
  whatever applies.

```
fix: anchor the npm global installed-check

A bare substring test matched "foo" against an installed "foobar", so the
package was never installed while the play reported success. Anchor the
match to /<name> at end of line.

Verified: two consecutive runs report changed=0 and the package installs on
a machine where a longer name is already present.
```

### Rewriting history

Fine on your own branch before review. Avoid it on `main` — it is published,
so rewriting requires a force-push and breaks every existing clone.

If you must, use `--force-with-lease` rather than `--force`, so the push aborts
if anything landed on the remote that you have not fetched. Note that
`git filter-branch -- --all` also rewrites remote-tracking refs, which
invalidates the lease check; run `git fetch` afterwards to repair it.
