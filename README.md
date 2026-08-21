# Runtime Fabric on OKD

This repository documents the provisioning of reusable environments for installing and exploring MuleSoft Anypoint Runtime Fabric. Every documented platform, configuration, and procedure has been successfully implemented and validated before being added to the repository.

> [!WARNING]
> **Supportability Notice**
>
> This repository demonstrates how to install Anypoint Runtime Fabric on an OKD Single Node OpenShift cluster for learning, experimentation, and proof-of-concept purposes.
>
> OKD is **not** a MuleSoft-certified or supported platform for Anypoint Runtime Fabric. MuleSoft officially supports Runtime Fabric only on supported Kubernetes distributions, including supported versions of Red Hat OpenShift Container Platform.
>
> Because OKD is not a supported Runtime Fabric platform, issues encountered while running Runtime Fabric on OKD are outside the scope of MuleSoft Support. This repository is intended solely for learning, experimentation, and proof-of-concept use.

## Philosophy

This repository documents working implementations rather than theoretical designs. New deployment platforms, configurations, and procedures are added only after they have been successfully implemented and validated. The goal is to build a practical engineering reference that evolves through real-world experience.

## Goals

- Build a repeatable OKD Single Node environment.
- Install MuleSoft Runtime Fabric.
- Deploy sample Mule applications.
- Capture validated procedures and reference configurations.

## Documentation

All repository documentation is maintained in the `docs` directory. The documentation is organized as a collection of focused installation guides, reference material, and management documentation. Each document covers a specific task or topic to keep the procedures concise, maintainable, and easy to validate.

## Repository Structure

`docs/`
: Installation guides, reference material, and management documentation.

`assets/`
: Images, diagrams, screenshots, and other documentation assets.

<!--
`docs/references/`
    Configuration references and environment baselines
-->

<!--
TODO: Determine if this folder is required
`notes/`
    Engineering notes and lessons learned
-->

## Current Scope

The current release documents the provisioning of an OKD Single Node (SNO) cluster and the installation of MuleSoft Anypoint Runtime Fabric on a Linux host using KVM/libvirt. The reference implementation was validated on a physical Linux workstation using KVM/libvirt. Future releases will document equivalent deployments on cloud-hosted virtual machines, such as Azure and AWS.

## Roadmap

### Release 1 (`v1.0.0`)

- Provision an OKD Single Node cluster on a Linux host using KVM/libvirt.
- Install MuleSoft Runtime Fabric.
- Validate the installation by deploying a Mule application.

### Release 2 (`v1.1.0`)

- Add support for using a local image registry with Runtime Fabric.
- Add procedures for synchronizing Runtime Fabric and Mule runtime images to the local registry.
- Add Runtime Fabric outbound connectivity testing and related management guidance.
- Incorporate installation procedure improvements identified while validating the local registry configuration.

### Release 3 (`v1.2.0`)

- Add a validated procedure for upgrading Runtime Fabric on OKD.
- Add support for upgrading Runtime Fabric when using a local image registry.
- Add pre-upgrade backup, health verification, upgrade monitoring, and post-upgrade validation procedures.
- Add Kubernetes version compatibility checks for Runtime Fabric installation and upgrades.

### Release 4 (`v1.3.0`)

* Add guidance for understanding Mule runtime Edge and LTS release channels.
* Add procedures for identifying and synchronizing selected Mule runtime and Anypoint Monitoring sidecar images when using a local image registry.

### Future Enhancements

Additional procedures and deployment scenarios will be documented as they are implemented and validated. Planned enhancements include additional Runtime Fabric management and lifecycle procedures and deployment on additional platforms, including Azure and AWS.

## About This Repository

The procedures and configurations in this repository are based on environments that have been successfully implemented and validated. As the repository evolves, additional deployment platforms and scenarios will be added only after they have been validated. This repository is intended as an engineering reference and learning resource rather than official product documentation.

---

Copyright © 2026 Alan Belisle. Licensed under the [MIT License](LICENSE).
