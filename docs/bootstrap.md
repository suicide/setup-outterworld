# Bootstrap

This repo bootstraps a fresh Ubuntu VPS that only has `root` password access and then manages it with Ansible over SSH key auth.

`playbooks/bootstrap.yml` is not just user creation. It prepares a fresh host for normal management by applying the baseline host configuration as well.

## Prerequisites

- `nix develop`
- A matching SSH host alias in local `~/.ssh/config`
- The fresh host is reachable as `root`
- The host vars file contains `admin_user` and `admin_authorized_keys`

## First boot for `outterworld3`

Run the bootstrap playbook and enter the root password when prompted:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/bootstrap.yml -k --limit outterworld3
```

The bootstrap playbook has a preflight check and will refuse to run if the host already has the repo-managed SSH hardening file. After first bootstrap, use `playbooks/site.yml` only.

What this does:

- installs baseline packages
- sets the hostname
- creates the admin user
- installs the configured SSH public key
- grants passwordless sudo
- disables root login and password SSH auth
- enables UFW and fail2ban

After bootstrap, log in with the configured SSH key as `ubuntu`.

## Existing hosts

Do not run `playbooks/bootstrap.yml` on hosts that already have a working non-root admin user.

For existing hosts:

- add them to the inventory
- set their `admin_user` and host-specific vars
- make sure Ansible can connect with your existing SSH config
- run `playbooks/site.yml`

`playbooks/site.yml` is the repeatable playbook for:

- common baseline settings
- SSH hardening
- UFW and fail2ban
- K3s
- cert-manager
- Let's Encrypt issuers

## Steady state configuration

After bootstrap, run the full site playbook:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/site.yml --limit outterworld3
```
