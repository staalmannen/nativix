# Nativix development roadmap

This roadmap describes a sensible progression from ABI research to a usable proof of concept. It is intentionally staged: distribution work should not get ahead of the underlying compatibility implementation.

## Guiding rule

**Prove that a guest userspace works before building a distribution around it.**

Nativix depends on Linux ABI emulation. The boot manager, installer, branding, and update framework are useful only after there is a sufficiently functional guest environment to boot.

## Stage 0 — Define the architecture

- [x] Establish the conceptual separation between Linux infrastructure, early boot, and native guest roots.
- [x] Establish the one-kernel/multiple-guest-root direction.
- [ ] Define the first concrete ABI and guest target.
- [ ] Record known ABI gaps and a test strategy.
- [ ] Define what counts as a successful userspace boot and a useful proof of concept.

## Stage 1 — ABI implementation

This work should remain the immediate technical priority.

- Build and maintain the ABI emulation modules against supported current Linux kernels.
- Define the supported syscall and kernel-interface surface for each target.
- Test executable loading and basic process creation.
- Implement and test signals, threads, memory mappings, file operations, sockets, and other required semantics.
- Identify required filesystem, procfs, sysctl, device, and ioctl behavior.
- Keep guest-specific compatibility work clearly separated where semantics differ.
- Create repeatable tests and document unsupported behavior.

Do not assume that compiling a module or starting a simple executable demonstrates broad ABI compatibility.

## Stage 2 — Start a native guest userspace

Choose one target for the first end-to-end proof of concept.

Potential targets have different trade-offs:

- **NetBSD:** A candidate for a focused first userspace experiment, subject to the actual ABI work and root-filesystem requirements.
- **FreeBSD:** Valuable as a comparison target because its Linux compatibility subsystem provides useful conceptual and behavioral reference material.
- **illumos:** A substantial target with its own syscall, process, and kernel-interface requirements.
- **9front:** Architecturally distinctive and likely to require a more specialized approach; treat it as a separate research target rather than assuming it resembles a BSD port.

Milestones:

- Mount a guest root under Linux.
- Run simple native guest executables.
- Start a meaningful portion of the guest userspace.
- Investigate and implement the requirements of the guest's native init.
- Execute guest init as PID 1 after root handover.
- Document what works, what does not, and what has been deliberately omitted.

Running native init is a major milestone, not proof of full operating-system compatibility.

## Stage 3 — Minimal handover environment

Once a guest userspace can be started reliably:

- Build a small initramfs.
- Evaluate statically linked musl BusyBox or Toybox.
- Implement a minimal, well-tested `/init`.
- Load the required ABI modules.
- Locate and mount the guest root using stable identifiers.
- Prepare required virtual filesystems and device interfaces.
- Use `switch_root` to execute the guest init.
- Provide useful diagnostics and a recovery shell.

Keep the early userspace independent of systemd and other general-purpose init systems unless a concrete requirement emerges.

## Stage 4 — Multiple guest definitions and selection

After one guest boots reliably:

- Define a versioned, declarative guest configuration format.
- Validate guest identifiers, root locators, module lists, mount options, and init paths.
- Add a simple text selection menu.
- Support a configurable default and optional timeout.
- Support one-time guest selection.
- Add a recovery entry.
- Test several guest roots sharing the same Linux kernel.
- Consider automatic discovery only after explicit configuration works reliably.

Guest selection changes the root filesystem and guest-specific preparation; it does not select or boot a different kernel.

## Stage 5 — Infrastructure security and updates

- Package the Linux kernel, matching ABI modules, and initramfs as a versioned release.
- Evaluate direct EFI-stub boot and Unified Kernel Images.
- Establish Secure Boot signing and key-management policy.
- Authenticate modules and any external boot configuration.
- Implement atomic infrastructure installation.
- Retain a known-good previous version and provide rollback.
- Test failed updates and interrupted writes.
- Keep guest OS update workflows independent.

The guest's native kernel package handling must be addressed per target so it cannot overwrite the Nativix Linux kernel.

## Stage 6 — System integration

- Storage and filesystem compatibility.
- Networking.
- Device access and permissions.
- User and group handling.
- Process supervision and service startup.
- Timekeeping and system clock behavior.
- Logging and diagnostics.
- Shutdown and reboot behavior.
- Guest-native package management.
- Shared data/home policy, if required.

This stage determines whether the system feels like a usable guest OS rather than merely reaching a shell.

## Stage 7 — Distribution engineering

Only after earlier stages have demonstrated a useful system:

- Installer and partitioning guidance.
- Guest root installation or import workflow.
- Boot entry and configuration management.
- Infrastructure update tooling and recovery UI.
- Documentation for supported guest systems and hardware.
- Release engineering, testing, and support policy.

## Suggested first proof of concept

Keep the first POC narrow:

1. One supported Linux kernel configuration.
2. One ABI implementation and one guest OS target.
3. One guest root filesystem on a simple, well-supported filesystem.
4. A minimal initramfs with static BusyBox or Toybox.
5. A deterministic root mount and `switch_root` handover.
6. Guest init started as PID 1, with a documented list of working and missing functionality.

Do not start with automatic guest discovery, a graphical menu, a universal installer, shared writable filesystems across guest OSes, or multiple guest targets. Those are follow-on features.

## Success criteria

A credible early Nativix proof of concept should demonstrate:

- Linux boots independently of the guest root.
- The ABI modules load successfully and match the running kernel.
- A native guest root is mounted without chrooting into a persistent Linux host installation.
- The guest's own init starts as PID 1.
- The system can perform useful operations within the guest userspace.
- The guest's native userland update mechanism can be used without replacing the Linux infrastructure kernel.
- Infrastructure and guest update responsibilities are clearly separated.

The longer-term success criterion is a system in which users select and operate native Unix-like environments while sharing a Linux kernel and its hardware support.
