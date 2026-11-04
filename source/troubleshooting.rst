Troubleshooting
###############

When software that works on another distribution does not work on Sulka, a hardening measure is usually responsible.
This page maps the symptoms you are likely to see onto the measure that causes them.

Narrowing It Down
*****************

Two checks narrow things down quickly.

First, build the same image with the development configuration described in :ref:`Building With Sulka` and disable SELinux and firewall to see whether the problem goes away:

.. code-block::

  setenforce 0
  nft flush ruleset

Second, look at the logs.
Most of the hardening in Sulka reports refusals there:

.. code-block::

  # Kernel log: module loading failures, failures caused by kernel hardening, read-only rootfs failures, etc.
  dmesg
  # Audit log: SELinux related failures, look for lines with denial messages
  grep -i avc /var/log/audit/audit.log
  # System log: User space failures, login failures, access control failures, etc.
  cat /var/log/syslog
  # On systemd systems, check journalctl as well
  journalctl

SELinux Denials
***************

**Symptoms.** A service fails to start for no obvious reason. A file cannot be opened even though the ownership and permission bits are correct. Something works when run by hand but not when run as a service.

Check whether SELinux is enforcing, and look for denials:

.. code-block::

  getenforce
  sestatus
  grep -i avc /var/log/audit/audit.log
  # Early SELinux denials may go to dmesg instead of audit log
  dmesg | grep -i avc

Denials are recorded as ``AVC`` messages naming the source context, the target context and the permission that was refused.

**Confirming the cause.** On an image built with development mode, put SELinux into permissive mode temporarily and retry:

.. code-block::

  setenforce 0

If the problem disappears, it is a policy problem rather than a bug in your software.
Note that this only works on an image built with development mode enabled, as described in :ref:`Development Mode`.
On a production image the kernel refuses the request, and the boot parameter ``selinux=0`` has no effect either.

**Fixing it.** Your own services will usually need a policy module of their own.
The ``audit2allow`` tool generates a starting point from the denials you have collected, but it is only installed by the development configuration, not on a default image.
Treat its output as a draft: it grants exactly what was asked for, which is not always what you should allow.

.. code-block::

  grep -i avc /var/log/audit/audit.log | sudo audit2allow

See :ref:`SELinux` for the policy Sulka ships and the patches applied to it.

Network Traffic Is Blocked
**************************

**Symptoms.** Nothing can reach the device, and the device cannot reach anything either. Outgoing connections, DNS lookups and package downloads all fail.

This is the default firewall doing its job.
Sulka ships a rule set that drops all traffic in every direction until you configure it.

.. code-block::

  nft list ruleset

To confirm this, you can try flushing the firewall:

.. code-block::

  nft flush ruleset

**Fixing it.** Select a template that suits you with ``SULKA_NFTABLES_CONF``, or ship your own rule set.
See :ref:`Firewall` for the bundled templates.

Write Failures On The Root File System
**************************************

**Symptoms.** Writes fail with a read-only file system error, or an application that expects to create files at startup fails immediately.

The root file system is mounted read-only by default:

.. code-block::

  mount | grep ' / '

**Fixing it.** Write to one of the designated writable locations instead (check the ``/var`` ``tmpfs`` mounts from ``mount`` output), or arrange one for your application.
:ref:`Read-Only Root File System` covers the options, including writable partitions, temporary file systems, bind mounts and symlinks.
Note that the ``overlayfs`` approach conflicts with SELinux, which is enabled by default.

Cannot Execute A Script From /tmp, /run Or /var
***********************************************

**Symptoms.** A script or binary with the execute bit set fails with a permission error, specifically when it lives in ``/tmp``, ``/run``, ``/var/lib`` or ``/var/cache``.

Sulka mounts the volatile file systems with ``noexec``, along with ``nodev`` and ``nosuid``:

.. code-block::

  mount | grep -E ' /tmp | /run | /var/volatile | /var/lib | /var/cache '

``/var/lib`` and ``/var/cache`` are affected as well, which is less obvious.
With the read-only root file system they are bind mounted from ``/var/volatile``, and they inherit the mount options from their source, so the ``noexec`` on ``/var/volatile`` carries over to both.

``/tmp`` reaches the same end result through two different routes, depending on the init manager.
On ``systemd`` systems ``/tmp`` is its own ``tmpfs``, mounted by systemd's ``tmp.mount`` unit, which Sulka patches to add ``noexec``.
On ``sysvinit`` systems ``/tmp`` is a symbolic link into ``/var/volatile``, so it inherits the ``noexec`` from that mount.

**Fixing it.** Prefer moving the executable somewhere persistent and running it from there.
If that is not possible, setting ``SULKA_HARDEN_MOUNTS`` to ``0`` disables the mount option hardening, but you then take on setting safe options yourself.
See the description of that variable in :ref:`Configuration Variables`.

ps Shows Only Your Own Processes
********************************

**Symptoms.** ``ps`` and ``top`` show almost nothing, and tools that inspect other processes report that they do not exist.

``/proc`` is mounted with ``hidepid=2``, which hides other users' processes.
Run the command through ``sudo`` to see the whole system, or remount with ``hidepid=0`` to verify.
This is also controlled by ``SULKA_HARDEN_MOUNTS``.

A Kernel Module Will Not Load
*****************************

**Symptoms.** ``modprobe`` or ``insmod`` fails, and the kernel log mentions a missing or invalid signature.

Module signing is enforced by default, so the kernel refuses any module it cannot verify against a trusted key.
Modules built through their own Yocto recipes are signed during the build.
Prebuilt binary modules and anything compiled by hand are not.

**Fixing it.** Sign the module with the same key pair the kernel trusts, as described in :ref:`Signing External Modules`.
If a correctly signed prebuilt or out-of-tree module still fails to load, ``CONFIG_RANDSTRUCT_FULL`` may be the cause.
It randomizes the internal structure layout of each kernel build, so a module built against another build of the kernel does not fit it.
Build the module against the exact kernel build, or disable ``CONFIG_RANDSTRUCT_FULL`` with the ``sulka-no-randstruct`` feature, as described in :ref:`Signing External Modules`.

If you cannot sign your modules at all, ``SULKA_ENABLE_MODULE_SIGNING`` can be disabled to disable the enforcement, at a real cost to the security of the system.

**Modules signed with other keys.**
Modules that ship in the image can also fail to load when they were signed with other keys than the ones built into the kernel.
The kernel log shows ``Loading of module with unavailable key is rejected``, and ``modprobe`` reports ``Key was rejected by service``.
The firewall is often the first to show it, as the ``nftables`` service fails at boot with errors like ``Could not process rule: No such file or directory``.

This happens when new keys are generated at the same path while the shared state cache holds modules signed with the old ones, for example when each build job generates its own keys and the jobs share a cache.
The task hashes of the kernel normally contain only the paths of the keys, so the module packages are restored from the cache, while the kernel can be built again with the new keys, for example when the initramfs is bundled into it.
``meta-sulka-kernel`` adds the contents of the keys to the task hashes of the kernels that inherit ``sulka-kernel-hardening``, so new keys build the kernel and its modules again.
A kernel recipe that does not inherit the class can still mix them.

**Fixing it.** When builds share a shared state cache, generate the keys once and point every build at them.
To recover a build, remove the kernel's entries from the cache and build again; pass the same configuration files you build with:

.. code-block::

  kas shell kas-sulka.yml -c 'bitbake -c cleansstate virtual/kernel'

A Kernel Feature Is Missing
***************************

**Symptoms.** A syscall returns ``ENOSYS``, a binary refuses to run, a file system or protocol is unavailable, or a library reports that a kernel facility is not supported.

Sulka removes a substantial amount of kernel functionality to cut the attack surface.
The options below are the ones most likely to affect ordinary software:

.. list-table::
   :header-rows: 1
   :widths: 35 65

   * - Disabled option
     - What stops working
   * - ``CONFIG_COMPAT``
     - 32-bit binaries on a 64-bit system
   * - ``CONFIG_USER_NS``
     - User namespaces, and with them containers and rootless sandboxing
   * - ``CONFIG_IO_URING``
     - Applications built on ``io_uring``
   * - ``CONFIG_AIO``
     - The POSIX asynchronous I/O interface
   * - ``CONFIG_COREDUMP``
     - Core dumps, in addition to the process limits that also disable them
   * - ``CONFIG_DEBUG_FS``
     - ``debugfs``, and the tools that rely on it
   * - ``CONFIG_FTRACE``, ``CONFIG_KPROBES``, ``CONFIG_UPROBE_EVENTS``
     - Kernel tracing, and tooling built on it
   * - ``CONFIG_CHECKPOINT_RESTORE``
     - Checkpoint and restore, such as CRIU
   * - ``CONFIG_MAGIC_SYSRQ``
     - The SysRq key combinations
   * - ``CONFIG_IP_SCTP``, ``CONFIG_TIPC``, ``CONFIG_RDS``, ``CONFIG_MPTCP``
     - Those network protocols

You can try disabling the kernel hardening to see if that helps by adding the following to your build configuration:

.. code-block::

  SULKA_HARDEN_KERNEL = "0"

**Fixing it.** The cuts live in the kernel metadata of `meta-sulka-kernel <https://codeberg.org/AltidSec/meta-sulka-kernel>`_, under the ``sulka-cut-attack-surface`` feature.
Re-enable what you genuinely need with a kernel configuration fragment of your own, from your own layer, rather than by editing the Sulka metadata.
Turning the whole feature off is possible with ``SULKA_HARDEN_KERNEL``, but that gives back far more attack surface than you probably intend.

Debuggers And Profilers Do Not Work
***********************************

**Symptoms.** ``gdb`` cannot attach to a running process, ``strace`` and ``ltrace`` fail immediately, ``perf`` reports insufficient permissions, and no core dump appears after a crash.

This is several measures acting together:

* ``kernel.yama.ptrace_scope`` is set to ``3``, which disables ``ptrace`` entirely. No process can attach to another, and a process cannot mark itself traceable either. This stops attaching debuggers and tracers outright, rather than merely restricting them.
* ``kernel.perf_event_paranoid`` is set to ``3``, which restricts performance monitoring.
* ``kernel.unprivileged_bpf_disabled`` is set to ``1``, so unprivileged BPF is unavailable.
* Core dumps are disabled both in the kernel and through the process limits.
* Kernel build configuration affects these as well.

**Fixing it.** These are runtime settings in the ``sysctl`` configuration that ``meta-sulka-kernel`` installs, so they can be overridden from your own layer for development images.
``ptrace_scope`` in particular cannot be relaxed by simply raising a limit at runtime once it has been set, so plan to build a development image rather than expecting to loosen it on a shipped device.

You can also try disabling the kernel hardening to see if that helps by adding the following to your build configuration:

.. code-block::

  SULKA_HARDEN_KERNEL = "0"

Consider whether you can reproduce the problem on a development host instead.
The restrictions exist precisely because these interfaces are powerful, and a device that permits debugger attachment in the field is a device that permits it for an attacker too.

U-Boot Does Not Boot The Kernel
*******************************

**Symptoms.** With verified boot enabled, U-Boot prints an error after loading the FIT image and resets, and the device ends up in a reset loop:

.. code-block::

  Verifying Hash Integrity ... sha256,rsa4096:sulka-fit-  error!
  Failed to verify required signature 'key-sulka-fit'
  Bad Data Hash
  ERROR -2: can't get kernel image!
  No CLI available
  resetting ...

This is verified boot refusing an image it cannot verify.
``Failed to verify required signature`` means that the configuration of the FIT image is not signed with the key built into U-Boot, and ``Bad hash value`` means that an image in the FIT image has been modified after signing.

**Confirming the cause.** Check whether the FIT image and U-Boot come from builds with the same key.
The most common cause is a FIT image signed with a new key on a device that still runs a U-Boot built with the old one, for example after regenerating the keys.
You can verify the FIT image against the U-Boot control device tree on the build host with ``fit_check_sign`` from the ``u-boot-tools`` package:

.. code-block::

  fit_check_sign -f fitImage -k u-boot.dtb

**Fixing it.** Update U-Boot together with the FIT image, or sign the FIT image with the key U-Boot was built with.
If U-Boot prints nothing about the FIT image, check that the boot command of the machine loads the FIT image and boots it with ``bootm``, as the verified boot feature of Sulka disables the commands that boot other image types.
See :ref:`Verified Boot`.

dm-verity Refuses The Root File System
**************************************

**Symptoms.** The console or the kernel log shows dm-verity refusing blocks of the root file system:

.. code-block::

  device-mapper: verity: 253:0: data block 10807 is corrupted

The reads of the refused blocks fail with input/output errors, and what happens next depends on what the blocks contain.
If they are needed to boot, the boot stops with a kernel panic.
For example, with a modified block in the C library, ``init`` cannot start:

.. code-block::

  /sbin/init: error while loading shared libraries: /lib/libc.so.6: cannot read file data: Input/output error
  Kernel panic - not syncing: Attempted to kill init! exitcode=0x00007f00

Otherwise the system boots, and only the programs that read the modified files fail.

When the whole root file system comes from another build than the root hash in the initramfs, dm-verity refuses the metadata blocks of the hash tree too, the initramfs cannot mount the root file system, and the kernel panics:

.. code-block::

  device-mapper: verity: 253:0: metadata block 52248 is corrupted
  erofs (device dm-0): cannot read erofs superblock
  mount: mounting /dev/mapper/rootfs on /rootfs failed: Invalid argument
  Kernel panic - not syncing: Attempted to kill init! exitcode=0x00000200

If the image on the storage has no hash tree at all, the console shows ``Device /dev/vda is not a valid VERITY device.`` instead, with the same kernel panic.

**Confirming the cause.** Check that the root file system on the storage and the kernel image, or the FIT image with verified boot, come from the same build.
The most common causes are a root file system image that has been modified after the build, a disk image that contains the plain ``erofs`` image instead of ``erofs.verity``, and updating only one of the two.
Under QEMU, ``runqemu`` can also boot the root file system image of an earlier build when only the image (but not rootfs) has been rebuilt, as it looks first for the image named in the ``qemuboot.conf`` file, which is only written when the root file system is built.

**Fixing it.** Write the root file system and the kernel image from the same build, and make your changes in the image recipes rather than in the built image.
If you need to modify the root file system after the build, disabling dm-verity is not enough, as the root file system is still a read-only ``erofs`` image.
Disable the read-only root file system with ``SULKA_ENABLE_READ_ONLY_ROOTFS``, which disables dm-verity too.
See :ref:`dm-verity`.

Under QEMU, give the root file system image to ``runqemu`` explicitly:

.. code-block::

  kas shell kas-sulka.yml \
    -c 'runqemu nographic slirp tmp/deploy/images/qemux86-64/core-image-base-qemux86-64.rootfs.erofs.verity'

The Build Fails
***************

A few build failures are Sulka-specific rather than ordinary Yocto problems:

* **Missing module signing keys.** Module signing is enabled by default and the build fails without keys. See :ref:`Module Signing`.
* **Incomplete verified boot configuration.** With verified boot enabled, the build fails if the key given with ``UBOOT_SIGN_KEYDIR`` and ``UBOOT_SIGN_KEYNAME`` does not exist, if ``SULKA_FIT_IMAGE_RECIPE`` is not set, or if the bootloader is not U-Boot. See :ref:`Signing Keys` and :ref:`Machine Integration`.
* **dm-verity with a writable root file system.** dm-verity requires the read-only root file system, so the build fails if ``SULKA_ENABLE_DM_VERITY`` is ``1`` while ``SULKA_ENABLE_READ_ONLY_ROOTFS`` is ``0``. See :ref:`dm-verity`.
* **dm-verity without the target image.** dm-verity protects the image set in ``DM_VERITY_IMAGE``, and the build fails if it is not set. Set it to your target image. See :ref:`dm-verity`.
* **Password expiration with the read-only root file system.** An expired password cannot be changed on a read-only root file system, so the build fails if ``SULKA_EXPIRE_PASSWORDS`` is ``1``, or ``SULKA_EXTRA_COMPLIANCY`` enables it, while ``SULKA_ENABLE_READ_ONLY_ROOTFS`` is ``1``. Disable one of them, or set ``SULKA_WRITABLE_PASSWORD_STORAGE`` to ``1`` if your partition layout keeps ``/etc`` writable. See :ref:`Configuration Variables`.
* **A dangling bbappend.** Sulka bbappends wildcard the patch release of the recipe version they target, so a version outside that range fails the build rather than silently dropping the hardening. Match the Yocto release that Sulka targets.
* **The fstab hardening did not apply.** If you ship your own ``fstab``, the hardening may not match it, and the build fails rather than leaving the mounts unhardened. Set ``SULKA_HARDEN_MOUNTS`` to ``0`` and harden your own ``fstab`` instead. See ``meta-sulka-distro`` for guidance.
* **Many kernel configuration warnings.** Disabling kernel modules converts many options from modules to built-ins, which Yocto reports as a mismatch between the requested and resulting configuration. See :ref:`Disabling Kernel Modules`.
