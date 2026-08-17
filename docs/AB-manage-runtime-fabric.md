# Managing Anypoint Runtime Fabric

## Purpose

Document management tasks for the validated OKD Single Node OpenShift and Anypoint Runtime Fabric environment described in this repository.

> [!NOTE]
> The current revision focuses on restoring previously created virtual machine backups. Additional management procedures will be added in future revisions.


## Restore a Virtual Machine Backup

Restore a previously created virtual machine disk backup.

The procedures in this section assume that:

- The `okd-sno` libvirt virtual machine already exists.
- The virtual machine uses `/data/vm/okd-sno.qcow2` as its disk.
- The verified OKD backup is stored at `/archives/okd/okd-sno-verified.qcow2`.
- The clean Runtime Fabric backup is stored at `/archives/rtf/okd-sno-rtf-clean.qcow2`.

> [!WARNING]
> Restoring a backup replaces the current `/data/vm/okd-sno.qcow2` virtual machine disk. Preserve the current disk before continuing if it may be needed later.

### Restore the Verified OKD Baseline

Restore the verified OKD baseline that completed the initial certificate rotation and does not include Runtime Fabric.

##### Procedure

Shut down the OKD virtual machine.

```bash
sudo virsh shutdown okd-sno
```

Monitor the virtual machine state.

```bash
watch -n 5 'sudo virsh list --all'
```

Continue when the virtual machine reports `shut off`, and then press `Ctrl+C`.

Preserve the current virtual machine disk in `/archives/pre-restore` before restoring a backup. This temporary safety-net copy allows the current state to be recovered if the restored baseline is not the desired one.

```bash
sudo mv /data/vm/okd-sno.qcow2 \
    "/archives/pre-restore/okd-sno-$(date +%Y%m%d-%H%M%S).qcow2"
```

Restore the verified OKD backup.

```bash
sudo cp /archives/okd/okd-sno-verified.qcow2 \
    /data/vm/okd-sno.qcow2
```

Start the virtual machine.

```bash
sudo virsh start okd-sno
```

##### Verification

Configure the OpenShift client.

```bash
export KUBECONFIG=~/Work/rtf-on-okd/cluster/auth/kubeconfig
```

Monitor the node until it reports `Ready`.

```bash
watch -n 15 'oc get nodes'
```

Continue when the node reports `Ready`, and then press `Ctrl+C`.

Verify that all cluster operators are available and are not progressing or degraded.

```bash
oc get clusteroperators
```

Confirm that:

- The node reports `Ready`.
- All cluster operators show `True` in the `AVAILABLE` column.
- All cluster operators show `False` in the `PROGRESSING` column.
- All cluster operators show `False` in the `DEGRADED` column.

### Restore the Clean Runtime Fabric Baseline

Restore the clean Runtime Fabric baseline that contains a validated Runtime Fabric installation with no Mule applications deployed.

##### Procedure

Shut down the OKD virtual machine.

```bash
sudo virsh shutdown okd-sno
```

Monitor the virtual machine state.

```bash
watch -n 5 'sudo virsh list --all'
```

Continue when the virtual machine reports `shut off`, and then press `Ctrl+C`.

Preserve the current virtual machine disk in `/archives/pre-restore` before restoring a backup. This temporary safety-net copy allows the current state to be recovered if the restored baseline is not the desired one.

```bash
sudo mv /data/vm/okd-sno.qcow2 \
    "/archives/pre-restore/okd-sno-$(date +%Y%m%d-%H%M%S).qcow2"
```

Restore the clean Runtime Fabric backup.

```bash
sudo cp /archives/rtf/okd-sno-rtf-clean.qcow2 \
    /data/vm/okd-sno.qcow2
```

Start the virtual machine.

```bash
sudo virsh start okd-sno
```

##### Verification

Configure the OpenShift client.

```bash
export KUBECONFIG=~/Work/rtf-on-okd/cluster/auth/kubeconfig
```

Monitor the node until it reports `Ready`.

```bash
watch -n 15 'oc get nodes'
```

Continue when the node reports `Ready`, and then press `Ctrl+C`.

Verify that all cluster operators are available and are not progressing or degraded.

```bash
oc get clusteroperators
```

Verify that the Runtime Fabric pods are running and ready.

```bash
oc get pods \
    --namespace rtf
```

In Runtime Manager, confirm that:

- `orion-okd-rtf` reports **Active**.
- The health summary reports **All systems operational**.
- All Runtime Fabric health checks report **Healthy**.
- No validation Mule application is deployed.

## Test Runtime Fabric Outbound Connectivity

Use `rtfctl` to verify that Runtime Fabric can reach the Anypoint control plane and external services required for operation.

### Install `rtfctl`

##### Procedure

Change to the downloads directory.

```bash
cd ~/Work/rtf-on-okd/downloads
```

Download the `rtfctl` command-line tool.

```bash
curl -L https://anypoint.mulesoft.com/runtimefabric/api/download/rtfctl/latest -o rtfctl
```

Install the `rtfctl` command-line tool.

```bash
sudo install -m 755 rtfctl /usr/local/bin/
```

##### Verification

Configure the OpenShift client to access the cluster.

```bash
export KUBECONFIG=~/Work/rtf-on-okd/cluster/auth/kubeconfig
```

Verify that `rtfctl` is installed.

```bash
rtfctl version
```

Example output:

```text
COMPONENT    VERSION
rtfctl       1.0.150
agent        3.0.277
kubernetes   1.35.5
```

### Test Outbound Network Connectivity

##### Procedure

Test Runtime Fabric connectivity to the Anypoint control plane.

```bash
rtfctl test outbound-network
```

##### Verification

Confirm that all required endpoints report successful connectivity and that the command completes with:

```text
Outbound network checks successful
Kubernetes test outbound-network successful
```

<!-- ## Update the Mule License -->
<!-- ### Install Helm -->
<!-- ### Update the Mule License -->

<!-- ## Upgrade Runtime Fabric -->

---

Copyright © 2026 Alan Belisle. Licensed under the [MIT License](../LICENSE).
