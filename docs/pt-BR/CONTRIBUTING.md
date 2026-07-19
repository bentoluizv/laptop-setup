# Contribuindo

*[English](../en/CONTRIBUTING.md) · [Português](CONTRIBUTING.md)*

Como adaptar este repositório para a sua máquina, estendê-lo e enviar mudanças.

## Adaptando para você

Isto é configuração pessoal de máquina. As listas de pacotes refletem o setup
de uma pessoa, então o caminho normal é fazer um fork e editar, não abrir um
pull request mudando o que é instalado.

```bash
git clone https://github.com/bentoluizv/laptop-setup.git
cd laptop-setup
git remote set-url origin https://github.com/<seu-usuario>/laptop-setup.git
```

Depois edite `group_vars/all.yml` e envie para o seu fork. Aponte o comando
`ansible-pull` do [README.md](README.md) para o seu fork.

Pull requests são bem-vindos para tudo que melhore o mecanismo em si: lógica
dos roles, falhas de idempotência, novos recursos para repositórios de
terceiros, documentação.

## Preparando o ambiente

```bash
sudo apt install -y ansible git ansible-lint yamllint
ansible-galaxy install -r requirements.yml
git config core.hooksPath .githooks
```

A última linha ativa dois hooks guardados em [`.githooks/`](../../.githooks):

- **pre-commit** roda yamllint, ansible-lint e uma verificação de sintaxe sobre
  os YAML no stage — as mesmas checagens do CI, antes de o commit existir. Se um
  linter não estiver instalado, ele é pulado em vez de falhar.
- **commit-msg** rejeita um assunto que não siga o Conventional Commits.

Use `git commit --no-verify` para ignorar os hooks quando precisar.

## Adicionando um pacote

Quase toda adição é uma linha em `group_vars/all.yml`. Escolha a lista certa —
o [README.md](README.md#adicionar-pacotes) documenta cada uma e as chaves
opcionais dos repositórios de terceiros.

| O quê | Onde |
| --- | --- |
| Pacote de linha de comando ou de sistema dos repositórios do Ubuntu | `apt_packages` |
| Pacote do repositório apt do próprio fornecedor | `apt_external_repos` + `apt_packages` |
| Aplicativo gráfico | `snap_packages` ou `flatpak_packages` |
| CLI npm global | `npm_global_packages` |
| Runtime de linguagem | `dev_tools_languages` |

**Confirme que o pacote existe e entrega o que você espera antes de
adicioná-lo.** Isso não é formalidade — o canal padrão de um snap não
necessariamente acompanha a versão que você imagina:

```bash
apt-cache policy <nome>        # está nos repositórios do Ubuntu? em que versão?
snap info <nome>               # leia o mapa de canais, não a descrição
npm view <nome> version
```

Para um repositório apt de terceiro, confirme que o pacote é publicado para a
sua arquitetura antes de escrever a linha do repositório:

```bash
curl -sf https://exemplo.com/apt/dists/stable/main/binary-amd64/Packages.gz \
  | gunzip -c | grep -A3 '^Package: <nome>$'
```

Depois adicione e teste conforme abaixo.

## Testando uma mudança

Nada é aceito sem uma verificação de idempotência. Quatro níveis, do mais
barato ao mais caro.

O [`.github/workflows/ci.yml`](../../.github/workflows/ci.yml) roda todos eles
a cada push e pull request, em três jobs:

| Job | Cobre |
| --- | --- |
| `Lint` | yamllint + ansible-lint |
| `Converge and idempotence (container)` | `molecule test` — tudo, exceto as tags `docker`, `apps` e `security` |
| `Full playbook on a real VM` | o playbook inteiro duas vezes em um runner, e depois as asserções de verificação |

Você não precisa de tudo isso localmente. O nível 1 e uma execução repetida na
sua máquina pegam a maior parte; deixe o container e a VM para o CI.

**1. Sintaxe e lint**

```bash
sudo apt install ansible-lint yamllint    # ou: pipx install ansible-lint yamllint
ansible-playbook -i inventory.ini playbook.yml --syntax-check
yamllint .
ansible-lint
```

A configuração de lint fica em [`.yamllint`](../../.yamllint) e
[`.ansible-lint`](../../.ansible-lint). Algumas regras são ignoradas de
propósito: `role-name` (os diretórios dos roles usam hífen por causa do layout
do projeto), `var-naming[no-role-prefix]` (`target_user`, `deb_arch` e
companhia são compartilhadas entre roles de propósito) e
`command-instead-of-module` / `risky-shell-pipe`.

**2. Execução simulada**

```bash
ansible-playbook -i inventory.ini playbook.yml -K --check --diff
```

O modo `--check` acusa falha nas tarefas que inspecionam uma ferramenta que só
seria instalada por uma tarefa anterior. Isso é esperado em uma máquina ainda
não configurada.

**3. Rode duas vezes — este é o teste de verdade**

```bash
ansible-playbook -i inventory.ini playbook.yml -K
ansible-playbook -i inventory.ini playbook.yml -K
```

A segunda execução **precisa** reportar `changed=0`. Uma execução que funciona
uma vez não é idempotente; uma tarefa que reporta `changed` toda vez é um bug,
mesmo que o estado final esteja correto.

**4. Molecule — um container descartável, convergido e conferido sozinho**

```bash
pip install molecule "molecule-plugins[docker]" docker
molecule test
```

Isso exige um Docker funcionando. Se você acabou de rodar o playbook pela
primeira vez, saia e entre na sessão antes — o grupo `docker` não vale na
sessão que adicionou você a ele.

A sequência padrão cria um container `ubuntu:26.04`, converge o playbook, roda
uma **segunda vez e falha se algo reportar `changed`**, e então executa as
asserções de `molecule/default/verify.yml`. Subcomandos úteis durante o
desenvolvimento:

```bash
molecule converge     # cria e roda, deixando o container de pé
molecule idempotence  # roda de novo no container existente
molecule verify       # só as asserções
molecule login        # abre um shell dentro dele
molecule destroy
```

O cenário usa `--skip-tags docker,apps,security`, porque Docker Engine, snapd e
unattended-upgrades precisam de systemd e nenhum funciona em um container
simples. Esses roles são cobertos pelo job `full-run` do CI, que roda o
playbook inteiro em uma VM de verdade.

**Verificando o estado final em qualquer lugar.** O
`molecule/default/verify.yml` é um playbook comum, então roda também contra a
sua máquina:

```bash
ansible-playbook -i inventory.ini molecule/default/verify.yml
```

Ele confere que as ferramentas existem, que o node reporta uma versão real, que
o go reporta uma versão completa, que o rust está no canal configurado, que os
blocos gerenciados do `~/.profile` aparecem exatamente uma vez cada e que todo
item de `npm_global_packages` está instalado.

**Testando tarefas de espaço de usuário sem container.** Tudo que instala
dentro do `$HOME` (uv, nvm, rustup, CLIs npm globais) pode rodar contra um
diretório home descartável, porque `target_home` origina todos os caminhos
derivados:

```yaml
- hosts: local
  become: false
  vars:
    target_home: /tmp/fakehome
  tasks:
    - ansible.builtin.include_tasks: roles/dev-tools/tasks/node.yml
```

Rode duas vezes, confirme `changed=0` e depois `rm -rf /tmp/fakehome`. Isso
pega erros de instalação sem exigir sudo nem sujar o seu `$HOME` real.

## Convenções

**Sem comentários nos arquivos de configuração.** Os nomes das tarefas dizem o
que elas fazem; o raciocínio por trás de uma escolha não óbvia vai na mensagem
de commit, onde `git blame` e `git log` o encontram. Se uma linha parece
arbitrária, a justificativa está a um `git log -p` de distância — consulte
antes de "simplificar".

**`become: true` em tarefas individuais, nunca na play ou no role.** Tudo que
escreve dentro do `$HOME` roda sem privilégio.

**Prefira módulos a `command`/`shell`.** Quando a chamada de shell for
inevitável, torne-a idempotente com `creates:`, um `when:` baseado em uma
verificação anterior ou um `changed_when:` correto.

**Todo role recebe uma tag**, para poder rodar sozinho via `--tags`.

**As variáveis moram em `group_vars/all.yml`**, espelhadas no
`defaults/main.yml` do role para que ele continue funcionando fora deste
repositório.

## Adicionando um role

```bash
mkdir -p roles/meu-role/{tasks,defaults}
```

Crie `tasks/main.yml` e `defaults/main.yml` e registre em `playbook.yml`:

```yaml
- role: meu-role
  tags: [meu-role]
```

Se o role precisar de um repositório apt de terceiro, não reescreva a lógica de
keyring e repositório. Chame a implementação compartilhada:

```yaml
- name: Register my vendor repository
  ansible.builtin.include_role:
    name: apt-packages
    tasks_from: external_repos.yml
  vars:
    external_repos: "{{ my_role_external_repos }}"
```

## Armadilhas

Todas estas já causaram bugs reais neste repositório.

**Teste de substring no `when:`.** Um `x not in output` casa com nomes
parciais. `foo` casa com um `foobar` instalado; `v2` casa com `v22.18.0`. A
tarefa é pulada e a play reporta sucesso sem ter instalado nada. Ancore a
comparação:

```yaml
when: not (output is search('(?m)/' ~ (name | regex_escape) ~ '$'))
```

**Saída de comando mais ampla do que o que você testa.** O `nvm ls` imprime a
tabela de apelidos LTS junto das versões instaladas, então uma comparação
ingênua enxerga versões que não estão instaladas. Restrinja o comando
(`--no-alias`) em vez de filtrar depois.

**Comandos que saem com código diferente de zero no caminho da máquina nova.**
O `nvm ls` sai com 3 quando nada está instalado; o `npm ls` sai diferente de
zero com qualquer dependência não satisfeita. Os dois ainda imprimem saída
útil. Use `failed_when: false` quando um código diferente de zero for
informação, não falha.

**Pacotes de terceiros que gerenciam a própria fonte apt.** Alguns scripts
`postinst` de `.deb` adicionam, migram ou apagam o próprio arquivo em
`/etc/apt/sources.list.d/`, brigando com o que o Ansible escreve e gerando um
`changed` permanente. Use `self_managed_marker` para que a nossa entrada apenas
viabilize a primeira instalação.

**Suposições de arquitetura e de versão.** O `deb_arch` mapeia apenas x86_64 e
aarch64. Strings de versão precisam ser completas onde o upstream não
redireciona uma versão parcial.

**Testes de sistema de arquivos que rodam na máquina errada.** Testes Jinja
como `'/algum/caminho' is exists` são avaliados no **controller**, não no alvo.
Em uma execução com conexão local os dois são o mesmo host, então o erro fica
invisível até o playbook rodar contra um container ou uma máquina remota.
Inspecione o alvo com `ansible.builtin.stat` e um `set_fact`. Foi exatamente
assim que o `ansible_become_exe` esteve errado até o molecule expor o problema.

## Fluxo de git

`main` é a única branch de longa duração e deve estar sempre funcionando.
Trabalhe em uma branch, com uma mudança lógica por commit.

```bash
git switch -c fix-node-version-match
# edite e teste conforme acima
git commit
git push -u origin fix-node-version-match
```

Abra um pull request contra `main`. Prefira rebase a merge, para manter o
histórico linear:

```bash
git fetch origin
git rebase origin/main
```

### Mensagens de commit

Este repositório segue o
[Conventional Commits 1.0.0](https://www.conventionalcommits.org/pt-br/v1.0.0/).
O hook `commit-msg` valida o assunto; o corpo é responsabilidade sua.

```
<tipo>[escopo opcional][!]: <descrição>

[corpo opcional]

[rodapé opcional]
```

| Tipo | Use para |
| --- | --- |
| `feat` | Uma capacidade nova — um role, uma ferramenta instalada, uma opção |
| `fix` | Um defeito: comportamento errado, execução quebrada, tarefa não idempotente |
| `docs` | Somente documentação |
| `test` | Cenários do molecule, asserções de verificação |
| `ci` | Mudanças de workflow e pipeline |
| `refactor` | Reestruturação sem mudança de comportamento |
| `style` | Formatação, tamanho de linha, espaços |
| `chore` | Ferramental, configuração de lint, manutenção |
| `build` | Dependências, `requirements.yml` |
| `perf` | Deixar algo mensuravelmente mais rápido |
| `revert` | Reverter um commit anterior |

O escopo é opcional e nomeia a área afetada: `fix(molecule):`,
`feat(docker):`, `docs(pt-br):`. Um `!` antes dos dois-pontos marca uma mudança
incompatível — aqui, qualquer coisa que altere o estado de uma máquina já
configurada em uma nova execução ou remova uma variável que as pessoas usam.

Como a configuração não tem comentários, o corpo é o único lugar onde o
raciocínio fica registrado. Escreva pensando em quem for depurar isto daqui a
um ano:

- Explique o **porquê** e o que quebra sem a mudança. Nomeie a falha concreta,
  não a edição — o diff já mostra a edição.
- Diga como foi verificado: `changed=0` na segunda execução, versões
  conferidas, o que se aplicar.

```
fix: anchor the npm global installed-check

A bare substring test matched "foo" against an installed "foobar", so the
package was never installed while the play reported success. Anchor the
match to /<name> at end of line.

Verified: two consecutive runs report changed=0 and the package installs on
a machine where a longer name is already present.
```

As mensagens de commit deste repositório são escritas em inglês, para ficarem
consistentes com o histórico existente.

### Reescrevendo o histórico

Tudo bem na sua própria branch antes da revisão. Evite na `main` — ela é
publicada, então reescrever exige um force-push e quebra todos os clones.

Se for necessário, use `--force-with-lease` em vez de `--force`, para que o
push seja abortado caso algo tenha chegado ao remoto sem você ter feito fetch.
Note que `git filter-branch -- --all` também reescreve as refs de rastreamento
do remoto, o que invalida a checagem do lease; rode `git fetch` depois para
corrigir.
