# Managing Anypoint Runtime Fabric

## Purpose

Document management tasks for the validated OKD Single Node OpenShift and Anypoint Runtime Fabric environment described in this repository.

> [!NOTE]
> This document contains management procedures that have been implemented and validated in the environment described in this repository. Additional management procedures will be added as they are implemented and validated.

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

## Upgrade Runtime Fabric

Upgrade an existing Anypoint Runtime Fabric installation on OKD to a newer Runtime Fabric version.

> [!NOTE]
> OKD is not a MuleSoft-certified or supported platform for Anypoint Runtime Fabric. Because the Red Hat certified Runtime Fabric Operator is not available through the OperatorHub catalog in OKD, this procedure uses `operator-sdk` to remove and install the Runtime Fabric Operator. The procedure follows the general approach described in MuleSoft’s Upgrade to an Intermediate Runtime Fabric on Red Hat OpenShift documentation.

> [!WARNING]
> If the installed Runtime Fabric version is several releases behind the target version, do not assume that a direct upgrade is supported. Review the applicable MuleSoft upgrade guidance and contact MuleSoft Support to confirm the required upgrade path before proceeding.

### Upgrade Requirements

Ensure the following requirements are met before upgrading Runtime Fabric.

- Review the applicable Runtime Fabric release notes.
- Verify that the Kubernetes version used by the OKD Single Node OpenShift cluster is listed as supported for the target Runtime Fabric version.
- Determine the current and target Runtime Fabric versions.
- Confirm that the existing Runtime Fabric is healthy.
- When using a local registry, synchronize the target Runtime Fabric images before modifying the existing Runtime Fabric installation.

> [!IMPORTANT]
> Upgrading Runtime Fabric does not upgrade the Mule runtime versions used by deployed applications. Mule runtime lifecycle management is separate from the Runtime Fabric upgrade process.

### Synchronize the Target Runtime Fabric Images

If Runtime Fabric retrieves images directly from the MuleSoft-hosted registry, skip this section and continue with **Verify the Current Runtime Fabric Installation**.

If Runtime Fabric is configured to use a local registry, synchronize the images required for the target Runtime Fabric version from the MuleSoft-hosted registry to the local registry.

> [!IMPORTANT]
> Commands in this section use credentials and authorization tokens. Be aware that commands containing sensitive values may be retained in shell history. Do not store them or their output in screenshots, shell scripts, documentation, or source-control repositories.

##### Procedure

Get an Anypoint authorization token.

```bash
ANYPOINT_TOKEN=$(
  curl -sS -X POST \
    -H "Content-Type: application/json" \
    -d '{"username":"<username>","password":"<password>"}' \
    "https://anypoint.mulesoft.com/accounts/login" \
  | jq -r '.access_token'
)
```

Verify the token was obtained successfully.

```bash
test -n "$ANYPOINT_TOKEN" \
  && test "$ANYPOINT_TOKEN" != "null" \
  && echo "Authorization token obtained successfully."
```

Set the target version.

```bash
RTF_VERSION=<target-runtime-fabric-version>
```

For example, the validated upgrade uses:

```bash
RTF_VERSION=3.0.277
```

Retrieve the list of OpenShift images required by the target Runtime Fabric version.

```bash
RTF_IMAGE_LIST=$(
  curl -sS \
    "https://anypoint.mulesoft.com/runtimefabric/api/agentmanifests/${RTF_VERSION}" \
    -H "Authorization: bearer ${ANYPOINT_TOKEN}" \
  | jq -r '.dependencies[]
      | select(.provider == "openshift")
      | "\(.artifact):\(.version)"'
)
```

Review the list before continuing.

```bash
printf '%s\n' "$RTF_IMAGE_LIST"
```

Example output:

```text
rtf-core-actions-ubi:1.0.219
dias-anypoint-monitoring-sidecar-ubi:2.2.36
rtf-cluster-ops-ubi:2.0.384
rtf-daemon-ubi:2.0.400
rtf-mule-clusterip-service-ubi:1.4.92
rtf-app-init-ubi:1.0.274
rtf-object-store-ubi:1.0.312
rtf-ubi-base-nginx:0.3.57
rtf-resource-fetcher-ubi:1.0.333
rtf-agent-ubi:3.0.277
```

Sign in to the MuleSoft Runtime Fabric registry.

```bash
podman login \
    rtf-runtime-registry.kprod.msap.io \
    --username '<docker-username>'
```

When prompted, enter the `docker-password` provided on the Runtime Fabric details page in Anypoint Runtime Manager.

Synchronize the images automatically.

```bash
for IMAGE in $RTF_IMAGE_LIST
do
    echo "Processing $IMAGE"
    podman pull rtf-runtime-registry.kprod.msap.io/mulesoft/$IMAGE
    podman tag \
        rtf-runtime-registry.kprod.msap.io/mulesoft/$IMAGE \
        orion.lan:5000/mulesoft/$IMAGE
    podman push orion.lan:5000/mulesoft/$IMAGE
done
```

##### Verification

Verify that each Runtime Fabric image in `RTF_IMAGE_LIST` is available in the local registry.

```bash
for IMAGE in $RTF_IMAGE_LIST
do
    REPOSITORY="${IMAGE%:*}"
    TAG="${IMAGE##*:}"

    printf 'Verifying %s ... ' "$IMAGE"

    if RESPONSE=$(curl -fsS \
        -u "rtf-registry:<registry-password>" \
        "https://orion.lan:5000/v2/mulesoft/${REPOSITORY}/tags/list")
    then
        if jq -e \
            --arg TAG "$TAG" \
            '.tags | index($TAG) != null' \
            <<< "$RESPONSE" \
            > /dev/null
        then
            echo "OK"
        else
            echo "FAILED"
        fi
    else
        echo "FAILED"
    fi
done
```

Confirm that the verification returns OK for all images.

Sign out of the MuleSoft Runtime Fabric registry.

```bash
podman logout rtf-runtime-registry.kprod.msap.io
```

Clear the authorization token.

```bash
unset ANYPOINT_TOKEN
```

### Verify the Current Runtime Fabric Installation

Verify that the current Runtime Fabric installation is healthy before creating the pre-upgrade backup and modifying the installation.

##### Procedure

Verify that the Runtime Fabric custom resource exists.

```bash
oc get runtimefabric \
    --namespace rtf
```

Verify that the Runtime Fabric pods are running and ready.

```bash
oc get pods \
    --namespace rtf
```

Verify the current Runtime Fabric Operator Lifecycle Manager resources.

```bash
oc get subscription,csv,catalogsource,installplan \
    --namespace rtf
```

In Anypoint Runtime Manager, open the Runtime Fabric and review the **Health Details** tab.

In Runtime Manager, select **Applications** and review the applications deployed to the Runtime Fabric.

##### Verification

Confirm that:

- The Runtime Fabric custom resource exists.
- The Runtime Fabric pods are running and all required containers are ready.
- The Runtime Fabric Operator CSV reports `Succeeded`.
- Anypoint Runtime Manager reports the Runtime Fabric as **Active**.
- The health summary reports **All systems operational**.
- All Runtime Fabric health checks report **Healthy**.
- Existing Mule applications deployed to the Runtime Fabric report **Running**.

### Back Up the Current Runtime Fabric Environment

Create a pre-upgrade backup of the verified Runtime Fabric environment before modifying the installation.

#### Shut Down the OKD Virtual Machine

##### Procedure

Shut down the OKD virtual machine.

```bash
sudo virsh shutdown okd-sno
```

##### Verification

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

#### Create the Pre-Upgrade Backup

##### Procedure

Back up the libvirt virtual machine definition.

```bash
sudo virsh dumpxml okd-sno \
    > /archives/rtf/okd-sno-pre-upgrade.xml
```

Back up the virtual machine disk image.

```bash
sudo cp /data/vm/okd-sno.qcow2 \
    /archives/rtf/okd-sno-rtf-pre-upgrade.qcow2
```

##### Verification

Verify that the backups exist.

```bash
ls -lh \
    /archives/rtf/okd-sno-pre-upgrade.xml \
    /archives/rtf/okd-sno-rtf-pre-upgrade.qcow2
```

Verify the size of the virtual machine disk image.

```bash
head -5 /archives/rtf/okd-sno-pre-upgrade.xml

du -h /archives/rtf/okd-sno-rtf-pre-upgrade.qcow2
du -h --apparent-size /archives/rtf/okd-sno-rtf-pre-upgrade.qcow2
```

#### Restart the OKD Virtual Machine

##### Procedure

Start the OKD virtual machine.

```bash
sudo virsh start okd-sno
```

##### Verification

Monitor the node until it reports `Ready`.

```bash
watch -n 15 'oc get nodes'
```

Verify that the Runtime Fabric pods are running and ready.

```bash
oc get pods \
    --namespace rtf
```

Confirm that the node reports `Ready` and that the Runtime Fabric pods are running and all required containers are ready.

### Remove the Current Runtime Fabric Operator

Remove the current Runtime Fabric Operator while preserving the existing Runtime Fabric instance.

> [!IMPORTANT]
> When uninstalling the operator below, ensure that the option **Delete all operand instances for this operator** is not selected. The existing Runtime Fabric instance must be preserved for the upgrade.

##### Procedure

Open the OKD web console URL in a browser and sign in as `kubeadmin`.

In the left navigation menu, expand **Ecosystem** and select **Installed Operators**.

Select the **Runtime Fabric Operator**.

On the **Details** tab, select **Actions**, and then select **Uninstall Operator**.

In the **Uninstall Operator?** dialog, ensure that **Delete all operand instances for this operator** is not selected.

Select **Uninstall**.

Remove the CatalogSource created for the current Runtime Fabric Operator bundle.

```bash
oc delete catalogsource runtime-fabric-operator-catalog \
    --namespace rtf
```

##### Verification

Verify the Operator Lifecycle Manager resources.

```bash
oc get subscription,csv,catalogsource,installplan \
    --namespace rtf
```

Confirm that the Runtime Fabric Operator resources associated with the previous operator installation have been removed.

Verify that the Runtime Fabric custom resource still exists.

```bash
oc get runtimefabric \
    --namespace rtf
```

Confirm that the existing Runtime Fabric custom resource remains present.

### Install the Target Runtime Fabric Operator

Install the target Runtime Fabric Operator bundle using Operator SDK.

> [!NOTE]
> MuleSoft’s OpenShift procedure installs the Runtime Fabric Operator from the Red Hat OperatorHub catalog. Because OKD does not include the Red Hat certified operator catalog, this guide installs the certified operator bundle through the Operator Lifecycle Manager using `operator-sdk`.

##### Procedure

In the [Red Hat Ecosystem Catalog, locate the Runtime Fabric Operator](https://catalog.redhat.com/en/software/containers/rtf/runtime-fabric-operator) and identify the complete Operator bundle image tag corresponding to the target Runtime Fabric version.

Set the target Operator bundle image tag.

```bash
RTF_OPERATOR_BUNDLE_TAG=<target-operator-bundle-image-tag>
```

For example, the validated upgrade uses:

```bash
RTF_OPERATOR_BUNDLE_TAG=3.0.277-1785441385
```

Sign in to the Red Hat registry so that operator-sdk can retrieve the Runtime Fabric Operator bundle.

```bash
podman login \
    registry.connect.redhat.com \
    --username '<red-hat-username>' \
    --password '<red-hat-password>'
```

Install the target Runtime Fabric Operator bundle.

```bash
operator-sdk run bundle \
    registry.connect.redhat.com/rtf/runtime-fabric-operator:${RTF_OPERATOR_BUNDLE_TAG} \
    --namespace rtf \
    --pull-secret-name redhat-connect-pull-secret \
    --timeout 10m
```

The command creates the Operator Lifecycle Manager resources required to install the target operator.

Sign out of the Red Hat registry after the operator installation completes.

```bash
podman logout registry.connect.redhat.com
```

##### Verification

Verify that the Runtime Fabric Operator ClusterServiceVersion (CSV) reports `Succeeded`.

```bash
oc get csv \
    --namespace rtf
```

Example output:

```text
NAME                                      DISPLAY                   VERSION   REPLACES   PHASE
runtime-fabric-operator.v3.0.277          Runtime Fabric Operator   3.0.277              Succeeded
```

Verify that the operator pod is running.

```bash
oc get pods \
    --namespace rtf
```

Confirm that the Runtime Fabric Operator pod reports `Running` and all its containers are ready.

### Monitor the Runtime Fabric Upgrade

Monitor the existing Runtime Fabric instance while the target Runtime Fabric Operator reconciles it to the target version.

##### Procedure

Monitor the Runtime Fabric pods while the upgrade is in progress.

```bash
watch -n 5 'oc get pods --namespace rtf'
```

During the upgrade, Runtime Fabric platform pods can be recreated as the target Operator reconciles the existing Runtime Fabric instance.

Continue monitoring until the Runtime Fabric pods are running and all required containers are ready, and then press `Ctrl+C`.

Monitor the Runtime Fabric custom resource.

```bash
oc get runtimefabric \
    --namespace rtf
```

In Anypoint Runtime Manager, on the **Runtime Fabrics** page, monitor Runtime Fabric status and version.

##### Verification

Confirm that:

- The Runtime Fabric pods are running and all required containers are ready.
- The Runtime Fabric custom resource remains present.
- Anypoint Runtime Manager reports the Runtime Fabric as **Active**.
- The Runtime Fabric version reports the target Runtime Fabric version.

Do not continue until the Runtime Fabric reports **Active**, reports the target Runtime Fabric version, and the Runtime Fabric platform pods are running and ready.

### Validate the Upgraded Runtime Fabric

Validate that the upgraded Runtime Fabric is healthy and that existing Mule applications remain operational after the upgrade.

##### Procedure

Verify the Runtime Fabric Operator Lifecycle Manager resources.

```bash
oc get subscription,csv,catalogsource,installplan \
    --namespace rtf
```

Verify that the Runtime Fabric pods are running and ready.

```bash
oc get pods \
    --namespace rtf
```

In Anypoint Runtime Manager, open the Runtime Fabric, review the **Health Details** tab, and confirm that:

- The health summary reports **All systems operational**.
- The following health checks report **Healthy**:
  - Create/Manage Application Deployments
  - Forwarding Logs To Anypoint Monitoring
  - Nodes

In Runtime Manager, select **Applications** and review the applications deployed to the Runtime Fabric.

##### Verification

Confirm that:

- The target Runtime Fabric Operator CSV reports `Succeeded`.
- The Runtime Fabric pods are running and all required containers are ready.
- Anypoint Runtime Manager reports the Runtime Fabric as **Active**.
- The Runtime Fabric version reports the target Runtime Fabric version.
- The health summary reports **All systems operational**.
- All Runtime Fabric health checks report **Healthy**.
- Existing Mule applications deployed to the Runtime Fabric report **Running**.

The Runtime Fabric upgrade is complete when all validation steps complete successfully.

## Manage Mule Runtime Versions

Manage the different versions of the Mule runtime and the corresponding Anypoint Monitoring sidecar images used by applications deployed to Runtime Fabric when a local container registry is used.

> [!IMPORTANT]
> Updating the Mule runtime versions available for deploying applications to Runtime Fabric is separate from upgrading Runtime Fabric. When Runtime Fabric is upgraded, the platform components are updated, but the available Mule runtime versions remain unchanged.

### Understand Mule Runtime Release Channels

MuleSoft provides Mule runtime releases through two release channels: **Edge** and **Long-Term Support (LTS)**. Both release channels are available for applications deployed to Runtime Fabric.

Edge releases provide access to new Mule runtime features on a more frequent release cadence and have a shorter support period. MuleSoft releases Edge versions up to three times per year.

LTS releases provide a longer support period and incorporate capabilities introduced through previous Edge releases. MuleSoft schedules LTS releases periodically in February.

When an LTS release occurs, the Edge and LTS releases can share the same Mule runtime minor version. The release channel is identified by the runtime build tag. For example:

```text
4.9.0:1e    Edge
4.9.0:1     LTS
```

The `e` suffix identifies an Edge build. An LTS build does not include the `e` suffix.

Mule runtime minor versions do not necessarily progress sequentially within the LTS channel. Edge releases can introduce intervening minor versions before the next LTS release. For example, Mule 4.6 LTS was followed by Mule 4.9 LTS, while Mule 4.7 and Mule 4.8 were Edge releases.

Organizations can choose Edge, LTS, or a combination of both release channels according to their own lifecycle, feature adoption, and support requirements.

For the current release cadence, support periods, and available Mule runtime versions, refer to MuleSoft’s [Edge and LTS Releases for Mule](https://docs.mulesoft.com/release-notes/mule-runtime/lts-edge-release-cadence) documentation.

### Identify Available Mule Runtime Versions

Organizations can synchronize any combination of supported Edge and LTS Mule runtime versions according to their own lifecycle and application requirements. They can also retain multiple Mule runtime versions in the local registry simultaneously. For the purpose of this guide, the validated environment maintains the current LTS release and the two most recent Edge releases. At the time this procedure was validated, this resulted in the following Mule runtime selections:

| Release channel | Mule runtime | Note                  |
| --------------- | ------------ | --------------------- |
| LTS             | 4.9          | Current LTS release   |
| Edge            | 4.11         | Previous Edge release |
| Edge            | 4.12         | Current Edge release  |

> [!NOTE]
> The Mule runtime versions shown above represent the versions selected to validate this guide. They are not recommendations for which Mule runtime versions an organization should maintain. Review the current MuleSoft release notes and select the versions appropriate for your environment.

As discussed in section **3.2 Mule Runtime Images** of `AC-configure-local-rtf-registry.md`, MuleSoft publishes the Mule runtime versions available for Runtime Fabric in the [Mule Runtime Patch Update Release Notes for Mule Apps on Runtime Fabric](https://docs.mulesoft.com/release-notes/runtime-fabric/runtime-fabric-runtimes-release-notes). The release notes identify the available Mule runtime versions, release channels, image builds, and supported Java versions. Runtime packages use the following format:

```text
<mule-version>:<image-build>-<java-version>
```

For example:

```text
4.12.2:8e-java17
```

As stated previously, the `e` suffix in the image build identifies an Edge release. LTS releases do not include the `e` suffix.

At the time this procedure was validated, the selected Mule runtime versions resulted in the following Mule runtime and Anypoint Monitoring sidecar images to synchronize. Only the latest patch and image build from each selected release line is synchronized.

| Mule runtime image                | Sidecar image |
| --------------------------------- | ------------- |
| poseidon-runtime-4.9.20:9-java17  | dias-anypoint-monitoring-sidecar-ubi:2.2.41 |
| poseidon-runtime-4.11.6:7e-java17 | dias-anypoint-monitoring-sidecar-ubi:2.2.33 |
| poseidon-runtime-4.12.2:8e-java17 | dias-anypoint-monitoring-sidecar-ubi:2.2.41 |

### Synchronize Mule Runtime Images

Synchronize the Mule runtime and OpenShift-compatible Anypoint Monitoring sidecar images required to deploy Mule applications to Runtime Fabric.

Synchronization is additive. Adding a Mule runtime version to the local registry does not replace or remove previously synchronized Mule runtime or Anypoint Monitoring sidecar images.

> [!IMPORTANT]
> Do not delete older Mule runtime or Anypoint Monitoring sidecar images from the local registry solely because newer versions have been synchronized; existing applications may depend on those images. Remove an image from the local registry only after you have confirmed that no deployed application still needs it.

##### Procedure

Sign in to the local registry.

```bash
podman login \
    orion.lan:5000 \
    --username rtf-registry
```

When prompted, enter the local registry password.

Sign in to the MuleSoft Runtime Fabric registry.

```bash
podman login \
    rtf-runtime-registry.kprod.msap.io \
    --username '<docker-username>'
```

When prompted, enter the MuleSoft Runtime Fabric registry password — that is, the `docker-password` provided on the Runtime Fabric details page in Anypoint Runtime Manager when the Runtime Fabric was created.

Define the list of images to synchronize.

```bash
MULE_IMAGE_LIST='poseidon-runtime-4.9.20:9-java17
poseidon-runtime-4.11.6:7e-java17
poseidon-runtime-4.12.2:8e-java17
dias-anypoint-monitoring-sidecar-ubi:2.2.33
dias-anypoint-monitoring-sidecar-ubi:2.2.41'
```

Review the list before continuing.

```bash
printf '%s\n' "$MULE_IMAGE_LIST"
```

Synchronize the images.

```bash
for IMAGE in $MULE_IMAGE_LIST
do
    echo "Processing $IMAGE"
    podman pull rtf-runtime-registry.kprod.msap.io/mulesoft/$IMAGE
    podman tag \
        rtf-runtime-registry.kprod.msap.io/mulesoft/$IMAGE \
        orion.lan:5000/mulesoft/$IMAGE
    podman push orion.lan:5000/mulesoft/$IMAGE
done
```

##### Verification

Verify that the Mule runtime and OpenShift-compatible Anypoint Monitoring sidecar images are available in the local registry.

```bash
for IMAGE in $MULE_IMAGE_LIST
do
    REPOSITORY="${IMAGE%:*}"
    TAG="${IMAGE##*:}"

    printf 'Verifying %s ... ' "$IMAGE"

    if RESPONSE=$(curl -fsS \
        -u "rtf-registry:<registry-password>" \
        "https://orion.lan:5000/v2/mulesoft/${REPOSITORY}/tags/list")
    then
        if jq -e \
            --arg TAG "$TAG" \
            '.tags | index($TAG) != null' \
            <<< "$RESPONSE" \
            > /dev/null
        then
            echo "OK"
        else
            echo "FAILED"
        fi
    else
        echo "FAILED"
    fi
done
```

Confirm that the verification returns OK for all images.

Sign out of the local registry.

```bash
podman logout orion.lan:5000
```

Sign out of the MuleSoft Runtime Fabric registry.

```bash
podman logout rtf-runtime-registry.kprod.msap.io
```

---

Copyright © 2026 Alan Belisle. Licensed under the [MIT License](../LICENSE).
