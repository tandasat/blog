---
layout: post
title: "Intel VT-rp - Part 2. Paging Write and Guest-Paging Verification"
---
- [Paging Write (PW)](#paging-write-pw)
  - [What protects the hypervisor-managed paging structures?](#what-protects-the-hypervisor-managed-paging-structures)
  - [PW as performance optimization](#pw-as-performance-optimization)
  - [Demo](#demo)
- [Guest-Paging Verification (GPV)](#guest-paging-verification-gpv)
  - [The aliasing attack](#the-aliasing-attack)
  - [Demo](#demo-1)
- [Discussions](#discussions)
  - [Bypassing Code-integrity protection with the remapping attack](#bypassing-code-integrity-protection-with-the-remapping-attack)
  - [Other mitigation technologies](#other-mitigation-technologies)
- [Conclusion](#conclusion)
  - [Notes](#notes)


This is the 2nd part of the series about the Intel VT-rp, redirection protection, technology. This post focuses on two of its features: paging write (PW) and guest-paging verification (GPV), and an exploitation technique can be prevented with them. We will also discuss how other protection mechanisms complement Intel VT-rp to protect the system from kernel-mode exploits.

For hypervisor-managed linear address translation (HLAT), please refer to the part 1.

## Paging Write (PW)

### What protects the hypervisor-managed paging structures?

The hypervisor-managed paging structures must be tamper resilient against a guest to enforce LA -> GPA translation. If the guest could modify the paging structures used by HLAT, there would be no point of using HLAT. For the hypervisor-managed paging structures to be tamper resilient, they have configured as read-only with EPT. The below illustrates this setup.

_(insert an illustration showing its permissions, along with guest-managed paging structures)_

This can cause extra VM-exit during HLAT paging because the processor may attempt to set "Accessed" and "Dirty" bits in the paging structures as needed. While this is done by hardware, it is still subject to EPT permissions.


### PW as performance optimization

To avoid those extra VM-exits, a hypervisor can set the "paging-write access" bit in the EPT entries that correspond to the hypervisor-managed paging structures. When this bit is set, write to "Accessed" and "Dirty" bits during paging bypasses EPT permissions.

![](/blog/img/posts/2023-06-02/eptpte_format.png)

_(insert illustration showing PW=1 in EPT PTe)_


### Demo

Let us observe this behaviour on the UEFI shell. We will:

1. Enable write-protection for the hypervisor-managed paging structures via a hypercall
2. Enable HLAT for LA 0x20000
3. Confirm that EPT violation occurs
4. Redo the above with PW bit in the EPT PTe and confirm EPT violation does not occur


## Guest-Paging Verification (GPV)

Guest-Paging Verification (GPV) is a mechanism to enforce that a given GPA can only be accessed through an indented LA. In the below diagram, access (a) is allowed but (b) is not.

_(insert illustration showing two path. normal and aliased)_

This works by the processor verifying that all leaf ETP entries involved to translate a LA to a PA have PW=1.

_(insert illustration showing two path. normal and aliased)_

To translate a LA to PA, the processor has to access paging structures (ie, PML4e, PDPTe, PDe and PTe) on memory. Access to each of those triggers EPT translation, ie:
1. to read PML4e, EPT PML4e, PDPTe, PDe and PTe are referenced
2. to read PDPTe, EPT PML4e, PDPTe, PDe and PTe are referenced, as well
3. and so on
4. Then, LA is translated to GPA
5. GPA is translated to PA with another set of EPT PML4e, PDPTe, PDe and PTe

The guest-paging verification process steps in when the "verify guest paging" bit is set in EPT PTe appeared in (5) above. When this happens, the processor checks that all EPT PTs appeared in (1), (2) and (3) have the "paging-write access" bit set. If not, VM-exit occurs.

![](/blog/img/posts/2023-06-02/eptpte_format.png)

_(Insert illustration showing the above)_

This mechanism enforces that a GPA is translated with intended paging structures and prevents a certain attack technique.


### The aliasing attack

The type of attack prevented by GPV is called "aliasing". When an attacker has an arbitrary kernel-mode read-write primitive, she can manipulate the guest paging structures and "dual-map" a single GPA onto two or more LAes (<a name="body1">[*1](#note1)</a>).

_(Insert illustration similar to the above one)_

This attack would be detected with GPV as access to the GPA through the aliased LA involves another paging structures. The hypervisor could set "paging-write access" bits only for the paging structures that would be used for the original LA.


### Demo


- Enable write-protection for hv managed paging structures
- Enable HLAT for 0x20000
- Set PW bit in the EPT PTe
- Set VPW bit in the EPT PTes
- Alias 0x20000
- Access the alias
  - Receive EPT violation


## Discussions

### Bypassing Code-integrity protection with the remapping attack

In the part 1, we discussed bypassing KDP, write protection for data, using the remapping attack. An astute reader might have wondered if the same technique could be used to bypass write protection for code. The answer is/should be "no".

The remapping attack is possible against data because it is trivial to find other GPA that is marked as RW in EPT. In fact, any GPA for data is marked as such by default. On the other hand, an attacker would have to find a GPA that is W+X in EPT for aliasing code. Such GPA should not exist in the first place, or an attacker would simply generate code in such GPA without even having to remap anything.

It is still possible to remap a code page to another code page *without modification* and make the processor execute different instructions than what the original GPA has. However, given that an attacker would have to find a code page with suitable instructions at the exact page offset or beginning of the page and still has no ability to generate any new code, it would be challenging to use this idea for anything useful.


### Other mitigation technologies

VT-rp mitigates only a subset of exploitation techniques, and other mitigation technologies are required for more comprehensive protection against kernel-mode exploits. Here is a few examples:

- W^X guarantee for kernel-mode code: If code is writable, an attacker can generate and execute her shell-code.
- SMEP: If a user-mode page is executable in kernel-mode, an attacker can generate and execute her shell-code in user-mode pages, where W^X is usually not enforced.
- Kernel-mode code flow integrity: If either forward or backward code flow is not protected through CFG and CET, an attacker can replace a function pointer or perform ROP to carry out desired operations.

Additionally, all of those configurations must be locked (protected) by a hypervisor. It is substantial work that is challenging to do without any bugs.


## Conclusion

In the part 2, we looked into two of features Intel VT-rp offered: PW and GPW. Specifically, how PW functioned to make the hypervisor-managed paging structures efficiently and how GPV can be used to detect the aliasing attack in combination with PW.

With a addition of Intel VT-rp, it is even more fascinating to follow evolution of hardware and hypervisors and how they play significant roles in the security scenes. It is also interesting to think about how many machines are unprotected even with code flow integrity given CET was released in only late 2020 with Intel 11th gen and AMD Ryzen 3 (<a name="body2">[*2](#note2)</a>).


### Notes

<a name="note1">*1</a> ([🔙](#body1)): To be honest, I am unclear how relevant the aliasing attack is, as it would be pointless if the GPA is properly protected with EPT. Even if the attacker aliases and accesses the GPA though the aliased LA, that is page protection would still be enforced by EPT anyway. I understand having another layer of assurance as defense-in-depth is nice, but it is hard to believe Intel invested money only for that. If anyone knows or think of scenarios where the aliasing attack may be relevant, please let me know.

<a name="note2">*2</a> ([🔙](#body2)): In fact, Windows does not enable kernel-mode CET by default, even with Secured-core PCs as of this writing (10.0.22621.1848). This can be enabled with the  `KernelShadowStacks` registry key as mentioned in [Exploit Development: No Code Execution? No Problem! Living The Age of VBS, HVCI, and Kernel CFG](https://connormcgarr.github.io/hvci/) if desired.


----

_Found this post interesting? We offer a training course about the Intel virtualization technology. [Check out the course syllabus](https://tandasat.github.io/)._
