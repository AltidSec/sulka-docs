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

There are multiple repositories related to Sulka. Here you can find the listing of them:

* `kas Sulka <https://codeberg.org/AltidSec/kas-sulka>`_

  This is the top-level repository of the project. It is the kas configuration repository.
  Kas is the build tool used to configure and build Sulka, and it is commonly used in Yocto projects.
  From this repository you can find the build configuration, and instructions on how to build the repository.

* `meta-sulka-distro <https://codeberg.org/AltidSec/meta-sulka-distro>`_

  This repository is the distro part of the Sulka.
  It defines the packages that get installed into the user space, and the hardening configurations for the packages.

* `meta-sulka-kernel <https://codeberg.org/AltidSec/meta-sulka-kernel>`_

  This repository is the kernel configuration part of the Sulka.
  It contains kernel metadata for creating a hardened kernel.

* `meta-sulka-bsp <https://codeberg.org/AltidSec/meta-sulka-bsp>`_

  This repository is the board support package part of the Sulka.
  It contains metadata for hardened bootloader, which in the reference implementation is U-Boot.

* `Sulka documentation <https://codeberg.org/AltidSec/sulka-docs>`_

  This is the documentation repository for Sulka.
  It is also the source for this very page you are reading!
  If you spot missing or incorrect documentation, please raise an issue in this repository.

* `Sulka tests <https://codeberg.org/AltidSec/sulka-tests>`_

  This is the test repository for Sulka.

Supported Yocto Versions
************************

The goal is to support the latest long-term support release of Yocto (currently Scarthgap).
