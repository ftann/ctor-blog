---
title: "Local storage and nextcloud encryption"
keywords: ["encryption", "nextcloud"]
tags: ["cloud", "encryption", "nextcloud"]
date: 2021-01-31T13:41:16+01:00
---

Nextcloud offers [server side encryption][0] for data that is stored on external storage providers (
e.g. amazon s3). Enabling the encryption module automatically stores new files encrypted on external
storage. This also applies to local external storage (mount points).

The encrypted data can only be shared via nextcloud's sharing functionality. This leads to problems
with local external storage that other applications might use. Other applications can't read the new
files.

If the garbage inside the garbage must be accessible then there are 2 solutions available.

## Situation

Let's assume that an existing local directory contains music which is accessible as local external
storage in nextcloud and is also accessible to plex and periodically scanned for new files. The user
is a huge Bieber fan and uploads an album to the shared directory.<br/>
Plex will only read garbage (rightfully so). Only nextcloud can read the correct garbage music.

## 1. Disable encryption for local mounts

The album can be made accessible to plex by [disabling the encryption for the shared directory][1]
only. Remove the mark on the `Enable encryption` checkbox.

This still enables the encryption for
other external storages. However, the album must be removed and uploaded again.

## 2. Disable server side encryption

If external storage is limited to local mounts only the whole encryption module can be disabled and
already encrypted content can be decrypted. Read the [nextcloud documentation][0] about the server
side encryption carefully!

Use these commands to decrypt all files for all users and disable the encryption.

```shell
occ encryption:status
occ encryption:decrypt-all
occ maintenance:mode --on
occ encryption:disable
occ maintenance:mode --off
```

Now all of your users can listen to that garbage music.

[0]: https://docs.nextcloud.com/server/20/admin_manual/configuration_files/encryption_configuration.html

[1]: https://docs.nextcloud.com/server/20/admin_manual/configuration_files/external_storage_configuration_gui.html#mount-options
