# Confidential Virtual Machine (CVM)

A **Confidential Virtual Machine (CVM)** is a virtual machine that runs inside a hardware Trusted Execution Environment (TEE), protecting the guest's memory and execution from inspection or tampering by the hypervisor, host OS, and other privileged software.

CVMs impose the least restrictions of any CC deployment model: existing applications run **unmodified** inside a CVM, with no code changes required.

```{figure} ../images/cvm.png
:alt: Remote Attestation — IETF RATS
:align: center
```

*References: [Red Hat — Introduction to Confidential Virtual Machines](https://www.redhat.com/en/blog/introduction-confidential-virtual-machines), [Edgeless Systems — CVMs](https://www.edgeless.systems/wiki/what-is-confidential-computing/cvms)*

---

## CVM Threat Model

The untrusted components include the host OS, KVM hypervisor, other VMs on the same host, other processes on the host machine.

Application code vulnerabilities, availability attacks, and software TEEs are out of scope.

### What a CVM Protects

CVMs provide **workload confidentiality**. The data in use is isolated from higher-privilege layers:

| Layer | Can it read CVM memory? |
|---|---|
| Hypervisor | ✗ No |
| Host OS | ✗ No |
| Host administrators | ✗ No |
| DMA-capable host devices | ✗ No |
| Other VMs on the same host | ✗ No |
| Guest OS (within the CVM) | ✔ Yes |
| The workload itself | ✔ Yes |

```{figure} ../images/cvm_protections.png
:alt: CVM protections
:align: center
```

### What CVMs Do NOT Protect Against

- Vulnerabilities within the guest workload (application bugs, remote exploits)
- Availability attacks (DoS)
- Attacks that originate inside the guest with legitimate access

---

## Memory Encryption

The hardware memory controller transparently encrypts all guest physical memory pages with a key held inside the CPU. The hypervisor sees only ciphertext when it accesses guest memory.



**Data at rest** in a CVM also requires protection — the host controls storage access, so full disk encryption (e.g., LUKS) is mandatory for any sensitive data persisted to disk.

---

## Boot Chain Security

A critical risk in CVMs: even with memory encryption, a malicious host could swap binaries before they are loaded into protected memory. Every executable must therefore be **authenticated** before execution.

### UEFI Secure Boot

Secure Boot ensures only vendor-signed code executes during boot:

```{mermaid}
flowchart LR
    FW["UEFI Firmware\n(Root of Trust)"]
    SH["Shim\nMicrosoft-signed\ncarries OS-vendor certs"]
    BL["Bootloader\nGRUB / systemd-boot"]
    KN["Kernel\nlockdown mode"]
    MOD["Kernel Modules\nsigned only"]

    FW -->|verifies| SH
    SH -->|verifies| BL
    BL -->|verifies| KN
    KN -->|allows| MOD
```

### Unified Kernel Image (UKI)

A UKI bundles the kernel, initramfs, and kernel command line into **one signed UEFI binary**, extending trust to components that were previously unsigned:

| Component | Standard Boot | UKI |
|---|---|---|
| Kernel | ✔ Signed | ✔ Signed |
| initramfs | ✗ Unsigned | ✔ Signed (part of UKI) |
| Kernel command line | ✗ Unprotected | ✔ Signed (part of UKI) |

**Trade-offs:** the initramfs and kernel command line are fixed at build time — they cannot be modified dynamically (e.g., `root=` cannot be set at runtime).

### Measured Boot and the Root Volume Key

Even with Secure Boot, a signed kernel could be paired with a crafted initramfs to extract a volume decryption key. The solution is **PCR-gated key unsealing**:

1. Each boot component extends a PCR register: `PCR_new = SHA256(PCR_old || component_hash)`
2. The volume decryption key is sealed to a TPM policy requiring specific PCR values
3. The hardware evaluates the policy — if any boot component changed, PCRs differ and the key is not released
4. A compromised guest OS cannot circumvent this because the hardware enforces it

Key PCRs for CVMs:

| PCR | What it covers |
|---|---|
| PCR4 | Hashes of loaded UEFI binaries |
| PCR7 | Secure Boot state + certificates used |
| PCR11 (UKI via systemd-stub) | Kernel, initramfs, command line, all UKI sections |

---

## The vTPM Role

A **virtual TPM backed by hardware** (rather than managed by the hypervisor) enables:

- **Automated (unattended) key unsealing** — no user password at boot (which the host could intercept via console emulation)
- **Attestable genuineness** — under SEV-SNP or TDX, a vTPM inside the TEE cannot be faked by the host
- **PCR-based policies** — the same policy model used for bare-metal TPMs works inside the CVM

When the vTPM is placed **inside the TEE** (via SVSM on AMD SEV-SNP, or natively on Azure CVMs), the hypervisor is completely excluded from the trust chain.

---

## Remote Attestation for CVMs

The hardware generates a **signed attestation report** containing the boot measurements. A remote party can use this to verify:

1. The VM is genuinely running inside a hardware TEE (not a software simulation)
2. The correct guest OS was booted (measurements match expected values)
3. The hardware is at the required firmware/patch level

```{mermaid}
sequenceDiagram
    actor Tenant
    participant CVM as CVM (TEE)
    participant HW as AMD / Intel

    Tenant->>CVM: Request attestation
    CVM->>HW: Get attestation report
    HW-->>CVM: Signed report (VCEK / IAK)
    CVM-->>Tenant: Attestation report
    Tenant->>Tenant: Verify report against AMD/Intel cert chain
    Tenant->>Tenant: Check measurements match expected values
    Tenant->>CVM: Send secrets (if verification passed)
```

---

## Hardware Technologies

With all three major CPU vendors supporting CVMs, they are expected to dominate the Confidential Computing landscape going forward.

| Vendor | Technology | Generation | Key Features |
|---|---|---|---|
| **AMD** | SEV-SNP | Current (EPYC Milan+) | Memory encryption + integrity; RMP table prevents remapping |
| **AMD** | SEV-ES | Previous | Encrypted registers; no integrity |
| **AMD** | SEV | Legacy | Memory encryption only |
| **Intel** | TDX | Sapphire Rapids (4th Gen Xeon+) | Trust Domains; RTMR-based measurement; TDX Module |
| **IBM Z** | Secure Execution | IBM Z / LinuxONE | Ultra-Violet guests; mainframe-grade isolation |
| **IBM Power** | PEF | IBM POWER | Protected Execution Facility |
| **ARM** | CCA (Confidential Compute Architecture) | In development | Realm VMs; Realm Management Monitor |
| **RISC-V** | CoVE (Confidential VM Extensions) | Proposed | Open ISA equivalent |

### AMD SEV-SNP vs Intel TDX

| Feature | AMD SEV-SNP | Intel TDX |
|---|---|---|
| **Memory integrity** | Reverse Map Table (RMP) | MAC-based per-page integrity |
| **Measurements** | Stored in AMD SP; in attestation report | RTMRs extended by OVMF; in event log |
| **Attestation signing key** | VCEK (per chip + firmware) | IAK (Intel-managed Quoting Enclave) |
| **vTPM support** | Via SVSM (inside TEE) or hypervisor-managed | Via TDX RTMR-based vTPM |
| **Availability** | AWS, GCP, Azure, bare metal | Azure, GCP |

---

## Cloud Availability

| Cloud | Technology | Status |
|---|---|---|
| **Microsoft Azure** | AMD SEV-SNP | GA |
| **Microsoft Azure** | Intel TDX | GA |
| **Google Cloud** | AMD SEV-SNP | GA |
| **Google Cloud** | Intel TDX | GA |
| **AWS** | AMD SEV-SNP | GA |
| **IBM Cloud** | IBM Secure Execution | GA |

---

## CVMs as the Foundation for CoCo

CVMs are **Pillar 1** of the Confidential Computing stack. The CNCF Confidential Containers (CoCo) project builds directly on CVMs:

- Each Kubernetes Pod runs inside its own CVM
- The CVM provides the hardware TEE boundary
- Attestation of the CVM is the root of trust for all CoCo secrets

The chapters that follow cover how CoCo and Trustee build on this foundation.
