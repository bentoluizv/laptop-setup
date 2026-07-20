# laptop-setup — usage

*[English](README.md) · [Português](../pt-BR/README.md)*

Ansible configuration for an Ubuntu laptop. Run it on a fresh machine, and
rerun it any time — it is idempotent.

To adopt it for your own machine or extend it, see
[CONTRIBUTING.md](CONTRIBUTING.md).

## Roles

| Role | Installs | Tags |
| --- | --- | --- |
| `apt-packages` | CLI tools, PPAs, vendor apt repos | `apt`, `packages`, `base` |
| `dev-tools` | build-essential, git, language runtimes, global npm CLIs | `dev`, `dev-tools` |
| `openspec` | Seeds openspec's global config and shell completion | `openspec`, `dev-tools` |
| `docker` | Docker Engine + compose plugin | `docker` |
| `dotfiles` | Clones a dotfiles repo and symlinks it into `$HOME` | `dotfiles` |
| `snap-flatpak-apps` | GUI apps via snap and flatpak | `apps`, `snap`, `flatpak`, `gui` |
| `claude-desktop` | Claude desktop app, from Anthropic's `.deb` | `claude-desktop`, `apps`, `gui` |
| `security-updates` | `unattended-upgrades`, restricted to the security pocket | `security`, `updates` |

## Prerequisites

```bash
sudo apt update
sudo apt install -y ansible git
ansible-galaxy install -r requirements.yml
```

## Configure

Edit [`group_vars/all.yml`](../../group_vars/all.yml). Everything intended to be
customised lives there.

## Run

```bash
ansible-playbook -i inventory.ini playbook.yml --ask-become-pass
```

Run a subset with tags:

```bash
ansible-playbook -i inventory.ini playbook.yml -K --tags docker
ansible-playbook -i inventory.ini playbook.yml -K --tags apt,apps
ansible-playbook -i inventory.ini playbook.yml -K --skip-tags dev-tools
```

Preview without changing anything:

```bash
ansible-playbook -i inventory.ini playbook.yml -K --check --diff
```

In check mode, tasks that inspect a tool an earlier task would have installed
report failures on a machine that has not been set up yet. That is expected.

Include a full `apt upgrade` in a run:

```bash
ansible-playbook -i inventory.ini playbook.yml -K -e apt_perform_upgrade=true
```

Update tools that are otherwise left at their installed version:

```bash
ansible-playbook -i inventory.ini playbook.yml -K -e uv_self_update=true
ansible-playbook -i inventory.ini playbook.yml -K -e rust_update=true
```

## Run on a brand-new machine with ansible-pull

`ansible-pull` clones this repo and runs it locally, with no prior checkout:

```bash
sudo apt update && sudo apt install -y ansible git
ansible-pull -U https://github.com/bentoluizv/laptop-setup.git \
  -i inventory.ini playbook.yml --ask-become-pass
```

It defaults to the `main` branch; use `-C <branch>` for another. It clones to
`~/.ansible/pull/<hostname>`, so reruns are a `git pull` and replay.

Running under `sudo` or as root requires naming the account to configure,
otherwise dotfiles and language runtimes land in `/root`:

```bash
sudo ansible-pull -U https://github.com/bentoluizv/laptop-setup.git \
  -i inventory.ini playbook.yml -e target_user=$USER
```

## Security

Vendor apt keys are **fingerprint-pinned**. Each entry in `apt_external_repos`
carries a `key_fingerprint`, checked against the downloaded key before the
repository is registered; a mismatch aborts the run. Keys are re-fetched every
run so rotation is picked up, which is exactly why the pin matters.

The uv and nvm installers are downloaded at a pinned version and verified
against a `sha256` recorded in `group_vars/all.yml`, rather than piped from a
URL into a shell. rustup downloads `rustup-init` and verifies it against the
checksum published beside it — that protects against corruption, not against a
compromised upstream host, since both come from the same origin.

`security-updates` installs `unattended-upgrades` limited to the security
pocket, so patches land without making a full playbook run non-deterministic.
Automatic reboots are off (`security_updates_reboot`).

Docker group membership is root-equivalent. Setting `docker_rootless: true`
switches to rootless mode and skips the group entirely; it is off by default
because it changes a working DDEV setup and is not exercised in CI.

## After the first run

- Log out and back in, or run `newgrp docker`, before using docker without
  sudo. Group membership does not apply to the current session.
- Run `mkcert -install` once if you want https on local DDEV sites. It adds a
  local root CA to your system trust store; `mkcert -uninstall` reverses it.
- Authenticate the CLIs you use: `gh auth login`, `aws configure`.
- Launch **Claude** and sign in with your Anthropic account. The desktop app is
  in beta on Linux and does not update itself — new versions arrive with
  `apt upgrade` like any other package.

## Add packages

**apt package** — append to `apt_packages`.

**snap** — append to `snap_packages` as a mapping: `name`, plus optional
`classic: true` and `channel:`.

**flatpak** — set `flatpak_enabled: true` and append the application ID to
`flatpak_packages`.

**Global npm CLI** — append to `npm_global_packages`. Pin with `@x.y.z`, or use
`@latest`.

**Language runtime** — set `enabled: true` under the relevant key in
`dev_tools_languages`.

**Python version** — normally you do not. uv downloads whatever a project's
`.python-version` or `requires-python` asks for on first use:

```bash
uv python pin 3.12
uv sync
```

Use `dev_tools_languages.python.versions` only to preinstall versions, for
example before working offline.

**Package from a vendor's own apt repo** — add an entry to
`apt_external_repos`:

```yaml
- name: example
  key_url: https://example.com/key.asc
  repo: >-
    deb [arch={{ deb_arch }} signed-by=/etc/apt/keyrings/example.asc]
    https://example.com/apt stable main
```

then add the package name to `apt_packages`. Optional keys:

| Key | Use |
| --- | --- |
| `key_armored: false` | Vendor serves a binary `.gpg` keyring rather than an armored `.asc` |
| `self_managed_marker: <path>` | Once this file exists the vendor manages the repo itself; ours is removed and no longer re-added |
| `debconf:` | Preseed a debconf answer before the package installs; takes `name`, `question`, `vtype`, `value` |

## Add a role

```bash
mkdir -p roles/my-role/{tasks,defaults}
```

Add `tasks/main.yml` and `defaults/main.yml`, then register it in
[`playbook.yml`](../../playbook.yml):

```yaml
- role: my-role
  tags: [my-role]
```

Put variables in `group_vars/all.yml`, and mirror them into the role's
`defaults/main.yml` so the role still works on its own.

Conventions: put `become: true` on individual tasks, never on the play or role;
prefer modules over `command`/`shell`, and where a shell call is unavoidable
guard it with `creates:`, a `when:` on a prior check, or `changed_when:`; give
every role a tag.

## Set up dotfiles

Set `dotfiles_enabled: true`, point `dotfiles_repo` at your repo, and list the
symlinks:

```yaml
dotfiles_links:
  - {src: gitconfig, dest: "{{ target_home }}/.gitconfig"}
  - {src: tmux.conf, dest: "{{ target_home }}/.tmux.conf"}
```

Use an `https://` URL so it works under `ansible-pull` before any SSH key
exists. A real file already at a link target is moved to
`<name>.bak.<timestamp>` first.

Leave your shell rc files out of the repo. The playbook writes every tool's
setup into a single `~/.config/laptop-setup/env.sh` (`shell_env_file`) and adds
a one-line hook to each rc file in `shell_rc_files` that exists — `.bashrc`,
`.zshrc`, `.profile` by default. So the shell you use does not matter, and a
zsh user needs no configuration change.

PATH entries go through a helper that only prepends when the entry is absent,
so sourcing the file twice — which happens, since `.profile` sources `.bashrc`
— cannot duplicate them.

Shell rc files only cover shells. Scripts, editor-integrated terminals and
coding agents run non-interactive shells, which read no rc file at all — by
design. So the playbook also writes
`~/.config/environment.d/10-laptop-setup.conf` (`session_env_file`), which
systemd applies to the entire user session. Every process started after that
inherits the tool paths, whatever shell it uses or does not use.

**It takes effect at the next login.** Until you log out and back in, agents
and scripts will still report the tools as missing even though your terminal
finds them. Set `session_env_enabled: false` to skip this.

If your repo has its own install script, set `dotfiles_install_script` to its
path relative to the repo root and it runs after cloning, instead of using
`dotfiles_links`.

## Troubleshooting

**`Timed out waiting for become success or become password prompt`**

Ubuntu 25.10+ ships `sudo-rs`, which Ansible cannot drive. The playbook detects
this before anything runs: it looks for the classic `/usr/bin/sudo.ws` on the
target and points `ansible_become_exe` at it. If that binary is missing you get
a preflight failure naming the fix rather than a timeout. Install it with:

```bash
sudo apt install sudo
```

Check which implementation you have with `readlink -f "$(which sudo)"`.

**`~/.local/bin/fd is a real file, not a symlink`**

Something else installed a binary at that path. Remove it, or set
`apt_link_fd: false`.

**Docker fails with a permissions error**

Log out and back in. See "After the first run".
