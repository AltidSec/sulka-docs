.. _quick-start:

Quick Start
###########

#. Follow instructions in `the kas-sulka repository <https://codeberg.org/AltidSec/kas-sulka/>`_ to install and activate `kas <https://github.com/siemens/kas>`_.

#. Clone the ``kas-sulka`` repository to build Sulka with kas:

   .. code-block::

     git clone https://codeberg.org/AltidSec/kas-sulka.git
     cd kas-sulka

#. Generate a password for the service user that can be used to log in.

   .. code-block::

     mkpasswd -m yescrypt -s -R 8 <SECRET_PASSWORD>

   When you update your password, the system requires that it be at least 14 characters long and include at least one character from at least three of the following four character classes: lowercase letters, uppercase letters, digits, and special characters. It is recommended that your initial password meets these requirements.

   For assigning the resulting encrypted password to a variable in a Yocto-style build, dollar signs have to be escaped with ``\``. This can be combined with the password creation process:

   .. code-block::

     mkpasswd -m yescrypt -s -R 8 test | sed 's/\$/\\$/g'

   This hashes the password "test" and prepares the resulting hash for pasting into a Yocto configuration file.

#. Add the password to ``kas-sulka-configuration.yml``. Escape the four dollar signs in hash with ``\`` if not done already:

   .. code-block::

     SULKA_SERVICEUSER_PASSWORD = "<HASH_FROM_PREVIOUS COMMAND>"

#. Checkout the meta-layers as module signing key generation script depends on meta-security:

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

#. Run the image, and login as the service user using the password defined earlier

