# hackathon-crc

Ansible playbook for installing OpenShift Local / CRC on fresh Ubuntu hosts and exposing it over a WireGuard-reachable network.

## Quick start

Run the full playbook, including Technitium DNS provisioning:

```bash
ansible-playbook playbooks/site.yml \
  -i inventory/hosts.yml \
  -e ansible_password='<lab-password>' \
  -e ansible_become_password='<lab-password>' \
  -e technitium_dns_api_token='<technitium-api-token>'
```

Or export the Technitium token instead of passing it on the command line:

```bash
export TECHNITIUM_DNS_API_TOKEN='<technitium-api-token>'
ansible-playbook playbooks/site.yml \
  -i inventory/hosts.yml \
  -e ansible_password='<lab-password>' \
  -e ansible_become_password='<lab-password>'
```

## What it installs

On each Ubuntu host in `inventory/hosts.yml`:

- downloads the latest CRC archive from Red Hat's `.../clients/crc/latest/crc-linux-amd64.tar.xz`
- extracts it into `~/Downloads`
- copies the `crc` executable into `~/bin`
- adds `~/bin` to `.bashrc`
- runs `crc config set consent-telemetry no` before start
- runs `crc config set preset openshift` before start
- runs `crc setup`
- runs `crc start`
- installs and enables a CRC user systemd autostart service so CRC starts again after host reboots
- downloads the latest `oc`/`kubectl` archive from `https://mirror.openshift.com/pub/openshift-v4/clients/oc/latest/linux/oc.tar.gz`
- extracts `oc` and `kubectl` into `~/bin`
- runs `oc login` as `developer` against the CRC API
- verifies the console UI and OAuth health route over HTTPS
- KVM/libvirt dependencies
- OpenShift CRC VM
- HAProxy TCP forwarding for the API server (`<ansible_host>:6443` -> `127.0.0.1:6443` in CRC user-networking mode)
- CRC's user-networking router exposure for HTTP/HTTPS (`<ansible_host>:80/443`)
- CRC CPU and memory sized dynamically to 80% of each host by default
- an 80 GiB CRC VM disk by default; smaller disks can hit OpenShift node `DiskPressure` after repeated restarts/log growth
- dnsmasq records for CRC names

Default targets:

```text
crc01 -> 192.168.10.74
crc02 -> 192.168.10.69
crc03 -> 192.168.10.72
crc04 -> 192.168.10.75
```

## Do I need DNS?

Yes. Clients need DNS for the CRC names.

The public/client-facing names are configurable per cluster. Defaults:

```text
api.crc01.testing
*.crc01.testing
*.apps-crc01.testing
api.crc02.testing
*.crc02.testing
*.apps-crc02.testing
```

The defaults come from these variables and can be overridden per host/group:

```yaml
crc_api_domain: "{{ crc_cluster_name }}.{{ crc_base_domain }}"
crc_apps_domain: "apps-{{ crc_cluster_name }}.{{ crc_base_domain }}"
```

Following the CRC engineering docs, the playbook configures:

- `ingresses.config.openshift.io/cluster spec.appsDomain`
- `ingresses.config.openshift.io/cluster spec.componentRoutes` for console/OAuth
- `apiserver/cluster spec.servingCerts.namedCertificates` for the custom API host
- `dns.operator/default spec.servers` so the cluster itself can resolve the custom domains
- `dnsmasq` records so WireGuard clients can resolve the custom domains
- optional Technitium DNS zone/record management on `https://dns01.lo.example-domain.eu:53443/` when `TECHNITIUM_DNS_API_TOKEN` is set

For `crc01`, point DNS to:

```text
api.crc01.testing     -> 192.168.10.74
*.crc01.testing       -> 192.168.10.74
*.apps-crc01.testing  -> 192.168.10.74
```

For `crc02`, point DNS to:

```text
api.crc02.testing     -> 192.168.10.69
*.crc02.testing       -> 192.168.10.69
*.apps-crc02.testing  -> 192.168.10.69
```

For `crc03`, point DNS to:

```text
api.crc03.testing     -> 192.168.10.72
*.crc03.testing       -> 192.168.10.72
*.apps-crc03.testing  -> 192.168.10.72
```

For `crc04`, point DNS to:

```text
api.crc04.testing     -> 192.168.10.75
*.crc04.testing       -> 192.168.10.75
*.apps-crc04.testing  -> 192.168.10.75
```

The playbook installs `dnsmasq` on the CRC host. When a Technitium API token is available, the `technitium_dns` role also creates the `crcNN.testing` and `apps-crcNN.testing` primary zones and manages wildcard `A` records pointing at each host's `ansible_host` IP.

Technitium configuration is disabled unless a token is provided:

```bash
export TECHNITIUM_DNS_API_TOKEN=...
ansible-playbook -i inventory/hosts.yml --tags technitium_dns playbooks/site.yml
```

## Requirements on the control machine

```bash
sudo apt-get install -y ansible sshpass
```

WireGuard must be connected so the control machine can reach the target host.

The playbook installs the CRC docs Ubuntu prerequisites before running `crc setup`:

```bash
sudo apt install qemu-kvm libvirt-daemon libvirt-daemon-system network-manager
```

It also installs supporting packages such as `libvirt-clients`, `virtinst`, `bridge-utils`, `dnsmasq`, and `haproxy`, then reboots after the first package install by default and tries to load the KVM kernel modules. The Ubuntu VM must still expose virtualization from the hypervisor:

```bash
test -e /dev/kvm
grep -E '(vmx|svm)' /proc/cpuinfo
```

If `/dev/kvm` is still missing after package install, fix the hypervisor settings. On Proxmox/VMware/etc. this usually means host CPU passthrough and nested virtualization enabled.

## Pull secret

Do **not** commit your pull secret.

Save it locally and point Ansible at it:

```bash
export CRC_PULL_SECRET_PATH=~/.hermes/secrets/hackathon-crc-pull-secret.json
chmod 600 "$CRC_PULL_SECRET_PATH"
```

## Inventory

Edit `inventory/hosts.yml` if addresses/users change. The committed inventory already includes:

```yaml
all:
  children:
    crc_hosts:
      hosts:
        crc01:
          ansible_host: 192.168.10.74
          ansible_user: lab
          crc_cluster_name: crc01
        crc02:
          ansible_host: 192.168.10.69
          ansible_user: lab
          crc_cluster_name: crc02
        crc03:
          ansible_host: 192.168.10.72
          ansible_user: lab
          crc_cluster_name: crc03
        crc04:
          ansible_host: 192.168.10.75
          ansible_user: lab
          crc_cluster_name: crc04
```

Add more hosts for more clusters:

```yaml
        crc05:
          ansible_host: 192.168.10.71
          ansible_user: lab
          crc_cluster_name: crc05
```

## CRC sizing

CRC CPU and memory are calculated per host from gathered Ansible facts:

```yaml
crc_resource_ratio: 0.8
crc_min_cpus: 4
crc_min_memory_mib: 14336
crc_disk_size_gib: 80
```

Effective values:

```text
crc_effective_cpus       = floor(host_vcpus * crc_resource_ratio)
crc_effective_memory_mib = floor(host_memory_mib * crc_resource_ratio)
```

If the existing CRC VM config differs from the calculated CPU, memory, or disk
size, the role stops and deletes the existing CRC VM before starting it again so
the new sizing takes effect.

Optional guard rails catch wrong hypervisor CPU exposure before CRC starts:

```yaml
crc_expected_host_cpus: 24
crc_expected_cpus: 19
```

For example, if you expected a 40-vCPU host and therefore a 32-vCPU CRC VM,
run with:

```bash
ansible-playbook playbooks/site.yml \
  -i inventory/hosts.yml \
  --limit crc01 \
  -e crc_expected_host_cpus=40 \
  -e crc_expected_cpus=32
```

The role also verifies the live OpenShift node reports the calculated CPU
capacity after CRC starts.

## Expand `/home` with a new disk

CRC's writable VM disk defaults to:

```text
/home/<crc-user>/.crc/machines/crc/crc.qcow2
```

So the immediate fix for the known qcow2 host-full failure is to expand
`/home`, not `/`, unless you intentionally move `.crc/machines` somewhere else.
The standalone expansion playbook auto-discovers exactly one safe whole-disk
candidate, adds it to the LVM volume group backing `/home`, and extends the
`/home` logical volume/filesystem with all free space. The device name and disk
size are discovered at runtime, so `/dev/sda`, `/dev/sdb`, `/dev/sdc`, NVMe,
128G/256G/larger disks all work when they satisfy the safety checks.

Run on one host first:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/expand-home-disk.yml \
  --limit crc01
```

If more than one safe blank disk exists, auto-discovery refuses to guess. Inspect
the host and select the intended disk explicitly:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/expand-home-disk.yml \
  --limit crc01 \
  -e crc_home_expansion_disk=/dev/sdc
```

Safety behavior:

- excludes the whole disk backing `/`
- requires a whole disk of at least 32 GiB by default
- requires exactly one safe candidate in automatic mode
- refuses mounted, partitioned, signed, or foreign-VG disks
- refuses to run unless `/home` is LVM-backed
- recognizes a previously consumed whole-disk PV in the `/home` VG for idempotent reruns
- consumes the full free size of the selected disk/VG
- supports ext2/ext3/ext4/xfs online filesystem growth through `lvextend --resizefs`

## Automated storage guard

Every CRC host gets a root systemd timer by default:

```text
crc-storage-guard.timer -> crc-storage-guard.service
```

It runs every 10 minutes and performs safe automatic cleanup before CRC fills
its VM or host filesystem. No human approval is required for cleanup.

What it checks:

```yaml
crc_storage_guard_host_path: "{{ crc_run_home }}/.crc/machines"
crc_storage_guard_vm_var_path: /var
crc_storage_guard_host_cleanup_used_percent: 85
crc_storage_guard_host_critical_used_percent: 92
crc_storage_guard_vm_cleanup_used_percent: 80
crc_storage_guard_vm_critical_used_percent: 90
```

What it may clean automatically:

```yaml
crc_storage_guard_enable_container_prune: true
crc_storage_guard_enable_image_prune: true
crc_storage_guard_enable_journal_vacuum: true
crc_storage_guard_enable_fstrim: true
crc_storage_guard_journal_vacuum_size: 500M
```

The image prune uses:

```bash
crictl rmi --prune
```

That removes **unused** CRI-O images only. It does not remove images referenced
by existing containers or pods. Running/used images are left alone.

The guard intentionally does not compact qcow2 files automatically:

```yaml
crc_storage_guard_auto_compact_qcow2: false
```

Compaction can require scratch space and can make a full host worse. The guard
prevents the common failure by pruning unused VM-side content early and running
`fstrim`, not by doing risky late-stage disk surgery.

Install or re-run only the guard:

```bash
ansible-playbook -i inventory/hosts.yml playbooks/storage-guard.yml
```

Run it on demand on a host:

```bash
sudo systemctl start crc-storage-guard.service
sudo journalctl -u crc-storage-guard.service -n 100 --no-pager
sudo tail -n 100 /var/log/crc-storage-guard/guard-$(date -u +%Y%m%d).log
```

## Run

For a password-based fresh Ubuntu host:

```bash
export CRC_PULL_SECRET_PATH=~/.hermes/secrets/hackathon-crc-pull-secret.json
ansible-playbook playbooks/site.yml \
  -i inventory/hosts.yml \
  -e ansible_password='YOUR_SSH_PASSWORD' \
  -e ansible_become_password='YOUR_SUDO_PASSWORD'
```

Or create a private, uncommitted `.local/hosts.local.yml` with secrets and run:

```bash
ansible-playbook -i .local/hosts.local.yml playbooks/site.yml
```

The playbook downloads `oc`/`kubectl` both on the CRC host and under `.local/bin/` on the Ansible controller. It then verifies `oc login` from the controller through the configured API domain.

## Client usage

While connected to WireGuard, configure DNS to use the CRC host IP, e.g. for crc01:

```text
DNS server: 192.168.10.74
```

Then open:

```text
https://console-openshift-console.apps-crc01.testing
```

or use `oc`:

```bash
oc login -u developer -p developer https://api.crc01.testing:6443 --insecure-skip-tls-verify=true
```

Get credentials on the CRC host:

```bash
crc console --credentials
```

A summary is written on the host:

```text
~/crc-access-crc01.txt
```

## Notes / limitations

- CRC is designed as a single-node local OpenShift environment. Treat this as one CRC instance per Ubuntu host.
- The playbook uses CRC `network-mode: user` because Ubuntu hosts with `systemd-networkd` are not supported by CRC system networking.
- By default the playbook creates `/etc/sudoers.d/99-crc-<user>` with `NOPASSWD:ALL` for the CRC run user because `crc setup/start` calls privileged helpers non-interactively. Set `crc_allow_passwordless_sudo_for_setup: false` if you handle that another way.
- The playbook supports multiple clusters through inventory entries (`crc01`, `crc02`, ...), not multiple CRC VMs on one host.
- The playbook configures custom CRC domains using the CRC engineering-docs approach: ingress `appsDomain`/`componentRoutes`, API named certificates, and cluster DNS forwarding. Override `crc_api_domain` and `crc_apps_domain` to change `*.crc02.testing` / `*.apps-crc02.testing` later.
- Use separate host IPs or DNS profiles when you want multiple CRC clusters reachable from the same client at the same time.
