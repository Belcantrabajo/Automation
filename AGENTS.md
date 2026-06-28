# Ansible Dual Stack — Project Context

## Layout

```
~/ansible/               # Stack moderno (RHEL 8, 9)
  .venv/                  # Python 3.12, ansible-core ~2.19
  ansible.cfg             # collections_path = collections
  collections/            # community.vmware 6.2.0, vmware.vmware 2.9.0,
                          # community.general 13.1.0, ansible.posix 2.2.0,
                          # ansible.utils 6.0.3
  group_vars/
    all/                  # (reservado)
    vars_rhel8.yml        # ansible_python_interpreter: /usr/bin/python3.12
    vars_rhel9.yml        # ansible_python_interpreter: /usr/bin/python3
    dev.yml               # vCenter vault bridge + vm_catalog
  inventory/
  roles/
    preflight/            # Pre-deploy validation (connectivity, placement, template, catalog)
    vm_postconfig/        # guest_upload.yml, guest_download.yml
  vars/vcenter_vars.yml
  vault/credentials.yml   # ansible-vault cifrado
  .vault_pass             # Missfortune.161988
  playbook_bootstrap_python.yml
  plabook_ad_join.yml
  playbook_mover_archivos.yml
  playbook_sellado.yml
  preflight.yml
  test_pivote.yml

~/ansible-legacy/         # Stack legacy (RHEL 7.9)
  .venv/                  # Python 3.9, ansible-core ~2.16
  ansible.cfg
  collections/            # Versiones compatibles con 2.16
```

## Usage

```bash
# Moderno (sin activar venv)
~/ansible/.venv/bin/ansible-playbook site.yml

# Legacy (sin activar venv)
~/ansible-legacy/.venv/bin/ansible-playbook -l rhel7 site.yml
```

## Key Config Notes

- RHEL 8 → `/usr/bin/python3.12` forzado (3.6 default está debajo del floor de ansible 2.19)
- RHEL 9 → `/usr/bin/python3` forzado explícito (determinismo)
- Vault: vars prefijo `vault_*` en credentials.yml → bridge a nombres limpios en dev.yml
- Bootstrap: `playbook_bootstrap_python.yml` usa `raw` module + `gather_facts: false`
- Host key checking deshabilitado (IPs efímeras de vCenter)
- Preflight role corre validaciones antes del deploy real
