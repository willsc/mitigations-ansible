# mitigations-ansible

Ansible playbooks that disable and blacklist specific kernel-module-based
mitigations on target hosts.

## Playbooks

| Playbook | Module | Config file written |
| --- | --- | --- |
| `disable-algif-aead.yml` | `algif_aead` | `/etc/modprobe.d/disable-algif-aead.conf` |
| `disable-copyfail.yml` | `copyfail` | `/etc/modprobe.d/copyfail-blacklist.conf` |
| `enable-mitigations.yml` | both | (removes the configs above) |

Each `disable-*` playbook:

1. Writes a `modprobe.d` config that `blacklist`s the module and rewires its
   `install` line to a no-op (`/bin/false` or `/bin/true`).
2. Runs `lsmod` to detect whether the module is currently loaded.
3. Unloads the module via `community.general.modprobe` if present.

`enable-mitigations.yml` reverses both: it removes the `modprobe.d` configs
and loads each module via `community.general.modprobe`. Module loads that
fail (e.g. module not built for the running kernel) are tolerated so the
revert still completes.

## Requirements

- Ansible 2.14+ on the control node (`ansible-core` package is enough).
- The `community.general` collection (bundled with the full `ansible` package;
  install separately with `ansible-galaxy collection install community.general`
  if running `ansible-core` only).
- Root / `sudo` on the target host (the plays use `become: true`).

## Inventory

The playbooks target `hosts: all`, so pass an inventory at runtime. A minimal
ad-hoc inventory works:

```bash
# single remote host
ansible-playbook -i 'host1.example.com,' disable-algif-aead.yml

# multiple hosts
ansible-playbook -i 'host1,host2,host3,' disable-algif-aead.yml

# inventory file
ansible-playbook -i inventory.ini disable-algif-aead.yml
```

The trailing comma in the ad-hoc form is required — it tells Ansible the
argument is a host list, not a file path.

## Running locally

To run against the control node itself:

```bash
ansible-playbook -i 'localhost,' -c local disable-algif-aead.yml
```

## Dry run (check mode)

Preview changes without applying them:

```bash
ansible-playbook -i 'localhost,' -c local --check --diff disable-algif-aead.yml
```

In `--check` mode the `lsmod` task is skipped (Ansible does not execute
`command` modules under check), so the unload step also reports as skipped —
this is expected.

## Verification

After a real run, confirm the config landed and the module is gone:

```bash
cat /etc/modprobe.d/disable-algif-aead.conf
lsmod | grep algif_aead   # should produce no output
```

## Reverting

Use the revert playbook to remove all configs and reload the modules:

```bash
ansible-playbook -i 'localhost,' -c local enable-mitigations.yml
```

Or revert a single host manually:

```bash
sudo rm /etc/modprobe.d/disable-algif-aead.conf
sudo modprobe algif_aead
```
