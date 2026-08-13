# Configuration Reference

## Purpose

Document the configuration used to validate the OKD Single Node OpenShift and Anypoint Runtime Fabric installation described in this repository.

## Validated Host Configuration

The validation was carried out on **Orion**, the workstation which is available in this lab environment, with the host configuration listed below.

| Resource | Validated configuration |
| -------- | ------------------------|
| Processor | AMD Ryzen Threadripper 2950X |
| CPU cores / threads | 16 cores / 32 threads |
| Memory | 128 GB |
| Operating system | Red Hat Enterprise Linux 9.8 |
| Virtualization | KVM / libvirt |
| Primary storage | 1 TB NVMe SSD |
| Virtual machine storage | 1 TB SATA SSD mounted at `/data` |
| Archive storage | 4 TB disk mounted at `/archives` |
| Network | NetworkManager bridge (`br0`) |

Orion offered sufficient compute, memory, and storage capacity to host the OKD Single Node OpenShift virtual machine and the installation environment supporting it, as described in this repository.

> [!IMPORTANT]
> Orion is the workstation used to author and validate the technical documentation. Its hardware specifications reflect the system available for this lab and should not be interpreted as recommended or ideal specifications for running OKD or Anypoint Runtime Fabric.
>
> Similarly, the storage devices and their capacities listed above show the amount of storage available at the time of validation; they do not constitute sizing recommendations. The storage requirements will vary depending on factors such as how virtual machine disks are allocated, how many backup copies are retained, the installation assets, the local container images, and how many environments are maintained.

## Validated Runtime Fabric Virtual Machine Sizing

The installation was validated using the following OKD Single Node OpenShift virtual machine configuration.

| Resource | Validated configuration |
|----------|-------------------------|
| Virtual CPUs | 16 |
| Memory | 64 GB |
| Virtual disk | 500 GB |
| CPU mode | Host passthrough |
| Machine type | Q35 |
| Firmware | UEFI |
| Disk interface | VirtIO |
| Network interface | VirtIO |
| Kubernetes topology | Single node with control-plane, master, and worker roles |

This configuration provided sufficient capacity to:

- Install and operate the OKD Single Node OpenShift cluster.
- Install the Runtime Fabric Operator and Runtime Fabric platform components.
- Register Runtime Fabric with Anypoint Platform.
- Deploy and run a validation Mule application with one replica.
- Run the OKD monitoring, ingress, console, and platform services required by the cluster.

> [!IMPORTANT]
> This configuration represents the environment validated by this guide. It is not a minimum sizing recommendation and does not establish capacity requirements for production workloads.
>
> Additional CPU, memory, and storage may be required depending on the number of Mule applications, replicas, runtime sizes, monitoring requirements, and operational headroom.

The virtual machine is created in `02-install-okd.md` with the following principal settings:

```bash
--memory 65536
--vcpus 16
--cpu host-passthrough
--machine q35
--boot uefi
--disk path=/data/vm/okd-sno.qcow2,size=500,bus=virtio,format=qcow2
--network bridge=br0,model=virtio
```

## Validated Network Configuration

The installation described in this repository was validated using the following network configuration.

| Component | Validated configuration |
|----------|-------------------------|
| Virtual machine connectivity | NetworkManager bridge (`br0`) |
| Host network configuration | DHCP |
| OKD node IP address | DHCP reservation |
| DNS configuration | Inherited from DHCP |
| DNS upstream resolvers | Quad9 (`9.9.9.9`) and Cloudflare (`1.1.1.1`) |
| DNS Operator customization | None |
| Base domain | `nip.io` |

The validated environment relied on standard DHCP-provided network settings. No changes were made to the OKD DNS Operator or the Fedora CoreOS DNS configuration after installation.

> [!IMPORTANT]
> Runtime Fabric requires reliable outbound DNS resolution. In the validated environment, DNS settings were inherited from DHCP, which supplied the external recursive DNS resolvers. No Fedora CoreOS DNS customization or OKD DNS Operator customization was required.

## Validated Software Versions

The procedures in this repository were validated using the following software versions.

| Component | Version |
|-----------|---------|
| Red Hat Enterprise Linux | 9.8 (Release 1) |
| CentOS Stream CoreOS (SCOS) | 10.0.20260614-0 (Coughlan) |
| OKD | 4.22.0-okd-scos.4 |
| Kubernetes | 1.35.5 |
| Runtime Fabric | 3.0.277 |
| Runtime Fabric Operator | 3.0.277 |
| Runtime Fabric Operator Bundle | 3.0.277-1785441385 |
| Operator SDK | 1.40.0 |

> [!IMPORTANT]
> These versions represent the validated software baseline for the procedures in this repository. Newer or older versions of OKD, CentOS Stream CoreOS, Runtime Fabric, Kubernetes, or the supporting tools may require changes to the documented procedures and should be validated before use.
