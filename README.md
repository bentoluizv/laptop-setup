# laptop-setup

[![CI](https://github.com/bentoluizv/laptop-setup/actions/workflows/ci.yml/badge.svg)](https://github.com/bentoluizv/laptop-setup/actions/workflows/ci.yml)

Ansible configuration for an Ubuntu laptop. Run it on a fresh machine, and
rerun it any time — it is idempotent.

Configuração Ansible para um notebook Ubuntu. Rode em uma máquina nova e
repita quando quiser — é idempotente.

## Documentation / Documentação

| | English | Português (pt-BR) |
| --- | --- | --- |
| Usage | [docs/en/README.md](docs/en/README.md) | [docs/pt-BR/README.md](docs/pt-BR/README.md) |
| Contributing | [docs/en/CONTRIBUTING.md](docs/en/CONTRIBUTING.md) | [docs/pt-BR/CONTRIBUTING.md](docs/pt-BR/CONTRIBUTING.md) |

## Quick start

```bash
sudo apt install -y ansible git
ansible-galaxy install -r requirements.yml
ansible-playbook -i inventory.ini playbook.yml --ask-become-pass
```
