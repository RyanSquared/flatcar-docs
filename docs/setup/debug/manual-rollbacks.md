---
title: Performing manual Flatcar Container Linux rollbacks
linktitle: Manual version rollbacks
description: How to manually rollback to a previous Flatcar version.
weight: 20
aliases:
    - ../../os/manual-rollbacks
    - ../../clusters/debug/manual-rollbacks
---

In the event of an upgrade failure, Flatcar Container Linux will automatically boot with the version on the rollback partition. Immediately after an upgrade reboot, the active version of Flatcar Container Linux can be rolled back to the version installed on the rollback partition, or downgraded to the version current on any lower release channel. There is no method to downgrade to an arbitrary version number.

This section describes the automated upgrade process, performing a manual rollback, and forcing a channel downgrade.

**Note:** Neither performing a manual rollback nor forcing a channel downgrade are recommended.

## Automated rollbacks

The rollback to the previously installed version is done by GRUB and happens automatically if `update-engine` had no chance to mark the version as successful.
This marking happens when the new version is booted and keeps running for around two minutes, at which point `update-engine` will mark the version as successful (how this works in detail is explained below).

To extend the automatic rollback logic to cover your important systemd services, you could make them as requirement for the `update-engine.service`.

Note that `update-engine` will still try to update which can cause a loop with disruptions due to the reboots.
You can disable automatic updates by setting `SERVER=disabled` in `/etc/flatcar/update.conf`.

## Rollback with `flatcar-update`

While you can rollback to the previously installed version manually with the rest of this guide, you can also install any version to the inactive partition with the `flatcar-update` tool.
To rollback to a known-good version, run it as follows:

```shell
$ sudo flatcar-update --to-version 2905.2.6 --disable-afterwards
```

The `--disable-afterwards` switch writes `SERVER=disabled` to `/etc/flatcar/update.conf` which disables updates.
This ensures that you will stay on the version you specified.

## How do updates work

The system's GPT tables are used to encode which partition is currently active and which is passive. This can be seen using the `cgpt` command.

```shell
$ cgpt show /dev/sda
       start        size    part  contents
           0           1          Hybrid MBR
           1           1          Pri GPT header
           2          32          Pri GPT table
        4096      262144       1  Label: "EFI-SYSTEM"
                                  Type: EFI System Partition
                                  UUID: 596FF08E-5617-4497-B10B-27A23F658B73
                                  Attr: Legacy BIOS Bootable
      266240        4096       2  Label: "BIOS-BOOT"
                                  Type: BIOS Boot Partition
                                  UUID: EACCC3D5-E7E9-461D-A6E2-1DCDAE4671EC
      270336     2097152       3  Label: "USR-A"
                                  Type: Alias for flatcar-rootfs
                                  UUID: 7130C94A-213A-4E5A-8E26-6CCE9662F132
                                  Attr: priority=2 tries=0 successful=1
     2367488     2097152       4  Label: "USR-B"
                                  Type: Alias for flatcar-rootfs
                                  UUID: E03DD35C-7C2D-4A47-B3FE-27F15780A57C
                                  Attr: priority=1 tries=0 successful=0
     4464640      262144       6  Label: "OEM"
                                  Type: Alias for linux-data
                                  UUID: 726E33FA-DFE9-45B2-B215-FB35CD9C2388
     4726784      131072       7  Label: "OEM-CONFIG"
                                  Type: Flatcar Container Linux reserved
                                  UUID: 8F39CE8B-1FB3-4E7E-A784-0C53C8F40442
     4857856    37085151       9  Label: "ROOT"
                                  Type: Flatcar Container Linux auto-resize
                                  UUID: D9A972BB-8084-4AB5-BA55-F8A3AFFAD70D
    41943007          32          Sec GPT table
    41943039           1          Sec GPT header
```

Looking specifically at "USR-A" and "USR-B", we see that "USR-A" is the active USR partition (this is what's actually mounted at /usr; you can verify this with `rootdev -s /usr`). Its priority is higher than that of "USR-B". When the system boots, GRUB (the bootloader) looks at the priorities, tries, and successful flags to determine which partition to use.

```shell
      270336     2097152       3  Label: "USR-A"
                                  Type: Alias for flatcar-rootfs
                                  UUID: 7130C94A-213A-4E5A-8E26-6CCE9662F132
                                  Attr: priority=2 tries=0 successful=1
     2367488     2097152       4  Label: "USR-B"
                                  Type: Alias for flatcar-rootfs
                                  UUID: E03DD35C-7C2D-4A47-B3FE-27F15780A57C
                                  Attr: priority=1 tries=0 successful=0
```

You'll notice that on this machine, "USR-B" hasn't actually successfully booted. Not to worry! This is a fresh machine that hasn't been through an update cycle yet. When the machine downloads an update, the partition table is updated to allow the newer image to boot.

```shell
      270336     2097152       3  Label: "USR-A"
                                  Type: Alias for flatcar-rootfs
                                  UUID: 7130C94A-213A-4E5A-8E26-6CCE9662F132
                                  Attr: priority=1 tries=0 successful=1
     2367488     2097152       4  Label: "USR-B"
                                  Type: Alias for flatcar-rootfs
                                  UUID: E03DD35C-7C2D-4A47-B3FE-27F15780A57C
                                  Attr: priority=2 tries=1 successful=0
```

In this case, we see that "USR-B" now has a higher priority and it has one try to successfully boot. Once the machine reboots, the partition table will again be updated.

```shell
      270336     2097152       3  Label: "USR-A"
                                  Type: Alias for flatcar-rootfs
                                  UUID: 7130C94A-213A-4E5A-8E26-6CCE9662F132
                                  Attr: priority=1 tries=0 successful=1
     2367488     2097152       4  Label: "USR-B"
                                  Type: Alias for flatcar-rootfs
                                  UUID: E03DD35C-7C2D-4A47-B3FE-27F15780A57C
                                  Attr: priority=2 tries=0 successful=0
```

Now we see that the number of tries for "USR-B" has been decremented to zero. The successful flag still hasn't been updated though. Once update-engine has had a chance to run, it marks the boot as being successful.

```shell
      270336     2097152       3  Label: "USR-A"
                                  Type: Alias for flatcar-rootfs
                                  UUID: 7130C94A-213A-4E5A-8E26-6CCE9662F132
                                  Attr: priority=1 tries=0 successful=1
     2367488     2097152       4  Label: "USR-B"
                                  Type: Alias for flatcar-rootfs
                                  UUID: E03DD35C-7C2D-4A47-B3FE-27F15780A57C
                                  Attr: priority=2 tries=0 successful=1
```

**Note:** You may also see `Alias for coreos-rootfs` shown for the `/usr` partition instead of the `flatcar-rootfs`. To refer to them you can use both names or the more appropriate `flatcar-usr` name which we will use from now on.

## Performing a manual rollback

So, now that we understand what happens when the machine updates, we can tweak the process so that it boots an older image (assuming it's still intact on the passive partition). The first command we'll use is `cgpt find -t flatcar-usr`. This will give us a list of all of the USR partitions available on the disk.

```shell
$ cgpt find -t flatcar-usr
/dev/sda3
/dev/sda4
```

To figure out which partition is currently active, we can use `rootdev`.

```shell
$ rootdev -s /usr
/dev/sda4
```

So now we know that `/dev/sda3` is the passive partition on our system. We can compose the previous two commands to dynamically figure out the passive partition.

```shell
$ cgpt find -t flatcar-usr | grep --invert-match "$(rootdev -s /usr)"
/dev/sda3
```

In order to rollback, we need to mark that partition as active using `cgpt prioritize`.

```shell
cgpt prioritize /dev/sda3
```

If we take another look at the GPT tables, we'll see that the priorities have been updated.

```shell
      270336     2097152       3  Label: "USR-A"
                                  Type: Alias for flatcar-rootfs
                                  UUID: 7130C94A-213A-4E5A-8E26-6CCE9662F132
                                  Attr: priority=2 tries=0 successful=1
     2367488     2097152       4  Label: "USR-B"
                                  Type: Alias for flatcar-rootfs
                                  UUID: E03DD35C-7C2D-4A47-B3FE-27F15780A57C
                                  Attr: priority=1 tries=0 successful=1

```

Composing the previous two commands produces the following command to set the currently passive partition to be active on the next boot:

```shell
cgpt prioritize "$(cgpt find -t flatcar-usr | grep --invert-match "$(rootdev -s /usr)")"
```

In the above scenario, _tries_ can stay 0 because the partition was marked as _successful_.
If the partition was not successfully booted, we also need to set the available _tries_ to 1 again:

```shell
cgpt add -T 1 /dev/sda3
```

## Forcing a Channel Downgrade

The procedure above restores the last known good Flatcar Container Linux version from immediately before an upgrade reboot. The system remains on the same [Flatcar Container Linux channel][relchans] after rebooting with the previous USR partition. It is also possible, though not recommended, to switch a Flatcar Container Linux installation to an older release channel, for example to make a system running an Alpha release downgrade to the Stable channel. Root privileges are required for this procedure, noted by `sudo` in the commands below.

First, edit `/etc/coreos/update.conf` to set `GROUP` to the name of the target channel, one of `stable` or `beta`:

```ini
GROUP=stable
```

Next, clear the current version number from the `release` file so that the target channel will be certain to have a higher version number, triggering the "upgrade," in this case a downgrade to the lower channel. Since `release` is on a read-only file system, it is convenient to temporarily override it with a bind mount. To do this, copy the original to a writable location, then bind the copy over the system `release` file:

```shell
cp /usr/share/coreos/release /tmp
sudo mount -o bind /tmp/release /usr/share/coreos/release
```

The file is now writable, but the bind mount will not survive the reboot, so that the default read-only system `release` file will be restored after this procedure is complete. Edit `/usr/share/coreos/release` to replace the value of `COREOS_RELEASE_VERSION` with `0.0.0`:

```ini
COREOS_RELEASE_VERSION=0.0.0
```

Restart the update service so that it rescans the edited configuration, then initiate an update. The system will reboot into the selected lower channel after downloading the release:

```shell
update_engine_client -update
```


[relchans]: ../releases/switching-channels
