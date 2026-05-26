# Role: `_common`

Shared task includes and helpers used by other roles. **Not a standalone role** — it has no `tasks/main.yml` entry point. Other roles include specific task files from `_common/tasks/<file>.yml` (e.g., `wait_for_winrm.yml`, `libvirt_domain_define.yml`).

Leading-underscore name marks it as private/internal so `ansible-galaxy role install` won't grab it.
