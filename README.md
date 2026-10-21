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

## Real DasharoPayloadPkg: built, integrated, two remaining gaps

A real `DasharoPayloadPkg` UEFI payload was built from `Dasharo/edk2` and integrated
(OVH, Debian trixie, 2026-10-21):

- Payload: `DasharoPayloadPkg/DasharoPayloadPkg.dsc -a IA32 -a X64 -t GCC5 -b RELEASE
  -D BOOTLOADER=COREBOOT -D TPM_ENABLE=FALSE -D SECURE_BOOT_ENABLE=FALSE`
  -> `UEFIPAYLOAD.fd` (14.8 MiB FV, mostly padding).
- Integrated into the 4 MiB coreboot ROM with
  `cbfstool coreboot.rom add-payload -f UEFIPAYLOAD.fd -n fallback/payload -c lzma`
  (the whole FV LZMA-compresses to ~910 KiB; load `0x800000`, entry `0x800860`).

Two gaps block a UEFI shell inside VirtualBox; both are tracked and reproducible:

1. VirtualBox side (`proof/dasharo-payloadpkg-vbox-serial.log`). The patched VBox
   loads and runs this ROM exactly like the `PAYLOAD_NONE` one -- coreboot reaches
   ramstage and enumerates the VBox chipset -- but coreboot's `emulation/qemu-i440fx`
   expects QEMU's PIIX4 ACPI function at `00:01.3` (`[8086/7113]`) and `fw_cfg`,
   neither of which VirtualBox emulates. Ramstage logs `PCI: Static device PCI:
   00:00:01.3 not found` and then trips an early ACPI S5 (`ACPI: Entering S5 power
   state` in VBox.log) -- before the payload is ever loaded.

2. Payload side (`proof/dasharo-payloadpkg-qemu-handoff.log`). Under QEMU (which has
   `fw_cfg` + PIIX4 ACPI) the identical ROM boots coreboot to completion and the
   coreboot -> payload hand-off is proven (`CBFS: Found 'fallback/payload'` ->
   `Loading Segment: addr 0x00800000` -> `Entry Point 0x00800860` -> `Jumping to boot
   code`). But `DasharoPayloadPkg` then resets immediately in early PEI (QEMU shows
   "Guest has not initialized the display"): it is built for real Dasharo boards, not
   the vanilla `emulation/qemu-i440fx` platform.

So the VBox-side loader patch is fully proven (it runs real coreboot firmware), and a
real Dasharo payload builds + integrates + hands off, but a UEFI shell inside VBox
needs the chipset + payload-platform work in the next section.

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
