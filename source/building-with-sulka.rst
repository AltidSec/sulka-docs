Building With Sulka
###################

If you are interested in using Sulka in your project, the usage is quite similar to the official reference distribution Poky.
In practice, this means building your solution on top of the Sulka meta-layers, and adding your own meta-layers alongside them.

Throughout this page, `the kas Raspberry Pi example repository <https://codeberg.org/AltidSec/kas-sulka-raspberrypi-example>`_ is used as the worked example.
It is a real port of Sulka onto real hardware, and it is built exactly the way this page describes.

This page covers setting up a project on top of Sulka, the tooling Sulka provides for development work, and how to get back to a production-grade image before you ship.

If something that works elsewhere does not work on Sulka, see :ref:`Troubleshooting` for the usual causes.

Setting Up Your Own Project
***************************

To start building your own system on top of Sulka, you will most likely want to create a kas configuration repository.
The easiest way to do this is to fork `the kas Sulka repository <https://codeberg.org/AltidSec/kas-sulka>`_, and add your own configuration files to the fork next to the Sulka configuration files.
This approach allows you to easily work with the Sulka distro, configure it as required, and merge your work on top of updates.
If you do not want to create a fork, you can manually set up the project structure yourself with a tool of your choosing.

Forking gives you the following files, which come from upstream Sulka:

``kas-sulka.yml``
  The top-level configuration that selects the machine, the distro and the target image, and includes the two files below.

``kas-layers.yml``
  The layer repositories and the revisions they are pinned to.

``kas-sulka-configuration.yml``
  The build configuration that gets written into ``local.conf``. This is the file the quick start has you edit, and it is safe to keep using for your own settings, though a separate file is usually tidier.

``extra_fragments/``
  Optional configurations that can be appended to a build command.

``scripts/``
  Helper scripts for generating the module signing keys and an SSH key pair.

What you add on top are usually configuration files of your own, which select your machine, pull in your layers, and set the configuration your product needs.
The Raspberry Pi example adds exactly one such file, ``kas-sulka-raspberrypi.yml``, and builds with it appended to the Sulka configuration:

.. code-block::

  kas build kas-sulka.yml:my-product.yml

Where Your Configuration Belongs
********************************

Putting your settings in ``kas-sulka-configuration.yml`` works, and nothing will break if you do.
It is still recommended to keep them in a configuration file of your own, so that the responsibilities stay separated: the Sulka files stay as they came from upstream, and everything specific to your product lives in one place that you control.
That separation is also what keeps merging upstream updates uneventful.

kas writes each ``local_conf_header`` block into ``local.conf`` in the order of the block keys, so the numeric prefix on the key decides who wins when two blocks set the same variable.
Sulka uses ``10-kas-sulka`` for its own base configuration, and the bundled fragments use ``30-`` and ``50-`` prefixes.
Use a prefix above those in your own configuration so that your settings are applied last:

.. code-block::

  header:
    version: 1
  machine: my-machine
  local_conf_header:
    60-my-product: |
      SULKA_SERVICEUSER_USERNAME = "operator"
      SULKA_DISABLE_GRAPHICS = "0"
  repos:
    meta-my-product:
      url: https://example.com/meta-my-product.git
      branch: main

The quick start suggests editing files inside the Sulka layers directly, for example the firewall template or the sudo configuration.
That is fine while you are trying Sulka out, but for a real project prefer overriding those from your own layer, so that you are not carrying local modifications to Sulka files across every update.

Adding Your Own Software
************************

Your application and any supporting recipes belong in a meta-layer of your own, added through the ``repos`` section of your configuration file as shown above.

A few things are worth keeping in mind while writing recipes for a Sulka system, as they are the usual reasons that software which works elsewhere does not work here.

SELinux is enabled by default with the ``targeted`` reference policy, so your own services can hit denials that never occur on an unhardened system.
When something fails for no apparent reason, the audit log (``/var/log/audit/audit.log``) is the first place to look, and services that need permissions the existing policy does not grant will need a policy module of their own.

The root file system is read-only by default, so anything that expects to write outside the designated writable locations will fail.

Kernel module signing is enforced, so modules built through their own Yocto recipes are signed for you, but prebuilt binary modules are not.

See :ref:`SELinux`, :ref:`Read-Only Root File System` and :ref:`Signing External Modules`.

Integration Layers
******************

Sometimes the Sulka metadata does not fit your hardware as-is, most often around the kernel and the bootloader, which are the most board specific parts of a build.
When that happens, the pattern is to add a small integration layer that reconciles the two, rather than to modify the Sulka layers.

`meta-sulka-raspberrypi <https://codeberg.org/AltidSec/meta-sulka-raspberrypi>`_ is the reference for this.
It applies the Sulka kernel hardening to the board's own kernel recipe, restores the security modules that the board's defconfig strips out, disables the options for hardware the board does not have, and supplies the bootloader configuration the board needs in order to still boot under the Sulka U-Boot hardening.

An integration layer has to win over the Sulka layers where they disagree, so give it a higher ``BBFILE_PRIORITY``.
The Sulka layers use priorities ``10`` and ``11``, and ``meta-sulka-raspberrypi`` uses ``15``.

Staying Up To Date
******************

Because your project is a fork, updating means merging the upstream kas Sulka repository into yours.
The Raspberry Pi example is maintained this way, and its history contains the merge commits from each upstream release.

A new Sulka release comes out roughly every three months.
Pin your build to a release tag rather than following a branch, so that updates happen when you choose.
Review the changes before merging, as version increases signal configuration changes that need your attention.

If you want to develop against the upcoming release, the ``kas-layers-development.yml`` fragment switches the Sulka layers from their pinned tags to the ``*-next`` development branches.
Those branches are also the targets for pull requests, if you intend to contribute changes back.

Using the Layers Without Forking kas-sulka
******************************************

Forking is the smoothest path, but it is not the only one.
The Sulka layers can be added to an existing build, provided that the recipe versions in your build line up with the ones the layers expect.
Many of the bbappends currently name an exact upstream version, so in practice this means matching the Yocto release that Sulka targets.
If a version does not line up, the build fails with a dangling bbappend rather than quietly dropping the hardening, so you will find out immediately.

Loosening this is planned. The intent is for the bbappends to follow the major version of each recipe rather than an exact one, which will make the layers considerably easier to use outside a Sulka build.

Using the Hardening Without the Sulka Distro
********************************************

Adding the layers does not oblige you to build with the ``sulka`` distro.
The hardening lives in ``meta-sulka-distro/conf/distro/include/``, and none of those files set ``DISTRO`` or any other distro identity variable, so your own distro configuration can require the whole of it in one line, exactly the way ``sulka.conf`` does:

.. code-block::

  require conf/distro/include/sulka-hardening.inc

The recipe metadata in ``meta-sulka-distro`` and ``meta-sulka-kernel`` keys off a ``sulka-hardening`` override that this file sets, rather than off the distro name, which is what makes the hardening work under a different ``DISTRO``.
Every feature it pulls in is still guarded by its own ``SULKA_`` variable, so the hardening can be relaxed from your build configuration as usual. See :ref:`Configuration Variables`.

The file is a thin wrapper over four topic includes, ``sulka-image.inc``, ``sulka-userspace.inc``, ``sulka-kernel.inc`` and ``sulka-gplv3.inc``, which can also be required individually if you only want part of the hardening.
Note that the ``sulka-hardening`` override is set by the wrapper alone, so requiring a topic include on its own leaves the recipe level hardening inactive unless you set the override yourself.

The Development Configuration Fragment
**************************************

The ``development.yml`` configuration fragment is the intended way to make a Sulka image workable during development.
Append it to your build command:

.. code-block::

  kas build kas-sulka.yml:extra_fragments/development.yml

It changes four things:

* Installs the OpenSSH server, which a default Sulka image does not have. It accepts key-based authentication only, so you will also need to install a key as described below.
* Switches the firewall to a template that allows SSH, ICMP, established connections and loopback traffic, so that the SSH server can actually be reached.
* Enables the Sulka development mode, described below.
* Installs the fuller SELinux tooling, including ``audit2allow`` and the ``setools`` utilities, neither of which is present in a default image.

This fragment lowers several security measures at once.
Do not ship a production image built with it.
For a production system that needs remote access, install and configure an SSH server deliberately, and open only what that access requires in the firewall.

Installing SSH Keys
*******************

Sulka configures the SSH server to accept key-based authentication only.
Password authentication is switched off and root login is refused, so the service user password gets you in on the serial console but not over the network.

This means an image that has an SSH server but no installed key has no way in over the network at all.
It is a common surprise the first time you build with the development fragment, so install a key at the same time.

Generate a key pair with the helper script:

.. code-block::

  ./scripts/generate_ssh_keys.sh

Then enable the installation in your build configuration:

.. code-block::

  SULKA_INSTALL_SSH_KEYS = "1"

The script writes the private key and the matching public key into an ``auth-keys`` directory, and the build picks the public key up from there.
This has no effect on an image that has no SSH server, so pair it with the development above or with an SSH server of your own.

Bear in mind that a key installed at build time is the same key on every device built from that image, so a leaked private key affects all of them.
See ``SULKA_INSTALL_SSH_KEYS`` in :ref:`Configuration Variables` for the details.

Development Mode
****************

``SULKA_DEVELOPMENT_MODE`` is enabled by the fragment above, and can also be set on its own.
On its own it is narrower than the name suggests: it does not disable hardening across the board.
It adds two kernel options, which together let you take SELinux out of the picture while diagnosing a problem:

* ``CONFIG_SECURITY_SELINUX_BOOTPARAM`` allows disabling SELinux from the kernel command line with ``selinux=0``.
* ``CONFIG_SECURITY_SELINUX_DEVELOP`` allows switching between enforcing and permissive at runtime with ``setenforce``.

This matters more than it first appears.
Neither option is set in a default build, so on a production image ``setenforce 0`` is refused and the boot parameters have no effect.
If you expect to debug SELinux behaviour on a device, you need an image built with development mode.

Development mode additionally relaxes the SELinux policy so that passwords can be changed over an SSH session.

The build prints a warning whenever development mode is enabled, so that it is hard to ship by accident.

Iterating On A Build
********************

``kas shell`` drops you into the build environment, where BitBake and the usual Yocto tooling are available:

.. code-block::

  kas shell kas-sulka.yml

From there you can build individual recipes, inspect the resolved configuration, and use ``devtool`` to work on a component without editing the layers themselves:

.. code-block::

  bitbake my-recipe
  bitbake -e my-recipe | grep ^SULKA_
  devtool modify my-recipe

To run the result, see the :ref:`quick-start`.

Tooling You Will Not Have
*************************

Sulka removes a considerable amount of debugging and introspection machinery from the kernel, and restricts more of it at runtime.
Debuggers cannot attach to processes, core dumps are disabled, the tracing infrastructure is compiled out, and profiling is restricted.

This is deliberate, but it does mean that some of the tools you would normally reach for are simply not available on target.
:ref:`Debuggers And Profilers Do Not Work` explains which ones and why, and where to change it if you need them during development.

If you create configuration fragments the help and ease the development work, those would be greatly appreciated in the upstream.

Moving Back To Production
*************************

Before shipping, rebuild without the development configuration and confirm the following:

* The build command no longer includes ``development.yml``, and ``SULKA_DEVELOPMENT_MODE`` is ``0``. The build warning should be gone.
* The firewall template is the one you intend to ship, rather than the permissive development one. See :ref:`Firewall`.
* The SSH server is either removed, or deliberately kept and configured for production use with key-based authentication.
* Any keys, passwords or authorized keys used during development have been replaced with production ones.
* SELinux is enforcing, and the policy modules you added during development are included in the image.

It is worth building the production image early and often rather than only at the end, so that hardening problems surface while there is still time to solve them.
