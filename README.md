# mssql_project

Ansible automation for deploying, configuring, and patching Microsoft SQL Server 2022 on Windows Server using Red Hat Ansible Automation Platform (AAP).

---

## Overview

This project provides a set of playbooks and roles to fully automate the lifecycle of a SQL Server instance on AWS Windows hosts — from provisioning to patching — all driven through an AAP workflow.

### AAP Workflow

```
[Provision EC2 (Windows)] --> [Install & Configure SQL Server 2022] --> [Patch OS + SQL Server CU]
```

---

## Repository Structure

```
mssql_project/
├── sql_install.yml                        # Playbook: install SQL Server 2022
├── sql_patch.yml                          # Playbook: patch OS + SQL Server CU
├── inventories/
│   └── hosts.yml                          # Inventory (aws_win1, etc.)
├── group_vars/
│   └── sql_servers.yml                    # Group variables for SQL hosts
└── collections/
    └── ansible_collections/
        └── automatesql/
            └── mssql/                     # Bundled automatesql.mssql collection
                └── roles/
                    ├── sql_install/       # Role: SQL Server install + post-config
                    └── sql_ag_patch/      # Role: SQL AG patching (reference only)
```

---

## Prerequisites

### AAP / Controller
- Ansible Automation Platform 2.x
- Windows host reachable via WinRM/CredSSP on port 5986
- Credential configured with AWS permissions and appropriate Windows credentials
- The `automatesql.mssql` collection is bundled in `collections/` — no `requirements_collections` needed

### Windows Host
- Windows Server 2019 or 2022
- WinRM enabled and configured for CredSSP
- PowerShell remoting enabled
- `C:\temp` will be created automatically by the playbooks

### Required PowerShell Modules (auto-installed by role)
- `SqlServerDsc` 17.5.x
- `StorageDsc`
- `NetworkingDsc`

---

## Playbooks

### `sql_install.yml` — Install SQL Server 2022

Installs and configures a standalone SQL Server 2022 Developer Edition instance using the `automatesql.mssql.sql_install` role.

**Workflow:**
1. Create `C:\temp` directory
2. Download SQL Server 2022 Developer ISO from Microsoft
3. Wait for WinRM connection to be available
4. Gather facts
5. Run `automatesql.mssql.sql_install` role:
   - Set power plan to high performance
   - Mount ISO
   - Generate configuration INI from template
   - Install SQL Server via DSC (`SqlSetup`)
   - Configure error logs, job history, database mail
   - Apply post-install SQL scripts

**Usage:**
```bash
ansible-playbook sql_install.yml -i inventories/hosts.yml
```

---

### `sql_patch.yml` — Patch OS + SQL Server CU

Backs up all user databases, applies Windows OS patches and an optional SQL Server Cumulative Update, reboots, and verifies all databases are online.

**Workflow:**
1. Create `C:\temp` and backup directories
2. Capture pre-patch SQL version and OS info
3. Back up all user databases to `C:\SQLBACKUP\PrePatch`
4. Stop SQL Server Agent and SQL Server
5. Apply Windows Security + Critical updates
6. Download and apply SQL Server Cumulative Update (if `sql_cu_url` is set)
7. Reboot
8. Start SQL Server and SQL Agent
9. Wait for port 1433 to be available
10. Verify all databases are `ONLINE`
11. Display before/after version diff summary

**Usage:**
```bash
ansible-playbook sql_patch.yml -i inventories/hosts.yml
```

---

## Variables

### `sql_install.yml` — Extra Variables

These variables must be set as **Extra Variables** in your AAP Job Template. The collection's `defaults/main.yml` values are overridden here.

| Variable | Required | Example Value | Description |
|---|---|---|---|
| `sql_install_iso_source` | Yes | See below | List defining the ISO name, SQL version, and config template |
| `sql_install_features` | Yes | `SQLENGINE,REPLICATION,FULLTEXT` | SQL Server features to install. Remove `IS` (Integration Services) unless needed |
| `sql_install_edition` | Yes | `Developer` | SQL Server edition. Use `Developer` for the Dev ISO |
| `sql_install_product_key` | Yes | `""` | Leave blank for Developer edition — the ISO encodes the edition |
| `sql_install_sapwd` | Yes | `CHANGEME` | SA account password. Use a vault-encrypted value in production |
| `sql_install_sql_svc_account` | Yes | `NT AUTHORITY\SYSTEM` | SQL Server service account |
| `sql_install_agent_svc_account` | Yes | `NT AUTHORITY\SYSTEM` | SQL Agent service account |
| `sql_install_installdb_path` | Yes | `C:` | Root path for SQL system databases |
| `sql_install_userdb_path` | Yes | `C:\SQLDATA` | Path for user database data files |
| `sql_install_userdblog_path` | Yes | `C:\SQLLOGS` | Path for user database log files |
| `sql_install_backup_path` | Yes | `C:\SQLBACKUP` | Path for SQL Server backups |
| `sql_install_tempdb_path` | Yes | `C:\SQLTEMPDB` | Path for TempDB data files |
| `sql_install_tempdblog_path` | Yes | `C:\SQLTEMPDB` | Path for TempDB log files |
| `sql_install_enableupdates` | Yes | `"false"` | Whether setup.exe should pull updates during install. Set to `"false"` to speed up install |
| `sql_install_update_source` | Yes | `[]` | List of update sources. Set to empty list `[]` when updates disabled |
| `sql_install_update_source_path` | Yes | `MU` | Update source path. Use `MU` (Microsoft Update) or a local path |
| `sql_install_prep_disks` | Yes | `"false"` | Whether to prep/format disks before install. Set `"false"` on AWS instances |
| `sql_install_install_ssisdb` | Yes | `"false"` | Whether to configure the SSIS catalog. Set `"false"` if IS feature not installed |

**`sql_install_iso_source` format:**
```yaml
sql_install_iso_source:
  - name: "SQLServer2022-x64-ENU-Dev.iso"
    version: "SQL2022"
    config: "config2025.j2"
```

> **Note:** Only `config2025.j2` exists in the collection's `templates/` directory. Use this template for SQL Server 2022 installs despite the name.

---

### `sql_patch.yml` — Extra Variables

| Variable | Required | Default | Description |
|---|---|---|---|
| `sql_cu_url` | No | `""` | Full download URL for the SQL Server CU `.exe`. Leave blank to skip SQL CU and apply OS patches only |
| `sql_cu_filename` | No | `SQLServer2022-CU-latest.exe` | Filename to save the CU installer as in `C:\temp` |
| `backup_dest` | No | `C:\SQLBACKUP\PrePatch` | Destination path for pre-patch database backups |

**Example extra vars for patching with SQL CU:**
```yaml
sql_cu_url: "https://download.microsoft.com/download/a89001cb-9c99-48d3-9f14-ded054b35fe4/SQLServer2022-KB5080999-x64.exe"
sql_cu_filename: "SQLServer2022-KB5080999-x64.exe"
```

---

## Known Issues & Fixes

| Issue | Fix |
|---|---|
| `win_get_url` fails: `C:\temp does not exist` | Add `win_file` task to create `C:\temp` before any download tasks |
| `ansible_hostname is undefined` | Ensure `gather_facts: true` is set in the playbook, or add an explicit `ansible.builtin.setup` task before the role |
| `sql_install_ssisdb_password is undefined` | Set `sql_install_install_ssisdb: "false"` in extra vars if not installing Integration Services |
| `conflicting action statements: win_copy, remote_src` | Remove `remote_src: true` from the Copy SQL scripts task in `prerequisites.yml` |
| SQL install fails: Setup exit code `-2054422505` | Invalid or unsupported product key. Leave `sql_install_product_key: ""` for Developer edition |
| DSC error: `Cannot bind argument to parameter 'Path'` | Usually caused by `sql_install_update_source_path` resolving incorrectly. Set explicitly to `MU` in extra vars |
| `(0 rows affected)` causes false failure in patch verify | Use `failed_when: offline_dbs.stdout_lines \| select('match', '^[A-Za-z]') \| list \| length > 0` |
| Check mode fails: `dict object has no attribute stdout_lines` | Add `check_mode: false` to all `win_shell` tasks that feed a `register` variable used downstream |
| EC2 instance still initializing when SQL install starts | Add `wait_for_connection` as first task in `sql_install.yml` |

---

## Collection Notes

The `automatesql.mssql` collection is bundled locally in `collections/ansible_collections/automatesql/mssql/` and can be edited directly. Key files:

| File | Purpose |
|---|---|
| `roles/sql_install/tasks/prerequisites.yml` | Creates temp dirs, mounts ISO, copies SQL scripts |
| `roles/sql_install/tasks/install.yml` | Runs DSC `SqlSetup` resource to install SQL Server |
| `roles/sql_install/tasks/databasemail.yml` | Configures database mail (requires `gather_facts: true`) |
| `roles/sql_install/defaults/main.yml` | All default variable values — override via AAP extra vars |
| `roles/sql_install/templates/config2025.j2` | Configuration INI template for SQL Server setup |
| `roles/sql_install/files/sql_scripts/` | SQL scripts copied to `C:\temp` during prerequisites |

---

## AAP Workflow Setup

1. Create three **Job Templates** in AAP:
   - `Provision EC2 Windows` — your EC2 provisioning playbook
   - `Install SQL Server` — points to `sql_install.yml`
   - `Patch SQL Server` — points to `sql_patch.yml`

2. Create a **Workflow Template** and connect the nodes:
   ```
   [Provision EC2] --On Success--> [Install SQL Server] --On Success--> [Patch SQL Server]
   ```

3. Enable **Update Revision on Launch** on the project so AAP always syncs from the latest `main` branch commit before running.

4. Optionally configure a **GitHub webhook** to auto-sync on push:
   - Payload URL: `https://<aap-host>/api/v2/projects/<project-id>/update/`
   - Content type: `application/json`
   - Token: AAP API Bearer token with Write scope

---

## Author

Joe Brown — Red Hat  
Repository: [https://github.com/joebrown-RH/mssql_project](https://github.com/joebrown-RH/mssql_project)
