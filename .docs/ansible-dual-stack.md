# Ansible Dual Stack — Documentación del Entorno

## Índice

1. [Arquitectura](#1-arquitectura)
2. [Stack Moderno — `~/ansible`](#2-stack-moderno--ansible)
3. [Stack Legacy — `~/ansible-legacy`](#3-stack-legacy--ansible-legacy)
4. [Configuración Compartida](#4-configuración-compartida)
5. [Uso Diario](#5-uso-diario)
6. [VM Remota (172.30.10.118)](#6-vm-remota-1723010118)
7. [Skills / AGENTS.md](#7-skills--agentsmd)
8. [Referencia Rápida](#8-referencia-rápida)

---

## 1. Arquitectura

Se mantienen dos entornos Ansible completamente aislados para cubrir el
espectro de versiones de RHEL en el parque:

| Stack | Ruta | `ansible-core` | Target |
|---|---|---|---|
| 🟢 Moderno | `~/ansible/.venv/` | 2.19.11 | RHEL 8, RHEL 9 |
| 🟡 Legacy | `~/ansible-legacy/.venv/` | 2.16.19 | RHEL 7.9 |

**Principio**: cada stack vive en su propio directorio-raíz con su propio
`.venv/`, `ansible.cfg`, `collections/` y vault password. No se mezclan.
Se invocan por ruta absoluta al binario, sin activar venvs.

**Stack moderno** provisiona VMs (clonado con `vmware_guest`) **y** hace
postconfig por SSH. **Stack legacy** solo hace postconfig por SSH contra
RHEL 7.9 (no provisiona — el clonado siempre lo hace el moderno).

---

## 2. Stack Moderno — `~/ansible`

### 2.1 Ruta base

```
/home/ansible/ansible/          (VM remota 172.30.10.118)
~/Automation/                   (replicado local en máquina Hermes)
```

### 2.2 Versiones instaladas

| Componente | Versión |
|---|---|
| Python | 3.12.13 |
| `ansible-core` | 2.19.11 |
| `community.vmware` | 6.2.0 |
| `vmware.vmware` | 2.9.0 |
| `community.general` | 13.1.0 |
| `ansible.posix` | 2.2.0 |
| `ansible.utils` | 6.0.3 |
| `vcf-sdk` | 9.1.0.0 |
| `jmespath` | (latest) |
| `netaddr` | (latest) |
| `govc` | 0.55.0 |

### 2.3 Estructura

```
~/ansible/
├── .venv/                         # Python 3.12 + ansible-core 2.19
├── ansible.cfg                    # Configuración local del proyecto
├── collections/
│   ├── requirements.yml           # Versiones pinneadas
│   └── ansible_collections/       # Instalado por ansible-galaxy (gitignored)
├── group_vars/
│   ├── all/                       # Reservado
│   ├── vars_rhel8.yml             # ansible_python_interpreter: /usr/bin/python3.12
│   ├── vars_rhel9.yml             # ansible_python_interpreter: /usr/bin/python3
│   └── dev.yml                    # vCenter vault bridge + vm_catalog
├── inventory/                     # Inventarios (dinámicos o estáticos)
├── roles/
│   ├── preflight/                 # Validación pre-deploy
│   └── vm_postconfig/            # guest_upload, guest_download
├── vars/
│   └── vcenter_vars.yml          # Variables de conexión vCenter
├── vault/
│   └── credentials.yml           # Cifrado con ansible-vault
├── .vault_pass                   # Contraseña del vault moderno
├── playbook_bootstrap_python.yml # Bootstrap con raw module para RHEL 8
├── plabook_ad_join.yml           # (nota: typo intencional en nombre original)
├── playbook_mover_archivos.yml
├── playbook_sellado.yml
├── preflight.yml
├── test_pivote.yml
└── preflight-role.tar
```

### 2.4 ansible.cfg

```ini
[defaults]
inventory          = inventory/
collections_path   = collections
vault_password_file = ~/.vault_pass
interpreter_python = auto
forks              = 10
host_key_checking  = False
[ssh_connection]
pipelining = True
```

---

## 3. Stack Legacy — `~/ansible-legacy`

### 3.1 Ruta base

```
/home/ansible/ansible-legacy/      (VM remota 172.30.10.118)
```

### 3.2 Versiones instaladas

| Componente | Versión |
|---|---|
| Python | 3.12.13 (mismo que moderno) |
| `ansible-core` | 2.16.19 |
| `community.general` | 8.6.11 |
| `ansible.posix` | 1.6.2 |
| `ansible.utils` | 5.1.2 |

Nota: no lleva `community.vmware`, `vmware.vmware` ni `vcf-sdk` porque el
legacy **no provisiona VMs**. Solo corre tareas por SSH.

### 3.3 Estructura

```
~/ansible-legacy/
├── .venv/                         # Python 3.12 + ansible-core 2.16
├── ansible.cfg                    # Configuración local del proyecto
├── collections/
│   ├── requirements.yml           # Versiones pinneadas
│   └── ansible_collections/       # Instalado por ansible-galaxy (gitignored)
├── group_vars/
│   └── rhel7.yml                  # ansible_python_interpreter: /usr/bin/python3
├── inventory/
│   └── hosts.yml                  # Placeholder
├── vault/                         # (vacío - para futuros secrets)
├── vars/                          # (vacío)
├── roles/                         # (vacío)
└── .vault_pass_legacy             # Contraseña del vault legacy
```

### 3.4 ansible.cfg

```ini
[defaults]
inventory          = inventory/
collections_path   = collections
vault_password_file = ~/.vault_pass_legacy
interpreter_python = auto
forks              = 10
host_key_checking  = False
[ssh_connection]
pipelining = True
```

---

## 4. Configuración Compartida

### 4.1 Vault Pattern

- Las **credenciales reales** se guardan cifradas en `vault/credentials.yml`
  con prefijo `vault_*` (ej: `vault_vcenter_hostname`).
- Los **nombres limpios** se exponen en `group_vars/dev.yml` mediante
  variables que referencian `{{ vault_* }}`.
- Cada stack tiene su propio vault password:
  - Moderno: `~/.vault_pass` → `Missfortune.161988`
  - Legacy: `~/.vault_pass_legacy` → generado con `openssl rand`

### 4.2 Python Interpreter

| Grupo | Forzado a | Razón |
|---|---|---|
| `rhel7` | `/usr/bin/python3` | Core 2.16 soporta Python 3.6 nativo |
| `rhel8` | `/usr/bin/python3.12` | RHEL 8 trae 3.6 por defecto, abajo del floor de 2.19 |
| `rhel9` | `/usr/bin/python3` | RHEL 9 trae 3.9, explícito por determinismo |

### 4.3 Bootstrap (solo RHEL 8)

`playbook_bootstrap_python.yml` instala `python3.12` en targets RHEL 8
que no lo tengan. Claves:
- `gather_facts: false` — no requiere python para recolectar facts
- Usa `ansible.builtin.raw` — único módulo que funciona sin python en target
- Es idempotente: si `python3.12` ya existe, no hace nada

### 4.4 Host Key Checking

Deshabilitado (`False`) porque las VMs clonadas desde vCenter rotan IPs
y nadie quiere fingerprint mismatches.

### 4.5 .gitignore Relevante

```
.venv/
collections/ansible_collections/   # Ignora artefactos instalados
vault/credentials.yml              # Tracking vía ansible-vault
```

Los `collections/requirements.yml` **sí** se trackean en git.

---

## 5. Uso Diario

### 5.1 Stack Moderno

```bash
# Desde cualquier lado
cd ~/ansible && .venv/bin/ansible-playbook site.yml

# Bootstrap de python en RHEL 8
cd ~/ansible && .venv/bin/ansible-playbook playbook_bootstrap_python.yml -l rhel8

# Instalar collections desde requirements
cd ~/ansible && .venv/bin/ansible-galaxy collection install \
  -r collections/requirements.yml -p collections
```

### 5.2 Stack Legacy

```bash
cd ~/ansible-legacy && .venv/bin/ansible-playbook -l rhel7 site.yml
```

### 5.3 govc (VMware CLI)

```bash
govc version   # → govc 0.55.0
# Instalado en /usr/local/bin/govc
```

---

## 6. VM Remota (172.30.10.118)

### 6.1 Acceso

```
Usuario:   ansible
Password:  Missfortune161988
```

### 6.2 Script de scaffold legacy

Ubicación: `/home/ansible/Script/Legacy_rhel7.sh`

Crea el entorno `~/ansible-legacy/` completo:
1. Crea estructura de directorios
2. Crea `.venv` con Python 3.12
3. Instala `ansible-core>=2.16,<2.17`
4. Instala collections pinneadas
5. Genera `ansible.cfg`, inventario placeholder y `group_vars/rhel7.yml`
6. Genera `.vault_pass_legacy` aleatorio

---

## 7. Skills / AGENTS.md

### 7.1 Hermes Skill

- Nombre: `ansible-dual-stack`
- Ruta: `~/.hermes/skills/devops/ansible-dual-stack/SKILL.md`
- Contiene layout completo, versiones, configs y patrones
- Se carga automáticamente al mencionar dual stack / legacy ansible

### 7.2 AGENTS.md (OpenCode / Claude Code / Cursor)

- Ruta: `~/Automation/AGENTS.md` (local) y `~/ansible/AGENTS.md` (VM)
- Contiene el mismo contenido del skill pero en formato más compacto
- Sirve como context file para agentes que operen el proyecto

---

## 8. Referencia Rápida

### 8.1 Comandos Frecuentes

| Acción | Comando |
|---|---|
| Ver versión moderno | `cd ~/ansible && .venv/bin/ansible --version` |
| Ver versión legacy | `cd ~/ansible-legacy && .venv/bin/ansible --version` |
| Listar collections moderno | `cd ~/ansible && .venv/bin/ansible-galaxy collection list` |
| Listar collections legacy | `cd ~/ansible-legacy && .venv/bin/ansible-galaxy collection list` |
| Instalar collections moderno | `.venv/bin/ansible-galaxy install -r collections/requirements.yml -p collections` |
| Ver collections con grep | `.venv/bin/ansible-galaxy collection list \| grep -Ei 'community.vmware\|vmware.vmware'` |
| Ejecutar playbook moderno | `cd ~/ansible && .venv/bin/ansible-playbook <playbook>.yml` |
| Ejecutar playbook legacy | `cd ~/ansible-legacy && .venv/bin/ansible-playbook -l rhel7 <playbook>.yml` |

### 8.2 Conexión SSH a VM

```bash
sshpass -p 'Missfortune161988' ssh -o StrictHostKeyChecking=no ansible@172.30.10.118
```

### 8.3 Rutas Clave en VM

| Recurso | Ruta |
|---|---|
| Stack moderno | `/home/ansible/ansible/` |
| Stack legacy | `/home/ansible/ansible-legacy/` |
| Script legacy | `/home/ansible/Script/Legacy_rhel7.sh` |
| Vault pass moderno | `/home/ansible/ansible/.vault_pass` |
| Vault pass legacy | `/home/ansible/ansible-legacy/.vault_pass_legacy` |
