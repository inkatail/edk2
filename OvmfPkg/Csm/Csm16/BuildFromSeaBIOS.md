# Building Csm16.bin from SeaBIOS

`OvmfPkg/Csm/Csm16/Csm16.inf` packages an external 16-bit BIOS payload
(`Csm16.bin`) into the OVMF flash image when OVMF is built with
`-D CSM_ENABLE`. The binary is intentionally **not** stored in this
repository. Build it from SeaBIOS as follows (see also
https://www.seabios.org/Build_overview.html and the Gerd Hoffmann
spec files referenced on the SeaBIOS mailing list):

## 1. Clone SeaBIOS

```
git clone https://github.com/coreboot/seabios.git
cd seabios
```

A release tarball works as well. Newer SeaBIOS releases are preferred;
the CSM interface (`CONFIG_CSM`) is stable.

## 2. Configure for CSM

Create `.config` with at minimum:

```
CONFIG_CSM=y
CONFIG_QEMU_HARDWARE=y
CONFIG_PERMIT_UNALIGNED_PCIROM=y
```

then:

```
yes "" | make oldconfig
```

`CONFIG_ROM_SIZE` controls the output size. The binary must fit the
space reserved in the OVMF flash map; historically 128 KB was used.
If legacy boot fails in `LegacyBiosDxe` with a `Not Found` ASSERT
while locating the Compatibility16 entry, the image is too large for
the reserved region — lower `CONFIG_ROM_SIZE` (e.g. 192 KB total budget
including padding, see tianocore/edk2 issue #82 discussion) and
rebuild.

## Staying within 128 KiB

The payload must fit the `0xE0000-0xFFFFF` shadow window, so
`CONFIG_ROM_SIZE=128` is effectively mandatory. A full-featured build
uses ~97% of that budget. If a future SeaBIOS update overflows it,
trim in this order (correctness impact smallest first):

1. `CONFIG_TCGBIOS=n` — drops legacy-OS TPM services (e.g. Win7
   BitLocker); OVMF keeps its own TPM stack for UEFI boot.
2. `CONFIG_S3_RESUME=n` — drops legacy-OS S3 resume.
3. `CONFIG_USB_XHCI=n` — drops USB3 boot (UHCI/EHCI remain).
4. `CONFIG_SERCON=n`, `CONFIG_LPT=n` — drops serial/parallel extras.

`CONFIG_VGAHOOKS` and `CONFIG_S3_RESUME` are default-on and were on in
all historical working CSM builds; leave them enabled. There are no
other OVMF-specific options: `CONFIG_CSM` + `CONFIG_QEMU_HARDWARE`
is the complete CSM contract (`PERMIT_UNALIGNED_PCIROM` no longer
exists upstream), and ACPI/SMBIOS are auto-excluded under `CONFIG_CSM`
because OVMF provides those tables.

## 3. Build and install into the edk2 tree

```
make
cp out/Csm16.bin <edk2>/OvmfPkg/Csm/Csm16/Csm16.bin
```

## 4. Build OVMF with CSM enabled

```
# from the edk2 root, after . edksetup.sh:
build -p OvmfPkg/OvmfPkgX64.dsc -t GCC5 -a X64 -b RELEASE -D CSM_ENABLE
# or: OvmfPkg/build.sh -a X64 -b RELEASE -t GCC5 -D CSM_ENABLE
```

`OvmfPkgIa32X64.dsc` supports `-D CSM_ENABLE` as well. Other OvmfPkg
targets (Xen, Bhyve, Microvm, CloudHv, TDX, SEV) do not wire the CSM
modules.

## 5. Run under QEMU

Point `-bios` at the resulting `OVMF_CODE.fd`. For emulated VGA cards
(cirrus/stdvga) also supply a matching SeaVGABIOS (`-L` path or
`-device ...,romfile=...`); QEMU's stock VGABIOS may predate the
bochs/cirrus mode fixes SeaBIOS-era guests expect.

## License note

SeaBIOS is LGPL. The `Csm16.bin` file is embedded as a discrete
freeform file in the flash volume and can be extracted/replaced with
UEFI firmware volume tools, satisfying the LGPL requirements
(see the SeaBIOS `README.CSM`).
