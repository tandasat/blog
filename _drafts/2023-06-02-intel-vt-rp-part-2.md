---
layout: post
title: "Intel VT-rp - Part 2. paging-write and guest-paging verification"
---
- [Paging-write (PW)](#paging-write-pw)
  - [Protecting the hypervisor-managed paging structures](#protecting-the-hypervisor-managed-paging-structures)
  - [PW as performance optimization](#pw-as-performance-optimization)
  - [Demo - read-only paging structures](#demo---read-only-paging-structures)
- [Guest-paging verification (GPV)](#guest-paging-verification-gpv)
  - [Demo - preventing aliasing](#demo---preventing-aliasing)
- [Side discussions](#side-discussions)
  - [Code-integrity protection v.s. the remapping attack](#code-integrity-protection-vs-the-remapping-attack)
  - [Relevant security features](#relevant-security-features)
- [Conclusion](#conclusion)
  - [Notes](#notes)

This is the 2nd part of the series about the Intel VT Redirection Protection (VT-rp) technology. This post focuses on two of its features: paging write (PW) and guest-paging verification (GPV). We will also discuss how other protection mechanisms complement Intel VT-rp to defend the system against kernel-mode exploits. For hypervisor-managed linear address translation (HLAT), please read [part 1](https://tandasat.github.io/blog/2023/07/05/intel-vt-rp-part-1.html).

As a reminder, source code of the sample hypervisor used in this blog series is available on [GitHub](https://github.com/tandasat/Hello-VT-rp/).

We will use following acronyms throughout this post:
- LA: linear address
- GPA: guest physical address
- PA: physical address

![](/blog/img/posts/2023-06-02/cat.jpg) _(I am intrigued...)_


## Paging-write (PW)

### Protecting the hypervisor-managed paging structures

As discussed in part 1 of this series, HLAT employs a new set of paging structures rooted from the HLATP VMCS field to "lock" translation of desired LAs. Those paging structures, which we refer to as the hypervisor-managed paging structures, must be tamper resilient against a guest; otherwise, there would be no point in using HLAT.

For them to be tamper resilient, they have to be marked as read-only with EPT. The below illustrates this setup.

![](/blog/img/posts/2023-06-02/hlat_protected.png)

This can cause extra VM-exit during HLAT paging because the processor attempts to set "accessed" and "dirty" bits in the paging structures as needed. This operation is called "paging write". While this is done by hardware, it is still subject to EPT permissions.


### PW as performance optimization

To avoid those extra VM-exits, a hypervisor can set the "paging-write access" (PWA) bit in the EPT entries that correspond to the hypervisor-managed paging structures (<a name="body1">[*1](#note1)</a>). When this bit is set, paging write is allowed regardless of the EPT read/write permission.

![](/blog/img/posts/2023-06-02/eptpte_format_pw.png)

![](/blog/img/posts/2023-06-02/pw.png)


### Demo - read-only paging structures

Let us observe this behaviour using the custom hypervisor and `uefi_client` on the UEFI shell.

First, we make the hypervisor-managed paging structures read-only using hypercall `1`, then, enable HLAT for LA 0x200000 using hypercall `0`. This immediately causes EPT violation VM-exit and panic as shown below.

![](/blog/img/posts/2023-06-02/pw_without_pw.jpg)

This is because the processor attempted to perform paging write for the hypervisor-managed paging structures, which were read-only.

Next, let us set the PWA bit in the EPT entry using hypercall `2`, then enable HLAT.

![](/blog/img/posts/2023-06-02/pw_with_pw.jpg)

This does not cause VM-exit due to paging write.


## Guest-paging verification (GPV)

The 3rd feature VT-rp provides is guest-paging verification (GPV). It is a mechanism to prevent accessing a given GPA through unintended LAs. For example, in the below diagram, two LAs translate to the same GPA due to aliasing, but access to the GPA through (b) can be detected and prevented using GPV.

![](/blog/img/posts/2023-06-02/aliasing.png)

In a nutshell, this works by the processor verifying that all leaf ETP entries used to translate a LA to a GPA set the PWA bit.

To take a closer look, let us remind ourselves that the processor has to access guest-managed paging structures (ie, PML4e, PDPTe, PDe and PTe) on memory to translate a LA to a PA, and that access to each of those requires GPA -> PA translation. Namely,
1. To translate a LA, the processor:
   1. needs to read PML4e on GPA, which requires EPT PML4e, PDPTe, PDe and PTe to translate it to PA
   2. needs to read PDPTe on GPA, which requires EPT PML4e, PDPTe, PDe and PTe to translate it to PA
   3. and so on
2. Then, LA (1) is translated to GPA
3. Finally, GPA (2) is translated to PA with another set of EPT PML4e, PDPTe, PDe and PTe

![](/blog/img/posts/2023-06-02/translation.png)

A process called guest-paging verification steps in when the "verify guest paging" (VGP) bit is set in EPT PTe appeared in (3) above. When this happens, the processor checks that all leaf EPT entries appeared in (1.1) ~ (1.3) have the PWA bit. If not, EPT violation VM-exit occurs.

![](/blog/img/posts/2023-06-02/eptpte_format_vgp.png)

The below illustration depicts this.

![](/blog/img/posts/2023-06-02/gpv.png)

1. If the VGP bit is set in the EPT entry for a GPA being accessed,
2. the processor looks at the paging structures referenced during LA -> GPA translation, and
3. makes sure the GPAs of the paging structures are marked as PWA=1 in EPT.

Like (2') and (3') above, if the processor encounters an EPT entry that is not marked as PWA=1, EPT violation VM-exit occurs. This mechanism allows an hypervisor to enforce that a given GPA is accessed only through an intended LA and prevent the aliasing attack (<a name="body2">[*2](#note2)</a>).

Similar to PW, this feature can be used independently of HLAT (<a name="body3">[*3](#note3)</a>).


### Demo - preventing aliasing

Let us prevent the aliasing attack against GPA 0x200000 with GPV.

First, we enable HLAT using hypercall `0` to make sure the GPA is accessed through intended permission. Next, using hypercall `3`, we enable GPV for the GPA and allow access through the hypervisor-managed paging structures, by setting the PWA bit into the EPT entries corresponding to them.

![](/blog/img/posts/2023-06-02/gpv_setup.jpg)

Then, we alias GPA 0x200000 with the `alias` command. In this demo, LA 0x46200000 is an alias and translates to GPA 0x200000. Access to LA 0x46200000, however, causes VM-exit and panic as shown below.

![](/blog/img/posts/2023-06-02/gpv_panic.jpg)

This is because LA 0x46200000 was translated to GPA 0x200000 using the paging structures which are not marked as
"ok" with the PWA bit in the corresponding EPT entries. If the GPA 0x200000 were accessed through an original LA, that would have been successful as the LA would be translated through the paging structures that are marked as "ok" (because of the hypercall `3` above).


## Side discussions

### Code-integrity protection v.s. the remapping attack

In part 1, we discussed bypassing KDP, write-protection for data, using the remapping attack. An astute reader might have wondered if the same technique could be used to bypass write-protection for code. The answer is/should be "no."

The remapping attack is possible against data because it is trivial to find other GPA that is marked as RW in EPT. In fact, any GPA for data is marked as such by default. On the other hand, an attacker would have to find a GPA that is W+X in EPT to apply the attack for code. Such GPA should not exist in the first place.

It is still possible to remap a code page to another code page *without modification* and make the processor execute different instructions than what the original GPA has. However, given that an attacker would have to find a code page with suitable instructions at the exact page offset or beginning of the page, it would be challenging to implement this idea for anything useful.


### Relevant security features

Keep it in mind that VT-rp mitigates only a subset of exploitation techniques, and other mitigation technologies are required for more comprehensive protection against kernel-mode exploits. Here is a few of them:

- W^X guarantee for kernel-mode code: If code is writable, an attacker can generate and execute her shell-code.
- SMEP: If a user-mode page is executable in kernel-mode, an attacker can generate and execute her shell-code in user-mode pages, where W^X is usually not enforced.
- Kernel-mode code flow integrity: If forward- or backward-edge code flow is unprotected using CFG and CET shadow stack, an attacker can replace a function pointer or perform ROP to do desired operations (<a name="body4">[*4](#note4)</a>).

Additionally, configurations of those must be protected by a hypervisor. If we consider securely starting up the hypervisor, we need to combine more technologies. It is substantial work that is challenging to do without any bugs.


## Conclusion

In part 2, we looked into two of the features Intel VT-rp offered: paging-write (PW) and guest-paging verification (GPV). Specifically, how PW helped protect the hypervisor-managed paging structures efficiently and how GPV can be used to detect the aliasing attack in combination with PW.

The addition of Intel VT-rp is one of the latest examples of how processors and hypervisors evolve and play significant roles in the security scenes. It is also interesting to think about how many machines lack some of those protections, given that it was only in late 2020 when Intel CET was released with 11th gen and AMD shadow-stack was released with Ryzen 3.


### Notes

<a name="note1">*1</a> ([🔙](#body1)): PW can be used for the guest-managed paging structures without enabling HLAT. For example, the hypervisor may make part of guest-managed paging structures read-only and PWA=1 with EPT. It would "lock" the translation of a given LA as HLAT does.

<a name="note2">*2</a> ([🔙](#body2)): Although it is [explained that preventing the aliasing attack is one of the motivations for GPV](https://kvmforum2020.sched.com/event/eE4F), I am unclear how relevant the attack is in the real-world, as aliasing would be pointless if the GPA is properly protected with EPT. Even if the attacker aliases and accesses the GPA through the aliased LA, page protection will still be enforced by EPT anyway.

I understand that GPV may be used to enforce permissions by asserting the use of the guest-managed paging structures that are marked as read-only via EPT. However, I do not think of when this is desired over protecting GPA with EPT. If anyone knows or thinks of scenarios where the aliasing attack is important, let me know.

<a name="note3">*3</a> ([🔙](#body3)): GPV can be used without HLAT. That is, a hypervisor can set the PWA bits to EPT entries that correspond to the guest-managed paging structures. This would enforce that a given GPA is accessed through an intended LA but would not enforce permissions as the guest would be free to change the permission bits in the guest-managed paging structures (unless paging structures themselves are set as read-only in EPT). For this reason, I think the primary use of GPV is with HLAT as done in the demo.

<a name="note4">*4</a> ([🔙](#body4)): In fact, Windows does not enable kernel-mode CET by default, even on secured-core PCs as of this writing (10.0.22621). This can be enabled with the `KernelShadowStacks` registry key as mentioned in [Exploit Development: No Code Execution? No Problem! Living The Age of VBS, HVCI, and Kernel CFG](https://connormcgarr.github.io/hvci/) if desired.


----

_Found this post interesting? We offer a training course about the Intel virtualization technology. [Check out the course syllabus](https://tandasat.github.io/)._
