---
layout: post
title: "Intel VT-rp with examples - Part 2. Paging Write and Guest-Paging Verification"
---
- [Paging Write (PW)](#paging-write-pw)
  - [What protects the hypervisor-managed paging structures?](#what-protects-the-hypervisor-managed-paging-structures)
  - [PW bit as performance optimization](#pw-bit-as-performance-optimization)
  - [Demo](#demo)
- [Guest-Paging Verification (GPV)](#guest-paging-verification-gpv)
  - [The aliasing attack](#the-aliasing-attack)
- [VWP and aliasing](#vwp-and-aliasing)
- [Demo](#demo-1)
- [Can remapping attack bypasses HVCI?](#can-remapping-attack-bypasses-hvci)
- [Remark on other mitigations](#remark-on-other-mitigations)
- [Final thoughts](#final-thoughts)
  - [Notes](#notes)


This is the 2nd part of the series about the Intel VT-rp, redirection protection, technology. This post focuses on two of its features: paging write (PW), guest-paging verification (GPV), and an exploitation technique can be prevented with them. We will also discuss how other protection mechanisms complement this feature to prevent exploitation techniques.

?: I suggest reading the part 1 if you have not, as this post assumes the readers are familiar with EPT and HLAT.

## Paging Write (PW)

### What protects the hypervisor-managed paging structures?

The hypervisor-managed paging structures must be tamper resilient against a guest to enforce LA -> GPA translation. This has to be implemented through EPT by making them read-only. The below illustrates this setup.
_(insert an illustration showing its permissions, along with guest-managed paging structures)_

This can cause extra VM-exit during HLAT paging because the processor may attempt to set `Accessed` and `Dirty` bits in the paging structures as needed. While this is done by hardware, it is still subject to EPT permissions.

### PW bit as performance optimization

To avoid this issue, a hypervisor can set the PW bit in the EPT entries that correspond to the hypervisor-managed paging structures. When this bit is set, write to `Accessed` and `Dirty` bits during paging bypasses EPT permissions (*1).
![](/blog/img/posts/2023-06-02/eptpte_format.png)
_(insert illustration showing PW=1 in EPT PTe)_

This behaviour is gated by one VM execution controls. See the bit position 2 below.
![](/blog/img/posts/2023-06-02/exec_control.png)


### Demo

Let us observe this behaviour on the UEFI shell. We will:

1. Enable write-protection for hv managed paging structures via a hypercall
2. Enable HLAT for LA 0x20000
3. Confirm that EPT violation occurs
4. Redo the above with PW bit in the EPT PTe and confirm no EPT violation occurs


## Guest-Paging Verification (GPV)

Guest-Paging Verification (GPV) is a mechanism to enforce that a given GPA can only be accessed through an indented LA. In the below diagram, access (a) is allowed, and (b) is not.
_(insert illustration showing two path. normal and aliased)_

This works by the processor verifying that all leaf ETP entries involved to translate a LA to a PA has PW=1.
_(insert illustration showing two path. normal and aliased)_

To translate a LA to PA, the processor has to access paging structures (ie, PML4e, PDPTe, PDe and PTe) on memory. Access to each of those triggers EPT translation. Thus:
1. to read PML4e, EPT PML4e, PDPTe, PDe and PTe are used
2. to read PDPTe, EPT PML4e, PDPTe, PDe and PTe are used, as well
3. and so on.
4. Then, LA is translated to GPA.
5. GPA is translated to PA with another set of EPT PML4e, PDPTe, PDe and PTe

GPV steps in when the "verify guest paging" bit is set in EPT PTe appeared in (5) above. When this happens, the processor checks that all EPT PTs appeared in (1), (2) and (3) has the "paging-write access" bit set.
![](/blog/img/posts/2023-06-02/eptpte_format.png)
_(Insert illustration showing the above)_

GPV enforces that the GPA is translated with intended paging structures and prevents an attack called "aliasing".

### The aliasing attack

when an attacker has an arbitrary kernel-mode read-write primitive, she can manipulate the guest paging structures and "dual-map" a single GPA onto two or more LAes.

Similar to the remapping attack we discussed in the part 1,

## VWP and aliasing
- explain aliasing
- explain that setting this bit in the EPT PTEs managing GPA of hypervisor-managed paging structure enforces that a given LA is translate to PA using paging structures that are (indirectly) marked as PW=1
- this prevents aliasing as it would use paging structure entries that are different from those used in the un-aliased memory access

## Demo


- Enable write-protection for hv managed paging structures
- Enable HLAT for 0x20000
- Set PW bit in the EPT PTe
- Set VPW bit in the EPT PTes
- Alias 0x20000
- Access the alias
  - Receive EPT violation


## Can remapping attack bypasses HVCI?

- Less likely
- Remapped GPA must be executable at the EPT level
- Also must be writable
  - Or reuse existing page
- W+KX pages are not supposed to exist




## Remark on other mitigations

VT-rp covers only subset of exploitation techniques, and taking advantages of other mitigations are required for more comprehensive security of the system.

For example, without kernel-mode CET, an attacker with arbitrary kernel-mode read and write primitives can corrupt kernel-thread stack and perform ROP (TBD ref). Without CFG, any unprotected function pointers can be updated to point to arbitrary code,


## Final thoughts
- AMD
- good learning resources of the state-of-art challenges
- older systems remain to be unprotected
- increasingly difficult to test


### Notes


----

_Found this post interesting? We offer a training course about the Intel virtualization technology. [Check out the course syllabus](https://tandasat.github.io/)._
