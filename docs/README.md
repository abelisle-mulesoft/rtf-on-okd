# Documentation

This directory contains the technical documentation for provisioning, validating, and managing an OKD Single Node OpenShift (SNO) environment running MuleSoft Anypoint Runtime Fabric. The documentation is organized as a collection of focused guides. Each document covers a specific task or reference topic to keep the procedures concise, maintainable, and easy to validate.

## Installation Guides

| Document | Description |
|----------|-------------|
| `01-prepare-host.md` | Prepare a Red Hat Enterprise Linux host for installing OKD. |
| `02-install-okd.md` | Install and validate an OKD Single Node OpenShift cluster. |
| `03-install-rtf.md` | Install, validate, and back up MuleSoft Anypoint Runtime Fabric. |

These documents should be completed in order.

## Supporting Documentation

| Document | Description |
|----------|-------------|
| `AA-configuration-reference.md` | Validated environment configuration and software versions. |
| `AB-manage-runtime-fabric.md` | Runtime Fabric management procedures. |
| `AC-configure-local-rtf-registry.md` | Configure a local image registry for Runtime Fabric. |

These documents provide reference information, configuration guidance, and management procedures for the validated environment. They can be consulted independently after completing the installation.

## Documentation Principles

The documentation in this directory follows a consistent set of engineering principles:

- Document one validated approach only.
- Validate every documented command on a clean environment before publication.
- Keep procedures concise and task-oriented.
- Prefer commands over concepts.
- Reuse previously established configuration whenever possible.
- Avoid optional paths and troubleshooting unless required.

## Validation

Documentation is added to this repository only after the procedures have been successfully implemented and validated.

Validated configuration details, including virtual machine sizing and software versions, are documented in `AA-configuration-reference.md`.

## Supportability

> [!WARNING]
> The Runtime Fabric installation documented in this repository uses OKD for learning, experimentation, and proof-of-concept purposes. OKD is not a MuleSoft-certified or supported Runtime Fabric platform.

For supported Runtime Fabric deployments, use a MuleSoft-supported Kubernetes distribution and version.

---

Copyright © 2026 Alan Belisle. Licensed under the [MIT License](../LICENSE).
