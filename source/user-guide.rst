User Guide
##########

This page provides more in-depth information about Sulka for the developers that are interested in using the distro.

Supported Configurations
************************

Sulka aims to provide support for some common alternatives related to Linux systems.
However, for time being the amount of supported options is limited to keep the development pace quick.

Init Managers
=============

Currently, Sulka primarily supports ``systemd`` as the init manager.
Using ``sysvinit`` as the init manager should be possible, and all the custom services have their ``sysvinit`` counterparts.
Basic testing is performed on both init managers, but ``systemd`` is prioritized in the development.

Mandatory Access Control Modules
================================

Sulka currently supports SELinux as its mandatory access control module.
While enabling support for AppArmor is a long-term goal, it is not actively being worked on at the moment.
You can use AppArmor to harden your own services, but a system-wide hardening policy for AppArmor is not currently available.

For more information about SELinux, see the :ref:`SELinux` section.

Installed Packages
******************

Sulka installs packages as a part of the distro. You can find the packages listed here, along with the explanation of what they do and why they're installed.

* ``audit``

  audit is an auditing package that can be used to watch files and syscalls. These actions taken on these files or syscalls can then be logged into the auditing log, detecting undesired behavior.

* ``cronie`` (installed if ``sysstat`` is installed)

  Cronie is a system utility that is used to run tasks periodically. This is useful for system monitoring, when activities are checked at specific times.

* ``dpkg-start-stop``

  This is a dependency for the ``audit`` init script, as the init script relies on options that are not available on Busybox's ``start-stop-daemon``.

* ``nftables``

  nftables is the packet filter / firewall used in Sulka.
  Firewall is a crucial part of the network security, and ``nftables`` provides a good performance and efficiency with a unified framework for packet filtering.
  Additionally, nftables is well-integrated with modern Linux kernels.

* ``nftables-configuration``

  nftables-configuration is a service that loads the firewall rules during boot.

* ``packagegroup-core-boot``

  This is the core packagegroup from Yocto project that includes the essentials for the system.

* ``packagegroup-selinux-minimal`` (installed if SELinux is enabled)

  The minimal set of SELinux userspace tooling and the reference policy itself.
  Sulka drops a few packages from this packagegroup, as they depend on components that Sulka avoids for licensing reasons.

* ``packagegroup-sulka-netfilter-modules`` (installed unless kernel modules are disabled)

  The kernel modules that the firewall rules need for connection tracking, rate limiting and logging.
  These are not installed when kernel modules are disabled, in which case the corresponding functionality has to be built into the kernel.

* ``passwdqc``

  passwdqc is a package that provides password quality enforcement. This should prevent users from using insecure passwords.

* ``restorecon-post-mount`` (installed if SELinux is enabled on a ``sysvinit`` system)

  Restores the SELinux file contexts after the file systems have been mounted, which ``systemd`` systems handle on their own.

* ``sudo``

  sudo is the package that is used to allow service user to perform actions with root capabilities.
  Since the root user is locked in Sulka, it is recommended to install sudo if there is a service user in the system

* ``syslog-ng``

  syslog-ng is the logging daemon used in Sulka. It is used to replace Busybox syslog daemon, as that lacks several security features like log signing and secure network transport.

* ``sysstat`` (installed if monitoring is enabled)

  sysstat is used for periodically checking the system resource usage to detect anomalous activity in the system.

Firewall
********

Sulka comes with ``nftables`` firewall installed and configured.
By default, the firewall drops all incoming and outgoing traffic.
There are a few different firewall templates that you can use to configure the firewall behavior.
The firewall template is selected with the ``SULKA_NFTABLES_CONF`` BitBake configuration variable.
You can either use your own configuration file and add it to the build by appending to the ``nftables-configuration`` recipe, or use one of the following configuration samples:

* ``nftables-drop-everything.conf``

  Drop all traffic: incoming, outgoing, and forwarding.

* ``nftables-allow-established-lo-outgoing.conf``

  Allow all outgoing and incoming established traffic (e.g., responses to outgoing traffic), and loopback traffic.

* ``nftables-allow-established-lo-ssh-icmp-outgoing.conf``

  Allow all outgoing and incoming established traffic (e.g., responses to outgoing traffic), ICMP (e.g., ping), SSH, and loopback traffic. Additionally, log dropped incoming traffic.

SSH
***

Sulka does not install an SSH server by default, as not every device needs remote access.
One is installed by the development configuration fragment, and you can of course add one to your own build.
Whenever it is present, Sulka hardens its configuration. Most of these values originate from ``meta-security/meta-hardening``.
This applies to OpenSSH only.

The most consequential change is that the server accepts key-based authentication only.
Password authentication is disabled and root login is refused, so the service user password works on the serial console but not over the network.
An image that has an SSH server but no authorized key installed cannot be reached over SSH at all.
See :ref:`Installing SSH Keys` for installing a key at build time.

In addition, the following hardening is applied to ``sshd`` configuration:

* ``MaxAuthTries 3`` limits the authentication attempts allowed per connection.
* ``MaxSessions 2`` limits the concurrent sessions per connection, down from the default of 10.
* ``AllowTcpForwarding no`` and ``AllowAgentForwarding no`` prevent the connection from being used to tunnel other traffic or to reach the client's authentication agent.
* ``LogLevel VERBOSE`` records the fingerprint of the key used for each login, which is what makes an SSH login traceable to a specific key afterwards.
* ``TCPKeepAlive no`` with ``ClientAliveCountMax 2`` drops unresponsive connections after two missed probes at the upstream fifteen second interval, relying on the authenticated keepalive rather than the plain TCP one.
* ``Banner /etc/issue.net`` presents the restricted system warning before login.

In addition, the ``sshd`` service user is given ``/sbin/nologin`` as its shell, and both configuration files are installed readable only by root.

The listening port can be moved off the default with ``SULKA_SSH_PORT``.
Note that this is obscurity rather than security, and the firewall still has to permit whichever port you choose. See :ref:`Firewall`.

Monitoring
**********

Sulka can install monitoring packages that are used to monitor the system and can be used to detect anomalies.
However, these are not installed by default as they are not essential for the operation and to get the most use of them the user would have to set up a remote logging system.
Currently, the monitoring packagegroup contains only ``sysstat`` package.

To enable monitoring, set the following flag in your build configuration

.. code-block::

  SULKA_ENABLE_MONITORING = "1"

This could be done for example in the ``local.conf``.

Note that enabling ``SULKA_EXTRA_COMPLIANCY`` option automatically enables the monitoring feature as well.

To get the most of these monitoring capabilities, your system should satisfy the following requirements:

* Persistent logging partition. Logging to volatile locations causes the logs to be lost in case of a power loss or reboot.
* Enough storage space. The monitoring tools can create large logs, so it is important to ensure there is enough storage for them.
* Remote logging. While this is not mandatory, it is useful for timely anomaly detection and log storage.

The monitoring can be useful without fulfilling these requirements, but the usefulness may be limited as the logs may be lost before analysis or cannot be analyzed remotely/automatically.

It is recommended that you run a long test with the system running the usual load to see how large the logs grow in your system and if the system properly rotates the logs.
After that, you can either configure or disable the monitoring functionality as required.

SELinux
*******

SELinux is the supported mandatory access control module in Sulka, enabling further hardening the system access controls.
SELinux is enabled by default.

To disable SELinux, add the following to your build configuration (for example, in the ``local.conf``):

.. code-block::

   SULKA_MANDATORY_ACCESS_CONTROL_MODULE = "none"

The default reference policy of the SELinux is set to ``targeted``.
This setting aims to protect core services while minimizing disruption to normal system operation.

A few patches have been applied to this reference policy to address certain denial issues.
These patches can be found from ``meta-sulka-distro/dynamic-layers/selinux/recipes-security/refpolicy/refpolicy-targeted``.
Note that several of them are applied conditionally, depending on the init manager, whether an SSH server is installed, and whether the read-only root file system and monitoring are enabled.
You should review these patches to ensure their changes align with your use case.

You may want to consider switching the policy to stricter ``standard`` for production systems.
This provides enhanced security, but for example interactive login sessions are limited in functionality.
Note that the Sulka specific patches apply only to the ``targeted`` refpolicy. If you change the refpolicy you need to edit the recipe files.

Kernel Modules
**************

Linux kernel modules provide a way to add code to highly-privileged kernel-space.
This is commonly used, for example, for drivers.
However, this also provides a dangerous attack surface, as adversaries can try to exploit this loading mechanism.

For this purpose, you will want to control the modules that can be loaded into the kernel.
Sulka provides two ways to control this: either by disabling the kernel modules completely, or by enforcing module signing.

Disabling Kernel Modules
========================

When Linux kernel modules are disabled, the kernel is fully defined during the build, meaning that no code can be loaded during runtime.
This is the most secure way to approach the module security issue.
However, if you have for example hardware that has binary drivers that you cannot compile into the kernel, or you have to build some modules out-of-tree, this is not a suitable option.
If that's the case, check the next section, :ref:`Module Signing`, for an alternative that could be more suitable.

To disable kernel modules, you can set the following in your build configuration:

.. code-block::

   SULKA_DISABLE_KERNEL_MODULES = "1"

Note that this feature will print out a lot of warnings during the build.
This happens because Yocto checks that the wanted build configuration matches the actual build configuration.
However, because the feature disables the modules, many ``m`` options get converted into ``y`` options, causing a mismatch between expectations and reality, which triggers the warnings.

Because of the build warnings and incompatibility with some systems, the feature is disabled by default.
Note that enabling ``SULKA_EXTRA_COMPLIANCY`` disables the kernel modules automatically.

Module Signing
==============

Module signing functionality allows signing kernel modules to prevent unauthorized code from being loaded into the kernel.
This helps preventing kernel-level attacks, like installing rootkits, keyloggers, or malicious drivers.

Module signing is enabled by default, as it is **strongly** recommended for securing the kernel.
Because the feature requires signing keys, you must generate them and point the build to them, otherwise the build will fail.

To provide the keys, first generate them using the ``generate_ima_evm_modsign_keys.sh`` script in `kas-sulka <https://codeberg.org/AltidSec/kas-sulka/src/branch/wrynose/scripts/generate_ima_evm_modsign_keys.sh>`_.
Then, add the location of the keys and the certificate authority to your build configuration:

.. code-block::

   MODSIGN_KEY_DIR = "/path/to/generated/keys"
   IMA_EVM_ROOT_CA = "${MODSIGN_KEY_DIR}/ima-local-ca.pem"

The script generates a certificate authority and signs both a module signing certificate and an IMA and EVM certificate with it, which is why the variable names mention IMA and EVM.
This is the shared signing key infrastructure only.
IMA and EVM themselves are not enabled in Sulka, as they conflict with SELinux, so the keys they would use are generated but not put to work.

Module signing is controlled with the ``SULKA_ENABLE_MODULE_SIGNING`` option, which defaults to ``1``.
If you cannot provide signing keys, you can disable the feature by setting ``SULKA_ENABLE_MODULE_SIGNING = "0"`` in your build configuration.
This is **not** recommended, as it leaves the kernel able to load unsigned modules.


Please note that this feature does not sign binary drivers that are not compiled during the build.
It is possible to sign these kind of drivers, but at the moment it has to be done manually before building the firmware image.
See :ref:`Signing External Modules` for instructions on how to do this.

Signing External Modules
------------------------

When module signing is enforced, the kernel refuses to load any module whose signature it cannot verify against a trusted key.
Modules that are compiled as part of the Yocto build are signed automatically.
This includes out-of-tree modules that are built through their own Yocto recipes, as they are signed during the build just like the in-tree modules.

The modules that are **not** signed for you are the ones that the Yocto build never builds itself.
Typical examples are prebuilt binary drivers shipped as ready-made ``.ko`` files, and modules that you compile by hand outside of the Yocto build.
These modules have to be signed manually with the same key that is built into the kernel keyring, otherwise the kernel will reject them at load time.

External modules are signed using the ``sign-file`` script that ships with the kernel source.
The script appends a signature to the ``.ko`` file using the module signing private key and certificate.
These are the same keys that you generated with the ``generate_ima_evm_modsign_keys.sh`` script and pointed to with ``MODSIGN_KEY_DIR``, so make sure you use the exact key pair that was used for the build.
If you sign a module with a different key, the kernel will not trust it.

To sign an external module, run the ``sign-file`` script with the hash algorithm, the private key, the certificate, and the module to sign:

.. code-block::

   scripts/sign-file sha512 "${MODSIGN_KEY_DIR}/privkey_modsign.pem" "${MODSIGN_KEY_DIR}/x509_modsign.crt" my-external-module.ko my-external-module-signed.ko

A few notes about the command:

* The hash algorithm (``sha512`` above) must match the one configured for module signing in the kernel. Check your kernel configuration if you are unsure. ``sha512`` is the default in Sulka.
* The ``sign-file`` script can be found under ``scripts/`` in the kernel source tree. In a Yocto build, you can locate it under the kernel recipe's build directory.

After signing, package the signed module into your firmware image as you normally would, for example through your own meta-layer recipe.
Because the signing has to happen before the image is assembled, it must be done before building the firmware image.

You can confirm that a module carries a signature by checking for the signature marker at the end of the file:

.. code-block::

   modinfo my-external-module-signed.ko | grep -E "^sig"

On the target, the kernel logs a message if it rejects an unsigned or incorrectly signed module.
If a module fails to load with module signing enabled, verify that it was signed with the correct key pair and that the same key is trusted by the running kernel.
You may also need to disable ``CONFIG_RANDSTRUCT_FULL`` enabled by Sulka configuration, as that may cause problems when loading external modules.

Read-Only Root File System
**************************

Sulka supports read-only root file system, and it is enabled by default to prevent modifications to the contents of the root file system.
Read-only root file system is controlled with ``SULKA_ENABLE_READ_ONLY_ROOTFS`` option.
To disable read-only root file system, set the option to ``0``.

This option adds ``erofs`` (enhanced read-only file system) to the ``IMAGE_FSTYPES``.
``erofs`` is used by default with ``runqemu`` command, but you'll need to manually add this to your Wic-images (or whatever you are using to build your images).
The option also adds ``read-only-rootfs`` to ``EXTRA_IMAGE_FEATURES``, in practice mounting the rootfs read-only, adding some read-only compatible configurations, and ensuring that there are no on-target post-installation tasks.
Finally, the option attempts to add ``ro`` kernel command-line parameter with ``APPEND`` and ``CMDLINE`` variables, but you should ensure that it gets properly added on your system.

You may want to add writable locations to your images. There are a few ways to achieve this:

* Writable partitions. In practice, adding extra partitions that are read-write (and preferably ``noexec``)
* Overlays. Adding a writable overlay with ``overlayfs`` to the root file system allows straightforward write support.
  Be aware that ``overlayfs`` does not work well with SELinux, which Sulka enables by default, because the overlay interferes with the file labelling.
  In practice this option is only open to you if you disable the mandatory access control, so treat it as a trade between write support and mandatory access control rather than as a drop-in choice.
  The Rugix firmware update reference makes that trade in the other direction: it disables its own overlay so that SELinux can stay enabled. See :ref:`Firmware Update Example`.
* Temporary file systems. If you want to create a completely stateless image, using temporary file systems is a good idea as it ensures that none of the written information is stored.
* Bind mounts. Bind mounts allow mounting individual directories as required. The mounted directory can be on an extra partition or ``tmpfs``, depending on whether you want to store the information.
* Symlinks. Symlinks can be created in the root file system for files that are expected to be writable. These links can then point to writable partitions or ``tmpfs`` locations.

Note that this feature does not prevent offline modifications to the file system.
To add integrity checking to the root file system, look into dm-verity.

Bootloader (U-Boot)
*******************

Sulka contains hardening features for the bootloader.
U-Boot is used as the reference bootloader as it is quite commonly used.

Locking Command Line Interface
==============================

In Sulka, the command line interface of U-Boot is locked behind the "stop string" functionality by default.
This "stop string" is a password in practice.
The default value of the stop string is an empty string, meaning that the command line interface should effectively be locked by default.

To set the password for the command line interface, use the ``SULKA_UBOOT_PASSWORD`` variable.
The value should be an SHA-512 password hash. You can generate a suitable password with the following command:

.. code-block::

   mkpasswd -m sha-512 -R 10000 PASSWORD

When setting the ``SULKA_UBOOT_PASSWORD`` variable, escape all dollar signs and slashes in the output with backslashes, for example:

.. code-block::

   # Do not use this password, it is just an example. Use the command above to generate a custom password
   SULKA_UBOOT_PASSWORD = "\$6\$rounds=10000\$UAOalptsTK98MxhA\$OE8G1lKVxgLt49fumHZUjtjLIp65Fwk.fKiJ8L6Ig2seKiG2iks5gWF\/GfCr0gg2RZNvEivVJ87gC\/0GPiiv3."

Commands
========

By default, U-Boot builds a lot of commands into its command line interface.
However, some commands can pose security risks and it is best to minimize the amount of commands.
Sulka disables some U-Boot commands to reduce attack surface.
However, some of these disabled commands may be required for your boot flows.
It is recommended to check the ``meta-sulka-bsp/recipes-bsp/u-boot/u-boot/sulka_harden_configuration.cfg`` configuration to see what commands are disabled.

In addition, Sulka adds a command allowlisting feature.
This feature allows defining the exact commands that are allowed to be executed during the autoboot process.
It is difficult to disable all the unnecessary commands, and sometimes certain commands have to be left in the bootloader for maintenance purposes.
The allowlisting feature allows executing all the commands if the CLI is opened, but prevents commands that are not in the allowlist during autoboot.

Allowlisting is disabled by default, as it is impossible to have a sane default value that would fit all the devices and use cases.
To enable the allowlist, add the following to your U-Boot configuration:

.. code-block::

   CONFIG_COMMAND_ALLOWLIST=y
   CONFIG_COMMAND_ALLOWLIST_CMDS="space separated list of allowed commands"

For a real example of an allowlist, see the Raspberry Pi reference, which enables the feature and lists the commands its boot flow needs in
``meta-sulka-raspberrypi/recipes-bsp/u-boot/files/sulka_raspberrypi.cfg``.

Environment
===========

U-Boot has an environment that can be used to control the boot.
The environment can be built-in, and it can also be loaded externally.
However, loading external environments can be a security risk as U-Boot does not verify its integrity.
Sulka disables the external environments by default, but if you need a writable external environment, enable it in the U-Boot configuration.
Check ``meta-sulka-bsp/recipes-bsp/u-boot/u-boot/sulka_harden_configuration.cfg`` for environment configurations.

Note that the external environment **should not** contain critical boot information that may result in losing control of the boot flow.

Configuration Variables
***********************

This chapter covers the configuration items in Sulka. The default value for each configuration is in the parentheses after the name

* ``SULKA_DEVELOPMENT_MODE`` (0)

  Enable a development mode where some security measures and configurations are lowered or disabled to make development and debugging work easier.
  This option should not be enabled on production builds, and a warning will be printed if this option is enabled.

* ``SULKA_DISABLE_GRAPHICS`` (1)

  This option allows enabling or disabling the graphics to reduce kernel attack surface.
  Requires `meta-sulka-kernel <https://codeberg.org/AltidSec/meta-sulka-kernel>`_ to be part of the build.
  If your device does not have a graphic output, you should be able to leave this to default.

* ``SULKA_DISABLE_KERNEL_MODULES`` (0)

  Set this option to ``1`` to disable loadable modules in the Linux kernel.
  Linux kernel modules allow loading code to highly privileged kernel space, potentially allowing dangerous exploits.
  Disabling kernel modules prevents this.
  If you have binary drivers that you cannot compile yourself, or you build out-of-tree modules, this option is not suitable for you.
  See :ref:`Disabling Kernel Modules` for more information.

* ``SULKA_ENABLE_MODULE_SIGNING`` (1)

  Set this option to ``0`` to disable module signing.
  Module signing is enabled by default and it is strongly recommended to keep it enabled, but it requires signing keys to be provided or the build will fail.
  See :ref:`Module Signing` for more information on providing the keys or disabling the feature.

* ``SULKA_ENABLE_MONITORING`` (0)

  Set this option to ``1`` to install ``packagegroup-sulka-monitoring``.
  This packagegroup contains utilities that can be used to monitor the system.
  Currently, this installs only ``sysstat``.
  See :ref:`Monitoring` for more information.

* ``SULKA_ENABLE_READ_ONLY_ROOTFS`` (1)

  Set this option to ``1`` to make the root file system read-only, or ``0`` to make it writable.
  For ensuring the integrity of the root file system, it is recommended to keep this ``1``.
  See :ref:`Read-Only Root File System` for more information.

* ``SULKA_EXPIRE_PASSWORDS`` (0)

  Set the passwords to expire in the system.
  This is disabled by default, as this requirement does not usually translate well into embedded systems.
  However, if you perform user management on the Linux user-space level, it is recommended to enable this.

* ``SULKA_EXTRA_COMPLIANCY`` (0)

  Enable the extra compliancy settings that may be required by some audits and compliancy checks.
  These are options that may make more sense in workstation or server use, or that need some integration work, so they are disabled by default.
  This option enables the following options:

  * ``SULKA_DISABLE_KERNEL_MODULES``
  * ``SULKA_ENABLE_MONITORING``
  * ``SULKA_EXPIRE_PASSWORDS``

  It is recommended to go through the options that are enabled by ``SULKA_EXTRA_COMPLIANCY``, and enable them manually if enabling the whole ``SULKA_EXTRA_COMPLIANCY`` feature is not possible.

* ``SULKA_HARDEN_KERNEL`` (1)

  Harden the kernel configuration. Requires `meta-sulka-kernel <https://codeberg.org/AltidSec/meta-sulka-kernel>`_ to be part of the build.

* ``SULKA_HARDEN_MOUNTS`` (1)

  This variable can be used to control whether the mount options are hardened or not.
  The hardening covers the default mounts of the system only.
  Mounts that you add yourself are never touched by this variable, and hardening them is always up to you.

  In ``fstab``, the hardening adds the ``hidepid=2`` option to the ``/proc`` mount, and the ``nodev,nosuid,noexec`` options to the ``/run`` and ``/var/volatile`` mounts.
  These shoud apply cleanly to the stock ``fstab`` from ``base-files`` recipe, but if you override it and your ``fstab`` differs, the hardening may not necessarily apply cleanly.
  In that situation, disable the hardening by setting this variable to ``0`` and ensure that your own ``fstab`` has secure options.

  On ``systemd`` systems the hardening also adds ``noexec`` to systemd's ``tmp.mount`` unit, which is what mounts ``/tmp``.
  ``systemd`` already mounts ``/tmp`` with ``nosuid`` and ``nodev``.
  On ``sysvinit`` systems ``/tmp`` is a symbolic link into ``/var/volatile`` instead, and it inherits the options from that mount.

  Note that ``noexec`` mount option may cause issues if you run scripts or programs in ``/run`` or ``/tmp``.
  The same applies to ``/var/lib`` and ``/var/cache``, which is easy to miss: with the read-only root file system those are bind mounted from ``/var/volatile``, and they inherit the mount options, so ``noexec`` reaches them too.
  First, consider if it is possible to modify your system so that the scripts can be run elsewhere.
  If not, you'll need to disable this feature and set the hardening flags yourself.

* ``SULKA_INSTALL_SSH_KEYS`` (0)

  Install SSH public key information into the firmware image.
  The key is searched from ``${SULKA_SSH_KEYS_DIR}`` directory, and is assumed to have a name in the form of ``${SULKA_SERVICEUSER_USERNAME}-auth-key.pub``.

  Note that installing the SSH key information to the firmware during build may pose a security risk.
  If the private key leaks, all the devices using the same firmware image become vulnerable.
  Consider generating unique SSH keys for each device if that is possible for your use case.

* ``SULKA_MANDATORY_ACCESS_CONTROL_MODULE`` ("selinux")

  Set the mandatory access control module to be used in the system.

  Mandatory access control architecture provides more control over the file and process permissions than the regular discretionary access control.
  This architecture increases the complexity of the system, and can result in problems if the system behavior is unpredicatable or changes often.
  However, enabling the mandatory access control is usually a good idea.

  Set this to ``none`` to disable the mandatory access control.

* ``SULKA_NFTABLES_CONF`` ("nftables-drop-everything.conf")

  The firewall configuration that gets installed to the system and is used as the default firewall configuration.
  See :ref:`Firewall` for more information.

* ``SULKA_RUGIX_ROOT_CERT`` (no default value)

  The path to the root certificate that is used to sign the update bundle signing certificate and that should be deployed to the firmware image.
  See :ref:`firmware-update` for more information.

* ``SULKA_SERVICEUSER_ENABLE_SUDO`` ("0")

  Enable default sudo configuration for the service user by setting this to ``1``.
  The default sudo configuration allows full root privileges for the service user when they use sudo, making them effectively a root user.
  You may want to consider more granular sudo configuration with multiple users on production systems.

* ``SULKA_SERVICEUSER_PASSWORD`` (no default value)

  The password that the service user uses to log in.
  This is not set by default, and if you do not set a password, the service user will not be added.
  See the instructions in the :ref:`quick-start` for the password creation and setting.

* ``SULKA_SERVICEUSER_USERNAME`` ("serviceuser")

  The name of the service user that can be used to log in to the system.
  It is recommended to change this into something else.

* ``SULKA_SSH_KEYS_DIR`` ("${TOPDIR}/../auth-keys")

  The directory where the public SSH key will be looked from if ``SULKA_INSTALL_SSH_KEYS`` is set to ``1``.

* ``SULKA_SSH_PORT`` (22)

  Allows configuring the SSH server to listen in a non-standard port.
  By default, the standard port 22 is used.

* ``SULKA_UBOOT_PASSWORD`` (no default value)

  The password that can be used to log in to the U-boot command line interface.
  By default, no password is set and the command line interface is inaccessible.
