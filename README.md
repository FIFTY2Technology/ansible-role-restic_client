[[_TOC_]]

# Description
Installs `restic` and configures it to do a full system backup every night. Each node starts its backup between 00:00 and 03:00 o'clock, always at the same point in time (e.g. always starting at 01:23 o'clock)[^1]. Backups are stored in `/srv/restic` by default, other repository backends can be configured (e.g. [restic REST server](https://github.com/restic/rest-server)).

The role creates a systemd service file `restic-backup.service` that runs periodically via a `.timer` file. A `restic-prune.service` runs the `restic prune --forget` command once a week to delete old backups.

## LVM snapshots
If LVM is detected on a node, automatic creation of snapshots is included as part of the backup process.
* `restic-backup.service` will be run inside a chroot of the mounted `/`-snapshot, with more snapshots mounted below, as applicable.
* LVM logical volumes with `swap` or `tmp` in their names are not included.
* At least 8 (eight) percent of the origins LV size must be free space in the LVM volume group for snapshots. A good estimate is to not let logical volumes (LVs) take more than 90 percent of the available space in a volume group (VG), or 10% free extents and you should be good to go. More is better, see point below.
* To prevent failing backups due to full and invalidated snapshots, LVM gets configured to automatically extend snapshot sizes once they reach 80 percent capacity (can be disabled).
* Only logical volumes are considered that are mounted at the time of the playbook run. To add or remove LVs, mount/umount them and rerun the playbook.
* Systems with existing logical volumes with names starting with `backup` are not tested and might cause name collision or even damage. This role creates LV snapshots named `backup*` per mount point, e.g. `backup_` for `/`, `backup_home` for `/home`, etc.
* Tested with ext4 and xfs filesystems on LVM.

## Remote pull-style backups
Backups can be triggered by the backup server, if a client cannot reach the backup server directly. (e.g. due to firewall rules, NAT, etc.) For this, the client must be deployed with `restic_client_is_remote: true` option. This functionality is only tested on a server deployed with the `restic_server` role.

The backup client will be configured with a not-activated systemd timer for `backup`/`prune` operations in this case. The backup server will be configured to periodically connect to the client via SSH and open a [Remote Port Forwarding](https://www.ssh.com/academy/ssh/tunneling/example#remote-forwarding) to the restic `rest-server`. The client is then able to reach the `rest-server` on `127.0.0.1` through the SSH tunnel.

# Requirements
* Requires `bunzip2` to be installed on the Ansible runner.
* Package `fuse` must be installed to use the `restic mount` command.

# Role Variables
All variables which can be overridden are stored in defaults/main.yml file as well as in table below.
| Name | Default value | Description |
| ------ | ------ | ----- |
| `restic_client_repository` | `templates/repository.j2` | Repository definition for restic backups. Must be a path to a Jinja2 template file. Default template sets `/srv/backup`. See template content for more info, and `templates/repository.example*.j2` for more examples. Content will be written to a file and used via `restic --repository-file`. |
| `restic_client_encryption_password` | "" | Password for encrypting the restic repository.  Must be set explicitly. Should be unique for every host. From `restic init` output: "_Please note that knowledge of your password is required to access the repository. Losing your password means that your data is irrecoverably lost._" |
| `restic_client_user` | "restic" | Username to use for running restic command |
| `restic_client_group` | "restic" | Groupname to use for running restic command |
| `restic_client_home` | "/home/restic" | Home directory of `restic_client_user` |
| `restic_client_config_dir` | `"{{ restic_client_home }}/.config/restic"` | Folder where the config and all restic-specific files will live in |
| `restic_client_bin_dir` | `/usr/local/bin"` | Folder the restic binary will be put in |
| `restic_client_use_lvm` | true | Use LVM if available on the client. Can only be disabled explicitly when the use of LVM snapshots is not wished (and in case LVM is present), but cannot enable LVM functionality when no LVM is present on the client. Useful e.g. if a pre-existing VG does not have any free physical extends (PE) available for snapshots. |
| `restic_client_lvm_snapshot_dir` | `"{{ restic_client_home }}/snapshots"` | Directory to mount snapshots into during backup process. Only used if LVM is detected on the target system |
| `restic_client_lvm_allow_edit_lvmconf` | true |  It's hard to guess whether a setting in `/etc/lvm/lvm.conf` is already set to a non-default value, so if you edited this file otherwise and want to keep your config, set this to `false`.<br/>Used for:<br/>* Configure LVM to automatically extend snapshot sizes if they reach 80 percent capacity. Can only extend snapshots as long as there are LVM extents available. |
| `restic_client_use_default_includes` | true | Set to false to ignore (not use) directories listed in `restic_client_default_includes`. You should define your own directories via `restic_client_custom_includes` then. |
| `restic_client_default_includes` | `/` | List of directories to include into backups by default, as long as not ignored via `restic_client_use_default_includes: false`. |
| `restic_client_custom_includes` | `[]` | List of directories to include into backups. Always used and appended to (if not ignored) `restic_client_default_includes`. |
| `restic_client_use_default_excludes` | true | Set to false to ignore (not use) directories listed in `restic_client_default_excludes`. You should define your own directories via `restic_client_custom_excludes` then. |
| `restic_client_default_excludes` | `/dev`<br>`/proc`<br>`/sys`<br>`/run`<br>`/tmp`<br>`/var/tmp`<br>`/var/lib`<br>`/var/cache`<br>`/usr` | List of directories to exclude from backups by default, as long as not ignored via `restic_client_use_default_excludes: false`.<br>The directory `restic_client_lvm_snapshot_dir` will be automatically added to the list of excluded directories if LVM is used. This prevents restic to backup those directories, which would result in roughly double the number of files per snapshot. (Snapshot size would not be affected thanks to deduplication) |
| `restic_client_custom_excludes` | `[]` | List of directories to exclude from backups. Always used and appended to (if not ignored) `restic_client_default_excludes`. |
| `restic_client_is_remote` | false | Enable to let the backup server connect via SSH to initiate a backup. Convenient for clients that cannot reach the backup server directly. If set to `true`, you should also run the `restic_remote` role. Default: false |
| `restic_client_no_log` | true | Do not show sensible content when printed by `ansible-playbook` runs. Set to `false` for debugging e.g. repository file templating. |
| `restic_client_custom_prepare_commands` | `[]` | Run custom commands as the `restic_client_user` prior to the backup. Output files will be included in the backup snapshot in that user's home directory, unless an absolute path is specified explicitly for the output file in a command. Stdout and stderr will be visible in logs (journald). Shell syntax is not fully supported, see: [systemd.service - Command lines](https://www.freedesktop.org/software/systemd/man/systemd.service.html#Command%20lines). Older distributions/versions of systemd don't accept relative paths (e.g. Ubuntu 18.04 with systemd 237), use absolute paths instead. If one command fails, the backup fails and won't start. |

# Dependencies
None.

# Example Playbook
```
- hosts: backupclients
  gather_facts: false
  become: true

  roles:
  - role: restic_client
```

See different molecule scenarios for more examples.

## Example CLI usage of `restic` command
To use the `restic` command on the client's CLI, it is best to switch to the `restic` user, since it will have `RESTIC_REPOSITORY_FILE` and `RESTIC_PASSWORD_FILE` environment variables sourced automatically.

For showing the output of all involved services, you can use `journalctl -ft restic`, which follows the syslog identifier `restic` (configured for all `restic-*` services).

# Known Limitations
## Failing backups
### LVM snapshot at 100%
Backups can fail due to filled-up LVM snapshots and exhausted LVM volume group extends. In this case, it must be recovered manually, otherwise subsequent backups will fail (because snapshots already exist):
* Make sure all snapshots are unmounted from the snapshot directory (default `/home/restic/snapshots`)
* Run `lvremove /dev/<vg-name>/backup_*` to remove all snapshots created by systemd service `restic-snapshot-create.service`.
  * In rare cases, LVM will report that an LV snapshot is still in use although unmounted and not showing anything in `lsof` output. Also, LVM snapshots can't get deactivated without deactivating their parent, so if temporarily unmounting the parent LV is not an option for you, the machine must be rebooted.
* _Optional_: Run `systemctl reset-failed` to clear `failed` state from systemd services `restic-snapshot-create.service` and `restic-snapshot-delete.service`.
* Run `systemctl start restic-backup --no-block` or wait for the `restic-backup.timer` script to run again.

### Permission denied
On some directories - e.g. `gvfs` mounts on Ubuntu - `restic` cannot read the metainformation and file contents:
```
restic[673470]: scan: lstat /home/user/.cache/gvfs: permission denied
restic[673470]: error: lstat /home/user/.cache/gvfs: permission denied
```
This will mark the systemd service as `failed`, although the backup will finish correctly. Add affected directories to `restic_client_custom_excludes` list.

### File disappeared
Occationally, files will disappear after restic scanned them but did not backup them yet. Backups will finish successfully, but the `restic-backup.service` will be marked as `failed`. To remedy, either **a**) rerun the `restic-backup.service`, **b**) reset the service state by running `systemctl reset-failed restic-backup.service`, or **c**) ignore and wait for next backup.

## Backup consistency
If LVM is not used on a node, there is no mechanism in place to ensure consistency for backups, so a backup might include files from two distinct application states.

# Footnotes
[^1]: Applies to clients with systemd version >= `247`. In systemd versions lower than `247` the corresponding option is ignored and a random point in time between 00:00 and 03:00 o'clock is picked on each run.

