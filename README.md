# sol1-proxmox-vm-deployment

An Ansible role that deploys a virtual machine to a Proxmox cluster by cloning a template, sizing the hardware, configuring cloud-init, and (optionally) creating the SDN zone and VNet the VM attaches to.

The role is intended to be looped over an inventory of VM hosts: each play targets one VM, the role talks to the Proxmox API over HTTPS, and the VM ends up on the requested node with the requested resources.

## Requirements

- Ansible `>= 2.12`
- Collection: `community.proxmox` (provides `proxmox_vm_info`, `proxmox_kvm`, `proxmox_disk`, `proxmox_zone_info`, `proxmox_zone`, `proxmox_vnet_info`, `proxmox_vnet`)
- A Proxmox API token with permission to create/clone VMs, resize disks, and (if `use_sdn: true`) manage SDN objects
- A VM template already present in the cluster, referenced by name

Install the collection if you don't already have it:

```bash
ansible-galaxy collection install community.proxmox
```

## What the role does

1. **`get_cluster_info.yml`** — queries the cluster for all VMs/templates, finds the template by name, and decides whether the target VM already exists. If it does, the existing node is used; otherwise the template's node is recorded as a starting point.
2. **`prepare_vm_networks.yml`** — runs once per entry in `vm_interfaces`. If `use_sdn: true` the SDN zone and VNet are created when missing. Either way, an interface string is assembled into `_pve_interfaces_dict` in the form Proxmox expects (`net0`, `net1`, ...).
3. **`create_vm.yml`** — for new VMs: clone the template, apply hardware settings, apply cloud-init, grow the boot disk, then start the VM. For existing VMs, hardware settings are updated in place.

## Role variables

### Required — Proxmox connection

| Variable | Description |
| --- | --- |
| `proxmox_api_host` | Hostname/IP of the Proxmox API endpoint (usually one of the cluster nodes). |
| `proxmox_api_user` | API user, e.g. `ansible@pve`. |
| `proxmox_api_token` | API token ID for that user. |
| `proxmox_api_token_secret` | API token secret. |
| `proxmox_target_host` | Name of the Proxmox node the VM should live on. If the VM already exists on a different node, the role will switch to that node automatically. |
| `proxmox_storage` | Storage pool to clone the VM disks onto (e.g. `local-lvm`, `ceph-vm`). |
| `template_name` | Name of the template VM in Proxmox to clone from. The role looks the template up by name and resolves its node and VMID. |

### Required — VM definition

| Variable | Description |
| --- | --- |
| `vm_interfaces` | List of NICs to attach to the VM. See [Network interfaces](#network-interfaces) below. |
| `vm_primary_ip` | Primary IP in CIDR form for cloud-init `ipconfig0` (e.g. `10.0.10.20/24`). |
| `vm_gateway_ip` | Default gateway for cloud-init. |
| `vm_ciuser` | Cloud-init default user. |
| `vm_cipassword` | Cloud-init password for that user. Use a vaulted value. |
| `vm_ssh_key` | SSH public key(s) to inject via cloud-init. Newline-separated string. |
| `vm_dns_servers` | List of DNS server IPs (e.g. `['1.1.1.1', '8.8.8.8']`). |
| `vm_disk_size` | Final size of the boot disk in GiB (e.g. `40`). The disk is grown to this size after clone; it will not shrink. |
| `vm_vcpus` | Cores per socket. |
| `vm_vsockets` | Number of CPU sockets. |
| `vm_memory` | RAM in MiB (e.g. `4096`). |

`inventory_hostname` is used as the VM name in Proxmox — set it to whatever you want the VM called.

### Defaults (override as needed)

Set in `defaults/main.yml`:

| Variable | Default | Notes |
| --- | --- | --- |
| `proxmox_validate_certs` | `false` | Set to `true` if your Proxmox cluster presents trusted certificates. |
| `vm_ostype` | `l26` | Proxmox OS type. `l26` for any Linux, `win11` etc. for Windows. |
| `vm_display` | `qxl` | Display adapter. Alternatives: `std`, `virtio`, `vmware`. |
| `vm_bootdisk` | `scsi0` | Boot disk identifier. Must match the disk type in your template. |
| `use_sdn` | `false` | When `true`, the SDN zone and VNet for each interface are created if missing. |
| `vm_cloud_init` | `true` | Reserved for gating the cloud-init step. |
| `vm_ci_upgrade` | `true` | Run `apt/yum upgrade` on first boot via cloud-init. |
| `vm_citype` | `nocloud` | `nocloud` for Linux, `configdrive2` for Windows. |

## Network interfaces

`vm_interfaces` is a list. Each item becomes one `netN` entry on the VM in the order listed.

### Without SDN (`use_sdn: false`)

The bridge is used directly, and `vlan_id` (if set) becomes a `tag=` on the interface.

```yaml
use_sdn: false

vm_interfaces:
  - type: virtio          # NIC model. virtio recommended for Linux. Optional, defaults to virtio.
    bridge: vmbr0         # Existing Linux bridge on the Proxmox node.
    vlan_id: 10           # Optional. Becomes the VLAN tag on the interface.
    firewall: "1"         # Optional. "1" enables the Proxmox firewall on this NIC. Default "0".
    mac: "52:54:00:12:34:56"   # Optional. Auto-assigned if omitted.
```

### With SDN (`use_sdn: true`)

The role will create a VLAN-type SDN zone and a VNet named `vlan<vlan_id>` if they don't already exist. The VM is then attached to the VNet (the VLAN tag is bound at the VNet, not the NIC).

```yaml
use_sdn: true

vm_interfaces:
  - type: virtio
    bridge: vmbr0                          # Bridge the SDN zone is built on top of.
    sdn_zone_id: "mgmt"                    # Max 8 characters. Created if missing.
    sdn_type: vlan                         # Only 'vlan' is currently supported. Optional, defaults to vlan.
    sdn_ipam: pve                          # Only 'pve' is currently supported. Optional, defaults to pve.
    vlan_id: 10                            # VLAN tag for the VNet.
    vlan_name: "mgmt"                      # Friendly alias on the VNet.
    firewall: "1"                          # Optional. Default "0".
    mac: "52:54:00:12:34:56"               # Optional.
```

## Example: inventory and playbook

`inventory/hosts.yml`:

```yaml
all:
  children:
    proxmox_vms:
      hosts:
        web01.example.com:
          vm_primary_ip: "10.0.10.21/24"
        web02.example.com:
          vm_primary_ip: "10.0.10.22/24"
      vars:
        vm_gateway_ip: "10.0.10.1"
        vm_dns_servers:
          - "1.1.1.1"
          - "8.8.8.8"
        vm_vcpus: 2
        vm_vsockets: 1
        vm_memory: 4096
        vm_disk_size: 40
        vm_interfaces:
          - type: virtio
            bridge: vmbr0
            vlan_id: 10
            firewall: "1"
```

`group_vars/proxmox_vms/vault.yml` (encrypt with `ansible-vault`):

```yaml
vm_cipassword: "s3cret"
proxmox_api_token_secret: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

`group_vars/proxmox_vms/main.yml`:

```yaml
proxmox_api_host: "pve01.example.com"
proxmox_api_user: "ansible@pve"
proxmox_api_token: "ansible-token"
proxmox_target_host: "pve02"
proxmox_storage: "local-lvm"

template_name: "debian-12-cloudinit"

vm_ciuser: "ansible"
vm_ssh_key: |
  ssh-ed25519 AAAA... ops@example.com
```

`deploy.yml`:

```yaml
- name: Deploy Proxmox VMs
  hosts: proxmox_vms
  gather_facts: false
  connection: local
  roles:
    - sol1-proxmox-vm-deployment
```

Run it:

```bash
ansible-playbook -i inventory/hosts.yml deploy.yml --ask-vault-pass
```

`gather_facts: false` + `connection: local` is important: the role talks to the Proxmox API from the controller, not to the VM itself (which may not exist yet).

## Behaviour notes

- **Idempotency.** Re-running the role against an inventory of existing VMs updates hardware settings in place; it does not re-clone or destroy. Cloud-init and the boot-disk grow only run on the first deployment.
- **Disk grow is one-way.** `vm_disk_size` can only increase the boot disk — shrinking is unsupported by Proxmox and the role will error.
- **Template location.** The role finds the template by `template_name` across the cluster, so you don't need to specify which node it lives on.
- **Node tracking.** If a VM already exists on a node different from `proxmox_target_host`, the role uses the actual current node — it will not migrate the VM for you.

## License

See `LICENSE.txt`.
