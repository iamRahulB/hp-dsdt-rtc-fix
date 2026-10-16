# HP Laptop 15-bs0xx — ACPI DSDT RTC/SystemCMOS Fix

## Overview

Machine-specific ACPI DSDT correction for an HP Laptop 15-bs0xx running Ubuntu 24.04 LTS.

The original HP firmware DSDT caused repeated ACPI failures involving the `SystemCMOS` operation region (`RCM0`), `ESDT`, `_Q42`, and related AML methods. Investigation traced the failure to the `PNP0B00` RTC device: its `_STA` behavior caused Linux to treat the RTC as unavailable while other firmware methods expected the CMOS/SystemCMOS path to exist.

The DSDT was corrected so the RTC is reported as present, then supplied to Linux through the ACPI initrd table-override mechanism.

This is a functional DSDT-level correction of the firmware behavior, not log suppression. The BIOS itself is not modified.

---

## Hardware and Software

- Laptop: HP Laptop 15-bs0xx
- CPU: Intel Core i3
- iGPU: Intel Skylake GT2 / Intel HD Graphics 520
- Wi-Fi: Realtek RTL8723DE
- BIOS: F.75
- BIOS date: 2024-04-26
- Architecture: x86-64
- OS: Ubuntu 24.04 LTS
- Working kernel: `6.14.0-37-generic`
- Untouched fallback: `7.0.0-38-generic`

---

# Original Problem

The original firmware generated repeated ACPI errors, including:

```text
ACPI Error: No handler for Region [RCM0]
ACPI Error: Region SystemCMOS (ID=5) has no handler
ACPI Error: Aborting method \_SB.WMID.ESDT
ACPI Error: Aborting method \_SB.PCI0.LPCB.EC0._Q42
```

On some tested kernels:

```text
ACPI BIOS Error (bug): Attempt to CreateField of length zero
WQBC
WQBD
WQBE
```

The behavior persisted across multiple kernels:

- `6.14.0-37-generic`
- `7.0.0-38-generic`
- `6.17.0-42-generic`
- `6.8.0-146-generic`
- `7.0.0-34-generic`

Changing kernels did not remove the underlying firmware ACPI problem.

---

# Root Cause Investigation

The DSDT was extracted and decompiled rather than simply suppressing the messages.

The firmware exposes:

```text
PNP0B00
```

at:

```text
\_SB_.PCI0.LPCB.RTC_
```

Its `_STA` behavior caused Linux to treat the RTC as unavailable.

The same firmware defines a SystemCMOS operation region:

```text
RCM0
```

and WMID/EC methods access that region.

The resulting failure chain was:

```text
HP firmware DSDT
      |
      +-- PNP0B00 RTC reports unavailable
      |
      v
Linux does not establish the expected ACPI RTC/CMOS path
      |
      v
SystemCMOS / RCM0 has no appropriate handler
      |
      v
ESDT / _Q42 and related AML methods fail
      |
      v
Repeated ACPI errors
```

This identified a specific firmware-table cause instead of treating the errors as unrelated symptoms.

---

# The Fix

The DSDT was modified so that the `PNP0B00` RTC device is reported as present.

The resulting table is:

```text
dsdt-patched.aml
```

Conceptually:

```text
Original:
PNP0B00 RTC -> unavailable

Patched:
PNP0B00 RTC -> present
```

This allows Linux to establish the RTC/CMOS path expected by the firmware's other ACPI methods.

The functional chain becomes:

```text
Corrected DSDT
      |
      v
PNP0B00 RTC available
      |
      v
rtc_cmos binds
      |
      v
Expected CMOS/SystemCMOS handling
      |
      v
Dependent ACPI methods execute
      |
      v
RCM0 / SystemCMOS / ESDT / _Q42 failures disappear
```

---

# Why This Is Not Error Suppression

The patch does not filter or hide kernel messages.

The actual ACPI table used by Linux is replaced during boot. The kernel confirmed:

```text
ACPI: DSDT ACPI table found in initrd [kernel/firmware/acpi/dsdt.aml]
ACPI: Table Upgrade: override [DSDT-HPQOEM-SLIC-MPC]
ACPI: DSDT ... Physical table override
```

The RTC also successfully bound to `rtc_cmos`.

Therefore this changes actual ACPI behavior rather than merely changing what appears in `dmesg` or `journalctl`.

---

# Linux ACPI Initrd Override

Linux supports replacing ACPI tables through the initramfs.

The patched DSDT is stored in the early CPIO portion at:

```text
kernel/firmware/acpi/dsdt.aml
```

The working initramfs therefore contains:

```text
initrd.img-6.14.0-37-generic
└── early CPIO
    └── kernel/
        └── firmware/
            └── acpi/
                └── dsdt.aml
```

Linux loads this replacement before normal ACPI initialization.

Reference:

https://docs.kernel.org/admin-guide/acpi/initrd_table_override.html

---

# Repository Files

```text
DSDT-original.dat
DSDT-original.dsl
dsdt-patched.aml
initrd.img-6.14.0-37-generic.patched
README.md
SHA256SUMS
```

`DSDT-original.dat` is the extracted original DSDT binary.

`DSDT-original.dsl` is the decompiled original DSDT source.

`dsdt-patched.aml` is the corrected DSDT.

`initrd.img-6.14.0-37-generic.patched` is the complete tested initramfs containing the DSDT override.

---

# Integrity Verification

Run:

```bash
sha256sum -c SHA256SUMS
```

Verified hashes:

```text
b7e303da44b90a6836306eb3de107ac4bb44e0c91cd0e06874daa6070f816953  DSDT-original.dat
5b996faa50456d3a363214b31c83a8cd7f4e5a0a865740dd33cb278c41020d34  DSDT-original.dsl
2905253c1c033c2ef341c64b6f231c7b4574a857779dcd154187f090a52fc1f3  dsdt-patched.aml
648833e6ea525ede67611e48e5b9ade8b7caa3a35a5f9f533883e2344454dd22  initrd.img-6.14.0-37-generic.patched
```

---

# Verify the Patched Initramfs

```bash
lsinitramfs initrd.img-6.14.0-37-generic.patched | grep 'kernel/firmware/acpi/dsdt.aml'
```

Expected:

```text
kernel/firmware/acpi/dsdt.aml
```

---

# Installing the Tested Initramfs

WARNING: The AML is specific to the tested platform and firmware. Do not blindly use it on unrelated machines.

Back up the current initramfs:

```bash
sudo cp /boot/initrd.img-6.14.0-37-generic /boot/initrd.img-6.14.0-37-generic.backup
```

Install the tested image:

```bash
sudo cp initrd.img-6.14.0-37-generic.patched /boot/initrd.img-6.14.0-37-generic
```

Verify:

```bash
sudo lsinitramfs /boot/initrd.img-6.14.0-37-generic | grep 'kernel/firmware/acpi/dsdt.aml'
```

Then reboot:

```bash
sudo reboot
```

---

# Verify the Running Kernel

```bash
uname -r
```

Expected:

```text
6.14.0-37-generic
```

The untouched fallback is:

```text
7.0.0-38-generic
```

---

# Verify the DSDT Override

```bash
journalctl -k -b | grep -iE 'DSDT.*initrd|Table Upgrade|Physical table override'
```

Expected messages include:

```text
ACPI: DSDT ACPI table found in initrd [kernel/firmware/acpi/dsdt.aml]
ACPI: Table Upgrade: override [DSDT-HPQOEM-SLIC-MPC]
ACPI: DSDT ... Physical table override
```

---

# Verify RTC Binding

```bash
cat /sys/class/rtc/rtc0/name
```

The tested system reported:

```text
rtc_cmos 00:03
```

Then:

```bash
readlink -f /sys/class/rtc/rtc0/device/driver
```

The tested system reported:

```text
/sys/bus/pnp/drivers/rtc_cmos
```

---

# Verify the Original ACPI Error Chain

```bash
journalctl -k -b | grep -iE 'RCM0|SystemCMOS|WQBC|WQBD|WQBE|ESDT|_Q42|CreateField|ACPI (Error|BIOS Error)|AE_[A-Z_]+'
```

For an explicit result:

```bash
journalctl -k -b | grep -iE 'RCM0|SystemCMOS|WQBC|WQBD|WQBE|ESDT|_Q42|CreateField|ACPI (Error|BIOS Error)|AE_[A-Z_]+' || echo "ALL CLEAN"
```

Expected:

```text
ALL CLEAN
```

---

# Verification Results

The patch was verified on the tested system.

### DSDT override

Confirmed by the kernel:

```text
ACPI: DSDT ACPI table found in initrd
ACPI: Table Upgrade: override [DSDT-HPQOEM-SLIC-MPC]
ACPI: DSDT ... Physical table override
```

### RTC

`rtc_cmos` successfully bound.

### Original ACPI errors

The following were absent after the patch:

```text
RCM0
SystemCMOS
ESDT
_Q42
WQBC
WQBD
WQBE
CreateField
ACPI Error
ACPI BIOS Error
AE_*
```

The targeted check returned:

```text
ALL CLEAN
```

---

# Reboot Test

After a normal reboot:

```text
uname -r
6.14.0-37-generic
```

The targeted ACPI check again returned:

```text
ALL CLEAN
```

The DSDT override was again confirmed in the boot log.

---

# Forced Power-Off Test

The system was also tested after a forced power-off followed by normal power-on.

After the system returned:

```text
uname -r
6.14.0-37-generic
```

and the ACPI check again returned:

```text
ALL CLEAN
```

This confirms that the patch remains active after an unclean shutdown and subsequent boot.

WARNING: Forced power-off can cause filesystem or data corruption and should not be used routinely.

---

# Thermal and Fan Verification

The thermal subsystem was checked after applying the patch.

Observed temperatures were approximately:

```text
CPU Package: 47 C
Core 0:       45 C
Core 1:       47 C
PCH:          45.5 C
ACPI TZ00:    27.8 C
ACPI TZ01:    29.8 C
ACPI TZ02:    10.0 C
```

The kernel registered the ACPI thermal zones normally.

A thermal/fan-related kernel check showed normal thermal initialization and no fan failure message.

This indicates that the DSDT correction did not break thermal handling.

---

# Broad Kernel Health Check

For serious kernel messages:

```bash
journalctl -k -b -p 0..3 --no-pager
```

This covers:

```text
0 Emergency
1 Alert
2 Critical
3 Error
```

On the tested boot, the only returned line was:

```text
x86/cpu: SGX disabled or unsupported by BIOS.
```

This is an unrelated BIOS capability message, not part of the ACPI failure chain.

A broader priority check also showed unrelated security/configuration warnings such as MDS, MMIO Stale Data, VMSCAPE, HPET, Intel virtual-button, and networking/power-management messages.

These are separate from the `RCM0/SystemCMOS` issue.

---

# Persistence and Initramfs Updates

The source DSDT:

```text
~/dsdt2/dsdt-rev.aml
```

is not directly loaded by Linux.

The boot-time copy is embedded inside:

```text
/boot/initrd.img-6.14.0-37-generic
```

at:

```text
kernel/firmware/acpi/dsdt.aml
```

A normal reboot does not remove it.

However, a kernel or initramfs update may regenerate the initramfs and replace the manually patched image.

Before modifying or regenerating the initramfs, preserve the known-good image:

```bash
sudo cp /boot/initrd.img-6.14.0-37-generic /boot/initrd.img-6.14.0-37-generic.working
```

Do not blindly run:

```bash
update-initramfs -u
```

against the patched kernel unless the early-CPIO DSDT override is automatically reinserted.

---

# Recovery

The system keeps an untouched fallback kernel:

```text
7.0.0-38-generic
```

If the patched kernel/initramfs fails, select the fallback kernel from GRUB Advanced Options.

The original DSDT is preserved as:

```text
DSDT-original.dat
DSDT-original.dsl
```

---

# Reproducing the Fix on Another Machine

Do not blindly reuse `dsdt-patched.aml` on another machine.

For another system:

1. Identify the exact laptop model.
2. Record the BIOS version.
3. Extract its own DSDT.
4. Decompile the DSDT.
5. Locate `PNP0B00`.
6. Inspect its `_STA` method.
7. Confirm the same `RCM0` / `SystemCMOS` structure.
8. Confirm the same ACPI error chain.
9. Apply the equivalent source-level correction.
10. Recompile the DSDT.
11. Put the resulting `dsdt.aml` in the early initrd CPIO.
12. Boot the matching kernel.
13. Verify the DSDT override.
14. Verify `rtc_cmos`.
15. Verify that the original ACPI errors disappear.
16. Keep an untouched fallback kernel.

---

# Scope and Compatibility

Verified configuration:

```text
HP Laptop 15-bs0xx
BIOS F.75
Ubuntu 24.04 LTS
Linux 6.14.0-37-generic
x86-64
```

The DSDT is firmware-specific.

Another HP model, BIOS revision, or hardware configuration may have a different DSDT and must be analyzed independently.

---

# What This Fix Changes

### Changes

- Corrects the DSDT RTC device availability behavior.
- Allows Linux to establish the expected RTC/CMOS path.
- Allows `rtc_cmos` to bind.
- Prevents the downstream ACPI method failures observed with the original table.

### Does not change

- BIOS firmware stored on the motherboard.
- Linux kernel source code.
- SSH configuration.
- Wi-Fi configuration.
- Network configuration.
- Hardware firmware.
- Kernel modules.

---

# Official vs Custom

The ACPI initrd table-override mechanism is supported by Linux.

This particular DSDT modification is custom and machine-specific.

It is:

- not an HP BIOS update;
- not an upstream Linux patch specifically for this laptop;
- not a kernel source modification.

The distinction is:

```text
Linux mechanism:
Officially supported ACPI table override

This DSDT:
Custom correction for the tested HP firmware
```

The motherboard BIOS itself remains unchanged.

---

# Why This Is a Root-Cause-Level Correction

The patch does not independently suppress each error.

The investigation identified a specific DSDT behavior:

```text
PNP0B00 RTC reported unavailable
```

The correction changes that firmware-table behavior.

Afterward:

```text
rtc_cmos binding                 WORKING
SystemCMOS / RCM0 errors         GONE
ESDT errors                      GONE
_Q42 errors                      GONE
WQBC/WQBD/WQBE errors            GONE
CreateField errors               GONE
```

This is why the patch is considered a DSDT-level root-cause correction for the observed firmware defect, delivered through Linux's ACPI table override mechanism.

The actual HP BIOS is not permanently modified.

---

# Current Status

```text
DSDT override loaded             YES
Patched initramfs                YES
rtc_cmos binding                 YES

RCM0 errors                     NONE
SystemCMOS errors               NONE
ESDT errors                     NONE
_Q42 errors                     NONE
WQBC errors                     NONE
WQBD errors                     NONE
WQBE errors                     NONE
CreateField errors              NONE
ACPI Error messages             NONE
ACPI BIOS Error messages        NONE

Thermal subsystem                WORKING
Normal reboot test               PASSED
Forced power-off test            PASSED
```

---

# Conclusion

The original ACPI problem was traced to the HP firmware DSDT.

The DSDT correction makes the `PNP0B00` RTC device available to Linux, allowing the expected RTC/CMOS handling to be established.

The corrected table is loaded through Linux's ACPI initrd override mechanism.

The resulting system was verified across normal reboot and forced power-off testing, with the original ACPI error chain remaining absent.

The actual HP BIOS is not modified.

This repository preserves the original DSDT, corrected DSDT, tested initramfs, documentation, and integrity information required to reproduce and study the fix.
