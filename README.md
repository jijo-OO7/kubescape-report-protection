# Kubescape Report Protection

> **A personal engineering case study documenting my upstream contributions to Kubescape's report anonymization, reversible transformation, encryption, and decryption workflow.**

[![Kubescape](https://img.shields.io/badge/Kubescape-v4.0.11-blue)](https://github.com/kubescape/kubescape/releases/tag/v4.0.11)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Security-326CE5)](https://kubernetes.io/)
[![Go](https://img.shields.io/badge/Go-1.x-00ADD8)](https://go.dev/)
[![License](https://img.shields.io/badge/License-MIT-green)](LICENSE)

---

## Overview

Kubernetes security reports can contain significantly more information than the security findings themselves.

Resource names, namespaces, container images, service accounts, repository paths, Git metadata, configuration references, and other report fields can expose information about an organization's internal infrastructure.

The original Kubescape feature request, [#1200 — Anonimization of Kubescape results](https://github.com/kubescape/kubescape/issues/1200), identified this problem:

> Kubescape reports can contain sensitive information such as namespace and workload names, and users may need to mask those values before sharing reports.

I picked up this long-running feature in May 2026 and developed it incrementally from a basic `--hide` mechanism into a broader **report-protection workflow**.

The work eventually covered:

- Deterministic resource anonymization
- Container name and image anonymization
- Secret and ConfigMap reference protection
- Annotation protection
- Service-account anonymization
- Repository and Git metadata protection
- Local filesystem path protection
- Transformation abstractions
- Reversible metadata transformation
- AES-GCM encryption
- Data Encryption Key (DEK) generation and wrapping
- Report decryption
- Resource metadata encryption/decryption
- Container metadata encryption/decryption
- CLI documentation and examples

The completed work shipped as part of **Kubescape v4.0.11**.

---

# The Problem

A Kubernetes security report can contain identifiers such as:

```text
namespace: production-internal
deployment: payments-api
image: registry.company.internal/payments/api
serviceAccount: payments-prod
repository: /home/developer/company-internal/project
```

The security finding itself may be safe to share, while the metadata surrounding it may reveal:

- Internal application names
- Infrastructure topology
- Private registries
- Environment naming conventions
- Repository structure
- Developer filesystem paths
- Workload identities
- Configuration relationships
- Git repository information

The challenge therefore wasn't simply:

> **"Remove names from the report."**

The real engineering problem was:

> **How can sensitive information be transformed consistently without breaking the structure, relationships, or usefulness of the security report?**

The original issue explicitly called for hiding sensitive fields such as namespace and workload names while generating a mapping that could later be used to decipher them.

**Original feature request:**  
[#1200 — Anonimization of Kubescape results](https://github.com/kubescape/kubescape/issues/1200)

---

# Engineering Journey

The feature evolved through several stages:

```text
Feature request
      │
      ▼
--hide pipeline
      │
      ▼
Resource identifier anonymization
      │
      ▼
Container names & images
      │
      ▼
Container configuration references
      │
      ▼
Annotations / source metadata
      │
      ▼
Service accounts / Git / filesystem paths
      │
      ▼
Transformer abstraction
      │
      ▼
Reversible transformations
      │
      ▼
AES-GCM encryption
      │
      ▼
DEK wrapping
      │
      ▼
Report decryption
      │
      ▼
CLI documentation
      │
      ▼
Kubescape v4.0.11
```

---

# Phase 1 — Establishing the Anonymization Pipeline

The first step was creating a proper anonymization boundary in the scan pipeline.

The `--hide` flag was introduced and connected to the scan flow so that anonymization could happen after report information had been collected.

### Initial flow

```text
kubescape scan --hide
        │
        ▼
     ScanInfo
     Hide=true
        │
        ▼
   Scan pipeline
        │
        ▼
    Anonymizer
        │
        ▼
Protected scan output
```

The first implementation established deterministic pseudonymization for Kubernetes resource identifiers.

For example:

```text
production
    ↓
namespace-1

payments-api
    ↓
resource-1
```

The important property was **consistency**.

If the same resource appeared in multiple places in the report, the same pseudonym had to be used everywhere.

### Upstream work

- [#2051 — feat(scan): add hide flag and anonymization pipeline scaffold](https://github.com/kubescape/kubescape/pull/2051)
- [#2090 — feat(scan): anonymize resource identifiers in scan results](https://github.com/kubescape/kubescape/pull/2090)

---

# Phase 2 — Container Names and Images

Resource identifiers were only one possible source of information leakage.

Container names and image references can expose:

- Application names
- Private registries
- Internal repository structures
- Environment identifiers
- Deployment conventions

The anonymizer was therefore extended to cover:

```text
containers
initContainers
ephemeralContainers
```

including both container names and image references.

### Upstream work

- [#2129 — feat(scan): anonymize container names and images for --hide](https://github.com/kubescape/kubescape/pull/2129)
- [#2155 — fix(anonymizer): support unstructured container metadata anonymization](https://github.com/kubescape/kubescape/pull/2155)

The second change was important because Kubernetes data can arrive through both **typed** and **unstructured** representations.

---

# Phase 3 — Expanding Container Configuration Coverage

The container identity surface was still incomplete.

References to configuration resources could also expose internal infrastructure relationships.

The anonymization coverage was extended to fields such as:

```text
spec.containers[].env[].valueFrom.secretKeyRef.name

spec.containers[].env[].valueFrom.configMapKeyRef.name

spec.containers[].envFrom[].secretRef.name

spec.containers[].envFrom[].configMapRef.name

spec.imagePullSecrets[].name
```

These values can reveal internal resource names even when the container itself has already been anonymized.

### Upstream work

- [#2300 — fix(anonymizer): extend --hide coverage for container config references](https://github.com/kubescape/kubescape/pull/2300)

---

# Phase 4 — Protecting Additional Metadata Surfaces

After the core resource and container paths were covered, I performed a broader audit of report surfaces that could still expose identifying information.

This led to several additional areas.

## Annotation values

Annotations can contain environment-specific metadata and identifiers.

- [#2316 — fix(anonymizer): anonymize annotation values for hidden scans](https://github.com/kubescape/kubescape/pull/2316)

## Resource source metadata

`ResourceSource` can contain information about where a resource originated, including repository and filesystem context.

- [#2326 — anonymize resource source metadata in hidden output](https://github.com/kubescape/kubescape/pull/2326)

## Git repository context

Git metadata can expose repository and contributor information.

- [#2327 — anonymizer: hide git repository context metadata](https://github.com/kubescape/kubescape/pull/2327)

## Service-account identities

Service-account names can expose workload identity and internal naming conventions.

- [#2333 — anonymizer: hide service account names in pod specs](https://github.com/kubescape/kubescape/pull/2333)

---

# Phase 5 — Protecting Local Filesystem Paths

One of the more interesting architectural cases involved `LocalRootPath`.

The path could contain sensitive information such as:

```text
/home/developer/company-internal/project
```

However, that value was still required internally by downstream consumers such as SARIF generation and FixHandler.

Simply replacing the value inside the session would therefore have broken legitimate internal consumers.

Instead, the protection was applied at the **report-export boundary**.

```text
Internal scan state
        │
        │ retain operational metadata
        ▼
Report generation
        │
        │ protect sensitive values
        ▼
Exported report
```

### Upstream work

- [#2344 — fix: anonymize LocalRootPath in hidden report output](https://github.com/kubescape/kubescape/pull/2344)

This established an important design principle:

> **Protect the exported representation without unnecessarily destroying information required by the internal processing pipeline.**

---

# Phase 6 — Strengthening the Anonymizer

As the number of supported transformation paths increased, the implementation and testing structure also evolved.

The anonymizer test suite was reorganized and expanded to make the transformation behavior easier to validate and maintain.

### Upstream work

- [#2202 — test(anonymizer): reorganize and expand unit coverage](https://github.com/kubescape/kubescape/pull/2202)

---

# Phase 7 — From Anonymization to Reversible Protection

Deterministic anonymization solves one problem:

> Hide sensitive values while keeping the report useful.

But another requirement emerged:

> **What if the report owner needs to recover the original values later?**

The initial idea was to maintain a separate mapping artifact.

During the design process, the architecture moved toward a **SOPS-inspired envelope-encryption model**.

The conceptual workflow became:

```text
Sensitive field
      │
      ▼
Generate DEK
      │
      ▼
AES-GCM encryption
      │
      ▼
Encrypted field
      │
      ▼
Encryption metadata
      │
      ▼
Protected report
```

The key architectural advantage is that the report structure remains intact while selected values are transformed.

This provides a path toward:

- Self-contained encrypted reports
- Reversible protection
- Future KMS integration
- Backward-compatible report structures

---

# Phase 8 — Transformer Abstraction and Crypto Foundation

A transformer abstraction was introduced for repository metadata together with the initial cryptographic primitives.

The implementation initially focused on establishing the extension point without changing the existing anonymization behavior.

### Upstream work

- [#2347 — refactor: introduce transformer abstraction for repo metadata](https://github.com/kubescape/kubescape/pull/2347)

This created the foundation for supporting multiple transformation strategies:

```text
                  Metadata
                     │
              Transformation
                  abstraction
                     │
          ┌──────────┴──────────┐
          │                     │
          ▼                     ▼
    Mapping / Hide         Encryption
```

---

# Phase 9 — Encryption Workflow

The encryption workflow was then implemented incrementally.

### Encrypted repository metadata

- [#2365 — feat: add encrypted repo metadata workflow](https://github.com/kubescape/kubescape/pull/2365)

### DEK wrapping

- [#2374 — reportcrypto: add DEK wrapping support](https://github.com/kubescape/kubescape/pull/2374)

### Integration into anonymization workflow

- [#2380 — Integrate report encryption metadata and DEK wrapping into anonymization workflow](https://github.com/kubescape/kubescape/pull/2380)

The resulting model separates the encryption of report values from the protection of the Data Encryption Key.

```text
Report metadata
      │
      ▼
Generate DEK
      │
      ├──────────────► Wrapped DEK
      │
      ▼
AES-GCM encryption
      │
      ▼
Encrypted report metadata
```

---

# Phase 10 — Testing and Fail-Closed Behavior

Security-sensitive transformations should not silently produce an apparently protected result when the transformation itself fails.

The encryption path therefore received dedicated integration tests and explicit fail-closed behavior.

### Upstream work

- [#2351 — Add encryption transformer integration tests](https://github.com/kubescape/kubescape/pull/2351)
- [#2353 — Fail closed on repo metadata transformation errors](https://github.com/kubescape/kubescape/pull/2353)

The latter was also included in the Kubescape v4.0.11 release changelog.

---

# Phase 11 — Report Encryption and Decryption

Once encrypted metadata could be produced, the corresponding decryption path was added.

### Initial report decryption

- [#2425 — feat(report): add encrypted report decryption](https://github.com/kubescape/kubescape/pull/2425)

The implementation was then expanded to cover resource-source metadata:

- [#2440 — feat(reportcrypto): decrypt resource source metadata in reports](https://github.com/kubescape/kubescape/pull/2440)

and broader resource metadata:

- [#2441 — Expand encryption and decryption coverage for resource metadata](https://github.com/kubescape/kubescape/pull/2441)
- [#2442 — Expand encryption and decryption coverage for resource metadata](https://github.com/kubescape/kubescape/pull/2442)

---

# Phase 12 — Reversible Container Metadata

Container metadata was subsequently integrated into the reversible transformation model.

- [#2473 — feat(anonymizer): support reversible container metadata transformation](https://github.com/kubescape/kubescape/pull/2473)

This allowed container-related sensitive metadata to follow the same transformation architecture rather than introducing a separate mechanism.

---

# Phase 13 — Encrypted Resource Metadata Decryption

The corresponding decryption support for encrypted resource metadata was completed with:

- [#2493 — feat(reportcrypto): add decryption support for encrypted resource metadata](https://github.com/kubescape/kubescape/pull/2493)

At this stage the feature had evolved from a simple anonymization flag into a broader report-protection workflow.

---

# Phase 14 — CLI Documentation

After the implementation was complete, the workflow was documented for users.

### Documentation

- [#2508 — docs(cli): document report protection workflow](https://github.com/kubescape/kubescape/pull/2508)
- [#2510 — docs(cli): improve report protection help and examples](https://github.com/kubescape/kubescape/pull/2510)

The original issue was subsequently closed as completed.

---

# Architecture

The resulting architecture can be viewed as two related protection paths.

## Anonymization

```text
Kubernetes report
       │
       ▼
Sensitive identifiers
       │
       ▼
Deterministic mapping
       │
       ▼
Protected identifiers
       │
       ▼
Shareable report
```

## Encryption

```text
Kubernetes report
       │
       ▼
Sensitive metadata
       │
       ▼
Generate DEK
       │
       ▼
AES-GCM encryption
       │
       ▼
Encryption metadata
       │
       ▼
Protected report
       │
       ▼
Decryption
       │
       ▼
Original metadata
```

---

# Anonymization vs Encryption

| Capability | Anonymization | Encryption |
|---|---:|---:|
| Hide sensitive identifiers | ✓ | ✓ |
| Preserve report structure | ✓ | ✓ |
| Deterministic mapping | ✓ | — |
| Recover original values | Via mapping | ✓ |
| Cryptographic confidentiality | — | ✓ |
| Suitable for sanitized reports | ✓ | ✓ |
| Requires decryption key | — | ✓ |

The two mechanisms therefore address related but different requirements.

**Anonymization** focuses on pseudonymizing identifiers while preserving report relationships.

**Encryption** provides confidentiality and a reversible path for sensitive metadata.

---

# Supporting Report Structures

Some report-structure additions required by this work were implemented in **OPA Utils**, where the relevant report types are owned.

Those changes are therefore not represented as Kubescape PRs in this repository.

This repository intentionally documents the **Kubescape-side engineering work and upstream integration**, while the actual implementation remains in the upstream projects.

---

# Release

The completed work was incorporated into:

## Kubescape v4.0.11

The official release contains multiple commits associated with this report-protection work, including the encryption transformer integration tests, fail-closed transformation behavior, encryption metadata/DEK integration, and `LocalRootPath` protection.

**Release:**  
https://github.com/kubescape/kubescape/releases/tag/v4.0.11

---

# Upstream Contribution Index

For easier review, here is the complete contribution trail documented by this case study.

## Anonymization

| PR | Contribution |
|---|---|
| [#2051](https://github.com/kubescape/kubescape/pull/2051) | Add `--hide` flag and anonymization pipeline scaffold |
| [#2090](https://github.com/kubescape/kubescape/pull/2090) | Anonymize resource identifiers |
| [#2129](https://github.com/kubescape/kubescape/pull/2129) | Anonymize container names and images |
| [#2155](https://github.com/kubescape/kubescape/pull/2155) | Support unstructured container metadata |
| [#2202](https://github.com/kubescape/kubescape/pull/2202) | Reorganize and expand anonymizer tests |
| [#2300](https://github.com/kubescape/kubescape/pull/2300) | Extend `--hide` to container configuration references |
| [#2316](https://github.com/kubescape/kubescape/pull/2316) | Anonymize annotation values |
| [#2326](https://github.com/kubescape/kubescape/pull/2326) | Anonymize resource source metadata |
| [#2327](https://github.com/kubescape/kubescape/pull/2327) | Hide Git repository context metadata |
| [#2333](https://github.com/kubescape/kubescape/pull/2333) | Hide service-account names |
| [#2344](https://github.com/kubescape/kubescape/pull/2344) | Anonymize `LocalRootPath` in hidden output |

## Encryption and reversible protection

| PR | Contribution |
|---|---|
| [#2347](https://github.com/kubescape/kubescape/pull/2347) | Transformer abstraction and crypto foundation |
| [#2351](https://github.com/kubescape/kubescape/pull/2351) | Encryption transformer integration tests |
| [#2353](https://github.com/kubescape/kubescape/pull/2353) | Fail closed on transformation errors |
| [#2365](https://github.com/kubescape/kubescape/pull/2365) | Encrypted repository metadata workflow |
| [#2374](https://github.com/kubescape/kubescape/pull/2374) | DEK wrapping |
| [#2380](https://github.com/kubescape/kubescape/pull/2380) | Integrate encryption metadata and DEK wrapping |
| [#2425](https://github.com/kubescape/kubescape/pull/2425) | Encrypted report decryption |
| [#2440](https://github.com/kubescape/kubescape/pull/2440) | Resource source metadata decryption |
| [#2441](https://github.com/kubescape/kubescape/pull/2441) | Resource metadata encryption/decryption |
| [#2442](https://github.com/kubescape/kubescape/pull/2442) | Resource metadata encryption/decryption follow-up |
| [#2473](https://github.com/kubescape/kubescape/pull/2473) | Reversible container metadata transformation |
| [#2493](https://github.com/kubescape/kubescape/pull/2493) | Encrypted resource metadata decryption |

## Documentation

| PR | Contribution |
|---|---|
| [#2508](https://github.com/kubescape/kubescape/pull/2508) | Document report protection workflow |
| [#2510](https://github.com/kubescape/kubescape/pull/2510) | Improve CLI help and examples |

---

# What This Work Demonstrates

This project involved more than adding a CLI flag.

It required reasoning about:

- Kubernetes resource structures
- Typed and unstructured Kubernetes objects
- Report transformation boundaries
- Deterministic pseudonymization
- Cross-resource consistency
- Sensitive metadata discovery
- Export-time data protection
- Go abstractions
- AES-GCM encryption
- Data Encryption Keys
- Key wrapping
- Reversible transformations
- Fail-closed security behavior
- Integration testing
- CLI design
- Backward-compatible report structures
- Security-focused upstream review
- Multi-stage feature development
- Documentation and release integration

Most importantly, the work required repeatedly asking:

> **What information are we actually protecting?**

> **Where can that information leak?**

> **What information must remain available internally?**

> **Where should transformation happen?**

> **What happens if protection fails?**

That progression turned a simple anonymization request into a broader security engineering problem spanning **data transformation, cryptography, report architecture, testing, and user-facing workflows**.

---

# Upstream Project

**Kubescape:**  
https://github.com/kubescape/kubescape

**Original feature request:**  
https://github.com/kubescape/kubescape/issues/1200

**Released in:**  
**Kubescape v4.0.11**

---

> This repository is a personal engineering case study. It does not contain a fork or replacement implementation of Kubescape. The authoritative implementation and project history remain in the upstream Kubescape repository.
