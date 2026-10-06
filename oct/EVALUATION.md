# Evaluation of `ansible_openchai.tgz` (current state)

Scope: the whole archive (608 entries: `inventory.py` 1141 lines, 63 playbooks, ~280 role/playbook YAML files).
Method: static review + **reproduction** of every inventory/vault defect in a scratch directory with a throw-away
password (`docs/evidence/legacy_defects_repro.py`) + Ansible 2.21 syntax-check sweep + pattern scans.
Your vault was **not** decrypted (only its header was read). No secret value is reproduced in this report.

## Verdict
The *idea* is sound (dynamic inventory + vault-backed credentials + GUI-managed node list) and several building
blocks are good (atomic-rename helper, flock, manifests, structured logging, no secrets in host_vars, fast parser:
10 000 hosts → 0.16 s). But the implementation is **not production-grade**: it can lose data silently, can be driven
into writing outside its directory, leaves inventory and vault inconsistent on any failure (your own logs show it),
and plaintext credentials exist at rest. Treat the credential items (S1) as urgent, independent of any redesign.

## Findings (all D-items reproduced; see evidence script)
| ID | Sev | Finding | Evidence |
|----|-----|---------|----------|
| S1 | **Critical** | Plaintext credentials at rest: `nodes.txt` holds an SSH password (the import path never deletes its source, the log even says "DELETE THE SOURCE FILE NOW"); `group_vars/all.yml` contains ~5 secret-like variables in clear text (kickstart root, pcs cluster, Slurm DB, LDAP root – lines ≈86–178) plus literal passwords in role defaults; `.vault_pass` sits in the project tree (a second, empty one under `inventory/`); no `.git`/`.gitignore` so nothing prevents these being committed/shipped (they were shipped in this archive). **Rotate all of them.** (One role-default password also scrolled through my own scan output.) | file review |
| D1 | **High** | `add_host` (the GUI-facing API) validates only the IP. A crafted node name wrote a file **outside** `host_vars/`; a crafted user injected `ansible_become_user:` into generated YAML and corrupted the inventory line. | E4 |
| D2 | **High** | Not transactional: inventory is saved, *then* vault sync runs. Any vault failure leaves host-in-inventory / no-credentials. Your `inventory.log` shows this repeatedly ("Inventory saved…" → `VaultError`). Reproduced: also blocks 30 s holding the lock when `ansible.cfg` isn't found from the caller's cwd. | E3 + log |
| D3 | **High** | `import` replaces the **entire** inventory with the file contents: no dry-run, no pre-backup, stale vault entries left behind. | E6 |
| D4 | **High** | `restore` joins manifest paths unchecked → arbitrary file write as the running user (root, per file ownership); no checksum; not atomic; leaves stale host_vars. | E7 |
| D5 | **High** | Silent data loss: `load()` skips malformed/duplicate lines with a warning; the next `save()` rewrites the file from valid records only → a hand-edit typo is **deleted** on the next GUI change; comments lost. | E5 |
| D6 | **High** | Vault YAML is hand-written/hand-parsed: passwords containing `"`, `\` or surrounding quotes are corrupted (3/10 test passwords) and re-escaped on every sync → silent wrong-password logins. | E1 |
| D7 | Med | Vault integration is fragile: the log shows the `default,default` failure (explicit `--vault-password-file` **plus** `vault_password_file` in `ansible.cfg` – reproduced, E8); the "fix" was commenting out the argument, so correctness now depends on `ansible.cfg` being discoverable from the caller's cwd. Plaintext temp file is written beside the vault before encryption; the zero-fill (`"\x00"*len`) is a no-op on immutable strings. | E3/E8 + code |
| D8 | Med | Blast radius: one `group_vars/all/vault.yml` with every node's password is in scope for every host in every play; one corrupted write loses all credentials; every sync re-encrypts everything. | design |
| D9 | Low | `^…$` + `re.match` accepts a trailing newline (`"root\n"`). | E2 |
| A1 | **High** | 61 of 65 playbooks are `hosts: all` + `become: true`; `host_key_checking = False`; NFS exports `clients: "*"` ×6 and `no_root_squash` ×9; SELinux disabled/permissive ×16; `gpgcheck` off ×15. One mistyped run (no `--limit`) changes the whole cluster. | scans |
| A2 | Med | 17 × `lookup('pipe', 'grep ^<node> inventory_def.txt …')` (13 in `group_vars/all.yml`): a shell per evaluation, couples every playbook to the storage file's column order, `^headnode` also matches `headnode2`. Replaced here by `inventories/production/group_vars/all/10_topology.yml`. | grep |
| A3 | Med | Host **and** group named `headnode` → Ansible warns "Found both group and host with same name". | `ansible-inventory --graph` |
| A4 | Med | No `requirements.yml`: 15 playbooks fail `--syntax-check` on a clean controller because `mount/sysctl/synchronize/selinux/ini_file/modprobe/openssh_keypair` live in `ansible.posix`/`community.*`. `storage_lustre_mount.yml` references a **role that does not exist**; `install_cuda_specific_version.yml` has a wrong `vars_files` depth. Roles are included via templated absolute paths instead of `roles_path`. | syntax sweep |
| A5 | Med | 0 `no_log` (while handling passwords), 79 `ignore_errors`, 267 `shell/command` tasks, a few `curl -k`/`validate_certs: false`. | scans |
| T1 | Med | 23 tests, all on pure helpers. Nothing covers locking, vault, CRUD, import, backup/restore, CLI or failure paths – exactly where every defect above lives. | tests/ |
| H1 | Low | No git/CI/lint; `.bk`/`.yml-new` leftovers; `__pycache__` shipped; paths inconsistent (`/opt/OpenCHAI` vs `/opt/OpenCHAI_Registry/OpenCHAI`); `[inventory] cache=yes` is inert for the `script` plugin and the `/tmp` cache dir is world-writable; docs claim Python 3.11+ while bytecode is 3.9; `nodes.txt` header lists columns in a different order than its data. | review |

## Not problems (checked)
Performance (parse 10k hosts: 0.16 s – no cache needed); `host_vars` stubs contain only template references, not secrets;
plain `os.replace` rename pattern is right (just missing fsync); logging/rotation is reasonable.

## Priority order
1. **Today:** rotate every password in S1; delete `nodes.txt`; remove `.vault_pass` from the tree; init git with a `.gitignore`.
2. **This sprint:** adopt this package (fixes D1–D9), migrate (docs/MIGRATION.md), switch the GUI to the stdio API.
3. **Next:** A1 (limit blast radius: no `hosts: all` defaults, `serial`, `--limit` guard, per-environment inventories),
   A4 (pin collections, fix the two broken playbooks), A5 (`no_log` on every credential-touching task), CI (`ansible-lint`,
   `yamllint`, `--syntax-check`, `pytest`).
