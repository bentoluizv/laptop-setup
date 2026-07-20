# laptop-setup — uso

*[English](../en/README.md) · [Português](README.md)*

Configuração Ansible para um notebook Ubuntu. Rode em uma máquina nova e
repita quando quiser — é idempotente.

Para adaptar a sua própria máquina ou estender o projeto, veja
[CONTRIBUTING.md](CONTRIBUTING.md).

## Roles

| Role | Instala | Tags |
| --- | --- | --- |
| `apt-packages` | Ferramentas de linha de comando, PPAs, repositórios apt de terceiros | `apt`, `packages`, `base` |
| `dev-tools` | build-essential, git, runtimes de linguagem, CLIs npm globais | `dev`, `dev-tools` |
| `openspec` | Semeia a configuração global e o autocompletar do openspec | `openspec`, `dev-tools` |
| `docker` | Docker Engine + plugin compose | `docker` |
| `dotfiles` | Clona um repositório de dotfiles e cria os links simbólicos no `$HOME` | `dotfiles` |
| `snap-flatpak-apps` | Aplicativos gráficos via snap e flatpak | `apps`, `snap`, `flatpak`, `gui` |
| `claude-desktop` | Aplicativo desktop do Claude, a partir do `.deb` da Anthropic | `claude-desktop`, `apps`, `gui` |
| `security-updates` | `unattended-upgrades`, restrito ao canal de segurança | `security`, `updates` |

## Pré-requisitos

```bash
sudo apt update
sudo apt install -y ansible git
ansible-galaxy install -r requirements.yml
```

## Configurar

Edite [`group_vars/all.yml`](../../group_vars/all.yml). Tudo que foi pensado
para ser personalizado está lá.

## Executar

```bash
ansible-playbook -i inventory.ini playbook.yml --ask-become-pass
```

Rodar apenas uma parte, usando tags:

```bash
ansible-playbook -i inventory.ini playbook.yml -K --tags docker
ansible-playbook -i inventory.ini playbook.yml -K --tags apt,apps
ansible-playbook -i inventory.ini playbook.yml -K --skip-tags dev-tools
```

Simular a execução sem alterar nada:

```bash
ansible-playbook -i inventory.ini playbook.yml -K --check --diff
```

No modo `--check`, tarefas que inspecionam uma ferramenta que só seria
instalada por uma tarefa anterior aparecem como falha em uma máquina ainda não
configurada. Isso é esperado.

Incluir um `apt upgrade` completo na execução:

```bash
ansible-playbook -i inventory.ini playbook.yml -K -e apt_perform_upgrade=true
```

Atualizar ferramentas que, por padrão, permanecem na versão já instalada:

```bash
ansible-playbook -i inventory.ini playbook.yml -K -e uv_self_update=true
ansible-playbook -i inventory.ini playbook.yml -K -e rust_update=true
```

## Executar em uma máquina nova com ansible-pull

O `ansible-pull` clona este repositório e executa localmente, sem precisar de
um checkout prévio:

```bash
sudo apt update && sudo apt install -y ansible git
ansible-pull -U https://github.com/bentoluizv/laptop-setup.git \
  -i inventory.ini playbook.yml --ask-become-pass
```

Ele usa a branch `main` por padrão; use `-C <branch>` para outra. O clone fica
em `~/.ansible/pull/<hostname>`, então repetir é apenas um `git pull` seguido
de nova execução.

Rodando com `sudo` ou como root é preciso informar qual conta configurar, senão
os dotfiles e os runtimes de linguagem vão parar em `/root`:

```bash
sudo ansible-pull -U https://github.com/bentoluizv/laptop-setup.git \
  -i inventory.ini playbook.yml -e target_user=$USER
```

## Segurança

As chaves apt de terceiros têm **fingerprint fixada**. Cada entrada em
`apt_external_repos` traz um `key_fingerprint`, conferido contra a chave baixada
antes de o repositório ser registrado; divergência aborta a execução. As chaves
são rebaixadas a cada execução, para que uma rotação seja percebida — e é
justamente por isso que a fixação importa.

Os instaladores do uv e do nvm são baixados em uma versão fixada e verificados
contra um `sha256` registrado em `group_vars/all.yml`, em vez de serem enviados
de uma URL direto para o shell. O rustup baixa o `rustup-init` e o verifica
contra a soma publicada ao lado dele — o que protege contra corrupção, mas não
contra um servidor comprometido, já que ambos vêm da mesma origem.

O role `security-updates` instala o `unattended-upgrades` limitado ao canal de
segurança, de modo que as correções chegam sem tornar uma execução completa do
playbook não determinística. O reinício automático está desligado
(`security_updates_reboot`).

Pertencer ao grupo `docker` equivale a ter root. Definir `docker_rootless: true`
muda para o modo rootless e dispensa o grupo por completo; vem desligado porque
altera um DDEV que já funciona e não é exercitado no CI.

## Depois da primeira execução

- Saia e entre na sessão de novo, ou rode `newgrp docker`, antes de usar o
  docker sem sudo. A troca de grupo não vale para a sessão atual.
- Rode `mkcert -install` uma vez se quiser https nos sites DDEV locais. Isso
  adiciona uma autoridade certificadora local ao seu sistema; `mkcert
  -uninstall` desfaz.
- Autentique as CLIs que for usar: `gh auth login`, `aws configure`.
- Abra o **Claude** e entre com sua conta Anthropic. O aplicativo desktop está
  em beta no Linux e não se atualiza sozinho — as versões novas chegam com
  `apt upgrade`, como qualquer outro pacote.

## Adicionar pacotes

**Pacote apt** — acrescente em `apt_packages`.

**snap** — acrescente em `snap_packages` como um mapa: `name`, mais os opcionais
`classic: true` e `channel:`.

**flatpak** — defina `flatpak_enabled: true` e acrescente o ID da aplicação em
`flatpak_packages`.

**CLI npm global** — acrescente em `npm_global_packages`. Fixe com `@x.y.z` ou
use `@latest`.

**Ferramenta CLI de Python** — acrescente o nome no PyPI em `uv_tools`; o uv a
instala como ferramenta isolada no `PATH`. `ansible-lint` e `yamllint` já vêm por
padrão para que o lint dos `.githooks` rode localmente, igual ao CI.

**Runtime de linguagem** — defina `enabled: true` na chave correspondente em
`dev_tools_languages`.

**Versão de Python** — normalmente você não precisa. O uv baixa a versão que o
`.python-version` ou o `requires-python` do projeto pedir, no primeiro uso:

```bash
uv python pin 3.12
uv sync
```

Use `dev_tools_languages.python.versions` só para pré-instalar versões, por
exemplo antes de trabalhar sem internet.

**Pacote vindo do repositório apt do próprio fornecedor** — acrescente uma
entrada em `apt_external_repos`:

```yaml
- name: example
  key_fingerprint: "0123456789ABCDEF0123456789ABCDEF01234567"
  key_url: https://example.com/key.asc
  repo: >-
    deb [arch={{ deb_arch }} signed-by=/etc/apt/keyrings/example.asc]
    https://example.com/apt stable main
```

e depois acrescente o nome do pacote em `apt_packages`. Chaves opcionais:

| Chave | Para que serve |
| --- | --- |
| `key_fingerprint` | Fingerprint esperada; a execução para se a chave baixada não bater |
| `key_armored: false` | O fornecedor serve um keyring binário `.gpg` em vez de um `.asc` |
| `self_managed_marker: <caminho>` | Assim que esse arquivo existe, o fornecedor passa a cuidar do repositório; o nosso é removido e não é recriado |
| `debconf:` | Responde uma pergunta do debconf antes de o pacote instalar; recebe `name`, `question`, `vtype`, `value` |

## Adicionar um role

```bash
mkdir -p roles/meu-role/{tasks,defaults}
```

Crie `tasks/main.yml` e `defaults/main.yml` e registre em
[`playbook.yml`](../../playbook.yml):

```yaml
- role: meu-role
  tags: [meu-role]
```

Coloque as variáveis em `group_vars/all.yml` e espelhe em `defaults/main.yml`
do role, para que ele continue funcionando sozinho.

Convenções: use `become: true` em tarefas individuais, nunca na play ou no
role; prefira módulos a `command`/`shell` e, quando o shell for inevitável,
proteja com `creates:`, um `when:` baseado em uma verificação anterior ou um
`changed_when:`; dê uma tag a todo role.

## Configurar os dotfiles

Defina `dotfiles_enabled: true`, aponte `dotfiles_repo` para o seu repositório
e liste os links:

```yaml
dotfiles_links:
  - {src: gitconfig, dest: "{{ target_home }}/.gitconfig"}
  - {src: tmux.conf, dest: "{{ target_home }}/.tmux.conf"}
```

Use uma URL `https://` para funcionar com `ansible-pull` antes de existir
qualquer chave SSH. Um arquivo real que já esteja no destino do link é movido
antes para `<nome>.bak.<timestamp>`.

Deixe os arquivos de configuração do seu shell fora do repositório. O playbook
escreve a configuração de todas as ferramentas em um único
`~/.config/laptop-setup/env.sh` (`shell_env_file`) e adiciona uma linha de
carregamento em cada arquivo listado em `shell_rc_files` que existir — por
padrão `.bashrc`, `.zshrc` e `.profile`. Assim o shell que você usa não
importa, e quem usa zsh não precisa mudar nada.

As entradas de PATH passam por uma função auxiliar que só acrescenta quando o
caminho ainda não está lá, então carregar o arquivo duas vezes — o que
acontece, já que o `.profile` carrega o `.bashrc` — não duplica nada.

Arquivos de configuração de shell só cobrem shells. Scripts, terminais
integrados de editor e agentes de código rodam shells não interativos, que não
leem nenhum desses arquivos — por definição. Por isso o playbook também escreve
o `~/.config/environment.d/10-laptop-setup.conf` (`session_env_file`), que o
systemd aplica à sessão inteira do usuário. Todo processo iniciado depois disso
herda os caminhos das ferramentas, use ele shell ou não.

**Só passa a valer no próximo login.** Até você sair e entrar de novo, agentes
e scripts ainda vão dizer que as ferramentas não existem, mesmo que o seu
terminal as encontre. Use `session_env_enabled: false` para desativar.

Se o seu repositório tiver um script de instalação próprio, defina
`dotfiles_install_script` com o caminho relativo à raiz dele; o script roda
depois do clone, no lugar de `dotfiles_links`.

## Solução de problemas

**`Timed out waiting for become success or become password prompt`**

O Ubuntu 25.10+ vem com o `sudo-rs`, que o Ansible não consegue controlar. O
playbook detecta isso antes de qualquer coisa rodar: procura o `sudo` clássico
em `/usr/bin/sudo.ws` na máquina alvo e aponta o `ansible_become_exe` para ele.
Se esse binário não existir, você recebe uma falha de verificação prévia
dizendo o que fazer, em vez de um timeout. Instale com:

```bash
sudo apt install sudo
```

Veja qual implementação você tem com `readlink -f "$(which sudo)"`.

**`~/.local/bin/fd is a real file, not a symlink`**

Algo instalou um binário nesse caminho. Remova o arquivo ou defina
`apt_link_fd: false`.

**O docker falha com erro de permissão**

Saia e entre na sessão de novo. Veja "Depois da primeira execução".
