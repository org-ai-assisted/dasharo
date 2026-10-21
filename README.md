# dasharo -- coreboot firmware for the VirtualBox machine

Our fork/configuration work to run real **coreboot** (and, as the next step, full
**Dasharo** = coreboot + `DasharoPayloadPkg`) as the firmware of an Oracle
VirtualBox guest.

This is the firmware side of a two-part effort. The VirtualBox side -- a small
`DevEFI` patch that lets VirtualBox load a raw coreboot image at all -- lives in the
companion repo `org-ai-assisted/virtualbox`. Neither half is useful without the
other.

This repo carries our delta and build recipe, not a full coreboot source mirror.

## Background

Dasharo is a coreboot distribution: coreboot does the platform init, and
`DasharoPayloadPkg` (a `UefiPayloadPkg` fork built as a coreboot payload) provides
the UEFI environment, Secure Boot, and the setup UI. It is **not** a standalone OVMF
build. VirtualBox's EFI slot only accepts an OVMF-shaped firmware volume, so running
coreboot there required the VirtualBox-side loader patch first (see the `virtualbox`
repo). With that patch in place, this repo is about producing a coreboot image that
initializes VirtualBox's emulated chipset.

## Current state: upstream coreboot `emulation/qemu-i440fx`, proven booting

VirtualBox emulates the same i440fx/PIIX3 chipset QEMU does, so upstream coreboot's
`emulation/qemu-i440fx` mainboard runs on it directly once VirtualBox can load the
image. `configs/config.vbox-i440fx` is the coreboot defconfig that was built and
proven booting inside the patched VirtualBox.

Key config choices:

- `CONFIG_MAINBOARD_DIR="emulation/qemu-i440fx"` -- matches VirtualBox's PIIX3.
- `CONFIG_COREBOOT_ROMSIZE_KB_4096` -- 4 MiB; the VirtualBox loader map loop caps the
  image just under 8 MiB, so 4 MiB is the safe size.
- `CONFIG_CONSOLE_SERIAL` -- COM1 log, so the boot can be captured headless.
- `CONFIG_PAYLOAD_NONE` -- in this defconfig, no payload; this de-risks the firmware
  init on its own. coreboot stops after device enumeration. A real `DasharoPayloadPkg`
  payload was built and integrated separately (see "Real DasharoPayloadPkg" below).

VirtualBox has no QEMU `fw_cfg` interface, so coreboot takes its documented CMOS
fallback for RAM sizing -- visible in the serial log:

```
QEMU: firmware config interface not found
QEMU: Falling back to RAM info in CMOS
```

## Build

coreboot's bundled toolchain is required; a host "any toolchain" fails to link the
32-bit stages (`undefined reference __udivdi3`, even with gcc-multilib):

```
cd coreboot
make crossgcc-i386 CPUS=8 BUILD_LANGUAGES=c        # ~7 min, builds util/crossgcc/xgcc
cp /path/to/configs/config.vbox-i440fx .config
make olddefconfig
make -j"$(nproc)"                                   # -> build/coreboot.rom (4 MiB)
```

Then load `build/coreboot.rom` in the patched VirtualBox per the `virtualbox` repo's
README (`EfiRom=<rom>` + `CorebootRom=1`).

## Proof

`proof/coreboot-on-vbox-serial.log` is the COM1 capture from the proven run
(OVH, Debian trixie, 2026-10-21). coreboot ran all stages
(bootblock -> romstage -> postcar -> ramstage) and enumerated VirtualBox's own PCI
devices:

```
PCI: 00:00:00.0 [8086/1237] enabled     # i440fx host bridge (VBox PIIX3)
PCI: 00:00:02.0 [80ee/beef] enabled     # VirtualBox VGA
PCI: 00:00:04.0 [80ee/cafe] enabled     # VirtualBox guest-service device
```

The corresponding VirtualBox log line was
`EFI: coreboot image, 0x400000 bytes mapped at 0xffc00000..4GB`.

## Real DasharoPayloadPkg + functional ACPI (2026-10-21)

A real `DasharoPayloadPkg` UEFI payload was built from `Dasharo/edk2`
(`-D BOOTLOADER=COREBOOT -D TPM_ENABLE=FALSE -D SECURE_BOOT_ENABLE=FALSE`
-> `UEFIPAYLOAD.fd`) and integrated into the 4 MiB ROM with
`cbfstool add-payload -f UEFIPAYLOAD.fd -n fallback/payload -c lzma` (the whole FV
LZMA-packs to ~910 KiB; load `0x800000`, entry `0x800860`).

### Gap 1 -- ramstage ACPI S5 -- SOLVED, with functional ACPI

Early on, with the guest's ACPI device enabled, coreboot powered the VM off during
ramstage. Traced on the wire (a temporary `LogRel` in VirtualBox's PM1a-control
handler): coreboot's PIIX4 southbridge writes `SUS_TYP_S0 | SCI_EN` = `0x1401` to
PM1a_CTL -- `SLP_TYP = 5`, `SLP_EN` (bit 13) **clear** -- a harmless "arm S0 + enable
SCI" write on real PIIX4. VirtualBox's `acpiR3PM1aCtlWrite` keyed the sleep state off
`SLP_TYP` **alone** and read 5 as S5, powering off. The fix (companion `virtualbox`
repo) gates the transition on `SLP_EN` per ACPI 1.0 4.7.2.2.1. With it, coreboot runs
to completion with ACPI **enabled**, writes its own ACPI tables, and reaches
`Jumping to boot code at 0x800860`. ACPI is **functional**, not merely disabled:
VirtualBox relocates its PM I/O decode from coreboot's `PMBA` config write, so the
FADT and the emulated ports line up, and a real shutdown (`SLP_TYP=5` with `SLP_EN`)
still powers off. (`--acpi off` also lets coreboot boot, but leaves ACPI non-functional
-- the `SLP_EN` fix is what makes it both present and working.)

### Gap 2 -- UEFI payload to an interactive shell -- SOLVED

With gap 1 fixed, a real coreboot `UefiPayloadPkg` (the classic, not universal, build
-- `CONFIG_PAYLOAD_EDK2` + `CONFIG_EDK2_UEFIPAYLOAD`, MrChromebox fork) now boots all
the way to an **interactive UEFI shell** inside the patched VirtualBox, with ACPI
functional. See `proof/uefi-shell-in-vbox.log`:

```
UEFI Interactive Shell v2.2
EDK II
UEFI v2.70 (EDK II, 0x00010000)
Welcome to the UEFI Shell!
Shell>
```

Getting there meant closing a chain of VBox-vs-QEMU/real-hardware differences the
vanilla `emulation/qemu-i440fx` board does not paper over. Each has a one-line fix:

1. **48-bit paging hang.** The payload identity-maps the full physical space; VBox
   passes the host's 48-bit width and masks the 1 GiB-page CPUID bit, so the map needs
   ~1 GiB of 2 MiB-page tables and wedges. Fix: report 36 physical address bits to the
   guest -- `VBoxManage modifyvm <vm> --cpuid-set 0x80000008 0x00003024 ...`.
2. **No HPET.** `HpetTimerDxe` asserts (`mHpetGeneralCapabilities == 0xFFFF...`). Fix:
   `VBoxManage modifyvm <vm> --hpet on`.
3. **HPET interrupt never fires.** Even present, VBox's HPET does not deliver the timer
   interrupt, so DXE's tick stalls. Fix: build the payload with the LAPIC timer instead
   -- `CONFIG_EDK2_USE_LAPIC_TIMER=y` (the LAPIC is on the CPU, handled by AMD-V).
4. **#GP in `PciHostBridgeDxe`.** coreboot emits an entryless MCFG (i440fx has no ECAM,
   and there is no `fw_cfg` to read QEMU's), and the payload reads past it into the
   adjacent "APIC" table as a bogus ECAM base. Fix: in coreboot `src/acpi/acpi.c`
   `acpi_create_mcfg`, skip emitting the MCFG when there are no entries (the memset-zeroed
   header then has `length < min_size` and the table loop drops it); the payload falls
   back to legacy PCI config. The one added guard, right after `current` is set:
   ```c
   if (!CONFIG(ECAM_MMCONF_SUPPORT) && !CONFIG(ACPI_CUSTOM_MCFG))
           return;
   ```
5. **Serial console + no DEBUG asserts.** Build RELEASE with `CONFIG_EDK2_SERIAL_SUPPORT=y`
   so the shell appears on COM1 (a DEBUG build also trips a TPM physical-presence
   `ASSERT` that is compiled out in RELEASE).

### Functional ACPI, proven end to end

With ACPI enabled, typing `reset -s` at the shell (an ACPI S5 shutdown: `SLP_TYP=5`
written **with** `SLP_EN`) powers the VM off cleanly -- `VBox.log` logs
`ACPI: Entering S5 power state`, and the VM transitions to OFF on its own -- while
coreboot's harmless boot-time `SLP_TYP=5 / SLP_EN=0` write is ignored. That is both
halves of the `SLP_EN` fix working.

## Reproduce (full recipe)

coreboot `.config` (on top of `configs/config.vbox-i440fx`): `CONFIG_PAYLOAD_EDK2=y`,
`CONFIG_EDK2_UEFIPAYLOAD=y`, `CONFIG_EDK2_USE_LAPIC_TIMER=y`, `CONFIG_EDK2_SERIAL_SUPPORT=y`,
`CONFIG_EDK2_RELEASE=y`; apply `patches/coreboot-skip-empty-mcfg.patch`; `make`.

Run in the patched VirtualBox (both `DevEFI` CorebootRom and `DevACPI` SLP_EN fixes):

```
VBoxManage modifyvm <vm> --firmware efi --chipset piix3 --hpet on --memory 2048
VBoxManage modifyvm <vm> --cpuid-set 0x80000008 0x00003024 0x791ef257 0x0000400f 0x00010000
VBoxManage modifyvm <vm> --uart1 0x3f8 4 --uartmode1 file <serial.log>
VBoxManage setextradata <vm> "VBoxInternal/Devices/efi/0/Config/EfiRom"      "<coreboot.rom>"
VBoxManage setextradata <vm> "VBoxInternal/Devices/efi/0/Config/CorebootRom" 1
```

## Next step: the real Dasharo payload + a first-class board

This proves the coreboot + UEFI-payload architecture and functional ACPI in VirtualBox
with upstream `UefiPayloadPkg`. The remaining polish is the **Dasharo** payload
specifically (`DasharoPayloadPkg` needs `-D CPU_TIMER_LIB_ENABLE=FALSE` plus the same
LAPIC/36-bit/MCFG treatment; its PEI ACPI-board-info HOB ordering still needs work),
and folding the five VM/build tweaks into a dedicated coreboot
`src/mainboard/emulation/qemu-vbox` board (VBox memory map, no `fw_cfg`, LAPIC timer,
no empty MCFG) plus a `configs/config.emulation_vbox`, so a stock VM boots it with no
per-VM `--cpuid-set`/`--hpet`. coreboot carries a Bhyve precedent for adding an
emulator board without `#ifdef`-ing shared code.

## License

coreboot and Dasharo are Free Software (GPLv2). This repo's configuration and build
recipe are offered under the same terms.
