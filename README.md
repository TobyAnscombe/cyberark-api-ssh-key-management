# cyberark-api-ssh-key-management

An Ansible role (`cyberark_ssh_key_rotation`) that discovers CyberArk Privilege Cloud accounts with a `cark_` name prefix, ensures the corresponding OS users exist on target hosts, and rotates their ED25519 SSH key pairs with a fail-safe two-key window.

Part of the [CyberArk Privilege Cloud Ansible suite](https://github.com/TobyAnscombe).

---

## How it works

1. **Authenticate** — the `cyberark_auth` role runs first, producing `cyberark_token`
2. **Discover** — queries the Privilege Cloud API for all accounts whose name starts with `cark_` and whose `secretType` is `key`
3. **Scope** — per inventory host, filters accounts by `remoteMachinesAccess.remoteMachines` (empty = all hosts)
4. **Ensure users** — builds `manage_user_accounts` dynamically from the discovered accounts and calls the `ubuntu_manage_user` role to create OS users, add them to the required supplementary groups (e.g. an sshd `AllowGroups` group), grant passwordless sudo, and add them to sshd `AllowUsers`
5. **Rotate keys** — for each account on each applicable host:
   - Captures any existing managed keys from `authorized_keys`
   - Generates a new ED25519 key pair (N+1) in a restricted temp directory
   - Converts to PPK v2 and asserts the format
   - **Adds N+1 to `authorized_keys`** — N and N+1 are both valid before CyberArk is updated
   - **Updates CyberArk** with the N+1 private key
   - **Removes all pre-rotation keys** from `authorized_keys` — exactly one managed key remains
   - Always deletes the temp directory, including on failure

### Fail-safe key model

During rotation, N+1 is added to `authorized_keys` **before** CyberArk is updated. If the CyberArk update fails, the current key N is still valid and the play fails loudly. Pre-rotation keys are only removed after CyberArk successfully holds N+1 — so connectivity is never interrupted regardless of where a failure occurs.

---

## Requirements

**Control node:**
- Ansible 2.14+
- `puttygen` installed (`apt install putty-tools`)
- SSH access to target hosts

**Target nodes:**
- Ubuntu 18.04, 20.04, 22.04, or 24.04
- Python 3 at `/usr/bin/python3`
- `become: true` — the role modifies system accounts, sshd, and sudoers

**Collections:**
```bash
ansible-galaxy collection install -r requirements.yml
```

**Role dependencies:**
```bash
ansible-galaxy role install -r requirements.yml
```

---

## Quick start

**1. Create a vault file:**
```bash
ansible-vault create ~/.ansible/vault/cyberark.yml
```
```yaml
cyberark_identity_tenant: "your-tenant.id.cyberark.cloud"
cyberark_client_id: "service-account@cyberark"
cyberark_client_secret: "..."
```

**2. Create an inventory:**
```ini
[linux_servers]
server1.example.com
server2.example.com
```

**3. Dry run (no changes):**
```bash
ansible-playbook -i inventory.ini examples/site.yml
```

**4. Apply:**
```bash
ansible-playbook -i inventory.ini examples/site.yml -e cyberark_dry_run=false
```

---

## Role variables

All variables are defined in `defaults/main.yml`.

| Variable | Default | Description |
|---|---|---|
| `cyberark_subdomain` | `""` | **Required.** Privilege Cloud subdomain (e.g. `myorg`) |
| `cyberark_safe` | `""` | Optional: restrict discovery to one safe. Empty = all safes. |
| `cyberark_ssh_key_name_prefix` | `"cark_"` | Account name prefix filter |
| `cyberark_ssh_key_secret_type` | `"key"` | Secret type filter — identifies SSH key accounts |
| `cyberark_ssh_key_platform_filter` | `[]` | Optional list of `platformId` values to restrict further |
| `cyberark_ssh_key_page_size` | `1000` | Accounts per API page |
| `cyberark_ssh_key_algorithm` | `"ed25519"` | SSH key algorithm |
| `cyberark_ssh_key_ensure_users` | `true` | Create OS users, grant sudo, add to sshd AllowUsers. Set `false` to skip user setup (rotation only). |
| `cyberark_ssh_key_sudo` | `true` | Grant passwordless sudo to discovered accounts |
| `cyberark_ssh_key_groups` | `[]` | Supplementary groups to assign to each discovered account. Set this in your playbook vars — group names are typically internal. If your sshd config uses `AllowGroups`, accounts must be members of a listed group or SSH login will be blocked. |
| `cyberark_ssh_key_home_base` | `"/home"` | Base home directory for managed accounts. The authorized_keys path is derived as `<home_base>/<userName>/.ssh/authorized_keys`. Override if accounts are homed outside `/home`. |
| `cyberark_dry_run` | `true` | Default safe — shows what would change without applying |
| `cyberark_validate_certs` | `true` | TLS certificate validation |

---

## Remote machine scoping

Each CyberArk account has a `remoteMachinesAccess.remoteMachines` field — a semicolon-separated list of hostnames or IPs. The role uses this to scope rotation:

- **Empty** → rotate on **all** inventory hosts
- **Set** → rotate only on inventory hosts whose `inventory_hostname` appears in the list

The same filter applies to OS user setup — a user is only created on hosts where their account's keys would be rotated.

---

## Consuming playbook

See `examples/site.yml`. The minimal form:

```yaml
- name: Rotate CyberArk SSH keys
  hosts: all
  gather_facts: false
  vars_files:
    - ~/.ansible/vault/cyberark.yml
  vars:
    cyberark_subdomain: "myorg"
    cyberark_dry_run: false
  roles:
    - cyberark_auth
    - cyberark_ssh_key_rotation
```

No static account list is needed — accounts are discovered from CyberArk at runtime.

---

## Dependencies

| Role | Source | Purpose |
|---|---|---|
| `cyberark_auth` | `cyberark-api-management` | OAuth2 authentication → `cyberark_token` |
| `ubuntu_manage_user` | `ubuntu-user-creation` | OS user creation, supplementary groups, sudo, sshd AllowUsers/AllowGroups |

`ubuntu_manage_user` is called conditionally when `cyberark_ssh_key_ensure_users: true` (the default). It requires v1.1.0+ for `manage_user_ssh_key_required: false` support — keys are managed by this role, not declared statically in the user list.

---

## Key comment convention

Every managed public key is written to `authorized_keys` with the comment `cark_managed_<account-name>`. This identifies which lines belong to CyberArk-managed accounts so the role can capture and remove old key generations accurately.

---

## Security notes

- Private key material is never written to `/tmp` — a `0700` temp directory is created via `ansible.builtin.tempfile` and deleted in an `always:` block
- Key material on the control node is held only for the duration of one account's rotation loop iteration
- All API calls use `no_log: true`
- `cyberark_dry_run: true` is the default — changes require an explicit opt-in
