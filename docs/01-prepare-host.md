# Preparing the Host for OKD

## 1. Purpose

Prepare a Red Hat Enterprise Linux 9.8 host for an OKD Single Node OpenShift (SNO) installation. Upon completion, the host is ready to create the OKD virtual machine described in the next guide.

> [!WARNING]
> This runbook prepares an OKD environment for installing Anypoint Runtime Fabric. OKD is not a MuleSoft-certified or supported Runtime Fabric platform. Use this implementation only for learning, experimentation, and proof-of-concept purposes.

## 2. Host Requirements

The host system must satisfy the following minimum requirements before installing an OKD Single Node (SNO) cluster.

> [!IMPORTANT]
> Although the minimum hardware requirements are sufficient to install OKD, additional CPU, memory, and storage may be required to run Runtime Fabric and Mule applications. The validated virtual machine sizing used throughout this guide is documented in `AA-configuration-reference.md`.

### Hardware

- 64-bit processor (`x86_64` recommended)
- Hardware-assisted virtualization (Intel VT-x or AMD-V) enabled in the system firmware
- Minimum **4 vCPUs**
- Minimum **16 GB RAM**
- Minimum **120 GB** of available storage
- Internet connectivity to download software and container images

### Operating System

- Red Hat Enterprise Linux 9.8 (Release 1)
- Administrative (`sudo`) access
- Latest system updates installed

> [!TIP]
> A Red Hat account with an active subscription is required to download and register Red Hat Enterprise Linux. For individual learning and development, Red Hat provides a no-cost Red Hat Developer Subscription for Individuals.

### Storage

Ensure the host has sufficient storage available for the OKD virtual machine, installation media, and snapshots.

### Virtualization

- KVM
- libvirt
- QEMU
- `virt-install`
- `virsh`

### Networking

- A bridged network interface for the virtual machine
- A static IP address or DHCP reservation for the OKD node
- Reliable DNS resolution for the cluster and outbound Internet connectivity. The OKD node and workloads must be able to resolve the external hostnames required by Runtime Fabric.

## 3. Install Red Hat Enterprise Linux

Install Red Hat Enterprise Linux 9.8 using the graphical installer and the configuration shown below.

| Category | Setting |
|----------|---------|
| Installation Source | Red Hat CDN |
| Software Selection | **Server with GUI** |
| Installation Destination | Automatic partitioning |
| Network | Enabled |
| Host Name | `orion` |
| Time Zone | America/Los_Angeles |
| Root Password | Configured |
| Administrative User | Created (`wheel` group) |
| Security Profile | None |
| KDUMP | Enabled (default) |

## 4. Prepare the Host

Perform the following tasks to prepare the host for the OKD installation.

### 4.1 Update the Operating System

Update the operating system to ensure the latest security updates and bug fixes are installed before configuring the host.

**Procedure**

```bash
sudo dnf update -y
sudo reboot # If required
```

**Verification**

Verify the installed Red Hat Enterprise Linux release and kernel version.

```bash
cat /etc/redhat-release
uname -r
```

### 4.2 Prepare Host Storage

Prepare the storage locations used throughout this guide.

The following directories are used:

| Directory | Purpose |
|-----------|---------|
| `/vm-storage` | Stores the OKD virtual machine disk and installation media. |
| `/archives` | Stores backups of the validated OKD and Runtime Fabric virtual machine disks. |

These directories may reside on dedicated storage devices, separate partitions, or the operating system volume, depending on the available hardware.

**Procedure**

Create the storage directories if they do not already exist.

```bash
sudo mkdir -p \
    /vm-storage \
    /archives \
    /archives/installation-assets \
    /archives/okd \
    /archives/pre-restore \
    /archives/rtf
```

Ensure the directories are owned by the administrative user so they can be used throughout this guide without additional permission changes.

```bash
sudo chown -R "$USER:$USER" \
    /vm-storage \
    /archives
```

If these directories are hosted on separate filesystems, configure those filesystems to mount automatically during system startup.

**Verification**

Verify that both directories exist.

```bash
ls -ld \
    /vm-storage \
    /archives \
    /archives/installation-assets \
    /archives/okd \
    /archives/pre-restore \
    /archives/rtf
```

If the directories are hosted on separate filesystems, verify that they are mounted.

```bash
findmnt /vm-storage
findmnt /archives
```

Verify that sufficient storage is available.

```bash
df -h \
    /vm-storage \
    /archives
```

Confirm that:

- `/vm-storage` exists.
- `/archives` exists.
- If separate filesystems are used, they are mounted.
- Sufficient free space is available for the virtual machine disks and backups.

### 4.3 Install the Virtualization Software

Install the virtualization software required to host the OKD Single Node (SNO) virtual machine.

**Procedure**

```bash
sudo dnf install -y \
  jq \
  qemu-kvm \
  libvirt \
  virt-install \
  virt-manager \
  cockpit \
  cockpit-machines

sudo systemctl enable --now libvirtd
sudo systemctl enable --now cockpit.socket
```

**Verification**

Verify that the virtualization tools are installed and the required services are enabled.

```bash
virsh --version
virt-install --version
systemctl is-enabled libvirtd
systemctl is-enabled cockpit.socket
```

### 4.4 Prepare the Workspace

Create a workspace to store the files used throughout the OKD installation.

**Procedure**

```bash
mkdir -p ~/Work/rtf-on-okd
cd ~/Work/rtf-on-okd

mkdir -p \
  cluster \
  downloads \
  installer \
  manifests \
  scripts
```

The workspace is used to organize installation assets, generated files, and helper scripts. It is independent of the virtual machine storage located under `/vm-storage`.

### 4.5 Create a Network Bridge

Create a NetworkManager bridge to allow the OKD virtual machine to connect directly to the physical network.

The virtualization host must have a stable IP address. This guide uses DHCP with a DHCP reservation, but a static IP address is equally acceptable.

**Procedure**

Create the bridge interface.

```bash
sudo nmcli connection add \
    type bridge \
    ifname br0 \
    con-name br0
```

Configure the bridge to obtain its IP address using DHCP and enable it automatically at boot.

```bash
sudo nmcli connection modify br0 \
    ipv4.method auto \
    ipv6.method ignore \
    connection.autoconnect yes
```

Attach the physical network interface to the bridge.

> [!NOTE]
> Replace `enp4s0` with the name of your physical network interface.

> [!TIP]
> Identify the physical network interface using:
>
> ```bash
> nmcli device status
> ```
>
> Use the interface listed as `ethernet` in the `TYPE` column.

```bash
sudo nmcli connection add \
    type bridge-slave \
    ifname enp4s0 \
    master br0
```

Activate the bridge.

```bash
sudo nmcli connection down enp4s0
sudo nmcli connection up br0
```

**Verification**

Verify that the bridge is active and has obtained an IP address.

```bash
ip addr show br0
```

Confirm the bridge configuration.

```bash
bridge link
```

### 4.6 Download the Installation Tools and Media

Download the tools and installation media required to deploy OKD and install Anypoint Runtime Fabric.

#### Download the OKD Toolchain

Change to the downloads directory.

```bash
cd ~/Work/rtf-on-okd/downloads
```

Set the OKD release to download.

```bash
OKD_VERSION=4.22.0-okd-scos.4
```

Download the OKD installer.

```bash
curl -LO \
  https://github.com/okd-project/okd/releases/download/${OKD_VERSION}/openshift-install-linux-${OKD_VERSION}.tar.gz
```

Download the OpenShift client tools.

```bash
curl -LO \
  https://github.com/okd-project/okd/releases/download/${OKD_VERSION}/openshift-client-linux-${OKD_VERSION}.tar.gz
```

Download the Kubernetes Operator SDK.

```bash
curl -LO \
  https://github.com/operator-framework/operator-sdk/releases/download/v1.40.0/operator-sdk_linux_amd64
```

#### Extract the Archives

Extract the OKD installer and OpenShift client archives.

```bash
tar -xzf openshift-install-linux-${OKD_VERSION}.tar.gz
tar -xzf openshift-client-linux-${OKD_VERSION}.tar.gz
```

#### Install the Binaries

Install the OKD installer and OpenShift client binaries in `/usr/local/bin`.

```bash
sudo install -m 755 openshift-install /usr/local/bin/
sudo install -m 755 oc /usr/local/bin/
sudo install -m 755 kubectl /usr/local/bin/
sudo install -m 755 operator-sdk_linux_amd64 /usr/local/bin/operator-sdk
```

#### Download the Matching SCOS Live ISO

Download the SCOS live ISO that corresponds to the selected OKD release.

```bash
ISO_URL=$(openshift-install coreos print-stream-json \
  | jq -r '.architectures.x86_64.artifacts.metal.formats.iso.disk.location')

curl -LO "$ISO_URL"
```

> [!NOTE]
> The `openshift-install coreos print-stream-json` command returns metadata for the SCOS release associated with the installed `openshift-install` version. The `jq` command extracts the Live ISO download URL.

#### Install `coreos-installer`

Install the `coreos-installer` package used to embed the Ignition configuration into the SCOS live ISO.

```bash
sudo dnf install -y coreos-installer
```

### 4.7 Verify the Installation Environment

Perform a final verification to confirm that the host is ready to create the OKD Single Node OpenShift virtual machine.

**Verification**

```bash
findmnt /vm-storage

systemctl is-active libvirtd
systemctl is-active cockpit.socket

ip addr show br0

jq --version

openshift-install version
oc version --client
kubectl version --client
operator-sdk version

ls -lh "$(basename "$ISO_URL")"

coreos-installer --version
```

Confirm that:

- `/vm-storage` is mounted.
- The `libvirtd` and `cockpit.socket` services are active.
- The `br0` bridge interface is active and has an IP address.
- `jq`, `openshift-install`, `oc`, `kubectl`, and `operator-sdk` are installed and available.
- The SCOS live ISO is available in the downloads directory.
- `coreos-installer` is installed and available.

If all verification steps complete successfully, the host is ready to install OKD.

