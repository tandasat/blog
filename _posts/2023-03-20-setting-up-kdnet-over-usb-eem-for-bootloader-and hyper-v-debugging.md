---
layout: post
title:  "Setting up KDNET over USB EEM for Bootloader and Hyper-V debugging"
date:   2023-03-20 17:10:30 -0700
categories: windows
---

This post notes how to enable a debugger for winload, tcblaunch and Hyper-V on a physical device over USB EEM. This instruction may be helpful when a target device cannot be debugged with any of other debugging interfaces like traditional KDNET and USB3.

![](/blog/img/posts/2023-03-20/debugging.jpg)

## No USB3 or KDNET working
I bought a Dell Latitude 7330 2-in-1 which supported various security features like PPAM and DRTM and wanted to debug interaction between Windows and relevant components.

As I used to for physical device debugging, I attempted to set up kernel debugging over USB3. Despite the USB ports being debug-capable according to USBView, it was not successful.

![](/blog/img/posts/2023-03-20/debug_capable.png)

I tried various combinations of USB C->C, C->A, A->C, A->A, using different hosts and different cables (including USB 2.0), and none worked. I was unable to see `USB Debug Connection Device` showing up even once. KDNET was unsupported based on KDNET.exe
```
>kdnet.exe

Network debugging is not supported on any of the NICs in this machine.
KDNET supports NICs from Intel, Broadcom, Realtek, Atheros, Emulex, Mellanox
and Cisco.
...
```

## USB EEM debugging
I moved onto USB EEM (Ethernet Emulation Model) debugging without much hope. While I had some success with it for Windows on ARM devices in the past, [the MSDN document](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/setting-up-kernel-mode-debugging-over-usb-eem-arm-kdnet) was specifically for an ARM device and my target was an Intel device. As expected, I could not make it work.

After asking several people, [@KelvinMsft](https://twitter.com/KelvinMsft) told me that USB EEM was what he used for x64 debugging as well. He was kind enough to walk though with me, and after some try and errors, boom💥, I was able to break-in to the target.

This is roughly our decision-making tree:

- Is there a debug-capable USB port exposed?
  - Y: Run KDNET.exe. Is KDNET supported on any of NICs?
    - Y: Try KDNET.
    - N: Is Windows 10 or earlier?
      - Y: Try upgrading it to Windows 11. Windows 11 supports more NICs.
      - N: Run KDNET.exe. Is KDNET supported on any of USB controllers?
        - Y: Try KD over USB EEM
        - N: Look into other unorthodox debugging.
  - N: Look into other unorthodox debugging.

Dell Latitude 7330 2-in-1's USB controllers did support KDNET:
```
>kdnet.exe
...
Network debugging is supported on the following USB controllers:
busparams=0.13.0, Intel(R) USB 3.20 eXtensible Host Controller - 1.20 (Microsoft)
busparams=0.20.0, Intel(R) USB 3.10 eXtensible Host Controller - 1.20 (Microsoft)
...
```

After double checking which USB controller corresponded to exposed USB ports, those were the `bcdedit` commands I ran for KDNET over USB EEM:
```
>bcdedit /dbgsettings net key:1.1.1.1 hostip:169.254.255.255 port:52000 busparams:0.20.0
>bcdedit /set {dbgsettings} dhcp no
>bcdedit /set loadoptions EEM
>bcdedit /set debug on

>bcdedit /dbgsettings
busparams               0.20.0
key                     1.1.1.1
debugtype               NET
hostip                  169.254.255.255
port                    52000
dhcp                    No
isolatedcontext         Yes
```
Then, made sure:
- Firewall on the host was completely disabled (it can be re-enabled later with exception rules once setup is confirmed).
- Both the host and target were connected through USB-A ports.
  - On the host, it was fine to use a C-to-A multi-function adapter.
- BitLocker was disabled. Not sure if this was required, but probably wise to do so anyway.


## Winload and tcblaunch debugging
Once kernel debugging is configured, simply enabling `bootdebug` lets the debugger break-in to Winload.exe and tcblaunch.exe at their startup.

```
>bcdedit /set debug off
>bcdedit /set hypervisordebug off

>bcdedit /set bootdebug on
```

![](/blog/img/posts/2023-03-20/winload.png)
![](/blog/img/posts/2023-03-20/tcblaunch.png)

## Hyper-V debugging
Again, once kernel-debugging is configured, simply enabling `hypervisordebug` with equivalent KDNET settings lets the debugger break-in to the Hyper-V.

```
>bcdedit /set debug off
>bcdedit /set bootdebug off

>bcdedit /hypervisorsettings net key:1.1.1.1 hostip:169.254.255.255 port:52000 busparams:0.20.0
>bcdedit /set {hypervisorsettings} hypervisordhcp no
>bcdedit /set hypervisordebug on

>bcdedit /hypervisorsettings
isolatedcontext         Yes
hypervisorbusparams     0.20.0
hypervisorusekey        1.1.1.1
hypervisordebugtype     NET
hypervisorhostip        169.254.255.255
hypervisorhostport      52000
hypervisordhcp          No
```

![](/blog/img/posts/2023-03-20/hyper-v.png)

That's it! I hope this note will help myself in the future and some others.
