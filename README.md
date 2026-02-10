# Ansible + Semaphore for Proxmox VM creation

This repository is prepared to run from Semaphore (v2.16.51) with Ansible (v2.20.2).

## What this setup does

Creates a VM in Proxmox from a Debian 13 template with these runtime fields:
- `vm_name` (required)
- `disk_size_gb` (default `20`)
- `cpu_sockets` (default `2`)
- `cpu_cores` (default `2`)
- `memory_mb` (default `2048`)
- `network_bridge` (default `vmbr1`)
- `vlan_tag` (default `10`)
- `vm_id` (default `auto`, can be overridden)
- `ip_mode` (`dhcp`, `static`, `phpipam`)
- `root_password` (optional; if set, cloud-init login user is set to `root`)
- `wait_for_cloudinit` (default `true`; waits for first-boot cloud-init to finish)
- `cloudinit_user_data_snippet` (optional pre-created Proxmox `cicustom` snippet)

Static mode fields:
- `static_ip_cidr`
- `static_gateway`
- `static_dns1`
- `static_dns2`

phpIPAM mode fields:
- `phpipam_vlan_id` (optional, defaults to `vlan_tag`)
- plus phpIPAM credentials in Semaphore Environment

Fixed VM config (not user-editable in form):
- start at boot: enabled
- qemu agent: enabled
- storage: `vm_storage`
- backup: disabled on primary disk
- cpu type: `host`
- numa: enabled
- firewall: disabled
- mtu: `1`
- hotplug: `disk,network,usb,memory,cpu`
- qemu guest agent can be auto-installed on first boot if `cloudinit_user_data_snippet` is set

## Prerequisites

1. Proxmox API token with permission to clone/configure/start VMs.
2. Debian 13 cloud-init template in Proxmox (example VMID: `9000`).
3. Semaphore can access this Git repository.
4. Optional: pre-create a cloud-init snippet and set `cloudinit_user_data_snippet`.
   In some Proxmox setups, `/storage/<id>/upload` does not allow `content=snippets`.
5. For automatic snippet creation, Semaphore runner must have `ssh`/`scp` access to the Proxmox node.

Example snippet (on Proxmox node):

```bash
mkdir -p /mnt/pve/cephfs/snippets
cat >/mnt/pve/cephfs/snippets/install-qga.yaml <<'EOF'
#cloud-config
package_update: true
packages:
  - qemu-guest-agent
runcmd:
  - systemctl enable --now qemu-guest-agent
EOF
```

Then set:

```yaml
cloudinit_user_data_snippet: "user=cephfs:snippets/install-qga.yaml"
```

Automatic mode (no manual snippet file):
- Set `auto_create_cloudinit_snippet: true`
- Set `proxmox_snippets_storage` and `proxmox_snippets_mount_path` (e.g. `cephfs` and `/mnt/pve/cephfs`)
- Ensure runner can SSH to Proxmox (`proxmox_ssh_user` / optional `proxmox_ssh_private_key_path`)

## Debian 13 template (one-time, on Proxmox node)

Run on the Proxmox host shell and adjust paths/IDs if needed:

```bash
TEMPLATE_ID=9000
TEMPLATE_NAME=debian-13-template
STORAGE=vm_storage
IMAGE=/var/lib/vz/template/iso/debian-13-genericcloud-amd64.qcow2

# Download official Debian 13 cloud image first and place at $IMAGE.
wget -O "$IMAGE" https://cloud.debian.org/images/cloud/trixie/latest/debian-13-genericcloud-amd64.qcow2

qm create $TEMPLATE_ID --name $TEMPLATE_NAME --memory 2048 --cores 2 --sockets 1 --net0 virtio,bridge=vmbr1
qm set $TEMPLATE_ID --scsihw virtio-scsi-pci --scsi0 ${STORAGE}:0,import-from=${IMAGE}
qm set $TEMPLATE_ID --ide2 ${STORAGE}:cloudinit
qm set $TEMPLATE_ID --boot order=scsi0
qm set $TEMPLATE_ID --serial0 socket --vga serial0
qm set $TEMPLATE_ID --agent enabled=1
qm template $TEMPLATE_ID
```

Notes:
- `scsi0` is the OS disk. `ide2` is the cloud-init metadata drive (not the system disk).
- `STORAGE` must support VM disks/images (`images` content). Do not use ISO-only storage for this.

If you prefer maximum robustness, use one-line `qm create` (avoid line-continuation issues):

```bash
qm create "$TEMPLATE_ID" --name "$TEMPLATE_NAME" --memory 2048 --cores 2 --sockets 1 --net0 "virtio,bridge=vmbr1"
```

If script execution fails with `--name: command not found`, run:

```bash
sed -i 's/\r$//' your-script.sh
```

## Semaphore setup

1. Key Store:
- Add SSH key or token required to read `https://github.com/Bondops/Ansible`.

2. Repository:
- URL: `https://github.com/Bondops/Ansible.git`
- Branch: `main`
- Link Key Store if repo is private.

3. Inventory:
- Type: Static
- Content from `inventories/semaphore/hosts.yml`

4. Environment:
- Create environment and paste values from `semaphore/environment.example.yml`.
- Replace secrets and hostnames.

5. Task Template:
- Type: Ansible Playbook
- Repository: `Bondops/Ansible`
- Inventory: the one above
- Environment: the one above
- Playbook: `playbooks/create_proxmox_vm.yml`
- Install collections before first run:
  - `ansible-galaxy collection install -r requirements.yml`
- Extra Variables default template:
  - use `semaphore/create-vm-vars.example.yml`
  - for `root_password`, use a Survey variable of type `Secret` (or pass in Secrets)

## Runtime behavior for `ip_mode`

- `dhcp`:
  - VM network is left as DHCP (`ip=dhcp`).
- `static`:
  - Requires `static_ip_cidr`, `static_gateway`, `static_dns1`, `static_dns2`.
- `phpipam`:
  - Finds subnet by VLAN in phpIPAM.
  - Reserves first free IP.
  - Applies static cloud-init network from phpIPAM subnet + your phpIPAM DNS values.

## Run locally (optional)

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/create_proxmox_vm.yml -e @semaphore/environment.example.yml -e @semaphore/create-vm-vars.example.yml
```

## Files

- `playbooks/create_proxmox_vm.yml`
- `roles/proxmox_vm/defaults/main.yml`
- `roles/proxmox_vm/tasks/main.yml`
- `inventories/semaphore/hosts.yml`
- `inventories/semaphore/group_vars/all.yml`
- `semaphore/environment.example.yml`
- `semaphore/create-vm-vars.example.yml`
