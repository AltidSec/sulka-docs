.. _quick-start:

Quick Start
###########

This page walks you through building Sulka for the ``qemux86-64`` reference machine and booting the result under QEMU.

Sulka requires some setup before the first build. Module signing keys have to be generated or the build fails, and a service user password has to be set or the resulting image has nothing that can log in.
Both of those are deliberate, and both are covered below.

Before You Start
****************

You will need a Linux host set up for Yocto builds. See the `Yocto system requirements <https://docs.yoctoproject.org/ref-manual/system-requirements.html>`_ for the supported distributions and the host packages to install.

You will also need ``mkpasswd`` for generating the password hash, which on Debian-based hosts is in the ``whois`` package.

Be aware that this is a full Yocto build. Expect it to need tens of gigabytes of disk space, and expect the first build to take hours, as nothing is cached yet.

Building Sulka
**************

#. Install `kas <https://github.com/siemens/kas>`_ by following the instructions in `the kas documentation <https://kas.readthedocs.io/en/latest/userguide/getting-started.html>`_.

#. Clone the ``kas-sulka`` repository to build Sulka with kas:

   .. code-block::

     git clone https://codeberg.org/AltidSec/kas-sulka.git
     cd kas-sulka

#. Generate a password hash for the service user.
   Dollar signs have to be escaped with ``\`` before the hash can be assigned to a variable in a Yocto-style build, so generate the hash and escape it in one go:

   .. code-block::

     mkpasswd -m yescrypt -s -R 8 <SECRET_PASSWORD> | sed 's/\$/\\$/g'

   When you later change the password on the running system, it is required to be at least 14 characters long and to include at least one character from at least three of the following four character classes: lowercase letters, uppercase letters, digits, and special characters.
   It is recommended that your initial password meets these requirements as well.

#. Add the resulting hash to ``kas-sulka-configuration.yml``:

   .. code-block::

     SULKA_SERVICEUSER_PASSWORD = "<HASH_FROM_PREVIOUS_COMMAND>"

#. Check out the meta-layers, as the module signing key generation script depends on ``meta-security``.
   Note that checking out the meta-layers may sometimes take a long while (up to a few minutes):

   .. code-block::

     kas checkout kas-sulka.yml

#. Generate the module signing keys. Module signing is required by default, so the build will fail if you do not provide the keys.

   .. code-block::

     ./scripts/generate_ima_evm_modsign_keys.sh

   Then point the build to the generated keys by adding the following to ``kas-sulka-configuration.yml``, replacing the path with the directory where the keys were generated:

   .. code-block::

     MODSIGN_KEY_DIR = "/path/to/generated/keys"
     IMA_EVM_ROOT_CA = "${MODSIGN_KEY_DIR}/ima-local-ca.pem"

   If you cannot use module signing, you can disable it instead by setting ``SULKA_ENABLE_MODULE_SIGNING = "0"`` in the configuration. However, this is not recommended. See :ref:`Module Signing` for more details.

#. (Optional) Change the default service user username ``serviceuser`` to something else by adding it to ``kas-sulka-configuration.yml``:

   .. code-block::

     SULKA_SERVICEUSER_USERNAME = "<USERNAME>"

#. (Optional) Enable the graphics support if your device requires it:

   .. code-block::

     SULKA_DISABLE_GRAPHICS = "0"

#. (Optional) Edit the firewall template in ``meta-sulka-distro/recipes-filter/nftables-configuration/files/nftables-drop-everything.conf``, or select one of the other templates with ``SULKA_NFTABLES_CONF`` configuration variable.

#. (Optional) Edit the sudo configuration for the service user in ``meta-sulka-distro/recipes-extended/sudo/files/serviceuser.conf`` to configure sudo, or set ``SULKA_SERVICEUSER_ENABLE_SUDO="1"`` in your build configuration.

#. Build the image:

   .. code-block::

     kas build kas-sulka.yml
     # or use kas-container for containerised builds
     kas-container build kas-sulka.yml

Running the Image
*****************

Boot the built image under QEMU with ``runqemu``, from inside the kas build environment:

.. code-block::

  kas shell kas-sulka.yml -c 'runqemu nographic slirp'

``nographic`` gives you a serial console, which is what you want because Sulka disables graphics by default.
``slirp`` enables user-mode networking.

Log in at the console prompt as ``serviceuser``, using the password you generated earlier.

The serial console is the only way in to a default image.
Sulka does not install an SSH server, and the firewall drops all traffic in every direction, so there is nothing listening and nothing that could reach it.
:ref:`The Development Configuration Fragment` covers getting network access while developing.

Next Steps
**********

* :ref:`Configuration Variables` lists everything that can be turned on and off in Sulka.
* :ref:`Firewall` covers configuring the firewall for your own use.
* :ref:`Building With Sulka` describes how to set up your own project on top of Sulka.
* :ref:`Firmware Update Example` demonstrates A/B firmware updates on the Raspberry Pi reference.
