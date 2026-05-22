# Ansible Lab

Ansible infrastructure lab.

## What It Does

Current automation:

- Defines lab infrastructure hosts in static inventory files.
- Uses `ansible.cfg` defaults for inventory, remote user, privilege escalation, output format, and vault password file location.
- Deploys Apache HTTPD to the Oracle DB host.
- Copies a static `index.html` page to `/var/www/html/index.html`.
- Starts and enables `httpd`.
- Opens HTTP traffic in `firewalld`.
- Verifies HTTP reachability with `ansible.builtin.uri`.
- Prepares Oracle 19c prerequisites on the `oracledb` group:
  - Installs prerequisite packages.
  - Creates Oracle groups and user.
  - Creates Oracle directories.
  - Applies kernel parameters.
  - Creates Oracle limits configuration.
  - Updates GRUB configuration for Transparent Huge Pages settings.
  - Opens Oracle listener port `1521/tcp`.

## Enterprise Assessment

Current state: **basic lab**, not enterprise-ready.

Why it is a basic lab:

- Playbooks are mostly direct task lists instead of reusable roles.
- Inventory is static and environment separation is limited.
- `vault_pass` exists in the repository path and `ansible.cfg` points to it directly.
- Host key checking is disabled.
- `become = true` is global in `ansible.cfg`.
- No CI pipeline is present.
- No `collections/requirements.yml` exists.
- Some playbooks reference inventory groups that are not consistently defined, such as `workstation` in `webserver.yml`.
- Some modules are not fully qualified collection names.
- No lint, syntax check, molecule, or idempotency testing is configured.
- No production promotion workflow exists.
- No change approval, rollback, or audit model is defined.

What is enterprise-like already:

- The repo uses inventory groups that resemble real platform service ownership.
- It separates variable files for web and Oracle configuration.
- It uses Ansible Vault configuration, though the current file placement should be changed.
- It uses handlers for GRUB regeneration.
- It automates OS state in a repeatable way.

## Repository Layout

```text
ansible_lab/
|-- README.md
|-- ansible.cfg
|-- hosts
|-- inventory.yml
|-- group_vars/
|   `-- all.yml
|-- files/
|   |-- index.html
|   |-- vars.yml
|   `-- oracle.yml
|-- site.yml
|-- webserver.yml
|-- oracle19c-prereq.yml
`-- vault_pass
```

| Path | Purpose |
| --- | --- |
| `ansible.cfg` | Local Ansible defaults for inventory, remote user, become, output, and vault password file. |
| `hosts` | INI inventory with service groups and IP addresses. |
| `inventory.yml` | YAML inventory organized by operating system and service group. |
| `group_vars/all.yml` | Shared Oracle user and group variables. |
| `files/vars.yml` | Apache package variable used by `site.yml`. |
| `files/oracle.yml` | Oracle prerequisite package, directory, and kernel parameter variables. |
| `files/index.html` | Static web content deployed by `site.yml`. |
| `site.yml` | Apache deployment and HTTP verification against `oracledb`. |
| `webserver.yml` | Alternate Apache deployment playbook with inline variables. |
| `oracle19c-prereq.yml` | Oracle 19c operating system prerequisite playbook. |
| `vault_pass` | Local vault password file reference. This should not be committed in enterprise use. |

## Ansible Configuration

Current `ansible.cfg` behavior:

```ini
[defaults]
inventory = hosts
remote_user = vagrant
become = true
host_key_checking = false
deprecation_warnings = false
command_warnings = false
retry_files_enabled = false
forks = 5
stdout_callback = yaml
vault_password_file = vault_pass
```

Enterprise concerns:

- `host_key_checking = false` is acceptable for disposable labs but should be enabled for managed environments.
- `become = true` globally increases blast radius. Prefer play-level or role-level privilege escalation.
- `vault_password_file = vault_pass` should point to an external protected path, not a repo file.
- `deprecation_warnings = false` hides upgrade signals.

## Inventory

Primary inventory:

```text
hosts
```

Main service groups:

- `awx`
- `gitlab`
- `jenkins`
- `artifactory`
- `database`
- `elk`
- `monitoring`
- `controller`
- `hub`
- `eda`
- `dns`
- `wiki`
- `oracledb`

YAML inventory:

```text
inventory.yml
```

This inventory groups hosts under operating system families such as:

- `rockylinux9`
- `redhat9`

Before using either inventory in production-like automation, confirm hostnames, IP addresses, SSH users, and group names are consistent.

## Playbooks

### `site.yml`

Purpose:

- Install Apache HTTPD on `oracledb`.
- Copy `files/index.html`.
- Start and enable HTTPD.
- Open HTTP service in `firewalld`.
- Verify the web server at `http://oracledb.example.com`.

Run:

```bash
ansible-playbook site.yml
```

Run with explicit inventory:

```bash
ansible-playbook -i hosts site.yml
```

### `webserver.yml`

Purpose:

- Install HTTPD, firewalld, and `python3-PyMySQL`.
- Start and enable firewalld and HTTPD.
- Write simple web content.
- Open HTTP in firewalld.
- Verify web reachability from the `workstation` group.

Important note:

`webserver.yml` references `hosts: workstation` in the verification play, but the current `hosts` inventory does not define a `workstation` group. Add the group or update the playbook before relying on it.

Run:

```bash
ansible-playbook -i hosts webserver.yml
```

### `oracle19c-prereq.yml`

Purpose:

- Prepare `oracledb` hosts for Oracle 19c installation.
- Install prerequisite packages from `files/oracle.yml`.
- Create Oracle user and groups.
- Create Oracle directories.
- Set kernel parameters.
- Configure Oracle user limits.
- Disable Transparent Huge Pages through GRUB configuration.
- Open Oracle listener port `1521/tcp`.

Run:

```bash
ansible-playbook -i hosts oracle19c-prereq.yml
```

Operational warning:

This playbook modifies system-level settings, GRUB configuration, kernel parameters, users, groups, directories, packages, and firewall rules. Run it only against disposable lab hosts or reviewed database build targets.

## Variables

### `group_vars/all.yml`

Shared Oracle identity values:

```yaml
oracle_user: oracle
oracle_groups:
  - oinstall
  - dba
```

### `files/vars.yml`

Apache package variable:

```yaml
apache_server: "httpd"
```

### `files/oracle.yml`

Oracle prerequisite data:

- `oracle_directories`
- `oracle_packages`
- `kernel_params`

## Running Safely

Start with connectivity:

```bash
ansible all -i hosts -m ping
```

Limit to a single host:

```bash
ansible-playbook -i hosts site.yml --limit oracledb.example.com
```

Run in check mode where supported:

```bash
ansible-playbook -i hosts oracle19c-prereq.yml --check
```

Show planned tasks:

```bash
ansible-playbook -i hosts oracle19c-prereq.yml --list-tasks
```

Show affected hosts:

```bash
ansible-playbook -i hosts oracle19c-prereq.yml --list-hosts
```

## Validation

Syntax checks:

```bash
ansible-playbook -i hosts site.yml --syntax-check
ansible-playbook -i hosts webserver.yml --syntax-check
ansible-playbook -i hosts oracle19c-prereq.yml --syntax-check
```

Inventory validation:

```bash
ansible-inventory -i hosts --list
ansible-inventory -i inventory.yml --list
```

HTTP validation:

```bash
curl -I http://oracledb.example.com
```

Oracle prerequisite validation examples:

```bash
ansible oracledb -i hosts -m command -a "id oracle"
ansible oracledb -i hosts -m command -a "sysctl fs.aio-max-nr"
ansible oracledb -i hosts -m command -a "firewall-cmd --list-ports"
```

## Enterprise Hardening Roadmap

To move this from basic lab to enterprise-ready:

1. Remove `vault_pass` from the repository and rotate the vault secret if it was ever shared.
2. Enable SSH host key checking and manage known hosts.
3. Split inventories into `dev`, `test`, and `prod` or use an inventory plugin.
4. Convert direct playbook tasks into roles:
   - `apache_web`
   - `oracle19c_prereq`
   - `firewall`
   - `common_packages`
5. Add `collections/requirements.yml`.
6. Use fully qualified collection names consistently.
7. Add `ansible-lint` and syntax checks in CI.
8. Add check-mode and idempotency validation.
9. Define rollback steps for system-level changes.
10. Replace global `become = true` with scoped privilege escalation.
11. Add tags for package, service, firewall, sysctl, user, and verification tasks.
12. Add documentation for supported operating systems and package repositories.
13. Remove hardcoded hostnames from validation tasks where possible.
14. Add ownership, approval, and promotion workflow.

## Recommended Enterprise Repository Shape

Recommended future structure:

```text
ansible_lab/
|-- README.md
|-- ansible.cfg
|-- collections/
|   `-- requirements.yml
|-- inventories/
|   |-- dev/
|   |-- test/
|   `-- prod/
|-- group_vars/
|-- host_vars/
|-- roles/
|   |-- apache_web/
|   |-- oracle19c_prereq/
|   `-- common_firewall/
|-- playbooks/
|   |-- site.yml
|   |-- webserver.yml
|   `-- oracle19c-prereq.yml
`-- .gitlab-ci.yml
```

## Governance

For lab use:

- Keep changes reviewed when shared with other operators.
- Use `--limit` for targeted runs.
- Avoid running Oracle system tuning against non-lab hosts.

For enterprise use:

- Require merge requests.
- Require CI validation.
- Require approval for database host changes.
- Store secrets in Ansible Vault, HashiCorp Vault, or an approved CI secret store.
- Capture playbook output for audit.
- Keep production inventory protected.

## Troubleshooting

### SSH Connection Fails

Check:

- Host IP address in inventory.
- `remote_user` from `ansible.cfg`.
- SSH key or password access.
- Firewall access to port `22`.

### HTTP Verification Fails

Check:

- `httpd` is installed and running.
- Firewalld allows HTTP.
- DNS or `/etc/hosts` resolves `oracledb.example.com`.
- The target host is in the `oracledb` group.

### Oracle Package Install Fails

Check:

- Target OS package repositories are enabled.
- Package names match the target distribution version.
- Host has internet or internal repo access.

### Sysctl or GRUB Update Fails

Check:

- Playbook is running with privilege escalation.
- Target OS uses `/boot/grub2/grub.cfg`.
- Kernel parameter values are valid for the host.

### `webserver.yml` Skips Verification

The playbook references the `workstation` group, which is not currently defined in `hosts`. Add a `workstation` group or update the verification play to run from `localhost`.

## Maintenance Notes

- Keep this repository labeled as lab automation until hardening is complete.
- Avoid adding real production secrets.
- Keep inventory and playbooks consistent.
- Prefer roles for reusable behavior.
- Document any destructive or reboot-impacting task before adding it.
