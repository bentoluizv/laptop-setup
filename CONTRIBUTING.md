# Contributing

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
sudo apt install -y ansible git
ansible-galaxy install -r requirements.yml
```

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
CI runs all of them on every push and pull request.

**1. Syntax and lint**

```bash
ansible-playbook -i inventory.ini playbook.yml --syntax-check
pipx install ansible-lint yamllint    # or: sudo apt install ansible-lint yamllint
yamllint .
ansible-lint
```

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

The scenario passes `--skip-tags docker,apps`, because Docker Engine and snapd
both need systemd and neither works in a plain container. Those two roles are
covered by the `full-run` CI job, which runs the whole playbook on a real VM
runner.

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

Because the config carries no comments, commit messages are the only place the
reasoning is recorded. Write them for someone debugging this in a year.

- Subject in the imperative, under ~72 characters: `Fix become on sudo-rs`,
  not `fixed sudo stuff`.
- Body explains **why**, and what breaks without the change. Name the concrete
  failure, not just the edit.
- State how it was verified — `changed=0` on a second run, versions confirmed,
  whatever applies.

```
Anchor the npm global installed-check

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
