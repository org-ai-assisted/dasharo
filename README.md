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

### Gap 2 -- UEFI payload bring-up on the emulated board -- open

With gap 1 fixed, coreboot hands off cleanly, but neither payload yet reaches a shell,
for VBox-vs-QEMU platform-model differences the vanilla `emulation/qemu-i440fx` board
does not paper over:

- `DasharoPayloadPkg` resets in early PEI. Root causes found with a DEBUG payload
  (serial): first `ASSERT CpuTimerLib.c(63): RegEax != 0` -- it reads the core-crystal
  clock from CPUID leaf `0x15`, which VBox's CPU does not implement (fix:
  `-D CPU_TIMER_LIB_ENABLE=FALSE`, the same coreboot uses); then
  `ASSERT AcpiTimerLib.c(57): GuidHob != NULL` -- the ACPI-board-info HOB
  (`gUefiAcpiBoardInfoGuid`, carrying the PM-timer base from the FADT) is never built,
  i.e. the payload's `ParseAcpiInfo` cannot find coreboot's ACPI RSDP (a
  coreboot->payload ACPI hand-off that works in QEMU but not here).
- Upstream `UefiPayloadPkg` (built natively by coreboot) fails earlier: a payload
  segment loads at `0x000da520`, inside the 640 KiB--1 MiB legacy hole that coreboot's
  VBox (CMOS-derived) memory map marks reserved, so `lzma: Decoding error` /
  `Payload not loaded`.

Both are the same root issue: coreboot's QEMU board assumes QEMU's memory map and
ACPI-RSDP hand-off. The clean fix is the dedicated board below, matched to VBox's
memory map and ACPI.

## Next step: a real Dasharo target

The two gaps above define it. The clean, idiomatic Dasharo target is a dedicated
mainboard rather than reusing the QEMU board:

- New coreboot mainboard `src/mainboard/emulation/qemu-vbox` (cloned from
  `qemu-i440fx`/`q35`, adapted to VirtualBox's CMOS RAM sizing and info port, no
  `fw_cfg`, and not driving the absent PIIX4 ACPI at `00:01.3`) -- this is what closes
  gap 1 (the ramstage ACPI S5). Alternatively, teach VirtualBox's `DevACPI` to expose
  a QEMU-compatible PIIX4 ACPI function + `fw_cfg`.
- The reused `DasharoPayloadPkg` as the coreboot payload. Gap 2 (the immediate PEI
  reset) needs a DEBUG payload build (`USE_CBMEM_FOR_CONSOLE=TRUE` or serial) to find
  the fault, then either adapt `DasharoPayloadPkg` to the board or fall back to
  upstream `UefiPayloadPkg` for the emulated target.
- A Dasharo `configs/config.emulation_vbox` wiring the two together.

That yields a true Dasharo firmware image for VirtualBox, loadable with the same
`virtualbox` patch. coreboot carries a Bhyve precedent for adding an emulator board
without `#ifdef`-ing the shared code.

## License

coreboot and Dasharo are Free Software (GPLv2). This repo's configuration and build
recipe are offered under the same terms.
