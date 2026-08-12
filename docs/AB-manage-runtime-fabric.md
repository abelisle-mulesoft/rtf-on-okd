# Managing Anypoint Runtime Fabric

## Purpose

Document management tasks for the validated OKD Single Node OpenShift and Anypoint Runtime Fabric environment described in this repository.

> [!NOTE]
> The current revision focuses on restoring previously created virtual machine backups. Additional management procedures will be added in future revisions.


## Restore a Virtual Machine Backup

Restore a previously created virtual machine disk backup.

The procedures in this section assume that:

- The `okd-sno` libvirt virtual machine already exists.
- The virtual machine uses `/vm-storage/okd-sno.qcow2` as its disk.
- The verified OKD backup is stored at `/archives/okd/okd-sno-verified.qcow2`.
- The clean Runtime Fabric backup is stored at `/archives/rtf/okd-sno-rtf-clean.qcow2`.

> [!WARNING]
> Restoring a backup replaces the current `/vm-storage/okd-sno.qcow2` virtual machine disk. Preserve the current disk before continuing if it may be needed later.

### Restore the Verified OKD Baseline

Restore the verified OKD baseline that completed the initial certificate rotation and does not include Runtime Fabric.

**Procedure**

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
sudo mv /vm-storage/okd-sno.qcow2 \
    "/archives/pre-restore/okd-sno-$(date +%Y%m%d-%H%M%S).qcow2"
```

Restore the verified OKD backup.

```bash
sudo cp /archives/okd/okd-sno-verified.qcow2 \
    /vm-storage/okd-sno.qcow2
```

Start the virtual machine.

```bash
sudo virsh start okd-sno
```

**Verification**

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

**Procedure**

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
sudo mv /vm-storage/okd-sno.qcow2 \
    "/archives/pre-restore/okd-sno-$(date +%Y%m%d-%H%M%S).qcow2"
```

Restore the clean Runtime Fabric backup.

```bash
sudo cp /archives/rtf/okd-sno-rtf-clean.qcow2 \
    /vm-storage/okd-sno.qcow2
```

Start the virtual machine.

```bash
sudo virsh start okd-sno
```

**Verification**

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
