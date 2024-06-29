# Self-Hosted Talos Cluster via Proxmox

Building out a local, self-hosted Talos cluster within Proxmox.

Official Guide for Talos via Proxmox: [Link](https://www.talos.dev/v1.7/talos-guides/install/virtualized-platforms/proxmox/)

## Prerequisites

This guide assumes that you already have Proxmox installed and running.
In my case, I am using three mini-pc's with Proxmox directly installed.
The three nodes are all joined together in a Proxmox cluster, thus enabling
features, such as live migration of virtual machines.

Since each of my machines are limited in CPU and memory, this guide assumes that
each Talos VM will be both a management and worker node.

## Desired Setup

- talos-ctl: 192.168.50.100 (talosctl/management node)
- talos-01: 192.168.50.101 (primary control plane node)
- talos-02: 192.168.50.102 (worker node)
- talos-03: 192.168.50.103 (worker node)

## Getting started with Talos

### Talosctl Host Setup

You can either use a pre-existing host environment or build one specifically for
the purposes of managing your Talos cluster. Either way, Talos has a
command-line tool `talosctl` that integrates with `kubectl` to enable cluster
management.

I prefer to use a dedicated Ubuntu server VM in my Proxmox cluster that I can
SSH into.

#### 1. Create a new VM in Proxmox

Many of these settings are the defaults and/or are subjective to your own
cluster resources and configuration.
As an example, I am using Ceph Storage (CephPool) for my VM storage.

```console
Node: pve0
VM ID: 1000
Name: TalosCTL

Storage: ISOs
ISO Image: ubuntu-24.04-live-server-amd64.iso
Type: Linux
Version: 6.x - 2.6 Kernel

Machine: q35
BIOS: OVMF (UEFI)
Add EFI Disk: Yes
EFI Storage: CephPool
Pre-Enroll Keys: Yes
SCSI Controller: VirtIO SCSI single
Qemu Agent: Yes
Add TPM: No

Bus/Device: SCSI 0
Storage: CephPool
Disk size: 50 (GiB)
SSD emulation: Yes
Discard: Yes
IO thread: Yes
Read-only: No
Backup: Yes
Skip replication: No
Async IO: Default

Sockets: 1
Cores: 4
Type: Host

Memory: 4096 (MiB)

Bridge: vmbr0
VLAN Tag: 50
Firewall: Yes
Model: VirtIO
```

#### 2. Run the VM and install Ubuntu

The default settings throughout the install process are fine.

Feel free to use the network configuration step of the install process to
set a static IPv4 address via either DHCP or within the host itself.
I prefer to assign IP addresses via DHCP, which I can manually assign within my
router's management settings.

> IPv4 Address: 192.168.50.100

IPv6 is not a necessity and is purely an optional if using IPv4.

Another optional item is the use of BTRFS instead of EXT4.

> Server Name: talos-ctl

> Username: nick

Remember to enable the installation of OpenSSH.
You can use Github to obtain your SSH keys.

(Optional) If you are using a DNS server, such as Pihole or Adguard, then I also
recommend taking the time to add your server, so that you reference it via its
hostname instead of IP address later when connecting via SSH.

#### 3. Ubuntu Configuration

At this point, you should have a fresh installation of Ubuntu Server.

> This is a good time to create a snapshot via Proxmox in case you ever need to
> start over.

You should be able to SSH into your newly created server.

- Update the system:

  ```console
  sudo apt update && sudo apt upgrade
  ```

- Restart (if needed):

  ```console
  sudo reboot now
  ```

- Install and Start the Qemu Guest Agent:

  ```console
  sudo apt install qemu-guest-agent -y && sudo systemctl start qemu-guest-agent
  ```

- (Optional) Add `sudoers` configuration to bypass needing a password for
  `sudo` (substitute `nick` for your respective username):

  ```console
  sudo echo 'nick ALL=(ALL:ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/nick > /dev/null
  ```

- (Optional) Use Cloudflare's NTP server:

  Uncomment `NTP=` and add `time.cloudflare.com` to `/etc/systemd/timesyncd.conf`:

  ```console
  sudo nano /etc/systemd/timesyncd.conf
  ```

  Restart the service:

  ```console
  sudo systemctl restart systemd-timesyncd.service
  ```

> Once you have finished configuring Ubuntu Server to your personal liking,
> ensure you take another Proxmox snapshot.

#### 4. Talosctl Installation

[Official Documentation](https://www.talos.dev/v1.7/talos-guides/install/talosctl/)

Download and run the installation script:

```console
curl -sL https://talos.dev/install | sh
```

Add Bash completion for `talosctl`

- Use `talosctl completion bash` to output the Bash completion file:

  ```console
  talosctl completion bash > ~/.talos/completion.bash.inc
  ```

- Enable Bash to load the information via `.bashrc`:

  ```console
  printf "
  # talosctl shell completion
  source ~/.talos/completion.bash.inc
  " >> ~/.bashrc
  ```

- Either exit and log back into your SSH session, or manually execute the
  updated `.bashrc`:

  ```console
  source ~/.bashrc
  ```

#### 5. Kubectl Installation

[Official
Documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

There are several options for installing `kubectl`, such as directly downloading
the binary or using a package manager.

The binary download is the most direct method.

- Download the latest binary using Curl:

  ```console
  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
  ```

- Validate the binary's checksum:

  Download the checksum file:

  ```console
  curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"
  ```

  Validate the binary against the checksum:

  ```console
  echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
  ```

  Successful check:

  ```console
  kubectl: OK
  ```

- Install `kubectl`:

  ```console
  sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
  ```

- Validate the install:

  ```console
  kubectl version --client
  ```

- Enable `kubectl` Bash completion in `.bashrc`:

  ```console
  printf "
  # kubectl shell completion
  source <(kubectl completion bash)
  " >> ~/.bashrc
  ```

  Either exit and log back into your SSH session, or manually execute the
  updated `.bashrc`:

  ```console
  source ~/.bashrc
  ```

> Again, snapshot your VM again as a clean install prior to any artifacts being
> created by either `talosctl` or `kubectl`.

### Talos Node Setup

#### Download a Talos ISO Image

[Talos Linux Image Factory website](https://factory.talos.dev/)

Configuration:

- Hardware Type: Bare-metal Machine
- Talos Linux Version: 1.7.5 (or latest)
- Machine Architecture: amd64
- SecureBoot: Yes (optional, requires VM in Proxmox to support via UEFI BIOS settings)
- System Extensions (use the search function to quickly find):
  - siderolabs/qemu-guest-agent
  - siderolabs/intel-ucode (or use amd version depending on your cpu)

This will result in a webpage populating download links for the ISO and
additional information. [Link based on above specs](https://factory.talos.dev/?arch=amd64&cmdline-set=true&extensions=-&extensions=siderolabs%2Fintel-ucode&extensions=siderolabs%2Fqemu-guest-agent&platform=metal&secureboot=true&target=metal&version=1.7.5)

Download the ISO from the page.
This can be done via Proxmox's user interface, if preferred.

> Make sure to note the Initial Install - Installer Image.  
> For example: `factory.talos.dev/installer-secureboot/e3fab82b561b5e559cdf1c0b1e5950c0e52700b9208a2cfaa5b18454796f3a7e:v1.7.5`

#### Talos Node VM Creation

VM Creation Options:

```console
Node: pve0
VM ID: 1001
Name: talos-01

Storage: ISOs
ISO Image: talos-v1.7.5-metal-amd64-secure-qemu-intelucode.iso

Machine: Q35
BIOS: OVMF (UEFI)
Add EFI Disk: Yes
EFI Storage: CephPool
Pre-Enroll keys: No
SCSI Controller: VirtIO SCSI single
Qemu Agent: Yes
Add TPM: No

Bus/Device: SCSI 0
Storage: CephPool
Disk size: 100 (GiB)
SSD emulation: Yes
Discard: Yes
IO thread: Yes
Read-only: No
Backup: Yes
Skip replication: No
Async IO: Default

Sockets: 1
Cores: 4
Type: x86-64-v2-AES (Default)

Memory: 4096 (MiB)

Bridge: vmbr0
VLAN Tag: 50
Firewall: Yes
Model: VirtIO
```

> Important Notes:
>
> - If you enable `Pre-Enroll Keys`, then the VM will fail to boot.
> - Using the `host` CPU option is not recommended if intending to live migrate
>   the VM's in the future.

- Repeat VM creation for Talos-02 and Talos-03 on Nodes pve01 and pve02,
  respectively.

To set node IPv4 addresses via DHCP:

- Inspect the MAC addresses via the Proxmox GUI for each VM to assign IPv4
  addresses via your router's management interface.

#### Use `talosctl` to Generate Machine Configurations

- Prior to generating the cluster configuration files, we will set a Bash variable
  for the `$CONTROL_PLANE_IP` i.e. `192.168.50.101`:

  ```console
  export CONTROL_PLANE_IP=192.168.50.101
  ```

- Generate configuration files:

  This outputs configuration files (`controlplane.yaml`, `worker.yaml`, and
  `talosconfig`) for cluster `talos-proxmox-cluster` to
  a `~/.talos/configs/talos-proxmox-cluster` directory.

  > Ensure to include the installer image respective to that displayed from the Talos Linux Image Factory

  ```console
  talosctl gen config talos-proxmox-cluster https://$CONTROL_PLANE_IP:6443 --output-dir ~/.talos/configs/talos-proxmox-cluster --install-image factory.talos.dev/installer-secureboot/e3fab82b561b5e559cdf1c0b1e5950c0e52700b9208a2cfaa5b18454796f3a7e:v1.7.5
  ```

  `talosctl gen config` documentation:
  [Link](https://www.talos.dev/v1.7/reference/cli/#talosctl-gen-config)

#### Create the Control Plane Node

Expect this to take several minutes as Talos will be installed and configure the
Kubernetes control plane.

- Apply the `controlplane.yaml` configuration:

  ```console
  talosctl apply-config --insecure --nodes $CONTROL_PLANE_IP --file ~/.talos/configs/talos-proxmox-cluster/controlplane.yaml
  ```

  `talosctl apply-config` documentation:
  [Link](https://www.talos.dev/v1.7/reference/cli/#talosctl-apply-config)

#### Create the Worker Nodes

- Start the worker node VM's.

- Apply the `worker.yaml` configuration for each worker node:

  ```console
  talosctl apply-config --insecure --nodes 192.168.50.102 --file ~/.talos/configs/talos-proxmox-cluster/worker.yaml
  ```

  ```console
  talosctl apply-config --insecure --nodes 192.168.50.103 --file ~/.talos/configs/talos-proxmox-cluster/worker.yaml
  ```

  `talosctl apply-config` documentation:
  [Link](https://www.talos.dev/v1.7/reference/cli/#talosctl-apply-config)

Expect each node to take several minutes to go through its setup and
configuration stages.

#### Bootstrap the Cluster

- Merge the `~/.talos/configs/talos-proxmox-cluster/talosconfig` into
  `talosctl`'s default `~/.talos/config`:

  ```console
  talosctl config merge .talos/configs/talos-proxmox-cluster/talosconfig
  ```

  `talosctl config merge` documentation:
  [Link](https://www.talos.dev/v1.7/reference/cli/#talosctl-config-merge)

- Verify that `talosctl` now shows the `Current Context` as
  `talos-proxmox-cluster`

  ```console
  talosctl config info
  ```

  `talosctl config info` documentation:
  [Link](https://www.talos.dev/v1.7/reference/cli/#talosctl-config-info)

- Configure `talosctl` to reference the `$CONTROL_PLANE_IP` i.e. `192.168.50.101`:

  ```console
  talosctl config endpoint $CONTROL_PLANE_IP
  ```

  `talosctl config endpoint` documentation:
  [Link](https://www.talos.dev/v1.7/reference/cli/#talosctl-config-endpoint)

  ```console
  talosctl config node $CONTROL_PLANE_IP
  ```

  `talosctl config node` documentation:
  [Link](https://www.talos.dev/v1.7/reference/cli/#talosctl-config-node)

- Bootstrap Etcd

  ```console
  talosctl bootstrap
  ```

  `talosctl bootstrap` documentation:
  [Link](https://www.talos.dev/v1.7/reference/cli/#talosctl-bootstrap)

> Expect the bootstrap process to take several minutes to complete across all
> the nodes.

Once the process is completed, you will see the the Talos Dashboards for each
node showing `Ready: True`.

- Retrieve and merge the `kubeconfig` with `~/.kube/config`:

  ```console
  talosctl kubeconfig
  ```

  `talosctl kubeconfig` documentation: [Link](https://www.talos.dev/v1.7/reference/cli/#talosctl-kubeconfig)

> At this point, it's worthwhile to take snapshots of each VM.

### Post-Cluster Creation

Congratulations on creating a Kubernetes cluster using Talos.

At this point, you can proceed to use your cluster to run workloads.

You can install Git, Helm, etc on the Talos-CTL node to help facilitate this.

When you want to shutdown the cluster gracefully:

```console
talosctl shutdown --nodes 192.168.50.101,192.168.50.102,192.168.50.103
```

`talosctl shutdown` documentation: [Link](https://www.talos.dev/v1.7/reference/cli/#talosctl-shutdown)
