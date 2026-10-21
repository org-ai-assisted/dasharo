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
- `CONFIG_PAYLOAD_NONE` -- no payload yet; this de-risks the firmware init on its own.
  coreboot stops after device enumeration. A real payload (next section) boots on.

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

## Next step: a real Dasharo target

`emulation/qemu-i440fx` proves the firmware path. The clean, idiomatic Dasharo
target is a dedicated mainboard rather than reusing the QEMU board:

- New coreboot mainboard `src/mainboard/emulation/qemu-vbox` (cloned from
  `qemu-i440fx`/`q35`, adapted to VirtualBox's CMOS RAM sizing and info port, no
  `fw_cfg`).
- The reused `DasharoPayloadPkg` as the coreboot payload, built board-agnostically
  via the coreboot table.
- A Dasharo `configs/config.emulation_vbox` wiring the two together.

That yields a true Dasharo firmware image for VirtualBox, loadable with the same
`virtualbox` patch. coreboot carries a Bhyve precedent for adding an emulator board
without `#ifdef`-ing the shared code.

## License

coreboot and Dasharo are Free Software (GPLv2). This repo's configuration and build
recipe are offered under the same terms.
