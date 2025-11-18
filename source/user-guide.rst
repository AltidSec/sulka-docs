User Guide
##########

This page provides more in-depth information about Sulka for the developers that are interested in using the distro.

Using Sulka For Your Project
****************************

If you are interested in using Sulka in your project, the usage is quite similar to the official reference distribution Poky.
In practice, this means building your solution on top of the Sulka meta-layers, and adding your own meta-layers alongside them.

To start building your own system on top of Sulka, you will most likely want to create a kas configuration repository.
The easiest way to do this to fork `kas Sulka repository <https://codeberg.org/AltidSec/kas-sulka>`_, and add your own configuration files to the fork next to the Sulka configuration files.
This approach allows you to easily work with the Sulka distro, configure it as required, and rebase your work on top of updates

You can find an example of how Sulka has been ported to a Raspberry Pi from `the kas Raspberry Pi example repository <https://codeberg.org/AltidSec/kas-sulka-raspberrypi-example.git>`_.

Supported Configurations
************************

Sulka aims to provide support for the common alternatives related to Linux systems.

Init Managers
=============

Currently, Sulka primarily supports sysvinit as the init manager.
Using systemd as the init manager should be possible, and all the custom initialization scripts have their systemd service counterparts. However, systemd is not actively used and tested, so there may be some things missing.
More complete systemd support is planned for the future. Please raise an issue in the ``meta-sulka-distro`` repository if getting this is urgent to you.

Mandatory Access Control Modules
================================

Sulka currently supports SELinux as its mandatory access control module.
While enabling support for AppArmor is a long-term goal, it is not actively being worked on at the moment.
You can use AppArmor to harden your own services, but a system-wide hardening policy for AppArmor is not currently available.

For more information about SELinux, see the :ref:`SELinux` section.

Installed Packages
******************

Sulka installs packages as a part of the distro. You can find the packages listed here, along with the explanation of what they do and why they're installed.


* ``acct`` (installed if monitoring is enabled)

  acct is the GNU Accounting Utilities package that is used to perform process monitoring. This can be used to check what processes have been started in the system, and when the processes have been started.

* ``aide`` (installed if monitoring is enabled)

  AIDE stands for Advanced Intrusion Detection Environment, and it is used as a file integrity monitor in Sulka. With periodic checks, AIDE can detect changes in files and report these.

* ``audit`` (installed if monitoring or SELinux is enabled)

  audit is an auditing package that can be used to watch files and syscalls. These actions taken on these files or syscalls can then be logged into the auditing log, detecting undesired behavior.

* ``cronie``

  Cronie is a system utility that is used to run tasks periodically. This is useful for system monitoring, when activities are checked at specific times.

* ``dpkg-start-stop``

  This is a dependency for the ``audit`` init script, as the init script relies on options that are not available on Busybox's `start-stop-daemon`.

* ``nftables``

  nftables is the packet filter / firewall used in Sulka.
  Firewall is a crucial part of the network security, and ``nftables`` provides a good performance and efficiency with a unified framework for packet filtering.
  Additionally, nftables is well-integrated with modern Linux kernels.

* ``nftables-configuration``

  nftables-configuration is a service that loads the firewall rules during boot.

* ``packagegroup-core-boot``

  This is the core packagegroup from Yocto project that includes the essentials for the system.

* ``passwdqc``

  passwdqc is a package that provides password quality enforcement. This should prevent users from using insecure passwords.

* ``rsyslog``

  rsyslog is the logging daemon used in Sulka. It is used to replace Busybox syslog daemon, as that lacks several security features like log signing and secure network transport.

* ``sudo``

  sudo is the package that is used to allow service user to perform actions with root capabilities.
  Since the root user is locked in Sulka, it is recommended to install sudo if there is a service user in the system

* ``sysstat`` (installed if monitoring is enabled)

  sysstat is used for periodically checking the system resource usage to detect anomalous activity in the system.

Recommended Packages
====================

In addition to the previously listed packages, Sulka recommends the following packages that are installed by default:

* ``bash``

  Bash is used in the testing images, as it is a dependency for some tools.
  Also, it fixes shell profile reading issue that occurs when Busybox shells are used with SELinux.
  Therefore it is recommended to be used with Sulka.

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

Monitoring
**********

Sulka can install multiple packages that are used to monitor the system and can be used to detect anomalies.
However, these are not installed by default as they require configuration.
These packages are ``acct``, ``aide``, ``auditd``, and ``sysstat``.

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
After that, you can either configure or disable some of the monitoring functionality as required.

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
These patches can be found from ``recipes-security/refpolicy/refpolicy-targeted``.
You should review these patches to ensure their changes align with your use case.

You may want to consider switching the policy to stricter ``standard`` for production systems.
This provides enhanced security, but for example interactive login sessions are limited in functionality.
Note that the Sulka specific patches apply only to the ``targeted`` refpolicy. If you change the refpolicy you need to edit the recipe files.

Module Signing
**************

Module signing functionality allows signing kernel modules to prevent unauthorized code from being loaded into the kernel.
This helps preventing kernel-level attacks, like installing rootkits, keyloggers, or malicious drivers.

The feature is disabled by default, as it breaks the build if no keys are provided.
However, it is **strongly** recommended that you enable module signing.

To enable the feature, first generate the keys using the ``generate_ima_evm_modsign_keys.sh`` script in `kas-sulka <https://codeberg.org/AltidSec/kas-sulka/src/branch/scarthgap/scripts/generate_ima_evm_modsign_keys.sh>`_.
Then, enable the key signing feature and add the location to the keys and certificate authority in your build configuration:

.. code-block::

   SULKA_ENABLE_MODULE_SIGNING = "1"
   MODSIGN_KEY_DIR = "/path/to/generated/keys"
   IMA_EVM_ROOT_CA = "${MODSIGN_KEY_DIR}/ima-local-ca.pem"


Please note that this feature does not sign binary drivers that are not compiled during the build.
It is possible to sign these kind of drivers, but at the moment it has to be done manually before building the firmware image.

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
This feature allows defining the commands that are allowed to be executed during the autoboot process.
It is difficult to disable all the unnecessary commands, and sometimes certain commands have to be left in the bootloader for maintenance purposes.
The allowlisting feature allows executing all the commands if the CLI is opened, but prevents commands that are not in the allowlist during autoboot.

Allowlisting is disabled by default, as it is impossible to have a sane default value that would fit all the devices and use cases.
To enable the allowlist, add the following to your U-Boot configuration:

.. code-block::

   COMMAND_ALLOWLIST=y
   COMMAND_ALLOWLIST_CMDS="space separated list of allowed commands"

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

* ``SULKA_ENABLE_MODULE_SIGNING`` (0)

  Set this option to ``1`` to enable module signing.
  It is strongly recommended to enable this feature, but it is disabled by default as it breaks the build if the signing keys are not provided.
  See :ref:`Module Signing` for more information on enabling this feature.

* ``SULKA_ENABLE_MONITORING`` (0)

  Set this option to ``1`` to install ``packagegroup-sulka-monitoring``. This packagegroup contains utilities that can be used to monitor the system.
  See :ref:`Monitoring` for more information.


* ``SULKA_EXPIRE_PASSWORDS`` (0)

  Set the passwords to expire in the system.
  This is disabled by default, as this requirement does not usually translate well into embedded systems.
  However, if you perform user management on the Linux user level, it is recommended to enable this.

* ``SULKA_EXTRA_COMPLIANCY`` (0)

  Enable the extra compliancy settings that may be required by some audits and compliancy checks.
  These are options that may make more sense in workstation or server use, or that need some integration work, so they are disabled by default.
  This option enables the following options:

  * ``SULKA_ENABLE_MONITORING``
  * ``SULKA_EXPIRE_PASSWORDS``

  It is recommended to go through the options that are enabled by ``SULKA_EXTRA_COMPLIANCY``, and enable them manually if enabling the whole ``SULKA_EXTRA_COMPLIANCY`` feature is not possible.

* ``SULKA_HARDEN_KERNEL`` (1)

  Harden the kernel configuration. Requires `meta-sulka-kernel <https://codeberg.org/AltidSec/meta-sulka-kernel>`_ to be part of the build.

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

  The firewall configuration template that gets installed to the system and is used as the default firewall configuration.
  See :ref:`Firewall` for more information.

* ``SULKA_SSH_KEYS_DIR`` ("${TOPDIR}/../auth-keys")

  The directory where the public SSH key will be looked from if ``SULKA_INSTALL_SSH_KEYS`` is set to ``1``.

* ``SULKA_SSH_PORT`` (22)

  Allows configuring the SSH server to listen in a non-standard port.
  By default, the standard port is used.

* ``SULKA_FSTAB_EXTRA_LINES`` ("")

  This variable can be used to add extra lines into the fstab file.
  Sulka installs an fstab file that contains some hardening options for the mounts.
  You can either override that file with your own, or append the required lines using this variable.

* ``SULKA_SERVICEUSER_USERNAME`` ("serviceuser")

  The name of the service user that can be used to log in to the system.
  It is recommended to change this into something else.

* ``SULKA_SERVICEUSER_PASSWORD`` (no default value)

  The password that the service user uses to log in.
  This is not set by default, and if you do not set a password, the service user will not be added.
  See the instructions in the :ref:`quick-start` for the password creation and setting.

* ``SULKA_UBOOT_PASSWORD`` (no default value)

  The password that can be used to log in to the u-boot command line interface.
  By default, no password is set and the command line interface is inaccessible.
