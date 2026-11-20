# MacBookAir5,1 Hibernation Resume Failure

## Overview

I have been investigating intermittent hibernation resume failures on an Apple MacBook Air (11-inch, Mid 2012, model identifier MacBookAir5,1) running Void Linux.

The system occasionally performs a normal boot instead of resuming from hibernation because the EFI/E820 memory map changes the classification of a single 4 KiB physical memory page between usable RAM and ACPI Reclaim Memory.

I have reproduced the issue without GRUB and without the proprietary Broadcom wl driver, with a kernel taint value of 0.

## System Information

- Hardware: Apple MacBook Air (11-inch, Mid 2012)
- DMI product name: MacBookAir5,1
- Firmware: Apple EFI 429.0.0.0.0 (2022-03-18)
- Architecture: x86_64, 64-bit UEFI
- RAM: 8 GiB
- Distribution: Void Linux (musl)
- Affected kernel versions: 6.18.62 and 6.18.63
- Tested kernel: 6.18.63_2 (Void Linux package revision)
- Swap: 16 GiB swap file
- Hibernation mode: shutdown
- Boot methods: GRUB and direct Linux EFI stub boot

The tested 6.18.63 kernel includes a small diagnostic patch that prints the saved and current E820 CRC32 values. The patch does not modify the integrity check or the resume behavior.

## Steps to Reproduce

1. Boot the system with hibernation configured.
2. Collect the EFI and E820 memory maps with efi=debug enabled.
3. Hibernate the system using shutdown mode.
4. Power on the machine.
5. Repeat until the system performs a normal boot instead of resuming.

The issue is intermittent. Multiple consecutive successful resumes are possible.

## Observed Error

A failed resume produces the following kernel messages:

```
Hibernate E820: saved CRC32=1eebf10d, current CRC32=f70aedce
Hibernate inconsistent memory map detected!
PM: hibernation: Image mismatch: architecture specific data
PM: hibernation: resume failed (-1)
```

The failure occurs during the E820 checksum comparison in arch_hibernation_header_restore() in arch/x86/power/hibernate.c.

## E820 Memory Map Comparison

I identified two recurring E820 memory map layouts.

Layout A:

```
0x8ad53000-0x8ad64fff usable
0x8ad65000-0x8ad8efff ACPI data
```

Layout B:

```
0x8ad53000-0x8ad63fff usable
0x8ad64000-0x8ad8efff ACPI data
```

The only difference between these two E820 layouts is the classification of the following 4 KiB page:

```
0x8ad64000-0x8ad64fff
```

I independently reconstructed the E820 tables from the boot logs and calculated the following CRC32 values:

```
Layout A: f70aedce
Layout B: 1eebf10d
```

These values match the checksums printed by the diagnostic patch.

## EFI Memory Map Comparison

With efi=debug enabled, I confirmed that the corresponding change is already present in the EFI memory map.

Layout A:

```
0x8ad53000-0x8ad64fff Conventional
0x8ad65000-0x8ad8efff ACPI Reclaim
```

Layout B:

```
0x8ad53000-0x8ad63fff Conventional
0x8ad64000-0x8ad8efff ACPI Reclaim
```

Thus, the difference is present in the EFI memory map available to the kernel and is not merely an artifact of the E820 conversion.

The physical addresses of the ACPI tables reported in the boot logs, including RSDP, XSDT, FACP, DSDT, FACS, DMAR and SSDTs, remained unchanged.

No ACPI table reported in the boot logs occupies the affected 4 KiB page. However, I cannot establish whether that page contains other firmware data.

## Reproduction without GRUB

To investigate whether GRUB contributes to the memory map variation, I configured the same kernel and initramfs to boot directly through the Linux EFI stub.

All initial warm and cold boot tests through the EFI stub produced layout A.

However, the first hibernation resume attempt through the EFI stub failed with the following checksum transition:

```
Hibernate E820: saved CRC32=f70aedce, current CRC32=1eebf10d
Hibernate inconsistent memory map detected!
```

The EFI memory map changed the affected page from Conventional Memory to ACPI Reclaim Memory.

The kernel command lines before and after the failed resume were identical and contained the EFI stub-specific initrd path.

This establishes that GRUB is not necessary to reproduce the issue.

## Reproduction without the Proprietary Broadcom wl Driver

I repeated the hibernation tests using direct EFI stub boot with the following additional kernel parameter:

```
module_blacklist=wl
```

I verified that the wl module was not loaded and that /proc/sys/kernel/tainted was 0.

The same hibernation resume failure was reproduced, and both directions of the E820 checksum transition were observed:

```
f70aedce -> 1eebf10d
1eebf10d -> f70aedce
```

Therefore, the problem is reproducible without GRUB, without the proprietary Broadcom wl driver, and with an untainted kernel.

## Experimental Logs

Representative before/after logs are available
for the following test conditions.

| Boot method | wl driver | Memory map transition | Logs |
|---|---|---|---|
| GRUB | Loaded | B → A | [Logs](logs/grub-failure/) |
| EFI stub | Loaded | A → B | [Logs](logs/efi-stub-failure/) |
| EFI stub | Not loaded | A → B | [Logs](logs/efi-stub-no-wl-failure/A-to-B/) |
| EFI stub | Not loaded | B → A | [Logs](logs/efi-stub-no-wl-failure/B-to-A/) |

The tests without the Broadcom wl driver were
performed with a kernel taint value of 0.

Each directory contains the logs collected before
hibernation and after the failed resume.

## Kernel Configuration and Diagnostic Patch

- [Kernel configuration](kernel/config-6.18.63_2)
- [E820 CRC32 diagnostic patch](kernel/hibernate-e820-debug.patch)

The diagnostic patch only prints the saved and
current E820 CRC32 values. It does not modify
the integrity check or the hibernation resume behavior.

## Related Discussion

I found the following related patch proposal:

[PATCH v2 0/4] PM: hibernate: Add relaxed_memmap option for x86 E820 integrity check

https://lists.openwall.net/linux-kernel/2026/06/23/1726

However, the proposed RAM-only checksum comparison would not appear to address this particular case, because the usable RAM boundary itself changes by 4 KiB.

I understand that ignoring the E820 checksum mismatch could be unsafe, particularly when a page classified as usable RAM during image creation is classified as ACPI Reclaim Memory during resume.

## Open Questions

- Is there a recommended way to investigate why the firmware memory map changes the classification of this particular page?
- Would additional EFI or ACPI diagnostics be useful?
- Could a platform-specific workaround safely handle this type of memory map variation, for example by consistently reserving the affected page before it can be used as normal RAM?

I have not yet tested the latest unmodified mainline kernel, and I have not established whether this is a regression.

I can provide complete before/after dmesg logs, EFI and E820 memory maps, kernel configuration, module and taint information, and the diagnostic patch.

I would appreciate any suggestions for further investigation.

## Current Status

Confirmed:

- [x] Identified the affected 4 KiB physical memory page.
- [x] Confirmed the E820 CRC32 mismatch.
- [x] Independently reproduced both CRC32 values.
- [x] Confirmed the change in the EFI memory map.
- [x] Reproduced the failure without GRUB.
- [x] Reproduced the failure without the Broadcom wl driver.
- [x] Reproduced the failure with an untainted kernel.

Pending:

- [ ] Reproduction on the latest unmodified mainline kernel.
- [ ] Identification of the underlying firmware or EFI cause.
- [ ] Identification of a safe workaround or fix.

[\[BUG\] x86/hibernate: Intermittent resume failure due to a 4 KiB EFI/E820 memory map change on MacBookAir5,1](https://lore.kernel.org/linux-pm/CAKPbQspktkzF-GH9728aG0mAiQGG9NQxbO15ruvcyaQGTm2y=w@mail.gmail.com/T/#u)
