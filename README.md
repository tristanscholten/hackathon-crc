# hackathon-crc

Ansible playbook for installing OpenShift Local / CRC on fresh Ubuntu hosts and exposing it over a WireGuard-reachable network.

## What it installs

On each Ubuntu host in `inventory/hosts.yml`:

- CRC binary
- KVM/libvirt dependencies
- OpenShift CRC VM
- HAProxy TCP forwarding for the API server (`192.168.10.55:6443` -> `127.0.0.1:6443` in CRC user-networking mode)
- CRC's user-networking router exposure for HTTP/HTTPS (`192.168.10.55:80/443`)
- dnsmasq records for CRC names

Default targets:

```text
crc01 -> 192.168.10.55
crc02 -> 192.168.10.56
```

## Do I need DNS?

Yes. Clients need DNS for the CRC names.

CRC itself uses these canonical names:

```text
api.crc.testing
*.apps-crc.testing
```

This playbook also exposes cluster-scoped aliases:

```text
api.crc01.testing
*.apps.crc01.testing
api.crc02.testing
*.apps.crc02.testing
```

But the cleanest browser/`oc` experience is still to resolve the canonical CRC names to the Ubuntu host running that cluster.

For `crc01`, point DNS to:

```text
api.crc.testing      -> 192.168.10.55
*.apps-crc.testing   -> 192.168.10.55
api.crc01.testing    -> 192.168.10.55
*.apps.crc01.testing -> 192.168.10.55
```

For `crc02`, point DNS to:

```text
api.crc.testing      -> 192.168.10.56
*.apps-crc.testing   -> 192.168.10.56
api.crc02.testing    -> 192.168.10.56
*.apps.crc02.testing -> 192.168.10.56
```

The playbook installs `dnsmasq` on the CRC host. While connected to WireGuard, either use the target host as DNS (`192.168.10.55` for crc01, `192.168.10.56` for crc02) or copy the same records into your existing DNS servers.

## Requirements on the control machine

```bash
sudo apt-get install -y ansible sshpass
```

WireGuard must be connected so the control machine can reach the target host.

The Ubuntu VM must expose KVM/nested virtualization:

```bash
test -e /dev/kvm
grep -E '(vmx|svm)' /proc/cpuinfo
```

On Proxmox/VMware/etc. this usually means host CPU passthrough and nested virtualization enabled.

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
          ansible_host: 192.168.10.55
          ansible_user: lab
          crc_cluster_name: crc01
          crc_bind_address: 192.168.10.55
        crc02:
          ansible_host: 192.168.10.56
          ansible_user: lab
          crc_cluster_name: crc02
          crc_bind_address: 192.168.10.56
```

Add more hosts for more clusters:

```yaml
        crc02:
          ansible_host: 192.168.10.56
          ansible_user: lab
          crc_cluster_name: crc02
          crc_bind_address: 192.168.10.56
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

## Client usage

While connected to WireGuard, configure DNS to use the CRC host IP, e.g. for crc01:

```text
DNS server: 192.168.10.55
```

Then open:

```text
https://console-openshift-console.apps-crc.testing
```

or use `oc`:

```bash
oc login -u developer https://api.crc.testing:6443
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
- The aliases `api.crc01.testing` and `*.apps.crc01.testing` are routed by HAProxy/dnsmasq, but CRC certificates/routes are canonical `crc.testing` / `apps-crc.testing`.
- Use separate host IPs or DNS profiles when you want multiple CRC clusters reachable from the same client at the same time.
