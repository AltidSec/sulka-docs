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
If a correctly signed external module still fails to load, kernel configuration ``CONFIG_RANDSTRUCT_FULL`` may be the cause, as it changes internal structure layout.

If you cannot sign your modules at all, ``SULKA_ENABLE_MODULE_SIGNING`` can be disabled to disable the enforcement, at a real cost to the security of the system.

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

The Build Fails
***************

A few build failures are Sulka-specific rather than ordinary Yocto problems:

* **Missing module signing keys.** Module signing is enabled by default and the build fails without keys. See :ref:`Module Signing`.
* **A dangling bbappend.** Many Sulka bbappends name an exact upstream recipe version, so a mismatched version fails the build rather than silently dropping the hardening. Match the Yocto release that Sulka targets.
* **The fstab hardening did not apply.** If you ship your own ``fstab``, the hardening may not match it, and the build fails rather than leaving the mounts unhardened. Set ``SULKA_HARDEN_MOUNTS`` to ``0`` and harden your own ``fstab`` instead. See ``meta-sulka-distro`` for guidance.
* **Many kernel configuration warnings.** Disabling kernel modules converts many options from modules to built-ins, which Yocto reports as a mismatch between the requested and resulting configuration. See :ref:`Disabling Kernel Modules`.
