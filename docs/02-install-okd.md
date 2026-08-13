# Installing OKD

## 1. Purpose

Install an OKD Single Node OpenShift (SNO) cluster as a virtual machine on the prepared Red Hat Enterprise Linux host.

## 2. Installation Requirements

Before proceeding, ensure the following requirements have been met:

- The host has been prepared as described in **Preparing the Host for OKD**.
- The OKD installation tools have been downloaded and installed.
- Internet connectivity is available to download the required container images during the installation.

> [!IMPORTANT]
> The installation creates an OKD Single Node OpenShift (SNO) cluster as a virtual machine using KVM and libvirt. Ensure that the virtualization services are running before proceeding.

## 3. Create the Installation Configuration

Perform the following tasks to create the installation configuration.

### 3.1 Create an SSH Key Pair

If you do not already have an SSH key pair, create one.

**Procedure**

```bash
ssh-keygen -t ed25519
```

> [!TIP]
> When prompted:
>
> - Accept the default location for the SSH key.
> - Leave the passphrase empty.

This public key (`~/.ssh/id_ed25519.pub`) will be referenced in `install-config.yaml`.

### 3.2 Create `install-config.yaml`

Create the `install-config.yaml` installation configuration file.

```bash
cd ~/Work/rtf-on-okd/cluster

SSH_KEY=$(cat ~/.ssh/id_ed25519.pub)

cat > install-config.yaml <<EOF
apiVersion: v1
baseDomain: 172.20.200.240.nip.io

metadata:
  name: okd

compute:
- name: worker
  replicas: 0

controlPlane:
  name: master
  replicas: 1

networking:
  networkType: OVNKubernetes
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  machineNetwork:
  - cidr: 172.20.200.0/24
  serviceNetwork:
  - 172.30.0.0/16

platform:
  none: {}

bootstrapInPlace:
  installationDisk: /dev/vda

pullSecret: '{"auths":{"fake":{"auth":"aWQ6cGFzcwo="}}}'

sshKey: |
  ${SSH_KEY}
EOF
```

#### Back Up `install-config.yaml`

Create a backup copy because the installer removes the original `install-config.yaml` file after generating the installation assets.

```bash
cp install-config.yaml install-config.yaml.backup
```

> [!NOTE]
> The `install-config.yaml` file above uses the following values throughout this guide:
>
> 1. The node IP address is `172.20.200.240`.
> 2. The cluster name is `okd`.
> 3. The base domain is `nip.io`. The API endpoint will be `api.okd.172-20-200-240.nip.io`.
> 4. The placeholder pull secret is sufficient for this installation because OKD does not require a Red Hat pull secret.

### 3.3 Generate the Ignition Configuration

Generate the bootstrap-in-place Ignition configuration.

```bash
openshift-install create single-node-ignition-config
```

The installer creates the Ignition configuration in the current directory and removes the original `install-config.yaml` file.

### 3.4 Configure the DHCP Reservation

Configure a DHCP reservation before starting the installation to ensure the OKD node always receives the same IP address.

This guide assumes the following reservation.

| Setting | Value |
|---------|-------|
| Hostname | okd-sno |
| IP Address | 172.20.200.240 |
| MAC Address | 52:54:00:8a:b3:1c |

> [!NOTE]
> If your DHCP server requires a device to obtain a lease before creating a reservation, create and start the virtual machine in the next section, wait for it to obtain an IP address, create the reservation, power off the virtual machine, and then continue with the installation.

## 4. Provision the OKD Virtual Machine

### 4.1 Prepare the Installation Media

Prepare the installation media used to boot the OKD virtual machine.

#### Copy the Installation ISO

Copy the downloaded SCOS live ISO to the virtual machine storage. The original downloaded ISO is preserved for future use.

```bash
cd ~/Work/rtf-on-okd

sudo cp "downloads/$(basename "$ISO_URL")" \
    /data/vm/okd-sno-install.iso
```

#### Embed the Ignition Configuration

Embed the bootstrap-in-place Ignition configuration into the installation ISO.

```bash
sudo coreos-installer iso ignition embed \
    -i cluster/bootstrap-in-place-for-live-iso.ign \
    /data/vm/okd-sno-install.iso
```

**Verification**

Verify that the installation ISO contains the embedded Ignition configuration.

```bash
sudo coreos-installer iso ignition show \
    /data/vm/okd-sno-install.iso |
    jq '{
        ignitionVersion: .ignition.version,
        files: (.storage.files | length),
        systemdUnits: (.systemd.units | length),
        users: (.passwd.users | length)
    }'
```

Example output:

```json
{
  "ignitionVersion": "3.2.0",
  "files": 113,
  "systemdUnits": 9,
  "users": 1
}
```

### 4.2 Create the Virtual Machine

Create the OKD virtual machine.

```bash
sudo virt-install \
    --name okd-sno \
    --memory 65536 \
    --vcpus 16 \
    --cpu host-passthrough \
    --machine q35 \
    --boot uefi \
    --disk path=/data/vm/okd-sno.qcow2,size=500,bus=virtio,format=qcow2 \
    --cdrom /data/vm/okd-sno-install.iso \
    --network bridge=br0,model=virtio,mac=52:54:00:8a:b3:1c \
    --graphics none \
    --console pty,target_type=serial \
    --osinfo detect=on,name=fedora-coreos-stable \
    --noautoconsole
```

> [!NOTE]
> Creating the virtual machine automatically boots the installation ISO and installs Fedora CoreOS. This process typically takes several minutes.

## 5. Install Fedora CoreOS

### 5.1 Wait for the Fedora CoreOS Installation to Complete

The virtual machine automatically boots from the installation ISO and installs Fedora CoreOS. The installation typically takes several minutes.

Monitor the virtual machine state as it powers off automatically when the installation completes.

```bash
watch -n 5 'sudo virsh list --all'
```

Example output:

```text
 Id   Name      State
--------------------------
 -    okd-sno   shut off
```

Press `Ctrl+C` to stop monitoring.

### 5.2 Start the Installed System

Start the virtual machine to boot the installed Fedora CoreOS system. During the first boot, Ignition applies the embedded configuration and initializes the OKD Single Node OpenShift cluster.

```bash
sudo virsh start okd-sno
```

## 6. Complete the OKD Installation

During the first boot, OKD initializes the cluster, starts the control plane services, and deploys the cluster operators. This process typically takes about 20-30 minutes, depending on the host system and environment.

### 6.1 Wait for the Node to Become Ready

Configure the OpenShift client to access the cluster.

```bash
export KUBECONFIG=~/Work/rtf-on-okd/cluster/auth/kubeconfig
```

Monitor the node until it reports `Ready` and list the `control-plane`, `master`, and `worker` roles.

```bash
watch -n 15 'oc get nodes'
```

Example output:

```text
NAME      STATUS   ROLES                         AGE
okd-sno   Ready    control-plane,master,worker
```

> [!NOTE]
> A `Ready` node indicates that the cluster is operational, but the OpenShift installation is not yet complete. Cluster operators continue to initialize after the node becomes ready.

Press `Ctrl+C` to stop monitoring.

### 6.2 Wait for the OpenShift Installation to Complete

In a separate terminal, wait for the OpenShift installation to complete.

```bash
openshift-install \
    --dir=cluster \
    wait-for install-complete \
    --log-level=info
```

Continue when the command reports:

```text
INFO Install complete!
```

### 6.3 Allow the Initial Certificate Rotation to Complete

Keep the OKD virtual machine running for at least 24 hours after the installer reports `INFO Install complete!` to allow the initial certificates to rotate.

Shutting down the cluster before the initial certificate rotation completes can result in pending certificate signing requests and certificate-related failures after the cluster is restarted.

> [!IMPORTANT]
> Do not shut down or back up the OKD virtual machine until at least 24 hours have elapsed and all verification steps in this section complete successfully.

**Verification**

Verify that the node and cluster operators remain healthy.

```bash
oc get nodes
oc get clusteroperators
```

Verify that no certificate signing requests remain pending.

```bash
oc get csr
```

Confirm that no requests show `Pending` in the `CONDITION` column.

Verify that the OpenShift Route API is available.

```bash
oc get routes.route.openshift.io \
    --namespace openshift-console
```

Verify that the API server can retrieve logs from the node.

```bash
oc logs \
    --namespace openshift-ingress \
    deployment/router-default \
    --tail=1
```

Display the OKD web console URL.

```bash
oc whoami --show-console
```
Open the returned URL and confirm that the OKD web console is accessible.

Do not continue until all verification steps complete successfully.

### 6.4 Shut Down the Virtual Machine

After the initial certificate rotation and verification steps complete successfully, shut down the OKD virtual machine.

```bash
sudo virsh shutdown okd-sno
```

Monitor the virtual machine state as it powers off.

```bash
watch -n 5 'sudo virsh list --all'
```

Example output:

```text
 Id   Name      State
--------------------------
 -    okd-sno   shut off
```

Press `Ctrl+C` to stop monitoring.

### 6.5 Back Up the Verified OKD Installation

Create a backup of the verified OKD installation after the initial certificate rotation has completed.

**Procedure**

Back up the libvirt virtual machine definition.

```bash
sudo virsh dumpxml okd-sno \
    > /archives/okd/okd-sno.xml
```

Back up the OKD virtual machine disk image.

```bash
sudo cp /data/vm/okd-sno.qcow2 \
    /archives/okd/okd-sno-verified.qcow2
```

> [!NOTE]
> This backup creates an independent copy of the virtual machine disk that can be restored at any time without relying on libvirt snapshot metadata.

**Verification**

Verify that the backups exist.

```bash
ls -lh /archives/okd/okd-sno.xml \
    /archives/okd/okd-sno-verified.qcow2
```

Verify that the virtual machine definition was exported successfully and display the size of the virtual machine disk image.

```bash
head -5 /archives/okd/okd-sno.xml

du -h /archives/okd/okd-sno-verified.qcow2
du -h --apparent-size /archives/okd/okd-sno-verified.qcow2
```

Together, these files preserve the verified OKD Single Node OpenShift cluster, including both the libvirt virtual machine definition and the virtual machine disk image after the initial certificate rotation. This backup serves as the validated baseline for future Runtime Fabric installations and testing.
