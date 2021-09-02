---
title: Building custom kernel modules
weight: 10
aliases:
    - ../../os/kernel-modules
---

## Create a writable overlay

The kernel modules directory `/usr/lib64/modules` is read-only on Flatcar Container Linux. A writable overlay can be mounted over it to allow installing new modules.

```shell
modules=/opt/modules  # Adjust this writable storage location as needed.
sudo mkdir -p "${modules}" "${modules}.wd"
sudo mount \
    -o "lowerdir=/usr/lib64/modules,upperdir=${modules},workdir=${modules}.wd" \
    -t overlay overlay /usr/lib64/modules
```

The following systemd unit can be written to `/etc/systemd/system/usr-lib64-modules.mount`.

```ini
[Unit]
Description=Custom Kernel Modules
Before=local-fs.target
ConditionPathExists=/opt/modules

[Mount]
Type=overlay
What=overlay
Where=/usr/lib64/modules
Options=lowerdir=/usr/lib64/modules,upperdir=/opt/modules,workdir=/opt/modules.wd

[Install]
WantedBy=local-fs.target
```

Enable the unit so this overlay is mounted automatically on boot.

```shell
sudo systemctl enable usr-lib64-modules.mount
```

An alternative is to mount the overlay automatically when the system boots by adding the following line to `/etc/fstab` (creating it if necessary).

```fstab
overlay /lib/modules overlay lowerdir=/lib/modules,upperdir=/opt/modules,workdir=/opt/modules.wd,nofail 0 0
```

## Prepare a Flatcar Container Linux development container

Read system configuration files to determine the URL of the development container that corresponds to the current Flatcar Container Linux version.

```shell
. /usr/share/flatcar/release
. /usr/share/flatcar/update.conf
url="https://${GROUP:-stable}.release.flatcar-linux.net/${FLATCAR_RELEASE_BOARD}/${FLATCAR_RELEASE_VERSION}/flatcar_developer_container.bin.bz2"
```

Download, decompress, and verify the development container image.

```shell
curl -LO https://kinvolk.io/flatcar-container-linux/security/image-signing-key/Flatcar_Image_Signing_Key.asc
gpg2 --import Flatcar_Image_Signing_Key.asc
curl -L "${url}" |
    tee >(bzip2 -d > flatcar_developer_container.bin) |
    gpg2 --verify <(curl -Ls "${url}.sig") -
```

Start the development container with the host's writable modules directory mounted into place.

```shell
sudo systemd-nspawn \
    --bind=/usr/lib64/modules \
    --image=flatcar_developer_container.bin
```

Now, inside the container, fetch the Flatcar Container Linux package definitions, then download and prepare the Linux kernel source for building external modules.

```shell
emerge-gitclone
emerge -gKv coreos-sources
gzip -cd /proc/config.gz > /usr/src/linux/.config
make -C /usr/src/linux modules_prepare
```

## Build and install kernel modules

At this point, upstream projects' instructions for building their out-of-tree modules should work in the Flatcar Container Linux development container. New kernel modules should be installed into `/usr/lib64/modules`, which is bind-mounted from the host, so they will be available on future boots without using the container again.

In case the installation step didn't update the module dependency files automatically, running the following command will ensure commands like `modprobe` function correctly with the new modules.

```shell
sudo depmod
```
