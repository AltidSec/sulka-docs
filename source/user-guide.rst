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

Supported Configurations
************************

Sulka aims to provide support for the common alternatives related to Linux systems.

Init Managers
=============

Currently, Sulka primarily supports sysvinit as the init manager.
Using systemd as the init manager should be possible, and all the custom initialization scripts have their systemd service counterparts. However, systemd is not actively used and tested, so there may be some things missing.
More complete systemd support is planned for the future. Please raise an issue in the ``meta-sulka-distro`` repository if getting this is urgent to you.

Installed Packages
******************

Sulka installs packages as a part of the distro. You can find the packages listed here, along with the explanation of what they do and why they're installed.


* ``acct``

  acct is the GNU Accounting Utilities package that is used to perform process monitoring. This can be used to check what processes have been started in the system, and when the processes have been started.

* ``aide``

  AIDE stands for Advanced Intrusion Detection Environment, and it is used as a file integrity monitor in Sulka. With periodic checks, AIDE can detect changes in files and report these.

* ``audit``

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

* ``sudo``

  sudo is the package that is used to allow service user to perform actions with root capabilities.
  Since the root user is locked in Sulka, it is recommended to install sudo if there is a service user in the system

* ``sysstat``

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

Monitoring
**********

Sulka installs multiple packages that are used to monitor the system and can be used to detect anomalies.
These packages are ``acct``, ``aide``, ``auditd``, and ``sysstat``.
To get the most of these monitoring capabilities, your system should satisfy the following requirements:

* Persistent logging partition. Logging to volatile locations causes the logs to be lost in case of a power loss or reboot.
* Enough storage space. The monitoring tools can create large logs, so it is important to ensure there is enough storage for them.
* Remote logging. While this is not mandatory, it is useful for timely anomaly detection and log storage.

The monitoring can be useful without fulfilling these requirements, but the usefulness may be limited as the logs may be lost before analysis or cannot be analyzed remotely/automatically.

It is recommended that you run a long test with the system running the usual load to see how large the logs grow in your system.
After that, you can either configure or disable some of the monitoring functionality as required.

To remove all the monitoring packages from the image, you should remove ``packagegroup-sulka-monitoring`` from ``DISTRO_EXTRA_RDEPENDS`` like this:

.. code-block::

  DISTRO_EXTRA_RDEPENDS:remove = "packagegroup-sulka-monitoring"

This could be done for example in the ``local.conf``.

Configuration Variables
***********************

This chapter covers the configuration items in Sulka. The default value for each configuration is in the parentheses after the name

* ``SULKA_DISABLE_GRAPHICS`` (1)

  This option allows enabling or disabling the graphics to reduce kernel attack surface.
  If your device does not have a graphic output, you should be able to leave this to default.

* ``SULKA_EXPIRE_PASSWORDS`` (0)

  Set the passwords to expire in the system.
  This is disabled by default, as this requirement does not usually translate well into embedded systems.
  However, if you perform user management on the Linux user level, it is recommended to enable this.

* ``SULKA_EXTRA_COMPLIANCY`` (0)

  Enable the extra compliancy settings that are required by some audits and compliancy checks.
  These are options that may make more sense in workstation or server use, but may still be required in embedded side to pass certain checks.
  Currently, this option enables the following options:

  * ``SULKA_EXPIRE_PASSWORDS``

* ``SULKA_INSTALL_SSH_KEYS`` (0)

  Install SSH public key information into the firmware image.
  The key is searched from ``${SULKA_SSH_KEYS_DIR}`` directory, and is assumed to have a name in the form of ``${SULKA_SERVICEUSER_USERNAME}-auth-key.pub``.

  Note that installing the SSH key information to the firmware during build may pose a security risk.
  If the private key leaks, all the devices using the same firmware image become vulnerable.
  Consider generating unique SSH keys for each device if that is possible for your use case.

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
