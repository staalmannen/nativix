# Boot process and guest selection

## 1. Goals

Nativix should use one shared Linux kernel and early-userspace image to start one of several installed guest root filesystems.

The boot environment should be small and purpose-built. Its job is to prepare the system and transfer control to the selected guest's native init. It should not provide a parallel package manager, service manager, or general-purpose host installation.

## 2. EFI boot options

### Direct Linux EFI stub

Linux can be built with EFI stub support (`CONFIG_EFI_STUB`) and launched directly by UEFI firmware as an EFI executable. This removes the need for a conventional bootloader when the firmware can launch the kernel image.

The EFI stub is not a full boot manager. It does not by itself provide a rich menu, guest discovery, version management, or rollback policy.

### Unified Kernel Image

A Unified Kernel Image (UKI) packages the Linux kernel and early-boot resources, commonly including the initramfs and command line, into one EFI executable. A signed UKI is attractive because the kernel and its early userspace can be authenticated as a single boot artifact.

The systemd project provides tooling such as systemd-stub and ukify for UKI creation. Using those tools does not require using systemd as the guest init system or systemd-boot as the boot manager.

### Optional boot manager

A separate UEFI boot manager can be added if users need boot-entry selection, multiple infrastructure versions, or a more polished startup experience. systemd-boot, rEFInd, and other EFI boot managers can be evaluated.

Keep this choice replaceable. The Nativix boot image and guest configuration model should not depend on one particular boot manager.

## 3. Early userspace

A minimal initramfs is a good fit for the handover stage. A statically linked musl build of BusyBox or Toybox can provide most of the required utilities without depending on a persistent Linux root filesystem.

The initial `/init` can be a small script or program that:

1. Mounts essential kernel interfaces needed by early boot.
2. Parses the kernel command line and boot configuration.
3. Locates and validates guest definitions.
4. Loads the required ABI modules.
5. Locates and mounts the selected guest root.
6. Prepares the guest's required mount points and runtime devices.
7. Moves or remounts required virtual filesystems as appropriate.
8. Executes `switch_root` and the configured guest init.

Use explicit error handling and provide a recovery shell when selection, module loading, or root mounting fails.

Do not assume that a shell script alone will remain sufficient forever. As complexity grows, a small dedicated helper may make parsing, validation, device discovery, and error handling safer and easier to test.

## 4. Guest configuration model

Guest definitions should be declarative data, not arbitrary shell scripts. This reduces the amount of executable configuration that must be trusted and makes entries easier to validate.

An illustrative guest definition:

```ini
[guest]
id=netbsd
name=NetBSD
root=UUID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
filesystem=ext4

[abi]
modules=netbsd_abi

[boot]
init=/sbin/init
```

A second guest could define a different root and ABI module:

```ini
[guest]
id=freebsd
name=FreeBSD
root=UUID=yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy
filesystem=ufs

[abi]
modules=freebsd_abi

[boot]
init=/sbin/init
```

These examples are conceptual. The actual module names, supported filesystems, init paths, and mount requirements depend on the implementations.

Potential fields include:

- Stable guest identifier.
- Human-readable display name and optional description.
- Root filesystem locator, preferably UUID or another stable identifier.
- Filesystem type and mount options.
- Required ABI modules and module parameters.
- Guest init path.
- Guest-specific kernel or boot parameters.
- Optional recovery or diagnostic behavior.

The format should have a defined schema and version. Reject unknown or malformed security-sensitive fields rather than interpreting them permissively.

## 5. Selection menu

The initramfs can present a simple text menu using BusyBox tools initially. A graphical menu is not required for the proof of concept.

Possible entries:

- NetBSD
- FreeBSD
- 9front
- Recovery environment

Useful menu behavior:

- Configurable default guest.
- Optional timeout before booting the default.
- Remember last successfully selected guest.
- One-time selection that does not change the default.
- Clear explanation when a guest is unavailable or its configuration is invalid.
- Recovery shell or diagnostic mode.

A one-time selection is especially useful for testing: the normal default remains unchanged, but a different guest can be selected for a single boot.

## 6. Guest discovery

Start with explicit configuration and stable root identifiers. Automatic discovery can be added later.

Scanning partitions and trying to infer operating systems introduces ambiguity and security concerns. A filesystem should not be considered bootable merely because it contains a familiar-looking directory or file.

A future discovery mechanism should use explicit, validated metadata and should present the detected guest to the user before booting it. It should not execute scripts found on an untrusted guest root during discovery.

## 7. Root handover details

The handover must account for the fact that the Linux kernel continues running. `switch_root` changes the root filesystem and replaces the initramfs process environment; it does not load a guest kernel.

Before handover, ensure the selected root is mounted at the intended location and that required virtual filesystems and device interfaces are available in the new root. The exact mount layout will depend on the guest OS and ABI implementation.

The configured init path must exist and be executable from the mounted root. Failure to execute it should lead to a clear diagnostic or recovery path, not an unexplained hang.

The guest init becomes PID 1. Its signal handling, child reaping, startup assumptions, and interactions with Linux PID 1 semantics need explicit testing.

## 8. Suggested repository evolution

As implementation begins, consider keeping boot components separate:

```text
boot/
  initramfs/
    init
    config/
  config-examples/
    netbsd.conf
    freebsd.conf
    9front.conf
  tools/
    guest-config-validator/
```

This is a possible future layout, not a requirement for the current documentation-only stage.
