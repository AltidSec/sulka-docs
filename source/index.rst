Sulka, Secure Yocto Distribution
################################

Hello, and welcome. You have reached the documentation page for Sulka, a Yocto Linux distribution that focuses on security hardening.

Sulka ships hardened defaults across the kernel, the bootloader and the userspace, and expects you to consciously relax the hardening where your product requires it, rather than the other way round.

Where to Start
**************

* :ref:`About Sulka` explains what Sulka is, why it exists, and what it deliberately does not do. Start here if you are deciding whether it fits your project.
* :ref:`quick-start` takes you from an empty directory to a running image under QEMU.
* :ref:`Building With Sulka` covers setting up your own project on top of Sulka, working on it day to day, and getting back to a production image before you ship.
* :ref:`User Guide` describes what the distro does, and everything that can be configured.
* :ref:`Troubleshooting` is where to look when something that works on another distribution does not work here.
* :ref:`Firmware Update Example` demonstrates A/B firmware updates on the reference hardware.

Both the documentation and the distro itself are under active development, so expect things to change between releases.
See :ref:`Maturity and Tested Scope` for what is currently tested, and :ref:`Releases & Versioning` for what a version number tells you.

See :ref:`Getting Help` for where to ask questions, report bugs and send patches, and :ref:`Repositories` for where the code lives.

Contents
********

.. toctree::
   :maxdepth: 2

   about
   quick-start
   building-with-sulka
   user-guide
   troubleshooting
   firmware-update
