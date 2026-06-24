About Sulka
###########

Sulka is a Yocto Linux distribution that aims to provide a secure base for embedded Linux systems.
The distro aims to be generic enough that it can fit most use cases, and provide enough configuration that it can be tailored for other systems.

The distro hardening is achieved by performing the following tasks by default:

* Installing firewall and monitoring packages
* Disabling root logins, and forcing all root actions to be done with ``sudo``
* Securing user logins with PAM
* Hardening the application configurations, like OpenSSH
* Minimizing the ``DISTRO_FEATURES`` of the image

Repositories
************

There are multiple repositories related to the Sulka project. This section contains a list of them, with a link to each repository.
In addition to the Codeberg repositories, you can find `GitHub mirrors of each repository here <https://github.com/AltidSec/>`_.

* `kas Sulka <https://codeberg.org/AltidSec/kas-sulka>`_

  This is the top-level repository of the project. It is the kas configuration repository.
  Kas is the build tool used to configure and build Sulka, and it is commonly used in Yocto projects.
  From this repository you can find the build configuration, and instructions on how to build the repository.

* `kas Sulka Raspberry Pi example <https://codeberg.org/AltidSec/kas-sulka-raspberrypi-example.git>`_

  Example repository of how the Sulka project can be ported on a custom hardware.
  The example ports the Sulka distro to Raspberry Pi 4 64-bit, but the instructions in the repo can be applied to other hardware as well.

* `meta-sulka-distro <https://codeberg.org/AltidSec/meta-sulka-distro>`_

  This repository is the distro part of the Sulka.
  It defines the packages that get installed into the user space, and the hardening configurations for the packages.

* `meta-sulka-kernel <https://codeberg.org/AltidSec/meta-sulka-kernel>`_

  This repository is the kernel configuration part of the Sulka.
  It contains kernel metadata for creating a hardened kernel.

* `meta-sulka-bsp <https://codeberg.org/AltidSec/meta-sulka-bsp>`_

  This repository is the board support package part of the Sulka.
  It contains metadata for hardened bootloader, which in the reference implementation is U-Boot.

* `meta-sulka-raspberrypi <https://codeberg.org/AltidSec/meta-sulka-raspberrypi.git>`_

  This is the integration layer that performs some modifications and additions to the Sulka that are required to port the distro to the Raspberry Pi.
  These actions mostly consist of editing the bootloader and kernel metadata, as they are quite board specific.

* `Sulka documentation <https://codeberg.org/AltidSec/sulka-docs>`_

  This is the documentation repository for Sulka.
  It is also the source for this very page you are reading!
  If you spot missing or incorrect documentation, please raise an issue in this repository.

* `Sulka tests <https://codeberg.org/AltidSec/sulka-tests>`_

  This is the test repository for Sulka.

Supported Yocto Versions
************************

The primary goal of Sulka is to support the latest long-term support release of Yocto (currently Wrynose).

In addition, limited support is currently given to the previous LTS release, Scarthgap.
The support is limited to the ``systemd`` init manager.
The Raspberry Pi 4 hardware reference has worked in the past but is no longer supported: it may or may not work, and it is not actively tested.
The same applies to ``sysvinit``.
Note that this support may end after any release without warning, so it is recommended to update to Wrynose as soon as possible.

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
The releases sharing the same row should be identical from the Sulka feature point of view, so for example documentation for the version 1.0.0 should apply to 0.6.0.

=============== =================
Wrynose (1.n.n) Scarthgap (0.n.n)
=============== =================
1.0.0           0.6.0
\               0.5.1
\               0.5.0
\               0.4.0
\               0.3.0
\               0.2.0
=============== =================

Roadmap & Following Development
*******************************

You can find the planned features from `the Trello board of the project <https://trello.com/b/4hZ5xmkg/sulka>`_.
Note that the planned content may be moved to a future or the next release if it cannot be completed in time for the release it was planned for.

To stay up-to-date on Sulka development, you can follow `the main developer's Mastodon account <https://infosec.exchange/@ejaaskel>`_.
