# Deploy Red Hat Satellite 6.19

Ansible automation to deploy and configure **Red Hat Satellite 6.19** on RHEL 9,
with separate **production** and **qualification** (staging) environments,
following Red Hat and Ansible best practices.

## What it does

| Stage | Role | Purpose |
|-------|------|---------|
| 1 | `satellite_prereqs` | Validate OS/sizing/FQDN/DNS, set time sync (chrony) & timezone, provision dedicated XFS storage, open firewall ports |
| 2 | `satellite_subscription` | Register to Red Hat (activation key), lock RHEL release, enable only the required repos, patch & reboot |
| 3 | `satellite_install` | Install the `satellite` package and run `satellite-installer` (idempotent, custom-cert aware, tuning profile) |
| 4 | `satellite_postconfig` | Upload subscription manifest and create lifecycle environments via the Satellite API |

## Deployment model (ansible-core "seed")

Satellite is deployed onto a **new infrastructure** from a central **ansible-core
"seed" server**, which also deploys Ansible Automation Platform (AAP) — kept in a
**separate project/repo**. The seed is the single bootstrap control node for the
whole new estate.

```
                +-------------------------+
                |  ansible-core "seed"    |
                |  (control node)         |
                |  - this repo (Satellite)|
                |  - AAP repo (separate)  |
                +-----------+-------------+
                            | SSH (svc_ansible + sudo)
             +--------------+--------------+
             |                             |
     +-------v--------+           +--------v-------+
     | Satellite host |           |  AAP host(s)   |
     | (already built)|           | (already built)|
     +----------------+           +----------------+
```

Key assumptions for this repo:

- **Targets already exist.** The seed only *configures* hosts; it does not
  provision the VMs. Ensure the Satellite host is a freshly installed RHEL 9
  with correct FQDN/DNS before running.
- **Connection identity** is declared per environment in
  `inventories/<env>/hosts.yml` under `all.vars` (`ansible_user`,
  `ansible_ssh_private_key_file`). Privilege escalation (sudo → root) is enabled
  globally in `ansible.cfg`.
- **First-run connectivity from the seed:**
  - Distribute the seed user's public key to each target's `authorized_keys`.
  - Pre-seed each target's host key (`host_key_checking` is on):
    `ssh-keyscan -H <ip_or_fqdn> >> ~/.ssh/known_hosts`.
- **Control-node-side files.** The manifest is uploaded through the Satellite
  API from the seed (`delegate_to: localhost`), so `satellite_manifest_path`
  points to a file **on the seed** (default: `files/manifest_<env>.zip` next to
  `site.yml`). Custom certificates are used **on the Satellite host**: set
  `satellite_custom_certs_deliver: true` to have the install role push the
  cert/key/CA from the seed (`satellite_custom_*_src`) to the target
  (`satellite_custom_*_file`), or leave it `false` if the host already carries
  them (e.g. from `rhel_post_install` PKI enrolment). Set
  `satellite_use_custom_certs: false` to use the installer's self-signed CA.

## Repository layout

```
.
├── ansible.cfg
├── requirements.yml            # Galaxy collections
├── site.yml                    # Main playbook
├── inventories/
│   ├── production/
│   │   ├── hosts.yml
│   │   └── group_vars/
│   │       ├── all.yml         # Non-secret prod settings
│   │       └── vault.yml       # ENCRYPTED secrets
│   └── qualification/
│       ├── hosts.yml
│       └── group_vars/
│           ├── all.yml
│           └── vault.yml
└── roles/
    ├── satellite_prereqs/
    ├── satellite_subscription/
    ├── satellite_install/
    └── satellite_postconfig/
```

## Prerequisites

- Control node with **Ansible 2.15+** (ansible-core) and Python 3.9+.
- Target host: freshly installed **RHEL 9**, correct **FQDN** with working
  forward/reverse DNS, and dedicated block devices/LVM volumes for
  `/var/lib/pulp`, `/var/lib/pgsql`, and `/var/log`.
- A valid Satellite subscription and (recommended) an **activation key**.
- SSH access with a privileged (sudo) account.

## Setup

1. Install the required collections:

   ```bash
   ansible-galaxy collection install -r requirements.yml -p collections
   ```

2. Edit the inventory for your environment:
   - `inventories/production/hosts.yml`
   - `inventories/qualification/hosts.yml`

   Set the connection identity (`ansible_user`,
   `ansible_ssh_private_key_file`) under `all.vars` so the seed can reach the
   already-provisioned target(s).

3. Adjust non-secret settings in `group_vars/all.yml` (org/location name,
   storage devices, tuning profile, certificates, repos, lifecycle envs).

4. Add secrets and **encrypt** the vault file:

   ```bash
   ansible-vault encrypt inventories/production/group_vars/vault.yml
   ansible-vault encrypt inventories/qualification/group_vars/vault.yml
   ```

## Run

Qualification first (recommended), then production:

```bash
# Qualification
ansible-playbook -i inventories/qualification/hosts.yml site.yml --ask-vault-pass

# Production
ansible-playbook -i inventories/production/hosts.yml site.yml --ask-vault-pass
```

Dry run / check mode:

```bash
ansible-playbook -i inventories/qualification/hosts.yml site.yml --ask-vault-pass --check --diff
```

Run a single stage with tags (`prereqs`, `subscription`, `install`, `postconfig`):

```bash
ansible-playbook -i inventories/production/hosts.yml site.yml --tags install --ask-vault-pass
```

## Best practices applied

- **Environment isolation** — independent inventories and `group_vars` for
  production and qualification; qualify changes before promoting to production.
- **Secrets management** — all passwords/keys kept in `ansible-vault`-encrypted
  `vault.yml`; `no_log` on sensitive tasks.
- **Idempotency** — `satellite-installer` is run with `--detailed-exitcodes`
  (rc 0 = no change, rc 2 = changed); filesystems are only created when absent.
- **Fail fast** — pre-flight assertions for OS version, CPU/RAM, FQDN and DNS
  before any change is made.
- **Right-sizing** — installer `--tuning` profile per environment
  (`large` for prod, `default` for qual).
- **Dedicated storage** — separate volumes for Pulp content, the database and
  logs to protect the root filesystem.
- **Minimal repos** — disable all repositories, then enable only the four
  required for Satellite to avoid dependency drift.
- **Custom certificates** — production uses organization-signed certs; the
  installer generates a self-signed CA for qualification.

## Notes

- Satellite 6.19 requires RHEL 9. The subscription role enables:
  `rhel-9-for-x86_64-baseos-rpms`, `rhel-9-for-x86_64-appstream-rpms`,
  `satellite-6.19-for-rhel-9-x86_64-rpms`,
  `satellite-maintenance-6.19-for-rhel-9-x86_64-rpms`.
- Always confirm the current sizing and port requirements against the official
  Red Hat Satellite 6.19 installation guide for your scale.
