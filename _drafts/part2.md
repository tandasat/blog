



# Part 2 outline

## How to set up HLAT
- mention that hypervisor-managed page tables have to be mapped to GPA in RO using the EPT level

## PW
- explain that setting this bit in the EPT PTEs managing GPA of hypervisor-managed paging structure allows the processor to set A and D bits without causing VM-exit
- performance improvement feature

## VWP and aliasing
- explain aliasing
- explain that setting this bit in the EPT PTEs managing GPA of hypervisor-managed paging structure enforces that a given LA is translate to PA using paging structures that are (indirectly) marked as PW=1
- this prevents aliasing as it would use paging structure entries that are different from those used in the un-aliased memory access


## Can remapping attack bypasses HVCI?

## Remark on other mitigations

VT-rp covers only subset of exploitation techniques, and taking advantages of other mitigations are required for more comprehensive security of the system.

For example, without kernel-mode CET, an attacker with arbitrary kernel-mode read and write primitives can corrupt kernel-thread stack and perform ROP (TBD ref). Without CFG, any unprotected function pointers can be updated to point to arbitrary code,


## Final thoughts





