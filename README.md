# laptop-setup

Ansible automation for configuring an Ubuntu laptop. Run it once on a fresh
machine, rerun it whenever you like — every task is idempotent, so a second run
changes nothing unless you changed the config.

## What it does

| Role | Purpose | Tags |
| --- | --- | --- |
| `apt-packages` | Base CLI/system packages, PPAs, vendor repos (Chrome) | `apt`, `packages`, `base` |
| `dev-tools` | build-essential, git, opt-in language runtimes (Python/Node/Go/Rust), global npm CLIs | `dev`, `dev-tools` |
| `docker` | Docker Engine + compose plugin from Docker's own apt repo | `docker` |
| `dotfiles` | Clone a git repo of shell/tool config (`.bashrc`, `.gitconfig`, …) and symlink it into `$HOME`. Off unless you keep such a repo. Unrelated to .NET | `dotfiles` |
| `snap-flatpak-apps` | GUI apps via snap and/or flatpak | `apps`, `snap`, `flatpak`, `gui` |

## Prerequisites

Ubuntu 22.04 or newer, plus:

```bash
sudo apt update
sudo apt install -y ansible git
ansible-galaxy install -r requirements.yml
```

Ansible 2.15+ is recommended. `requirements.yml` pulls in `community.general`
(for the `snap` and `flatpak` modules) and `ansible.posix`.

## Configure it first

Open [`group_vars/all.yml`](group_vars/all.yml) and fill in the parts marked
`TODO` — package lists, which language runtimes you want, your dotfiles repo
URL. The lists ship empty on purpose; nothing gets installed that you didn't
ask for.

## Running it

```bash
ansible-playbook -i inventory.ini playbook.yml --ask-become-pass
```

`--ask-become-pass` (`-K`) is needed because some tasks escalate to root.
Escalation is per-task, not blanket — anything that writes inside `$HOME`
(dotfiles, uv, nvm, rustup) runs as you.

### If become times out on Ubuntu 25.10 or newer

Ubuntu 25.10+ replaced the default `sudo` with `sudo-rs`, which renders
Ansible's `-p` prompt inside its own template rather than using it verbatim.
Ansible cannot match the resulting prompt and every privileged task fails with:

```
Timed out waiting for become success or become password prompt
```

`group_vars/all.yml` handles this by pointing `ansible_become_exe` at the
classic sudo (`/usr/bin/sudo.ws`) when that binary exists, falling back to
plain `sudo` otherwise. If you are on such a release and the classic binary is
missing, install it with `sudo apt install sudo`. To check what you have:

```bash
readlink -f "$(which sudo)"    # /usr/lib/cargo/bin/sudo => sudo-rs
```

Useful variations:

```bash
# Preview without changing anything
ansible-playbook -i inventory.ini playbook.yml -K --check --diff

# Just one part
ansible-playbook -i inventory.ini playbook.yml -K --tags docker
ansible-playbook -i inventory.ini playbook.yml -K --tags apt,apps

# Everything except the slow language builds
ansible-playbook -i inventory.ini playbook.yml -K --skip-tags dev-tools

# Include a full apt upgrade this run
ansible-playbook -i inventory.ini playbook.yml -K -e apt_perform_upgrade=true
```

Note: `--check` mode reports failures on tasks that depend on something an
earlier task would have installed (e.g. checking a uv that isn't there yet).
That's expected on a machine that hasn't been set up.

## Bootstrapping a brand-new machine with `ansible-pull`

`ansible-pull` clones this repo and runs it against the local machine — no
prior checkout needed. On a fresh install:

```bash
sudo apt update && sudo apt install -y ansible git
ansible-pull -U https://github.com/<you>/laptop-setup.git \
  -i inventory.ini playbook.yml \
  --ask-become-pass
```

Two things to know:

- `ansible-pull` defaults to the `main` branch; add `-C <branch>` for another.
- It clones to `~/.ansible/pull/<hostname>` and reruns are just `git pull` +
  replay, so it's safe to run repeatedly.

If you run it under `sudo` or as root, pass the account you're configuring
explicitly, or dotfiles and language runtimes land in `/root`:

```bash
sudo ansible-pull -U https://github.com/<you>/laptop-setup.git \
  -i inventory.ini playbook.yml -e target_user=$USER
```

To keep a machine continuously in sync, put that command in a cron entry or a
systemd timer.

## Adding things later

**A new apt package** — append to `apt_packages` in `group_vars/all.yml`. Done.

**A new snap or flatpak** — append to `snap_packages` (a mapping: `name`, plus
optional `classic: true` and `channel:`) or `flatpak_packages` (the full
application ID, e.g. `com.spotify.Client`). Set `flatpak_enabled: true` to turn
flatpak support on.

**A new language runtime** — flip `enabled: true` under the relevant key in
`dev_tools_languages` and set the version.

**A Python version** — you generally don't. Pin it in the project instead:

```bash
cd ~/code/my-project
uv python pin 3.12     # writes .python-version
uv sync                # fetches that interpreter if it's missing
```

`dev_tools_languages.python.versions` exists only to warm the cache ahead of
time (offline work, slow link).

**A global npm CLI** — append to `npm_global_packages`. Pin with `@x.y.z` for
reproducibility, or use `@latest` to take whatever's current at install time.
These install into the nvm-managed Node, never with sudo.

**An app only shipped through a vendor's own apt repo** — add an entry to
`apt_external_repos` (`name`, `key_url`, `repo`) and put the package name in
`apt_packages`. Chrome and `gh` are the worked examples. If the vendor serves a
binary keyring rather than an armored one, set `key_armored: false`.

**A whole new role**:

```bash
mkdir -p roles/my-role/{tasks,defaults}
```

Add `tasks/main.yml` and `defaults/main.yml`, then register it in
[`playbook.yml`](playbook.yml):

```yaml
- role: my-role
  tags: [my-role]
```

Put the knobs in `group_vars/all.yml` (the single place anyone customizing this
should have to look) and mirror them into the role's `defaults/main.yml` so the
role still works standalone.

Conventions worth keeping when you do:

- `become: true` on individual tasks, never on the play or the role.
- No `command`/`shell` where a module exists. Where a shell call is
  unavoidable, guard it with `creates:`, a `when:` on a prior check, or
  `changed_when:` so reruns stay clean.
- Give every role a tag.

## What's currently configured

- **git** — installed by `dev-tools` (base packages) and listed in
  `apt_packages`, so it's present whether you run the whole playbook or just
  `--tags apt`.
- **CLI tools** — `curl`, `wget`, `ripgrep`, `fd-find`, `fzf`, `tree`, `htop`,
  `jq`, `unzip`, `vim`, `tmux`.
- **Python** — uv only; interpreters arrive per-project (see below).
- **Node** — nvm, tracking the current LTS (`version: "lts"`).
- **Go** — current stable, into `/usr/local/go` (`version: "latest"`).
- **Rust** — rustup on the `stable` channel.
- **OpenSpec** — `@fission-ai/openspec`, a global npm CLI on that Node.
- **VS Code** — `code` from Microsoft's apt repo.
- **Google Chrome** — `google-chrome-stable` from Google's apt repo.
- **GitHub CLI** — `gh` from GitHub's apt repo.
- **AWS CLI v2** — the official `aws-cli` snap.
- **DDEV** — `ddev` from DDEV's apt repo, plus `mkcert`. See the Drupal
  section below.

## Drupal development

Only DDEV is installed machine-wide. Everything else — Drush, Drupal itself,
PHP, Composer — lives inside the project, which is how both tools intend it.

**Drush is per-project, by design.** Upstream is explicit that it supports one
install method: Composer, as a project dependency. There is no supported global
install, and Drush Launcher is not part of the current story. Each site gets the
Drush version matching its codebase:

```bash
ddev composer require drush/drush
ddev drush status
```

**You don't need PHP or Composer on the host.** DDEV runs both inside the
container, so `ddev composer` and `ddev drush` work on a machine with no PHP
installed. Keeping PHP off the host also avoids a host version silently
mismatching what a project expects.

**Using `~/.drush` site aliases.** A project-local Drush reads global alias
files, but *only* if you tell it where to look — Drush does not search
`~/.drush/sites` by default. Create `~/.drush/drush.yml`:

```yaml
drush:
  paths:
    alias-path:
      - '${env.HOME}/.drush/sites'
```

With that in place, alias files at `~/.drush/sites/*.site.yml` resolve from any
project, and remote aliases (those with `host:`/`user:`) dispatch over SSH.
Note that a global Drush install is not the answer here: upstream supports only
the per-project Composer install, and Drush Launcher — the usual way to get a
global `drush` — was archived in August 2023.

Starting a new site:

```bash
mkdir my-site && cd my-site
ddev config --project-type=drupal --docroot=web
ddev start
ddev composer create drupal/recommended-project
ddev composer require drush/drush
ddev drush site:install --account-pass=admin -y
ddev launch
```

**One manual step: `mkcert -install`.** The playbook installs `mkcert` but does
not run it. That command adds a new root certificate authority to your system
trust store — a machine-wide security change, and the sort of thing that should
be a deliberate act rather than a side effect of a setup run. Run it once when
you want https on local sites:

```bash
mkcert -install     # `mkcert -uninstall` reverses it
```

DDEV works over http without it; you'll just get certificate warnings on https.

**DDEV needs Docker**, which the `docker` role provides. On a first run the
docker group membership isn't active until you log out and back in, so `ddev
start` will fail with a permissions error until you do.

## Notes

- **Docker group**: joining the `docker` group grants root-equivalent access to
  the machine. It's on by default for convenience; set
  `docker_add_user_to_group: false` if you'd rather use `sudo docker`. Group
  membership only takes effect after logging out and back in.
- **Shell environment**: uv, nvm, Go, and cargo append blocks to
  `~/.profile`, each fenced with `# BEGIN/END ANSIBLE MANAGED` markers so
  reruns update in place rather than duplicating. If you use zsh or fish,
  either source `~/.profile` from your rc file or adjust the `blockinfile`
  paths in `roles/dev-tools/tasks/`.
- **Python via uv**: this role installs uv and nothing else — no interpreter is
  provisioned at setup time. uv's `python-downloads` setting defaults to
  `automatic`, so `uv run`, `uv sync`, and `uv venv` download whatever version
  a project's `.python-version` or `requires-python` asks for, on first use.
  Which Python a project needs is the project's business; the laptop only
  needs to be able to get it. Consequences worth knowing:
  - Nothing here touches the system `python3`, and no bare `python` is created
    unless you set `dev_tools_languages.python.default`.
  - The first `uv sync` in a new project pauses briefly to download an
    interpreter. Subsequent projects on the same version reuse it.
  - This needs network on first use of a given version. To work offline,
    preload with `uv python install 3.12`, or list versions in
    `dev_tools_languages.python.versions` so the playbook does it.
  - uv is installed with `UV_NO_MODIFY_PATH=1` so its installer doesn't rewrite
    `~/.profile` behind Ansible's back; the PATH entry is managed here instead.
    `uv self update` is opt-in via `-e uv_self_update=true`.
- **Renamed binaries**: two apt packages don't install under the name you'd
  expect. `ripgrep` provides `rg`. `fd-find` provides `fdfind`, because Debian
  already had an `fd` package — the role symlinks `~/.local/bin/fd` to it
  (`apt_link_fd: false` to opt out). Ubuntu's stock `~/.profile` puts
  `~/.local/bin` on `PATH` when it exists.
- **Chrome**: Google has no official snap or flatpak, so it comes from their
  apt repo. The `.deb` normally re-adds its own source file on every upgrade,
  which would collide with the one Ansible writes and report a change on every
  run; the role writes `/etc/default/google-chrome` with `repo_add_once=false`
  to prevent that.
- **Signing key formats**: vendors ship apt keys either ASCII-armored (`.asc`,
  e.g. Google) or as a binary keyring (`.gpg`, e.g. GitHub). apt reads both,
  but the file extension has to match the content or `signed-by` verification
  fails, so each entry in `apt_external_repos` declares which it serves via
  `key_armored`.
- **AWS CLI**: not available in apt in any form, so it comes from the snap AWS
  officially publishes and supports, which auto-refreshes. The alternative is
  AWS's zip installer, which needs `--update` passed on any reinstall (it
  errors out otherwise) and tracks no version state — worth switching to only
  if you need to pin a specific minor version.
- **VS Code**: installed from Microsoft's apt repo, not the snap. The `code`
  `.deb` normally prompts via debconf to add that repo itself, which would
  leave a `vscode.sources` next to the `vscode.list` Ansible writes — two
  source files for one repo, and an apt warning on every update. The role
  preseeds `code/add-microsoft-repo` to `false` so Ansible owns the repo.
  Don't also add the `code` snap: both provide `/usr/bin/code` and would
  fight over the `.desktop` handler.
- **Node LTS tracking**: `dev_tools_languages.node.version: "lts"` resolves the
  newest LTS release at run time via `nvm version-remote --lts` and installs it
  only if it's missing, with `nvm alias default 'lts/*'`. So a new major shows
  up on the first run after release, and reruns in between change nothing.
  The tradeoff: two teammates running this months apart get different Node
  majors. Pin `version: "22"` if you'd rather have reproducibility. Old
  versions are left installed — `nvm uninstall vX` to clean up.
- **Go and Rust versions**: neither language has an LTS line. Go supports the
  two most recent majors, so `version: "latest"` resolves the current stable
  release from `https://go.dev/VERSION?m=text` at run time and installs it only
  if it differs from what's there. Rust's `stable` is a moving channel, but
  rustup only advances it when told — `-e rust_update=true` does that. Both can
  be pinned (`"1.23.4"`, `"1.83.0"`) if you need reproducibility.
- **OpenSpec**: installed globally into the nvm Node, so it's only on `PATH`
  in a shell where nvm has been sourced. A fresh login handles that; the
  Ansible run itself won't have it on `PATH`. Note that npm globals are
  per-Node-version: after an LTS bump the playbook reinstalls everything in
  `npm_global_packages` into the new Node, but globals you installed by hand
  won't carry over.
- **Dotfiles**: the clone pulls updates on rerun but never force-resets, so
  local commits survive. Any real file sitting where a symlink should go is
  moved to `<name>.bak.<timestamp>` first. The timestamp matters: if you ever
  replace a managed symlink with a real file, a later run backs that up too
  instead of overwriting the earlier backup and losing your edits.
