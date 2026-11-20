# Security and update model

## 1. Separate update domains

Nativix has two independently managed software domains:

1. **Linux infrastructure:** Linux kernel, ABI modules, initramfs, and boot configuration or metadata needed to start guests.
2. **Guest operating system:** The selected guest's native userland, applications, configuration, and package database.

The guest should continue to use its native update mechanisms wherever possible. Nativix must update the Linux infrastructure independently and must not let a guest's ordinary kernel package replace the running Linux kernel.

This boundary is fundamental to the project.

## 2. Infrastructure update contents

Treat a kernel, its ABI modules, and its initramfs as one compatible release unit.

An infrastructure release may contain:

- Linux kernel image.
- Matching ABI modules.
- Initramfs containing the required early-userspace tools and boot logic.
- Boot metadata and configuration schema version.
- Version and integrity metadata.

Modules must match the exact kernel build and configuration. If modules are embedded in the initramfs, the initramfs becomes part of the signed boot artifact. If modules are loaded later from disk, they need their own integrity and signature policy, and the kernel must enforce the intended trust model.

## 3. Secure Boot and UKIs

A Unified Kernel Image (UKI) can package the kernel and early-boot resources into one EFI executable. Signing the UKI allows firmware Secure Boot policy to authenticate the boot artifact before it is executed.

A UKI is a promising fit for Nativix because the early userspace is security-critical: it chooses which root filesystem will become the system root and determines which ABI modules are loaded.

Important distinctions:

- Secure Boot authenticates the boot chain according to the configured trust policy; it does not automatically make every later file or guest root trustworthy.
- A signed UKI does not automatically authenticate external configuration files, external kernel modules, or guest filesystems.
- Firmware trust databases, signing-key enrollment, revocation, and recovery procedures need an explicit design.
- If the initramfs contains all required modules and boot logic, the authenticated artifact boundary is easier to reason about.

The Linux EFI stub can also boot a kernel directly from UEFI. It is a kernel-loading mechanism, not a complete boot-entry manager or update system.

## 4. External guest configuration

Guest configuration may be embedded in the signed initramfs or stored externally.

### Embedded configuration

Advantages:

- Authenticated as part of the UKI.
- Simple trust model.
- Configuration and boot code can be versioned together.

Disadvantages:

- Adding or changing a guest requires rebuilding and signing the boot image.
- Less convenient for systems where guest installations change frequently.

### External configuration

Advantages:

- Guests can be added or removed without rebuilding the kernel image.
- Configuration can be maintained separately from infrastructure releases.

Disadvantages:

- The initramfs must authenticate and validate configuration before using it.
- A modified configuration could redirect boot to another root or alter boot parameters.
- Key management and configuration rollback become part of the security design.

A later design could use signed manifests or a signed configuration bundle. Do not execute arbitrary scripts from external configuration or guest filesystems as part of early boot.

For an initial proof of concept, keep configuration embedded or otherwise within the trusted boot artifact. External signed configuration can follow when the need is established.

## 5. Atomic updates and rollback

Use versioned infrastructure artifacts and retain a known-good version.

A possible update sequence:

1. Download or install a new infrastructure release.
2. Verify its signature and version metadata.
3. Confirm that the kernel and ABI module set are a compatible release.
4. Install the new signed UKI alongside the currently working one.
5. Set the new image as the next boot target.
6. Boot it and perform an explicit health check.
7. Mark it as the known-good default only after successful validation.
8. Retain the previous image for recovery.

The exact mechanism for one-time boot, boot counting, and fallback depends on firmware and the chosen boot manager. Do not assume that every UEFI implementation provides equivalent rollback behavior.

The guest root should not need to be modified by a Linux infrastructure update.

## 6. Guest OS updates

Guest userland updates should use the guest's native package manager and tools.

The main integration issue is the guest's expectation that it owns its kernel. A native update may install a guest kernel, update bootloader configuration, or assume a native kernel ABI. Nativix must prevent those operations from replacing or interfering with the Linux kernel.

Possible strategies, to be evaluated per guest:

- Exclude guest-kernel packages from normal updates.
- Provide a documented package-manager policy or package selection.
- Supply compatibility metadata or a narrowly scoped integration package where appropriate.
- Preserve native userland updates and security fixes without pretending the guest kernel is running.

Avoid silently modifying a guest's package database in ways that make its native package manager believe an unsupported guest kernel is installed.

## 7. Recovery and failure handling

A boot failure should not make the guest data inaccessible.

Useful recovery paths include:

- Previous known-good Linux infrastructure image.
- Minimal maintenance shell in initramfs.
- Diagnostics for missing root devices, invalid configuration, missing modules, and failed init execution.
- A way to mount and inspect guest roots without starting them.
- Independent backup and recovery procedures for guest data.

Recovery should not depend on the guest init or guest package manager working.

## 8. Threat model questions to resolve

Before a production-oriented implementation, document:

- Who can modify the ESP and infrastructure storage?
- How are Secure Boot keys enrolled, rotated, and revoked?
- Are guest roots trusted, locally administered, or potentially removable/untrusted?
- Can guest configuration select arbitrary kernel parameters or module options?
- Are external ABI modules permitted, and how are they authenticated?
- How are rollback attacks handled?
- What happens if the system clock or entropy source is unavailable during early boot?
- How are guest data and configuration backed up?
- What security guarantees are provided by the underlying filesystem and its mount options?

Secure Boot protects the boot chain, but it does not turn a guest root filesystem into a trusted or isolated environment. A guest running on the shared Linux kernel is not isolated from that kernel in the way a virtual machine would be.
