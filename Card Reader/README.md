power management needs looked into for d3cold 
disabling d3cold is a bandaid fix need to find root cause

This turned into FULL AI PROJECT as i got lost and it seems a bios or a firmware update may just fix this issue.

## AI's Project Mess

# Handoff: Realtek RTS525A PCIe card-reader D3cold resume failure — ASUS GX651AR / Fedora 44

**Status date:** 2026-10-22  
**Purpose:** Transfer this investigation to another kernel/ACPI engineer without repeating earlier blind kernel builds or already-completed checks.

## 1. Problem statement

On an ASUS ROG Zephyrus Duo GX651AR running Fedora 44, the Realtek RTS525A PCIe card reader fails to resume after D3cold. The device becomes inaccessible at PCI configuration space level; `lspci` has shown `!!! Unknown header type 7f`. Preventing D3cold prevents the failure. The reader is supported by `rtsx_pci`, and the user wants a permanent, reproducible root-cause fix, not a permanent D3cold-disable quirk or repeated test-kernel builds.

## 2. Hardware and topology

- Laptop: ASUS ROG Zephyrus Duo GX651AR.
- Firmware: ASUS BIOS GX651AR.309 (updated previously; boot-freeze improvement was being monitored separately).
- OS/kernel at the latest ACPI trace: Fedora 44, `7.2.11-200.fc44.x86_64`.
- Card reader: `0000:57:00.0`, Realtek `10ec:525a`, ASUS subsystem `1043:202f`, driver `rtsx_pci`.
- Parent PCIe Root Port: `0000:00:1c.0`, Intel `8086:e33a`; ACPI name `\_SB.PC00.RP03`, `_ADR = 0x001C0000`; secondary bus 57.
- The ACPI field layout in RP03 reads DLL Active (`LASX`) from PCIe Link Status offset `0x52`, bit 13, and reads firmware-specific LTSSM byte `LTSM` at offset `0x32B`.
- RP03 ACPI register mapping method `PC2M(Arg0, Arg1)` computes the address from `GPCB()`, bus argument, `_ADR` device/function fields. `PBNU` inside RP03 is `Zero`, consistent with bus 0 for RP03.

## 3. Reproduced symptoms / known findings

1. D3cold is the failure trigger. Keeping the reader out of D3cold prevents the issue.
2. After the failure, PCI configuration space is invalid (`lspci -vv -s 57:00.0` reports `!!! Unknown header type 7f`).
3. Kernel log includes:
   - `pcieport 0000:00:1c.0: Data Link Layer Link Active not set in 100 msec`
   - PCI core reports ACPI power state changes to D3cold then D0 in earlier traces.
   - `ACPI _REG disconnect evaluation failed (5)` / `ACPI _REG connect evaluation failed (5)` were present. These errors are not independently proven to be causal.
   - After power-up, PCI core reported readiness `0ms after power up D3hot->D0uninitialized`, then restored config and eventually timed out waiting for an active downstream link.
4. After a failed attempt, `setpci -s 00:1c.0 52.w 328.l` returned:
   ```text
   1011
   13080000
   ```
   `0x1011` has bit 13 clear, so DLL Active is not set. In little-endian dword `0x13080000` at offset `0x328`, the byte at offset `0x32B` is `0x13`.
5. ACPI probe results during the power-on method show that the clock override write takes effect and both `LASX` and `LTSM` remain steady for the sample interval, **before the following `L23D()` method**:
   - At 06:20:27: clock register before/after `0x10A`; reset released; `LASX=1`, `LTSM=0x34`; ten samples over 500 ms remained `LASX=1`, `LTSM=0x34`.
   - At 06:20:50: clock register `0x102` before and `0x10A` after; reset released; `LASX=1`, `LTSM=0x48`; ten samples over 500 ms remained `LASX=1`, `LTSM=0x48`.
   - At 06:20:52: Linux reports `Data Link Layer Link Active not set in 100 msec`.
   These samples establish that ACPI sees the port as link-active during `PON()` sampling, but do not prove the link has completed the following recovery sequence or remains active when Linux checks it.

## 4. Relevant ACPI source and sequence

Working source: `~/acpi-work/dsdt-probe2.dsl`.

- RP03 scope starts at line ~16082; its power resource is around line ~71230 in the full source (line numbers can shift as edits are made).
- `PowerResource (RPR2, 0x00, 0x0000)` is defined under RP03.
- `RPR2._ON`:
  ```asl
  SHPO (WAKG, One)
  PON ()
  L23D ()
  ```
- `RPR2._OFF`:
  ```asl
  FLGW = WKEN
  DL23 ()
  POFF ()
  ```
- RP03 `_PR0`, `_PR2`, and `_PR3` each contain `RPR2`; `_S0W = 0x03`.
- `PON()` optionally waits using `PWRG` / `WOFF`, enables clock via `SPCO(SCLK, One)`, releases reset via `\PIN.OFF(RSTG)`, then returns. The investigation added temporary `RTS-PROBE` direct `Debug` statements around the clock write and around the reset-release/link state. Those direct messages do show in the kernel log.
- `POFF()` asserts reset using `\PIN.ON(RSTG)`, disables the clock if applicable, and updates wake/power bookkeeping.
- **RP03's actual `L23D()` is at original source line ~16206, not the method around 16891 (that one is RP04).** Current method logic:
  ```asl
  ADBG (Concatenate ("PCIe RP LTSSM at L23D Start  -", ToHexString (LTSM)))
  If ((SCB0 != One)) {
      ADBG (Concatenate ("PON SCB ", ToHexString (SLOT)))
      Return (Zero)
  }
  L23R = One
  Local0 = Zero
  While (L23R) {
      If ((Local0 > 0x04)) {
          ADBG (Concatenate ("PCIE L23R TO ", ToHexString (SLOT)))
          Break
      }
      Sleep (0x10)
      Local0++
  }
  SCB0 = Zero
  Local0 = Zero
  While ((LASX == Zero)) {
      If ((Local0 > 0x08)) {
          ADBG (Concatenate ("PCIE LASX TO ", ToHexString (SLOT)))
          Break
      }
      Sleep (0x10)
      Local0++
  }
  ADBG (Concatenate ("PCIe RP LTSSM at L23D End -", ToHexString (LTSM)))
  ```
- In RP03 `PXCS`, `SCB0` is at offset `0xE0`, bit 0; `L23R` is at offset `0xE2`, bit 3; `LTSM` is at offset `0x32B`, 8 bits. The field `LASX` is bit 13 at offset `0x52`.
- The main unresolved question is what `L23D()` observes and does after `PON()` returns. It may be skipped if `SCB0 != 1`, may time out waiting for `L23R` to clear, or may observe/stage a transient `LASX` state that is lost by the time Linux polls the link. None of these outcomes has yet been demonstrated.
- Earlier kernel log filtering did not show `L23D`'s `ADBG` strings, but that absence is inconclusive by itself; direct `Debug` instrumentation was added to RP03's `L23D()` to settle this.

## 5. Current diagnostic artifacts and exact state

The latest step has **already installed a new DSDT-TEST diagnostic initramfs**. No kernel was built for this step; normal Fedora initramfs and GRUB entries were not changed.

- Source: `~/acpi-work/dsdt-probe2.dsl`
- Pre-instrumentation source backup: `~/acpi-work/dsdt-probe2-before-l23d-probe.dsl`
- Compiled AML: `~/acpi-work/dsdt-probe2.aml`
- AML compiler result after instrumentation: `Compilation successful. 0 Errors, 94 Warnings, 655 Remarks, 546 Optimizations, 2 Constants Folded`; output AML size reported 409,313 bytes.
- Staged AML: `~/acpi-work/l23d-overlay/kernel/firmware/acpi/dsdt.aml`
- Replacement cpio: `~/acpi-work/acpi-override-l23d.cpio`; contains exactly:
  ```text
  kernel
  kernel/firmware
  kernel/firmware/acpi
  kernel/firmware/acpi/dsdt.aml
  ```
- SHA-256 of the instrumented `dsdt-probe2.aml` and AML extracted from the new overlay both matched:
  `920839282596956c5144c3d4d96adfd4bcddafb5a564436a9d90f9017a58e9fb`
- SHA-256 of the replacement overlay archive: `f18f972d9b46e7bbe2ea213b245f3e0a0d94c3628e9bc5b7e5fe1ab00b2ab3da`.
- The `.new` initramfs was assembled by prefixing that overlay cpio to the original Fedora initramfs. Both checks were verified byte-for-byte: overlay prefix hash matched; original base initramfs tail hash matched.
- Original stock Fedora image used as base: `/boot/initramfs-7.2.11-200.fc44.x86_64.img`; SHA-256 `c19c095bf077615cd589fe3ad44764f40e9a6852a0b55674c06bdbcfd338b843`.
- Active diagnostic image now at `/boot/initramfs-7.2.11-200.fc44.x86_64-dsdt.img` (149M; moved into place at about 07:11 on Oct 22).
- Rollback copy of the former diagnostic image: `/boot/initramfs-7.2.11-200.fc44.x86_64-dsdt.img.before-l23d` (149M; former probe2 image, from 06:16).
- Older diagnostic backup: `/boot/initramfs-7.2.11-200.fc44.x86_64-dsdt.img.before-probe2` (149M; from Oct 21 04:53).
- DSDT-TEST GRUB entry is already configured to use `/boot/initramfs-7.2.11-200.fc44.x86_64-dsdt.img` and kernel `7.2.11-200.fc44.x86_64`, with `acpi.debug_layer=0xffffffff acpi.debug_level=0x2`. No GRUB edit is required.
- The last shell output showed the active image and both backups in place. The latest image has **not yet been booted** after insertion of `RP03-L23D` instrumentation.

## 6. Next decisive collection (one boot, no kernel build)

If a follow-up test is acceptable, boot **DSDT-TEST** once, reproduce the card-reader D3cold failure/resume, then collect this log:

```bash
sudo journalctl -k -b --no-pager | grep -E \
'RP03-L23D|RTS-PROBE|PCIe RP LTSSM at L23D|PCIE L23R TO|PCIE LASX TO|Data Link Layer Link Active not set|power state changed by ACPI|_REG.*evaluation failed'
```

Also record:

```bash
uname -r
cat /proc/cmdline
```

Interpretation:
- `RP03-L23D SKIPPED SCB0=...`: firmware skipped recovery because `SCB0` was not set. Investigate why `_OFF`/`DL23` did not leave the required sticky bit set across D3cold, including whether ACPI power-resource transitions match the real hardware power state.
- `RP03-L23D after-request L23R=1` plus `PCIE L23R TO`: firmware's request did not clear; target the root-port/L23 request handshake or its power/clock ordering, not an arbitrary Linux delay.
- `L23R` clears, but `LASX` waits/times out: link retraining fails within ACPI method; inspect `LTSM` transition and hardware power/reset/clock state.
- `L23D` exit shows `LASX=1` but Linux still says link-active not set: the port loses active state between ACPI return and Linux poll; inspect immediately subsequent root-port power/config changes and Linux PM ordering.
- No `RP03-L23D` messages despite `RTS-PROBE` messages: check whether the DSDT override actually loaded and whether the instrumented RP03 `L23D` method is the one invoked. Confirm the OEM revision / direct probe marker; do not infer failure state from the absence of old `ADBG` messages alone.

## 7. Prior fixes and failed experiments — do not repeat blindly

- Existing project workaround patch `95a06a7ca1ad` disables D3cold for RTS525A on ASUS GX651AR. It prevents the failure but is not accepted as the preferred root fix.
- An earlier attempt added an `acpi_device_set_power(ACPI_COMPANION(&pci_dev->dev), ACPI_STATE_D0)` call in `drivers/pci/pci-driver.c` around `pci_pm_bridge_power_up_actions()` before `pci_bridge_wait_for_secondary_bus()`, constrained to Intel RP03 `00:1c.0` (`8086:e33a`). It was built under kernel release `7.2.5-gx651ar-rp03-test-gx651ar-rp03-rootfix` but did not boot. Do not repeat without diagnosing why that candidate failed to boot.
- User reports approximately 30 installed kernels, around 15 test kernels, after repeated exploratory builds. Avoid creating new kernel builds until the ACPI probe gives a specific result. Avoid cleanup/deletion unless explicitly requested; preserve currently working boot entries.
- Repeated checksum comparisons are already complete and passed. Do not repeat them unless the image is changed.
- Clock-write success already demonstrated (`0x102` to `0x10A` on the second PON); more clock-only experiments are low value.
- `LASX=1` and stable LTSM during `PON()` for 500 ms already demonstrated; more sampling exclusively inside `PON()` is low value. The unknown transition lies after `PON()`, around `L23D()`/handoff to Linux.

## 8. Kernel PCI source context

Linux's `pci_bridge_wait_for_secondary_bus()` is the code that checks the downstream link after D3cold-to-D0/recovery, and the PCI resume path may mark subordinate devices disconnected when the link is not active. Source references:

- https://code.googlesource.com/linux/torvalds/linux/%2B/b8910630c967ffee582289451ddb5f9f19c26872/drivers/pci/pci.c
- https://github.com/torvalds/linux/blob/master/drivers/pci/pci-driver.c

A 2026-08-31 upstream patch discussion says `pci_pm_bridge_power_up_actions()` is run for bridges returning from D3cold as part of noirq resume so mandatory post-power-on link-train waits occur before accessing downstream devices. This is context, not proof of the reader's root cause: https://lists.openwall.net/linux-kernel/2026/08/31/3252

## 9. Current conclusion

The evidence localizes the failure to the transition between the ACPI RP03 power-on sequence and Linux's post-D3cold link check. It does **not yet prove** whether firmware's `L23D()` handshake is skipped/failing or whether the link drops after the method returns. A root-cause fix cannot be responsibly claimed from the currently captured pre-`L23D()` samples alone. The newly installed diagnostic image adds direct `Debug` instrumentation to answer this with one targeted boot and does not require another kernel build.
