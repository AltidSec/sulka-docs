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

#. Add the password to `kas-sulka-configuration.yml`. Escape the four dollar signs in hash with ``\`` if not done already:

   .. code-block::

     SULKA_SERVICEUSER_PASSWORD = "<HASH_FROM_PREVIOUS COMMAND>"

#. (Optional) Change the default service user username ``serviceuser`` to something else by adding it to ``kas-sulka-configuration.yml``:

   .. code-block::

     SULKA_SERVICEUSER_USERNAME = "<USERNAME>"

#. (Optional) Enable the graphics support if your device requires it:

   .. code-block::

     SULKA_DISABLE_GRAPHICS = "0"

#. (Optional) If editing the files in ``meta-sulka-distro``, checkout the meta-layers first:
   
   .. code-block::

     kas checkout kas-sulka.yml

#. (Optional) Edit the firewall template in ``meta-sulka-distro/recipes-filter/nftables-configuration/files/nftables-drop-everything.conf``, or select one of the other templates with ``SULKA_NFTABLES_CONF`` configuration variable.

#. (Optional) Edit the sudo configuration for the service user in ``meta-sulka-distro/recipes-extended/sudo/files/serviceuser.conf`` to enable sudo.

#. Build the image:

   .. code-block::

     kas build kas-sulka.yml

#. Run the image, and login as the service user using the password defined earlier

