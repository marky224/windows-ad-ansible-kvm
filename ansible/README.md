# Ansible — AD Lab IaC

Provisions a 4-VM Active Directory lab on KVM/libvirt: 1 Server 2025 DC, 2 Win 11 Enterprise clients, 1 Ubuntu 24.04 server, all on `corp.markandrewmarquez.com`.

## Layout

```
ansible/
├── ansible.cfg            inventory + vault password + ssh args
├── requirements.yml       collections (microsoft.ad, ansible.windows, …)
├── inventory/lab.yml      static inventory: [dc], [clients], [linux_clients]
├── group_vars/
│   ├── all/
│   │   ├── main.yml       shared constants (forest, subnet, paths, ISOs)
│   │   └── vault.yml      ansible-vault encrypted secrets
│   ├── windows.yml        WinRM connection (parent of dc + clients)
│   ├── dc.yml             DC-specific vars
│   ├── clients.yml        Win 11 client-specific vars
│   └── linux_clients.yml  Ubuntu SSH config
├── playbooks/             NN-verb-noun.yml (see playbooks/README.md)
├── roles/                 one role per README, see each roles/<role>/README.md
└── files/                 templates, GPO baselines, etc.
```

## First-time setup

```bash
ansible-galaxy collection install -r requirements.yml
echo 'your-vault-password' > ~/.ansible-vault-pass-corp-lab && chmod 600 ~/.ansible-vault-pass-corp-lab
ansible-vault edit group_vars/all/vault.yml   # replace the PLACEHOLDER values
```

## Run the full lab

```bash
cd ansible/
ansible-playbook playbooks/00-libvirt-network.yml
ansible-playbook playbooks/01-provision-dc.yml
ansible-playbook playbooks/02-configure-dc.yml
ansible-playbook playbooks/03-provision-clients.yml
ansible-playbook playbooks/04-join-domain.yml
ansible-playbook playbooks/05-provision-linux.yml
ansible-playbook playbooks/06-join-linux.yml
ansible-playbook playbooks/99-smoke-test.yml
```

## Conventions

- **Playbooks** are numbered `NN-verb-noun.yml` to imply execution order; `99-` is verification.
- **Roles** are named `<provider>_<resource>` (infra) or `<system>_<feature>` (config) or `<verb>_<target>` (action).
- **Secrets** live only in `group_vars/all/vault.yml`. Reference as `vault_*` keys.
- **Idempotency**: every playbook is safely re-runnable.
- **Tags** on major sections for partial runs: `--tags provision`, `--tags configure`, …
