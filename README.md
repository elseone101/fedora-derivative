# fedora-derivative (Name TBD)
This is a derivative of fedora, with many simple QOL modifications like extra packages, removal of unneeded packages, change in a bit of default config etc.

It uses [bootc](https://bootc-dev.github.io/bootc), a tool which manages "atomic" and "immutable" systems using container images, bult in similar ways.
Atomic distros have many advantages, like the important parts being read-only, being able to rollback to the previous version if things break, seamlessly switch the "base" images etc...

(References and extra info are documented at the end of this README)

Basically an attempt to document how everyone can easily do this, as well as a template which anyone can fork and modify for their personal use (OR even prepare a derivative distribution with many modifications).

See [sigulete/kde-bootc](https://github.com/sigulete/kde-bootc) for a similar attempt.

## Building and publishing the image
`.github/` contains a simple action file which automatically builds the container image, and publishes it for potential users to pull from and use.
It is self-explanatory with many comments.
Just go to the `Actions` tab in github and enable it as needed.

## Containerfile
It is the ultimate source of truth.
It pulls an upstream fedora image, copies files from the repository root, and runs commands to modify the image as needed.

The rest of this README virtually explains how the Containerfile works

## The "base image"
We pull a base image of fedora, which we modify as per our needs.
It already contains a basic system with `systemd`, `NetworkManager`, `podman`, `dnf` etc...
AND the maintainer of the final modified image can optionally be written.
```
FROM quay.io/fedora/fedora-kinoite:42
MAINTAINER First Last
```
(`fedora-kinoite` has KDE included, for a DE-less one use `fedora-bootc` instead. Kindly open <quay.io/fedora> in your browser to see all of their images)

## Making /opt atomic
The default is to keep `/opt` a symlink on `/var/opt`, in order to allow free installation of software in there.
But `/opt` needs to be read-only just like `/usr`, logs etc.. symlinked to equivalents in `/var/opt`, config files symlinked to equivalents in `/etc/opt`
```
# /opt needs to be atomic
RUN <<EOOPT
mkdir /usr/opt
[ -d /opt ] && mv /opt/* /usr/opt
ln -sf usr/opt /opt
EOOPT
```
Linking `/opt` to `/usr/opt` allows it to be seamlessly handled as part of `/usr` by bootc.
It can be modified by `systemd-sysext` just like `/usr` (even when symlinked and `/usr` is already mounted-or-not).

## DNF extra plugins
The dnf plugins are useful. We install them (even though dnf is mostly useless outside the Containerfile)
```
# DNF plugins
RUN dnf -y install dnf5-plugins
```

<!-- Not now... This is wrong and incomplete
## Enabling kernel-install integration
In order to be able to modify the kernel setup process, for eg. to use UKIs or custom secureboot keys
Ref: https://docs.fedoraproject.org/en-US/bootc/building-containers/#_kernel_management
```
# kernel-install
RUN dnf -y downgrade kernel
```
-->

## fedora-sysexts
[fedora-sysexts](https://extensions.fcos.fr/) are a collections of sysexts, which are meant for fedora.

[sysexts-manager](https://github.com/travier/sysexts-manager) is a simple tool which helps you to install, enable/disable, remove, update and manage sysexts.
It is installed as a sysext itself, but it "starts" other sysexts (installed through it) dynamically.

**It is the only tool you use in order to manage sysexts**

```
RUN <<EOSYSEXT
# Install the sysext which manages all other sysexts
VERSION="0.3.0" # sysexts-manager version
VERSION_ID="42" # Fedora release
ARCH="x86-64"   # or arm64
URL="https://github.com/travier/sysexts-manager/releases/download/sysexts-manager/"
NAME="sysexts-manager-${VERSION}-${VERSION_ID}-${ARCH}.raw"
install -d -m 0755 -o 0 -g 0 "/var/lib/extensions"{,.d} "/run/extensions"
curl --silent --fail --location "${URL}/${NAME}" | tee /var/lib/extensions.d/${NAME}
ln -snf "/var/lib/extensions.d/${NAME}" "/var/lib/extensions/sysexts-manager.raw"
restorecon -RFv "/var/lib/extensions"{,.d} "/run/extensions"
systemctl enable systemd-sysext.service sysext-manager.service
EOSYSEXT
```

"sysexts" are simple bundles of non-essential software which can be installed on top of a system.
They are essentially disk images with the sofware, overlayed on `/usr` and `/opt`.
Since they are mounted/overlayed relatively late during boot, they can't be used for kernel, udev rules etc.
The fedora-sysexts collection contains software like libvirt, fd-find, etc.

## Incompleteness is a part of development; Kindly wait until this is complete

# Extra info references
