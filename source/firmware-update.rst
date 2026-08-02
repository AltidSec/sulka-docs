.. _firmware-update:

Firmware Update Example
#######################

This page contains information about the firmware update references provided by Sulka.

Motivation
**********

While it is impossible to provide a generic, fully hardened one-size-fits-all solution for the firmware upgrades, Sulka provides an example firmware update mechanism for the reference hardware.
The purpose of it is to demonstrate firmware upgrades, and to demonstrate how to achieve them on the hardened Sulka system.
To adapt this into your needs, you should carefully consider the security features required from your firmware update flow.

The example is a demonstration rather than a production update system.
In particular, it signs bundles with a key on your build machine and transports them on a USB drive, both of which you would most likely do differently on a real product.

Rugix
*****

Rugix provides atomic A/B updates with automatic rollback, delta updates, cryptographic verification, and state management.

The example implementation of Rugix performs A/B updates on Raspberry Pi 4 reference, using U-Boot as the bootloader.

How to Run the FW Update Example
================================

This demo performs a simple firmware update using Rugix running on Sulka.
To follow along you are going to need the following:

#. Raspberry Pi 4
#. SD Card
#. USB stick
#. USB-TTL adapter
#. Computer to build the Sulka with

The actual steps to perform the update are as follows:

#. Clone the ``kas-sulka-raspberrypi-example`` repository:

   .. code-block::

     git clone https://codeberg.org/AltidSec/kas-sulka-raspberrypi-example.git
     cd kas-sulka-raspberrypi-example

#. Generate the firmware bundle signing keys:

   .. code-block::

     openssl ecparam -name prime256v1 -genkey -noout -out root.key
     openssl req -x509 -new -key root.key -out root.crt -days 3650 \
         -subj "/CN=Update CA" \
         -addext "basicConstraints=critical,CA:TRUE" \
         -addext "keyUsage=critical,keyCertSign,cRLSign" \
         -addext "subjectKeyIdentifier=hash"
     openssl ecparam -name prime256v1 -genkey -noout -out signer.key
     openssl req -new -key signer.key -out signer.csr -subj "/CN=Update Signer"
     openssl x509 -req -in signer.csr \
         -CA root.crt -CAkey root.key -CAcreateserial \
         -out signer.crt -days 365 \
         -extfile <(printf "basicConstraints=critical,CA:FALSE\nkeyUsage=critical,digitalSignature\nsubjectKeyIdentifier=hash\nauthorityKeyIdentifier=keyid:always")

#. Follow instructions from the :ref:`quick-start` to configure the build.
   You need to generate the module signing keys, as the build fails without them, and to set the login password with ``SULKA_SERVICEUSER_PASSWORD``.
   The update commands below are run through ``sudo``, so enable it with ``SULKA_SERVICEUSER_ENABLE_SUDO``.

   Note that these signing keys are for the kernel modules, and are unrelated to the update bundle signing keys generated in the previous step.

#. Configure the path to the root certificate that will be deployed to the image. Add the following for example to ``kas-sulka-configuration.yml``:

   .. code-block::

     SULKA_RUGIX_ROOT_CERT = "<PATH_TO_KEYS>/root.crt"

   This installs the root certificate to ``/etc/rugix/root.crt`` on the device, and Rugix is configured to trust that certificate.
   An update bundle is only accepted if its signature chains up to it, which is what stops the device from installing a bundle you did not sign.

#. Build the image:

   .. code-block::

     kas build kas-sulka.yml:kas-sulka-raspberrypi.yml:extra_fragments/fw-update-rugix.yml

#. Sign the resulting ``.rugixb`` image with keys you generated before.
   For this, you'll need ``rugix-bundler`` command. If you don't want to install it you can use the built binary as follows:

   .. code-block::

     <PATH_TO_TOPDIR>/tmp/sysroots-components/x86_64/rugix-bundler-native/usr/bin/rugix-bundler signatures sign \
         <PATH_TO_IMAGES>/core-image-base-raspberrypi4-64.rootfs.rugixb \
         <PATH_TO_KEYS>/signer.crt \
         <PATH_TO_KEYS>/signer.key \
         core-image-base-raspberrypi4-64.rootfs.signed.rugixb

   For production setups, it is recommended to use PKCS#11 for signing. For further details, see `Rugix's documentation <https://rugix.org/docs/ctrl/signed-updates/>`_.

#. Copy the signed update bundle ``core-image-base-raspberrypi4-64.rootfs.signed.rugixb`` to a USB thumb drive.
   In this demo we'll use USB to transport the update bundle to the device, but network transportation methods are also possible.
   For example, Rugix supports streaming the update from an HTTP server directly to its destination while verifying individual blocks before writing them.

#. Flash the built ``.wic`` image to an SD card. You can use the ``bmaptool`` to flash the SD card as follows:

   .. code-block::

     sudo bmaptool copy core-image-base-raspberrypi4-64.rootfs.wic.bz2 /dev/<SD_CARD_DEVICE>

#. Connect to the Raspberry Pi using UART.
   `This post <https://www.jeffgeerling.com/blog/2021/attaching-raspberry-pis-serial-console-uart-debugging/>`_ contains instructions how to connect the cable to a Raspberry Pi.
   Note that you do not have to modify the image contents to enable UART.

#. Insert SD card to Raspberry Pi and turn it on.
   Log in to the system using the password you defined with the instructions in the quick start.

#. If your device does not have network connection, you'll need to manually set up the system time so that the certificate is active.
   You can use the following command to set the time, adjust the echoed timestamp to your current time:

   .. code-block::

     # The format is YYYYMMDDHHmmss:
     sudo date -s "$(echo 20260225213500 | sed 's/\(....\)\(..\)\(..\)\(..\)\(..\)\(..\)/\1-\2-\3 \4:\5:\6Z/')"

#. Plug in the USB drive containing the update bundle, and mount it to the system:

   .. code-block::

     sudo mount /dev/sda1 /media

#. Install the update package:

   .. code-block::

     sudo rugix-ctrl update install /media/core-image-base-raspberrypi4-64.rootfs.signed.rugixb

#. Wait for the device to reboot after the bundle is installed.
   After boot, check the system status with the following command, and commit if you are happy with the results of the update:

   .. code-block::

     # Print system info
     # activeGroup should be b, default group a
     sudo rugix-ctrl system info
     # Commit the active group as default
     sudo rugix-ctrl system commit

   The commit step is what makes the update permanent.
   Until you commit, the newly booted group is active but the other one is still the default, so a reboot returns the device to the previous firmware.
   This is the safety net that A/B updates exist to provide: an update that boots into a broken system is recovered by a power cycle rather than by a visit to the device.

Limitations / Modifications to Sulka
====================================

There are a few modifications related to using Rugix:

* FAT U-Boot environment is enabled and additional U-Boot commands are allowed

  To switch between boot partitions, Rugix requires access to the U-Boot environment from the user space.
  This is common with A/B update schemes, but may be dangerous if the environment is not carefully protected, so ensure no unauthorized parties can modify it.

  Similarly, to allow modifying the U-Boot environment from the U-Boot itself, some commands are added to the U-Boot command allowlist.

* ``/run`` is not mounted through ``/etc/fstab`` anymore

  As Rugix's state management feature is used, Rugix will run early during the boot process, before the init manager.
  As a part of its initialization process, Rugix will take care of mounting ``/run``.
  To avoid re-mounting the tmpfs during the boot, ``/run`` is removed from ``fstab``.

* State management overlay of Rugix is disabled

  Rugix state management relies on mounting an overlayfs on top of the root file system.
  However, this is incompatible with SELinux, as the overlayfs loses the SELinux labeling when files are modified and copied up.
  If you want to use state management overlay, disable SELinux.
  The state management binds should still work with SELinux.
