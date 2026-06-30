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
  collections/requirements.yml    # community.vmware 6.2.0, vmware.vmware 2.9.0, etc

~/ansible-legacy/         # Stack legacy (RHEL 7.9)
  .venv/                  # Python 3.12, ansible-core 2.16.19
  ansible.cfg             # collections_path = collections
  collections/            # community.general 8.6.11, ansible.posix 1.6.2, ansible.utils 5.1.2
  group_vars/
    rhel7.yml             # ansible_python_interpreter: /usr/bin/python3
  inventory/hosts.yml
  collections/legacy/requirements.yml  # community.general 8.6.11, ansible.posix 1.6.2, etc
  .vault_pass_legacy      # openssl rand -base64 48
```

## Local Aliases (~/.bashrc)

```bash
alias venv-core='source ~/Automation/.venv/bin/activate && echo "✅ Ansible Core 2.19.11 activado"'
alias venv-legacy='source ~/ansible-legacy/.venv/bin/activate && echo "✅ Ansible Legacy 2.16.19 activado"'
```

## Usage

### Local (Hermes host)

```bash
# Moderno — desde ~/Automation
cd ~/Automation && ~/Automation/.venv/bin/ansible-playbook playbooks/site.yml
# o con alias
venv-core
ansible-playbook -i inventory/ playbooks/site.yml

# Legacy — desde ~/ansible-legacy
cd ~/ansible-legacy && ~/ansible-legacy/.venv/bin/ansible-playbook -l rhel7 playbooks/site.yml
# o con alias
venv-legacy
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

### VM (172.30.10.118)

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
- Legacy (2.16): no necesita bootstrap — soporta Python 3.6/2.7 nativos de RHEL 7. Vault password separado en `.vault_pass_legacy`.