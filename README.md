# Nativix

Nativix is an experimental project exploring a different way to run Unix-like operating systems: use the Linux kernel as a shared kernel and hardware-compatibility layer, while running the native userland of another operating system as the active root filesystem.

The intended result is not a conventional Linux distribution with a BSD compatibility layer exposed as an optional feature. The goal is for a selected guest environment—such as NetBSD, FreeBSD, illumos, or 9front—to remain the operating environment the user interacts with, including its native tools, configuration conventions, service management, and package-management workflow, while Linux supplies the running kernel.

This is an architectural proposal and long-term direction. It depends on the development of working Linux ABI emulation implementations; it is not yet a working distribution.

## Core idea

A single Linux kernel and a small, independent boot environment can start one of several installed guest root filesystems:

```text
UEFI
  |
  v
Linux kernel + early userspace (initramfs)
  |
  +-- load required ABI modules
  +-- discover guest installations
  +-- present guest selection menu
  +-- mount the selected guest root
  |
  v
switch_root
  |
  v
Selected guest's native init (PID 1)
  |
  v
Guest OS userland and native update mechanisms
```

The Linux environment is infrastructure. It should be small, replaceable, independently updateable, and mostly invisible after the handover.

## Design principles

- **Native guest experience:** The selected guest root owns its userland, configuration, applications, service management, and package database.
- **Linux as the shared kernel:** Hardware support, kernel facilities, and filesystem support come from Linux, subject to the capabilities and limitations of the ABI implementation.
- **One kernel, multiple roots:** Multiple guest installations can share one Linux kernel and boot infrastructure. The selected root determines which userland starts.
- **A minimal handover environment:** Early userspace should do only what is needed to prepare the system and hand over to the guest.
- **Independent lifecycle:** Linux infrastructure updates must not be conflated with native guest OS updates.
- **Explicit, inspectable configuration:** Guest definitions should be declarative and validated, rather than arbitrary shell scripts.
- **Incremental development:** Prove ABI functionality and guest booting before investing in installer, distribution, or polished boot-menu work.

## Documentation

- [Architecture overview](docs/architecture.md)
- [Boot process and guest selection](docs/boot-and-guest-selection.md)
- [Security and update model](docs/security-and-updates.md)
- [Development roadmap](docs/roadmap.md)

## Status and scope

Nativix is currently a design direction and proof-of-concept concept, not a promise of broad compatibility. Each guest OS requires a sufficiently complete ABI implementation and integration work. Running a guest's native init is a major milestone, but does not by itself establish full compatibility with its kernel, drivers, system calls, or expected kernel interfaces.

The initial priority should remain the underlying ABI work. Distribution engineering should follow only after a guest userspace can be started and exercised reliably.

## Terminology

- **Host kernel / Linux infrastructure:** The Linux kernel, its ABI modules, and the early userspace used to prepare the system.
- **Guest OS / guest root:** The non-Linux operating system's root filesystem and native userspace. “Guest” describes its relationship to the Linux kernel; it does not imply a virtual machine.
- **Early userspace / initramfs:** The temporary environment started by Linux before the real root filesystem is selected.
- **Handover:** The transition from initramfs to the selected guest root, normally using `switch_root` and executing the guest's init process.

Nativix does not, in this design, boot a FreeBSD or NetBSD kernel. It boots Linux and attempts to provide the ABI and kernel interfaces needed by the selected operating system's userland.
