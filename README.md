# borgsnap - Backups using ZFS snapshots, borg, and (optionally) rsync.net

This is a simple script for doing automated daily backups of ZFS filesystems.
It uses ZFS snapshots, but could easily be adaptable to other filesystems,
including those without snapshots.

[Borg](https://www.borgbackup.org/) has excellent deduplication, so unchanged
blocks are only stored once. Borg backups are encrypted and compressed
(borgsnap uses lz4).

Unlike tools such as Duplicity, borg uses an incremental-forever model so you
never have to make a full backup more than once. This is really great when
sending full offsite backups might take multiple days to upload.

Borgsnap has optional integration with rsync.net for offsite backups. rsync.net
offers a [cheap plan catering to borg](http://www.rsync.net/products/attic.html).
As rsync.net charges nothing in transfer fees nor penalty fees for early
deletion, it's a very appealing option that is cheaper than other cloud storage
providers such as AWS S3 or Google Cloud Storage once you factor in transfer
costs and fees for deleting data.

Borgsnap automatically purges snapshots and old borg backups (both locally
and remotely) based on retention settings given in the configuration.

This assumes borg version 1.0 or later (I'm using the version shipped with
Debian Stretch).

Finally, these things are probably obvious, but: Make sure your local backups
are on a different physical drive than the data you are backing up and don't
forget to do remote backups, because local backups isn't disaster proofing
your data.

# Backup flow

Borgsnap is pretty simple, it has the following basic flow:

+ Read configuration file and encryption key file
+ Validate output directory exists and a few other basics
+ For each ZFS filesystem do the following steps:
  + Initialize borg repositories if local one doesn't exist
  + Take a ZFS snapshot of the filesystem
  + Run borg for the local output
  + Run borg for the rsync.net output (if configured)
  + Delete old ZFS snapshots
  + Prune local borg
  + Prune rsync.net borg

That's it!

If things fail, it is not currently re-entrant. For example, if a ZFS snapshot
already exists for the day, the script will fail. This could use a bit of
battle hardening, but has been working well for me for several months already.

# Restoring files

Borgsnap doesn't help with restoring files, it just backs them up. Restorations
are done directly from borg (or ZFS snapshots if it's a simple file deletion to
be restored). A backup that can't be restored from is useless, so you need to
test your backups regularly.

For Borgsnap, there are three ways to restore, depending on why you need to:

+ Use the local ZFS snapshot (magic .zfs directory on each ZFS filesystem).
This is the way to go if you simply deleted a file and there is no hardware
failure.

+ Use the local borg repository. If there is data loss on the ZFS filesystem,
but the backup drive is still good, use "borg mount" to mount up the directory
and restore files. See example below.

+ Use the remote borg repository. As with a local repository, use "borg mount"
to restore files from rsync.net.

## Examples

Note: Instead of setting BORG_PASSPHRASE as done here, with an exported
environment variable, you can paste it in interactively.

Also note that borgsnap does backups directly from the ZFS snapshot, using
the magic .zfs mount point, hence the borg snapshot preserves this directory
structure. Don't worry, borg is still deduplicating files, even though the
directory changes each time. Also, don't panic if you do "ls /mnt" and don't
see anything - try "ls -a /mnt" or you might miss seeing that .zfs directory.

```
$ sudo -i

# export BORG_PASSPHRASE=$(</path/to/my/super/secret/myhost.key)

# borg list /backup/borg/zroot/root
week-20171008                        Sun, 2017-10-08 01:07:29
day-20171009                         Mon, 2017-10-09 01:07:54
day-20171010                         Tue, 2017-10-10 01:07:48
day-20171011                         Wed, 2017-10-11 01:07:57

# borg mount /backup/borg/zroot/root::day-20171011 /mnt

# ls /mnt/.zfs/snapshot/day-20171011/
backup	bin   etc  home	 lib64  proc  root  sbin  tmp  var

# borg umount /mnt
```

Restoring from rsync.net is nearly the same, just a change in the path, and
passing --remote-path=borg1 since we are using a modern borg version:

```
# borg mount --remote-path=borg1 XXXX@YYYY.rsync.net:myhost/zroot/root::day-20171011 /mnt
```

I used "borg mount" above, where we would, simply "cp" the files out. See
the borg manpages to read about other restoration options, such as
"borg extract".
