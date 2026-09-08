<div align="center">

<h1>Yudistira Putra</h1>

<p><strong>Systems Software Engineer — Compilers · GPU · Firmware · Correctness</strong></p>

<p>
<strong>C · C++ · Rust</strong><br>
LLVM / AMDGPU · Vulkan · QEMU / x86_64 · RISC-V / OpenSBI · secp256k1
</p>

<p>
Upstream work across LLVM, Khronos Vulkan Validation Layers, OpenSBI, and Linux,<br>
plus independent QEMU validation and security research.
</p>

<p>
<a href="mailto:pyudistira519@gmail.com">
  <img src="https://img.shields.io/badge/Email-Contact-24292F?style=flat-square&logo=gmail&logoColor=white" alt="Email">
</a>
<a href="https://www.linkedin.com/in/yudistira-putra-dev/">
  <img src="https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat-square&logo=linkedin&logoColor=white" alt="LinkedIn">
</a>
<a href="https://github.com/Yudis-bit">
  <img src="https://img.shields.io/badge/GitHub-Yudis--bit-181717?style=flat-square&logo=github&logoColor=white" alt="GitHub">
</a>
</p>

</div>

---

## `$ whoami`

I work on **correctness-critical systems software**: compiler backends, validation layers, emulators, firmware, cryptographic code, and regression infrastructure.

Most of my work starts with a failure that is difficult to explain and ends with something another engineer can verify.

**reproduce → reduce → understand → test → upstream**

The work below links directly to the patch, commit, report, or repository behind it.

---

## Upstream Engineering

| Project | Contribution | Result |
|:---|:---|:---:|
| [**LLVM · AMDGPU**](https://github.com/llvm/llvm-project/pull/210583) | Prevented TFE/LWE image loads from entering invalid `SILoadStoreOptimizer` merge candidates; added MIR regression coverage | `MERGED` |
| [**Khronos Vulkan Validation Layers**](https://github.com/KhronosGroup/Vulkan-ValidationLayers/pull/12743) | Fixed an out-of-bounds crash in static descriptor validation and added C++ regression coverage | `MERGED` |
| [**OpenSBI · SBI ecall**](https://github.com/riscv-software-src/opensbi/commit/f95648d3955d72f77e13315a990a6135303978a5) | Prevented a caller-buffer boundary bug that could lead to an out-of-bounds write; added SBIUNIT redzone coverage | `MERGED` |
| [**QEMU · x86_64**](https://github.com/qemu/qemu/commit/3589cd995b4facf34071e944fd8ec2294524e25a) | Independently validated a long-mode segment-prefix decoding fix | `TESTED-BY` |
| [**bitcoin-core/secp256k1**](https://github.com/bitcoin-core/secp256k1/pull/1893) | Extended constant-time test coverage for `schnorrsig_sign_custom` | `OPEN` |
| [**OpenSBI · RPMI mailbox**](https://github.com/riscv-software-src/opensbi/pull/423) | Proposed bounded handling for fixed-size shared-memory queue names | `OPEN` |
| **Linux · tracing / ftrace** | Focused documentation and sample cleanups — [e5d8524](https://github.com/torvalds/linux/commit/e5d8524) · [8a66c09](https://github.com/torvalds/linux/commit/8a66c09) | `MERGED ×2` |
| [**Code4rena · Swafe**](https://code4rena.com/reports/2025-11-swafe) | Co-found replayable account-recovery state issue, M-04 | `CO-FINDER` |

---

## Selected Upstream Work

### 01 · LLVM / AMDGPU

**`SILoadStoreOptimizer` image-load merge correctness**

TFE/LWE image loads carry additional status-result semantics.

The image-load merge path cannot safely reconstruct those result lanes, which means these instructions should not enter ordinary merge-candidate collection.

```text
ordinary image load
        │
        └── merge candidate

TFE / LWE image load
        │
        └── additional status-result semantics
                    │
                    ▼
          merge path cannot preserve
            the required result shape
                    │
                    ▼
          reject before collection
```

I added the rejection guard before candidate collection, covered the asymmetric ordinary → TFE/LWE ordering with MIR regression tests, preserved the existing positive merge behavior, and iterated through AMDGPU review and CI.

**Artifact:** [llvm-project #210583](https://github.com/llvm/llvm-project/pull/210583) — merged into `llvm:main`.

---

### 02 · Khronos Vulkan Validation Layers

**Static descriptor validation crash**

An invalid pipeline could expose a mismatch between a shader-declared descriptor array and the pipeline-layout binding count.

That path could eventually reach:

```cpp
binding.descriptors[index]
```

with an out-of-range `index`.

```text
shader descriptor array
          │
          ▼
pipeline-layout binding
          │
          ▼
       mismatch
          │
          ▼
static validation
          │
          ▼
descriptors[index]
          │
          ▼
out-of-range access
```

I reproduced the failure, isolated the descriptor-array / binding-count boundary, added a narrow early-termination guard, added dedicated C++ regression coverage, and worked through maintainer review and upstream CI.

**Artifact:** [Vulkan-ValidationLayers #12743](https://github.com/KhronosGroup/Vulkan-ValidationLayers/pull/12743) — merged into `KhronosGroup:main`.

[Read the case study](https://github.com/Yudis-bit/opencode/blob/main/case-studies/vulkan-validation-crash-fix.md)

---

### 03 · OpenSBI / RISC-V

**Caller-buffer boundary in `sbi_ecall_get_extensions_str()`**

The extension-string helper could advance its offset using the nominal extension-name length before confirming that the next entry fit inside the caller-provided destination.

With a sufficiently small destination, the offset could move beyond the supplied capacity and make the remaining-size calculation invalid.

```text
registered SBI extensions
          │
          ▼
caller-provided buffer
          │
          ▼
next entry does not fit
          │
          ▼
offset would exceed capacity
          │
          ▼
boundary check rejects append
```

The patch adds the boundary guard before appending the next extension name.

The regression coverage uses a **16-byte destination with a redzone** to detect writes beyond the caller-provided boundary, together with a larger-buffer control case.

The patch was reviewed by **Anup Patel**, landed in OpenSBI `master`, and closed the corresponding upstream issue.

**Artifacts:**  
[Upstream commit](https://github.com/riscv-software-src/opensbi/commit/f95648d3955d72f77e13315a990a6135303978a5) ·
[Patch thread](https://lore.kernel.org/r/20260719101125.190314-1-pyudistira519@gmail.com)

---

### 04 · QEMU / x86_64

**Long-mode segment override prefix decoding**

I independently tested an upstream fix for x86 long-mode handling of legacy segment override prefixes.

The work focused on checking emulator behavior against the expected architectural semantics.

The final QEMU commit records:

```text
Tested-by: Yudistira Putra
```

This is **independent validation credit**, not authorship of the underlying patch.

**Artifact:** [QEMU commit 3589cd9](https://github.com/qemu/qemu/commit/3589cd995b4facf34071e944fd8ec2294524e25a)

---

## Active Upstream Work

### bitcoin-core / secp256k1

**Constant-time coverage for `schnorrsig_sign_custom`**

Work in progress extending constant-time test coverage around custom Schnorr signing behavior, including custom nonce callbacks, auxiliary data, variable-length messages, and CHECKMEM-backed execution.

**Artifact:** [secp256k1 #1893](https://github.com/bitcoin-core/secp256k1/pull/1893)

---

### OpenSBI / RPMI mailbox

**Fixed-size queue-name boundary handling**

Proposed bounded handling for shared-memory queue names where external input meets a fixed-size firmware buffer.

**Artifact:** [OpenSBI #423](https://github.com/riscv-software-src/opensbi/pull/423)

---

## Projects

### [`ecc-audit-engine`](https://github.com/Yudis-bit/ecc-audit-engine)

**Reproducible differential-testing infrastructure for secp256k1 implementations.**

```text
deterministic corpus
        │
        ▼
target execution
        │
        ▼
behavior comparison
        │
        ▼
failure detection
        │
        ▼
minimization
        │
        ▼
deterministic replay
        │
        ▼
structured evidence
```

The project focuses on:

- deterministic test corpora
- differential execution
- correct, corrupted, and synthetic targets
- failure detection and minimization
- deterministic replay
- structured evidence and reporting
- dynamic-trace experiments
- CI-backed regression verification

The useful output is not simply _“these implementations disagree.”_

It is a reduced case that another engineer can reproduce and inspect.

[Repository](https://github.com/Yudis-bit/ecc-audit-engine) ·
[Latest release](https://github.com/Yudis-bit/ecc-audit-engine/releases/latest) ·
[Quick demo](https://github.com/Yudis-bit/ecc-audit-engine/blob/main/examples/quick-demo.sh)

---

### [`ArkheionX`](https://github.com/Yudis-bit/arkheionx)

**Local-first security-review infrastructure.**

ArkheionX builds structured review artifacts from local repositories so reviewers can reason about:

- scope
- value flow
- protocol behavior
- assumptions
- invariants
- supporting evidence
- unresolved review gaps

without requiring sensitive source code to leave the local environment.

The design keeps strong boundaries:

```text
no RPC
no live-chain scanning
no auto-submit
no automatic vulnerability confirmation
human review required
```

The intent is to improve review structure and reproducibility without pretending that automated tooling can replace engineering judgment.

[Repository](https://github.com/Yudis-bit/arkheionx) ·
[Releases](https://github.com/Yudis-bit/arkheionx/releases)

---

### [`bitpeek`](https://github.com/Yudis-bit/bitpeek)

**Small, client-side byte inspector.**

Paste bytes and inspect what they mean under a specific representation.

```text
DE AD BE EF

hex
binary
ASCII / UTF-8
signed integer
unsigned integer
big endian
little endian
individual bits
```

It supports:

- hex, binary, decimal-byte, and UTF-8 input
- byte-range selection
- signed and unsigned integer interpretation
- big-endian and little-endian interpretation
- interactive bit mutation
- exact handling of 64-bit integer values
- fully client-side execution

No backend. No upload.

[Live](https://bitpeek-seven.vercel.app/) ·
[Source](https://github.com/Yudis-bit/bitpeek)

---

## Security Research

### Code4rena · Swafe

**M-04 co-finder · Medium severity · tied #30**

My submission **S-855** was credited in Code4rena's final Swafe report as a co-finding of **M-04**.

The affected recovery flow authenticated the request but did not sufficiently bind it to freshness or current account state.

That meant a previously valid recovery request could be replayed against later recovery state.

```text
valid recovery request
        │
        ▼
insufficient freshness /
state binding
        │
        ▼
old request remains valid
        │
        ▼
later recovery state affected
        │
        ▼
legitimate recovery disrupted
```

The issue affected recovery-state integrity and could be used to interfere with legitimate account recovery.

Code4rena's mitigation review later marked **M-04 as mitigated**.

[Final Swafe report](https://code4rena.com/reports/2025-11-swafe) ·
[Submission S-855](https://code4rena.com/audits/2025-11-swafe/submissions/S-855)

---

## How I Work

I prefer small changes with evidence attached.

For correctness work, I usually want to know:

```text
Can I reproduce it?
        │
        ▼
Can I reduce it?
        │
        ▼
Which assumption or invariant failed?
        │
        ▼
Can the correction stay narrow?
        │
        ▼
Can a regression test preserve it?
        │
        ▼
Upstream review
```

That process matters more to me than the size of the patch.

A three-line fix can represent a substantial investigation if the failure is subtle enough.

---

## Current Focus

| Area | Work |
|:---|:---|
| **Compilers** | LLVM backend correctness · machine-level optimization · MIR regression testing |
| **GPU / Graphics** | Vulkan validation · descriptor and pipeline boundaries · crash debugging |
| **Virtualization** | QEMU · x86_64 ISA behavior · decoder validation |
| **Firmware** | RISC-V · OpenSBI · caller-provided and fixed-size buffer boundaries |
| **Cryptographic Software** | secp256k1 · constant-time testing · differential testing |
| **Verification** | reproduction · minimization · deterministic replay · regression infrastructure |
| **Security-Critical Software** | invariants · freshness · state transitions · replay resistance |

---

## Open to Engineering Work

I'm interested in **full-time, contract, and scoped engineering work** involving:

- systems software in **C, C++, or Rust**
- compiler engineering and backend correctness
- GPU / graphics infrastructure
- virtualization and ISA-level debugging
- RISC-V firmware
- cryptographic software
- regression and verification infrastructure
- security-critical systems
- difficult cross-layer debugging

The problems I enjoy most are the ones where the failure is subtle, the boundary is not obvious, and correctness has to be demonstrated rather than assumed.

---

<div align="center">

<strong>Systems · Compilers · GPU · Virtualization · Firmware · Cryptographic Software</strong>

<br><br>

<a href="mailto:pyudistira519@gmail.com">
  <img src="https://img.shields.io/badge/Email-Let's_Talk-24292F?style=flat-square&logo=gmail&logoColor=white" alt="Email">
</a>
<a href="https://www.linkedin.com/in/yudistira-putra-dev/">
  <img src="https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=flat-square&logo=linkedin&logoColor=white" alt="LinkedIn">
</a>

</div>
