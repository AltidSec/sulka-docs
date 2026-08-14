About Sulka
###########

Sulka is a Yocto Linux distribution that aims to provide a secure base for embedded Linux systems.
The distro aims to be generic enough that it can fit most use cases, and provide enough configuration that it can be tailored for other systems.

Being a Yocto distribution, Sulka is something you build rather than something you download.
You assemble the image from source and tailor it to your product as you go, which means a first build takes hours and a fair amount of disk space.
This documentation assumes you are comfortable with Yocto and BitBake, and uses concepts like layers, recipes and ``local.conf`` without explaining them.
If Yocto is new to you, start with `the Yocto Project documentation <https://docs.yoctoproject.org/>`_ and come back once you have built an image with it.

The distro hardening is achieved by performing the following tasks by default:

* Minimizing the ``DISTRO_FEATURES`` of the image
* Installing a firewall that drops all traffic until it is configured
* Disabling root logins, and forcing all root actions to be done with ``sudo``
* Securing user logins with PAM
* Hardening the application configurations, like OpenSSH
* Enforcing mandatory access control with SELinux
* Hardening the kernel configuration and the runtime ``sysctl`` settings
* Enforcing kernel module signing to prevent unsigned code from being loaded into the kernel
* Mounting the root file system read-only
* Locking down the bootloader console on the reference hardware
* Collecting audit logs, and producing a software bill of materials for every image

Further hardening, such as the monitoring tooling and the security scanners, is available but left off by default.
See :ref:`Configuration Variables` for everything that can be turned on and off.

Why Sulka
*********

The default reference distribution of Yocto, Poky, is a general-purpose distribution that is suitable for getting started with Yocto.
However, as it is meant mostly to be a reference, it has to make some compromises on security.

Sulka does not have to make such compromises, and it can focus on security.
The goal is to make a distro that is as hardened as possible, and then let you make a conscious decision to lower the hardening if your product requires it.
Every hardening measure listed above is on by default and can be turned off, rather than being an opt-in extra that is easy to forget.

Sulka is aimed at the people building embedded products that have to hold up to scrutiny: devices that ship to customers, live on untrusted networks, and have to answer questions from auditors and security reviews.

Building on meta-security
=========================

Yocto already has `meta-security <https://git.yoctoproject.org/meta-security>`_, which collects security tooling for embedded Linux: scanners, intrusion detection, compliance tooling, mandatory access control and integrity measurement.
Sulka is not an alternative to it. Sulka is built on top of it, and pulls in ``meta-security`` and ``meta-integrity`` as dependencies.

The difference is in what you are handed.
``meta-security`` gives you building blocks and leaves the decisions to you: which tools to install, how to configure them, how the pieces interact, and what the resulting system's defaults should be, all of which you work out yourself and keep coherent as the layers move underneath you.

Sulka makes those decisions and ships them as distro policy.
The firewall is installed and loading a rule set, SELinux is enabled with a policy that has been patched until the system boots cleanly under it, the kernel configuration is hardened, module signing is enforced, and the pieces are tested together at each release.
If you want different decisions, you change the configuration or drop a layer, but you start from a coherent whole rather than an empty ``local.conf``.

What to Expect
==============

Sulka deliberately requires some setup before it is usable.
The firewall drops all traffic in every direction, including outgoing.
Root login is disabled, so nothing can log in until you configure a user.
Module signing is enforced, so the build fails until you provide the signing keys.

This is the point rather than an oversight.
An out-of-the-box Poky image boots and works; the hardening it lacks is invisible until something goes wrong in the field.
A Sulka image makes you decide what to open up, and each thing you open is a decision you made on purpose.
The cost is some work before the first boot, and in return the parts you never got around to are closed rather than open.

If you want the friction lowered while developing, the ``development.yml`` configuration fragment and other development settings relax the defaults.
These are not meant for production images.

What Sulka Does Not Do
======================

Sulka hardens the software it ships. There are neighbouring problems that it leaves to you, and you should plan for them separately:

* **Verified boot.** Sulka locks down the U-Boot console, but it does not set up a verified boot chain. Nothing in the default build verifies the kernel or the root file system image before booting it.
* **Offline tampering.** A read-only root file system stops a running system from modifying itself, but it does not stop anyone from modifying the storage directly. If you need that, look into dm-verity, which Sulka does not provide.
* **Runtime file integrity.** IMA and EVM are not enabled, as they conflict with SELinux, which Sulka does enable by default. File integrity measurement and appraisal are therefore not part of the system today.
* **Storage encryption.** No disk or file system encryption is configured.
* **Hardware roots of trust.** TPM and secure element integration are out of scope. ``meta-security`` ships ``meta-tpm`` and ``meta-parsec`` layers that Sulka does not include, and you can add them to your own build.
* **Your application.** Sulka secures the platform underneath your software. What you put on top of it, and the services you open up in order to run it, stay your responsibility.

None of these are ruled out by the design, and some may be added later.
They are simply not part of what Sulka gives you today, so do not assume them.

Auditing and Compliance
=======================

The hardening is not invented in-house.
The kernel configuration and the runtime ``sysctl`` settings follow the suggestions from `kernel-hardening-checker <https://github.com/a13xp0p0v/kernel-hardening-checker>`_, and the userspace hardening applies the findings of `Lynis <https://cisofy.com/lynis/>`_ and `OpenSCAP <https://www.open-scap.org/>`_.
Those tools can be installed into an image with the ``audit.yml`` configuration fragment so that you can run the same scans yourself, and the Sulka test suite runs the Lynis and OpenSCAP scans against every release.
Note that there are still some unaddressed hardening points reported by these tools.

Sulka does not certify your product against any particular standard, and using it is not a substitute for your own security work.
What it aims to do is start you from a defensible baseline.

Licensing
=========

The Sulka metadata is licensed under the MIT license, so you can fork it, adapt it, and ship products built on it.

Sulka also avoids GPLv3-licensed components, as those licenses are often a problem for shipped devices.
Notably, ``uutils-coreutils`` replaces GNU ``coreutils``, and ``readline`` is removed from the packages that would otherwise pull it in.
Note that Sulka does not set ``INCOMPATIBLE_LICENSE`` for you.
If your product must be free of a particular license, verify it against the bill of materials of your own image.

Maturity and Tested Scope
*************************

Sulka is under active development, and the interfaces and defaults still change between releases.
Go through the changes when updating the version of Sulka, and pin to a release tag rather than tracking a branch.

Every release is exercised by an automated test suite covering the build itself, the Lynis and OpenSCAP audit scans, SELinux enforcement, the running processes, the logging, the kernel configuration, the file system, and the alternative build configurations.

The actively tested scope is:

* ``qemux86-64`` as the default reference machine with the full testing, and the Raspberry Pi 4 64-bit as the hardware reference with a build test
* ``x86_64`` and ``arm64`` architectures
* ``systemd`` as the init manager, with ``sysvinit`` tested as the secondary option

Other machines and architectures are expected to work, since nothing in the distro is specific to the reference targets, but you should expect to do some porting and testing work yourself.
See :ref:`Supported Configurations` for the details.

Repositories
************

There are multiple repositories related to the Sulka project. This section contains a list of them, with a link to each repository.
In addition to the Codeberg repositories, you can find `GitHub mirrors of each repository here <https://github.com/AltidSec/>`_.

* `kas Sulka <https://codeberg.org/AltidSec/kas-sulka>`_

  This is the top-level repository of the project. It is the kas configuration repository.
  kas is the build tool used to configure and build Sulka, and it is commonly used in Yocto projects.
  From this repository you can find the build configuration, the optional configuration fragments, and instructions on how to build Sulka.

* `kas Sulka Raspberry Pi example <https://codeberg.org/AltidSec/kas-sulka-raspberrypi-example>`_

  Example repository of how the Sulka project can be ported on a custom hardware.
  The example ports the Sulka distro to Raspberry Pi 4 64-bit, but the instructions in the repo can be applied to other hardware as well.
  It also carries the Rugix firmware update example described in :ref:`Firmware Update Example`.

* `meta-sulka-distro <https://codeberg.org/AltidSec/meta-sulka-distro>`_

  This repository is the distro part of the Sulka.
  It defines the distro itself, the packages that get installed into the user space, and the hardening configurations for those packages.

* `meta-sulka-kernel <https://codeberg.org/AltidSec/meta-sulka-kernel>`_

  This repository is the kernel configuration part of the Sulka.
  It contains kernel metadata for creating a hardened kernel, along with the matching runtime ``sysctl`` settings.

* `meta-sulka-bsp <https://codeberg.org/AltidSec/meta-sulka-bsp>`_

  This repository is the board support package part of the Sulka.
  It contains metadata for hardened bootloader, which in the reference implementation is U-Boot.

* `meta-sulka-raspberrypi <https://codeberg.org/AltidSec/meta-sulka-raspberrypi>`_

  This is the integration layer that performs some modifications and additions to the Sulka that are required to port the distro to the Raspberry Pi.
  These actions mostly consist of editing the bootloader and kernel metadata, as they are quite board specific.

* `Sulka documentation <https://codeberg.org/AltidSec/sulka-docs>`_

  This is the documentation repository for Sulka.
  It is also the source for this very page you are reading!
  If you spot missing or incorrect documentation, please raise an issue in this repository.

* `Sulka tests <https://codeberg.org/AltidSec/sulka-tests>`_

  This is the test repository for Sulka. The tests are written for the Robot Framework.

* `Sulka release scripts <https://codeberg.org/AltidSec/sulka-release-scripts>`_

  The scripts and the checklist used to cut a Sulka release across all of the repositories above.

Supported Yocto Versions
************************

The primary goal of Sulka is to support the latest long-term support release of Yocto (currently Wrynose).

In addition, limited support is currently given to the previous LTS release, Scarthgap.
The support is limited to the ``systemd`` init manager.
The Raspberry Pi 4 hardware reference has worked in the past but is no longer supported: it may or may not work, and it is not actively tested.
The same applies to ``sysvinit``.
The last release supporting Scarthgap will be the 2026.12 release (see :ref:`Releases & Versioning`), so it is recommended to start updating to Wrynose as soon as possible.

Releases & Versioning
*********************

A new version of Sulka is released roughly every four months, towards the end of March, June, September and December.

The versioning scheme goes roughly as follows:

* Major version indicates the version of Yocto that Sulka is based on (0.n.n is for Scarthgap, 1.n.n for Wrynose).
  Updating to a new major version will certainly require fixing things.

* Minor version increases when there are significant configuration changes to Sulka, or other known breaking changes from Yocto.
  Updating to this version will require your attention.
  Reviewing your configuration and testing the functionality is required, but the release does not necessarily break things if the default settings work for you.

* Patch version increases after every release if there are no changes that are considered major or minor.
  Note that minor Yocto updates fall into this category. Updating the Yocto minor releases is usually straightforward, but may cause issues in some situations.
  In general, updating to this version should be safe, but there may still be unexpected complications, so plan accordingly.

The following table shows the Sulka releases that are based on different versions of Yocto.
Starting from 1.0.0 / 0.6.0, the matching releases share a common name of the form year.month, named after the month they were released in: 1.0.0 and 0.6.0 are together the 2026.06 release.
Earlier releases have no common name.
The releases sharing the same common release name should be identical from the Sulka feature point of view, so for example documentation for the version 1.0.0 should apply to 0.6.0.

=========== =============== =================
Common name Wrynose (1.n.n) Scarthgap (0.n.n)
=========== =============== =================
2026.06     1.0.0           0.6.0
\           \               0.5.1
\           \               0.5.0
\           \               0.4.0
\           \               0.3.0
\           \               0.2.0
=========== =============== =================

Roadmap & Following Development
*******************************

You can find the planned features from `the Trello board of the project <https://trello.com/b/4hZ5xmkg/sulka>`_.
Note that the planned content may be moved to a future or the next release if it cannot be completed in time for the release it was planned for.

To stay up-to-date on Sulka development, you can follow `the main developer's Mastodon account <https://infosec.exchange/@ejaaskel>`_.

Getting Help
************

Questions, bug reports and feature ideas all belong in the issue tracker of the repository they concern.
The :ref:`Repositories` section above lists them, and issues are as welcome for starting a discussion as they are for reporting a defect.

Patches and pull requests are welcome, and should target the ``*-next`` branch of the repository you are contributing to.
See :ref:`Staying Up To Date` for how to follow those branches from your own build.
