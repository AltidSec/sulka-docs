Release Notes
#############

Releases that share a common name are identical from the Sulka feature point of view, so entries may cover multiple releases that are based on different versions of Yocto.
See :ref:`Releases & Versioning` for what the version numbers tell you.

2026.09 (1.1.0 / 0.7.0)
***********************

.. important::

   Two changes in this release need action when you update:

   * Kernel module signing is now enforced by default, and the build fails until you provide the signing keys.
     See :ref:`Module Signing`.
   * ``SULKA_HARDEN_FSTAB`` has been renamed to ``SULKA_HARDEN_MOUNTS``.
     If you set the old variable in your configuration, rename it.

   The next release, due by the end of 2026, is the last one to support Scarthgap.
   If you are on the 0.n.n line, now is a good time to move to Wrynose.

.. rubric:: Features

* The distro configuration and the bbappends have been refactored so that the meta-layers can be used as generic hardening layers, by including a single configuration file in your own distro configuration.
  See :ref:`Using the Hardening Without the Sulka Distro`.
* Kernel module signing is now enforced by default.
  The documentation walks through generating the keys, see :ref:`Building Sulka` and :ref:`Module Signing`.
* A new option, ``SULKA_DISABLE_KERNEL_MODULES``, builds a kernel without loadable module support.
  It is not enabled by default, except when ``SULKA_EXTRA_COMPLIANCY`` is enabled.
  See :ref:`Disabling Kernel Modules`.
* The ``meta-selinux`` dependency is now genuinely optional, and only required when SELinux is the selected mandatory access control module.
* ``/tmp`` is now mounted with ``noexec``, ``nodev`` and ``nosuid`` on systemd systems as well, as it already was on sysvinit systems.
  As the option now covers more than the ``fstab``, ``SULKA_HARDEN_FSTAB`` has been renamed to ``SULKA_HARDEN_MOUNTS``.
  A test checking the mount options has been added.
  See :ref:`Cannot Execute A Script From /tmp, /run Or /var` if this affects your software.
* ``auditd`` is now part of the core packages that are always installed.

.. rubric:: Bug Fixes

* Fixed build warnings about an incorrect ``S`` variable in two recipes.
  The warnings had no effect beyond the message itself.
  A test that catches build warnings has been added so they do not return unnoticed.

.. rubric:: Maintenance

* All the dependency layers, such as ``openembedded-core`` and ``meta-openembedded``, have been updated.

.. rubric:: Other

* The documentation and the READMEs have been revamped.
  Two new pages have been added: :ref:`Building With Sulka` and :ref:`Troubleshooting`.
