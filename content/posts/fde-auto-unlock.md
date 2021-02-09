---
title: "Full-disk-encryption with auto unlock and fallback"
keywords: ["server", "encryption", "linux", "auto", "zfs"]
tags: ["cryptsetup", "dracut", "encryption", "linux", "systemd", "zfs"]
date: 2021-02-06T10:39:38+01:00
---

Your precious server is already secured by full disk encryption (FDE) and now it's time to take back
some comfort you knew from the dark ages of clear text? Look no further and learn how to boot your
machine with an usb-key-stick or via SSH as fallback.

Be warned this comes with drawbacks to keep in mind. These are discussed in the last
section [further reading](#further-reading).

# Requirements

This guide is based on [fedora][0] linux and luks full disk encryption. The initramfs is modified to
contain a simple ssh server setup.

- __Fedora with luks FDE__
- Systemd
- Dracut
- SSH access
- USB stick

## Optional encrypted storage

This part isn't mandatory for the unlocking process.

In addition to the encrypted root filesystem a separate data storage based on a mirrored zfs array
is used. This storage contains mount points used by docker containers. In this guide docker is
configured to be dependent on the encrypted storage.

# Key files

The first step is to generate the necessary key files. Two reasonably long keys are generated from
random data. One for our root file system and the other for the data storage.

```shell
dd bs=1024 count=8 if=/dev/random of=root.key
dd bs=1024 count=8 if=/dev/random of=data.key
```

## Format the USB stick

The weakness in this setup is the usb stick. It isn't encrypted and therefore the key files are
accessible to everybody. __DON'T LOSE YOUR KEYS!__.
Fedora needs `exfatprogs` installed for `mkfs.exfat` being available

```shell
dnf install exfatprogs
mkfs.exfat /dev/sd<Xn>
```

Then mount the usb stick and copy the keys to the stick:

```shell
mkdir /mnt/usb
mount /dev/sd<Xn> /mnt/usb
cp *.key /mnt/usb
```

### USB stick uuid

During the guide the uuid of the partition is needed. Henceforth, it shall be known as `stick-uuid`.
It is retrieved by issuing the following command:

```shell
blkid -s UUID -o value /dev/sd<Xn>
```

# Add keys to LUKS

The second step is to add the key files to the luks partitions.

```shell
cryptsetup luksAddKey /dev/sd<A> root.key
```

## Optional data storage

Add the key file to the data storage partitions as well.

```shell
cryptsetup luksAddKey /dev/sd<D1> data.key
cryptsetup luksAddKey /dev/sd<D2> data.key
```

# Use key files

In the third step the kernel command line is adjusted to use the key files present on the USB stick,
then the `/etc/crypttab` is adjusted to use the key files for the data storage too.

## Kernel command line

To configure the kernel command line in fedora edit the `/etc/default/grub` file. Beside the
existing option that instructs the kernel on which partition contains the encrypted root file system
several additional commands are added.

The entry to be modified is `GRUB_CMDLINE_LINUX`:

```shell
GRUB_CMDLINE_LINUX="rd.luks.uuid=root-uuid rd.luks.key=root-uuid=/root.key:UUID=stick-uuid rd.luks.options=root-uuid=discard,keyfile-timeout=20s rd.luks.crypttab=no"
```

- __rd.luks.uuid=root-uuid__
  sets the partition uuid of the root file system.
- __rd.luks.key=root-uuid=/root.key:UUID=stick-uuid__
  sets where the kernel finds the key for the root file system. In this case it is `/root.key` on
  the partition with the `stick-uuid` id.
- __rd.luks.options=root-uuid=discard,keyfile-timeout=20s__
  sets the discard option, and the timeout to wait for the key file
- __rd.luks.crypttab=no__
  the crypttab generator now ignores in the [initrd][2] the `/etc/crypttab` entries. This important
  because these are supplied directly.

After editing the kernel command line update the grub configuration:

```shell
grub2-mkconfig -o /etc/grub2-efi.cfg
```

## Crypttab

The `/etc/crypttab` defines the encrypted file systems the system shall unlock during boot. In this
file the root file system should be excluded because it is supplied directly in the command line.
Other file systems shall be unlocked after early boot. That is why `rd.luks.crypttab=no` is set.

Remove the root file system line (or comment out) from `/etc/crypttab`:

```shell
# Exclude root volume because unlocking is done by kernel arguments.
#<name> UUID=root-uuid none
```

### Optional data storage

However, the data storage should be listed because after the root file system is mounted systemd
uses the crypttab as instructed (`rd.luks.crypttab` only applies to initrd unlike `luks.crypttab`)
to unlock encrypted file systems.

Edit according to your need but ensure that `nofail`, `keyfile-timeout` and `timeout` are set.

```shell
# Exclude root volume because unlocking is done by kernel arguments.
#<name> UUID=root-uuid none
data1 UUID=<data1-uuid> /data.key:UUID=stick-uuid discard,keyfile-timeout=10s,timeout=10s,nofail
data2 UUID=<data2-uuid> /data.key:UUID=stick-uuid discard,keyfile-timeout=10s,timeout=10s,nofail
```

- __keyfile-timeout=10s__
  waits for 10 seconds before giving up on the key file
- __timeout=10s__
  waits for 10 seconds for user input before giving up
- __nofail__
  This is important because it instructs systemd to not fail if this encrypted file system could not
  be mounted

It is mandatory to set both timeout options or else the kernel will wait indefinitely for user input
to unlock the encrypted partitions.

#### ZFS

Note that if using zfs it's mandatory to use the [zfs mount generator][3]. The generator provides
the option to set `nofail` for zfs mounts. This option is required to let the system successfully
boot. Else it will wait for the zfs volumes to appear.

Set the `nofail` option for all datasets.

```shell
zfs set org.openzfs.systemd:nofail=on <volume>
```

## Update initrd

Ensure that the initial ram disk contains all the changes so far.

```shell
dracut -f -v
```

# Automatic unlock

From now on it should be possible to start the machine with the usb-key-stick inserted. The current
fallback is the user supplied password which requires local access to the machine.

The next steps focus on the fallback with an early boot SSH server. This enables a remote-only boot
process.

___

# Add SSH to early boot

Step five is to add SSH to early boot. There exists a [dracut ssh module][4] that easily adds an SSH
server to the initial ram disk. It is easy to add. The module is then enabled by default.

```shell
dnf copr enable gsauthof/dracut-sshd
dnf install -y dracut-network
```

## Network connectivity

This guide doesn't use systemd's networkd in early boot. To start network connectivity in early boot
add the following kernel command line options:

```shell
rd.neednet=1 ip=dhcp
```

## SSH key

In early boot the system has no access to the authorized SSH keys of the root user. The dracut
module incorporates the list of authorized keys from several locations. This guide uses
`/root/.ssh/dracut_authorized_keys` as location for these.

This could be a copy of your existing user's list of authorized keys. The content should look
similar:

```shell
ssh-ed25519 <pubkey> <name>
```

### ! (IMPORTANT) Permission denied (RTFM)

Some setups leave the root account locked or without a password, and the admin is the user created
during setup. This prevents SSH access in early boot. Because the SSH server by default uses `root`
for the login and `root` doesn't have a password set the login fails.

_Change the account to have no password, but be unlocked. An account has no password if the password
hash in the password database is not the hash of any string. Traditionally, a one-character string
such as * or ! is used for that._ (https://unix.stackexchange.com/a/193131)

To solve this situation the root user needs to have a special password set. Read the [FAQ][5] for
more information.

```shell
usermod -p '*' root
```

## Update initrd again

[Update the initial ram disk again](#update-initrd)

# Automatic unlocking with SSH fallback

This marks the point where this guide was intended to lead to. The machine boots with the
usb-key-stick automatically or else with manual unlocking of the encrypted root filesystem by ssh.

If the optional encrypted storage is present the few additional steps will show how to account for
the missing mount points with docker (and zfs).

___

# Optional docker and zfs

If there are containers that depend on mount points from encrypted storage, then prevent docker from
starting altogether. This will prevent situations where these directories are created and used
instead of the real data from mount points.

Create a [systemd target][6] `container-data.target` in `/etc/systemd/system`. Add `.mount`s based
on the mount points used by containers.

```shell
[Unit]
Description=Container data mounts
BindsTo=zfs.target
Requires=zfs.target mnt-data.mount mnt-data-x.mount mnt-data-y.mount

[Install]
RequiredBy=docker.service
```

Then enable the target.

```shell
systemctl enable container-data.target
```

Now when the encrypted storage is not mounted because the usb-key-stick and in turn the key is
missing, docker won't start. The system will still be started and accessible by SSH. With a little
script the volumes can be unlocked and docker started.

```shell
#!/usr/bin/env bash

systemctl restart systemd-cryptsetup@data1.service
systemctl restart systemd-cryptsetup@data2.service
systemctl restart zfs-import.target container-data.target docker.service
```

___

# Further reading

There are several topics to investigate further. Certainly the most important is all the drawbacks.
Having the keys accessible on a USB stick is a serious threat to consider. This guide shows how to
unlock a headless system but for top secure systems this is not an option.<br/>
More sophisticated tokens are available in the form of yubikeys for example.
[Systemd integrates those.][7] When this is still an issue, then only the remote unlocking by SSH is
applicable.

This guide assumes that FDE is deployed, but the boot partition is still not encrypted. From a
technical point this is option is available. However, grub only allows unlocking the boot partition
with local user input. This won't work for remote, headless systems. The term _evil maid_ rings a
bell?<br/>
[chkboot][8] is a small utility that is able to provide tamper detection of the boot (efi)
partition. However, this is beyond the scope of this guide (already tl;dr).

Even without the usb-key-stick and with chkboot in place _cold boot attacks_ could reveal the
encryption keys. Have a look at [TRESOR][9]. This tunes the kernel to save the encryption keys in
x86 debug registers instead of the ram.

To further worsen the situation one could assume the very core of the system could be rotten.
Nothing will then stop attackers from gaining access to the supposedly encrypted data. Demand
open-source efi implementations and get rid of intel me and the likes. Good luck.

# Thanks for all the fish

Do you like the guide and want to give feedback or found a mistake? Then send me a mail to `f4ntasyland /at/ protonmail /dot/ com`

You can always buy me a beer.
`（ ^_^）o自自o（^_^ ）`

_[xmr][10]: 473WTZ1gWFdjdEyioCQfbGQKRurZQoPDJLhuFJyubCrk4TRogKCtRum63bFMx2dP2y4AN1vf2fN6La7V7eB2cZ4vNJgMAcG_

[0]: https://docs.fedoraproject.org/en-US/fedora/f33/install-guide/

[1]: https://openzfs.github.io/openzfs-docs/Getting%20Started/index.html

[2]: https://en.wikipedia.org/wiki/Initial_ramdisk

[3]: https://wiki.archlinux.org/index.php/ZFS#Automatic_Start

[4]: https://github.com/gsauthof/dracut-sshd

[5]: https://github.com/gsauthof/dracut-sshd#faq

[6]: https://www.freedesktop.org/software/systemd/man/systemd.target.html

[7]: http://0pointer.net/blog/unlocking-luks2-volumes-with-tpm2-fido2-pkcs11-security-hardware-on-systemd-248.html

[8]: https://github.com/grazzolini/chkboot

[9]: https://en.wikipedia.org/wiki/TRESOR

[10]: https://www.getmonero.org/
