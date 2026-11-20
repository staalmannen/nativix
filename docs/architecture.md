# Nativix architecture

## 1. Purpose

Nativix explores using Linux as a common kernel substrate for native Unix-like userlands. The Linux kernel provides hardware drivers and kernel facilities; an ABI compatibility implementation translates the guest operating system's expected system-call and related interfaces into Linux behavior.

The intended user experience is that of the selected guest OS, not that of a Linux distribution. The guest root filesystem remains authoritative for its native utilities, package management, configuration, services, and applications.

This is not virtualization. There is no separate guest kernel: the Linux kernel remains the only running kernel.

## 2. Architectural layers

### Nativix Core

The infrastructure layer consists of:

- Linux kernel, configured for the target hardware.
- ABI emulation modules and any supporting kernel interfaces.
- Required kernel configuration and drivers.
- A secure, independently managed kernel/module/initramfs update mechanism.

ABI modules must match the kernel they are built for. They should be treated as part of one versioned infrastructure release.

### Nativix Boot

The boot layer is a minimal early-userspace environment responsible for:

- Reading and validating guest definitions.
- Loading the required ABI modules.
- Discovering and mounting the selected guest root.
- Preparing essential virtual filesystems and device nodes.
- Handing over to the guest's native init process.

It should not become a second general-purpose operating system. A small statically linked musl-based BusyBox or Toybox environment is a reasonable starting point.

### Native guest systems

Each guest installation is an ordinary root filesystem for its respective operating system, subject to the compatibility limitations of the Linux ABI implementation.

Examples of possible research targets include NetBSD, FreeBSD, illumos, and 9front. These are separate compatibility targets, not a claim that one implementation will support all of them.

The guest should retain its native package manager and normal userspace update workflow. Guest kernel packages require special handling: they must not replace or interfere with the Linux infrastructure kernel.

## 3. Boot and root handover

The intended sequence is:

1. UEFI launches the Linux kernel, directly through its EFI stub or through a boot manager.
2. Linux starts the initramfs as PID 1.
3. Early userspace loads the ABI modules needed by the selected guest.
4. The boot manager presents available guest installations or selects the configured default.
5. The selected root filesystem is located and mounted.
6. Required mount points and runtime resources are prepared.
7. Early userspace performs `switch_root` and executes the guest's init.

`switch_root` changes the root filesystem and replaces the initramfs environment; it does not change the running kernel. The guest init becomes PID 1 in the same Linux kernel.

PID 1 behavior is a significant compatibility milestone. Linux gives PID 1 special signal and process-reaping responsibilities, and guest init programs may assume kernel semantics that differ from Linux. Starting init is therefore necessary but not sufficient to demonstrate a usable system.

## 4. One kernel, multiple guest roots

A central Nativix feature should be the ability to install several guest roots and select one at boot while sharing the same Linux kernel and early userspace.

For example:

```text
UEFI
  |
  v
Shared Linux kernel + Nativix initramfs
  |
  v
Guest selection
  |             |              |
  v             v              v
NetBSD root   FreeBSD root   9front root
  |             |              |
  v             v              v
native init    native init     native init
```

The selection changes the root filesystem and guest-specific boot parameters, not the kernel. Different guests may require different ABI modules, module parameters, filesystem drivers, or mount preparation.

The boot manager should therefore treat each guest as a validated configuration entry, not simply as a partition label.

## 5. Filesystem and storage considerations

The Linux kernel determines which filesystems can be mounted during early boot and after handover. This may offer broader hardware and filesystem choices than the guest OS would otherwise support, but compatibility is not automatic.

Important considerations include:

- The filesystem must be supported by the Linux kernel and configured with the necessary drivers.
- Filesystem semantics must be suitable for the guest userland.
- Ownership, permissions, extended attributes, links, locking, timestamps, and special files must behave as expected.
- A filesystem writable by both a guest OS and Linux must be tested carefully.
- Native guest tools may assume filesystem features or utilities that are not present in the Linux environment.

For an initial proof of concept, use a well-supported Linux filesystem such as ext4 for the guest root if that is acceptable to the target userspace. More specialized options can be evaluated later.

ZFS deserves particular caution. Linux and FreeBSD ZFS implementations and feature sets may differ, and a pool's compatibility depends on its enabled features and implementation versions. Do not assume that a pool can safely be shared read-write between different implementations without explicit testing.

A possible disk layout is:

- EFI System Partition (ESP): EFI boot artifacts.
- Optional infrastructure storage: versioned kernel/UKI images, recovery artifacts, and configuration if not embedded.
- One partition or volume per guest root.
- Optional shared data/home volume, only after UID/GID and filesystem semantics are addressed.

The exact partitioning scheme should remain flexible; UUID-based root references are preferable to unstable device names.

## 6. Hardware and system integration

The Linux kernel may provide drivers for hardware unsupported by a guest OS, but the guest still needs suitable interfaces to use that hardware. ABI compatibility may need to cover more than system calls, including:

- Process and thread behavior.
- Signals and wait semantics.
- Memory mapping and shared memory.
- Device nodes and ioctl interfaces.
- procfs, sysctl, and other kernel-facing interfaces.
- Networking and socket behavior.
- Filesystem behavior and file locking.
- Event notification mechanisms such as kqueue equivalents.
- User, group, permissions, and credential semantics.
- Expectations of the guest's init and service manager.

A successful boot should be treated as an early milestone. A useful system also needs working storage, networking, devices, process management, and native service startup.

## 7. Design boundaries

Keep these responsibilities separate:

| Responsibility | Nativix or guest |
|---|---|
| Running kernel | Nativix |
| ABI emulation | Nativix |
| Early boot and guest selection | Nativix |
| Kernel and ABI updates | Nativix |
| Guest root filesystem | Guest OS |
| Guest package management | Guest OS |
| Guest services and configuration | Guest OS |
| Guest applications | Guest OS |
| Guest kernel replacement | Not used as the running kernel |

The boundary around guest kernel packages will need deliberate integration with each guest's native package manager. The goal is to preserve the guest's normal userland workflow without allowing its kernel update process to overwrite the Nativix kernel.
