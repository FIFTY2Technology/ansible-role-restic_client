# Description
Installs `restic` and configures it to do a full system backup every night. Each node starts its backup between 00:00 and 03:00 o'clock, always at the same point in time (e.g. always starting at 01:23 o'clock).[^1] Backups are pushed to a [restic REST server](https://github.com/restic/rest-server), other repository backends are not (yet) supported.

It creates a systemd service file `restic-backup.service` that runs periodically via a `.timer` file. A `restic-prune.service` runs the `restic prune --forget` command once a week to delete old backups.

## LVM snapshots
If LVM is detected on a node, automatic creation of snapshots is included as part of the backup process.
* `restic-backup.service` will be run inside a chroot of the mounted `/`-snapshot, with more snapshots mounted below, as applicable.
* LVM logical volumes with `swap` or `tmp` in their names are not included.
* At least 8 (eight) percent of the origins LV size must be free space in the LVM volume group for snapshots. A good estimate is to not let logical volumes (LVs) take more than 90 percent of the available space in a volume group (VG), or 10% free extents and you should be good to go.
* Only logical volumes are considered that are mounted at the time of the playbook run. To add or remove LVs, mount/umount them and rerun the playbook.
* Tested with ext4 and xfs filesystems on LVM.

## Remote pull-style backups
Backups can be triggered by the backup server, if a client cannot reach the backup server directly. (e.g. due to firewall rules, NAT, etc.) For this, the client must be deployed with `restic_client_is_remote: true` option. This functionality is only tested on a server deployed with the `restic_server` role.

The backup client will be configured with a not-activated systemd timer for `backup`/`prune` operations in this case. The backup server will be configured to periodically connect to the client via SSH and open a [Remote Port Forwarding](https://www.ssh.com/academy/ssh/tunneling/example#remote-forwarding) to the restic `rest-server`. The client is then able to reach the `rest-server` on `127.0.0.1` through the SSH tunnel.

# Requirements
* Requires `bunzip2` to be installed on the Ansible runner.
* [pass](https://www.passwordstore.org/) installed on the Ansible runner and configured for password storage. ([See example configuration](https://www.fifty2.eu/innovation/how-we-provide-i-t-secrets-through-passwordstore-in-ansible-at-fifty2/)) Can be un-configured.

# Role Variables
`restic_client_backup_server_hostname` (set to `backup_server` here as a default) must be set to a hostname if a remote repository is specified.

Passwords are stored:
* on the client:
  * backup encryption:`/home/restic/.config/restic/password`
  * REST credentials: `/home/restic/.config/restic/repository`
* in the password-store:
  * backup encryption: `IT/backup/encryption/$HOSTNAME` (configurable)
  * REST credentials: `IT/backup/auth/$HOSTNAME` (configurable)

All variables which can be overridden are stored in defaults/main.yml file as well as in table below.
| Name | Default value | Description |
| ------ | ------ | ----- |
| `restic_client_user` | "restic" | Username to use for running restic command |
| `restic_client_group` | "restic" | Groupname to use for running restic command |
| `restic_client_home` | "/home/restic" | Home directory of `restic_client_user` |
| `restic_client_config_dir` | `"{{ restic_client_home }}/.config/restic"` | Folder where the config and all restic-specific files will live in |
| `restic_client_bin_dir` | `/usr/local/bin"` | Folder the restic binary will be put in |
| `restic_client_backup_server_hostname` | `"{{ backup_server }}"` | Specify backup REST server hostname to use as remote target |
| `restic_client_backup_server_port` | 3000 | Specify backup REST server port to use as remote target |
| `restic_client_use_lvm` | true | Use LVM if available on the client. Can only be disabled explicitly when the use of LVM snapshots is not wished (and in case LVM is present), but cannot enable LVM functionality when no LVM is present on the client. Useful e.g. if a pre-existing VG does not have any free physical extends (PE) available for snapshots. |
| `restic_client_lvm_snapshot_dir` | `"{{ restic_client_home }}/snapshots"` | Directory to mount snapshots into during backup process. Only used if LVM is detected on the target system |
| `restic_client_use_default_includes` | true | Set to false to ignore (not use) directories listed in `restic_client_default_includes`. You should define your own directories via `restic_client_custom_includes` then. |
| `restic_client_default_includes` | `/` | List of directories to include into backups by default, as long as not ignored via `restic_client_use_default_includes: false`. |
| `restic_client_custom_includes` | `[]` | List of directories to include into backups. Always used and appended to (if not ignored) `restic_client_default_includes`. |
| `restic_client_use_default_excludes` | true | Set to false to ignore (not use) directories listed in `restic_client_default_excludes`. You should define your own directories via `restic_client_custom_excludes` then. |
| `restic_client_default_excludes` | `/dev`<br>`/proc`<br>`/sys`<br>`/run`<br>`/tmp`<br>`/var/tmp`<br>`/var/lib`<br>`/var/cache`<br>`/usr` | List of directories to exclude from backups by default, as long as not ignored via `restic_client_use_default_excludes: false`.<br>The directory `restic_client_lvm_snapshot_dir` will be automatically added to the list of excluded directories if LVM is used. This prevents restic to backup those directories, which would result in roughly double the number of files per snapshot. (Snapshot size would not be affected thanks to deduplication) |
| `restic_client_custom_excludes` | `[]` | List of directories to exclude from backups. Always used and appended to (if not ignored) `restic_client_default_excludes`. |
| `restic_client_is_remote` | false | Enable to let the backup server connect via SSH to initiate a backup. Convenient for clients that cannot reach the backup server directly. If set to `true`, you should also run the `restic_remote` role. Default: false |
| `restic_client_custom_prepare_commands` | `[]` | Run custom commands as the `restic_client_user` prior to the backup. Output files will be included in the backup snapshot in that user's home directory, unless an absolute path is specified explicitly for the output file in a command. Stdout and stderr will be visible in logs (journald). Shell syntax is not fully supported, see: [systemd.service - Command lines](https://www.freedesktop.org/software/systemd/man/systemd.service.html#Command%20lines). Older distributions/versions of systemd don't accept relative paths (e.g. Ubuntu 18.04 with systemd 237), use absolute paths instead. If one command fails, the backup fails and won't start. |

# Dependencies
REST-passwords of clients are rolled out to the REST server as part of the `restic_server` role. It's sufficient to run the `add_rest_accounts` tasks file from the `restic_server` role (see below for example).

# Example Playbook
```
- hosts: backupclients
  gather_facts: false
  become: true

  roles:
  - role: restic_client
```

## Example REST Accounts Rollout
```
- hosts: backupservers
  gather_facts: false
  become: true

  tasks:
    - import_role:
        name: restic_server
        tasks_from: add_rest_accounts
```

## Example CLI usage of `restic` command
To use the `restic` command on the client's CLI, it is best to switch to the `restic` user, since it will have `RESTIC_REPOSITORY_FILE` and `RESTIC_PASSWORD_FILE` environment variables sourced automatically.

# Known Limitations
## Single backup server
Currently, in our infrastructure, there is only one (productive) backup server and restic itself doesn't know how to handle multiple backup targets.

Thus, the `backupservers` inventory group has only one member and this role (`restic_client`)  queries for the first member of that group via: `"{{ groups['backupservers'] | first }}"` to get the hostname of our backup server (in `roles/restic_client/defaults/main.yml` this is hidden behind `restic_client_backup_server_hostname: "{{ backup_server }}"`).

## Backup consistency
If LVM is not used on a node, there is no mechanism in place to ensure consistency for backups, so a backup might include files from two distinct application states.

# Footnotes
[^1]: Applies to clients with systemd version >= `247`. In systemd versions lower than `247` the corresponding option is ignored and a random point in time between 00:00 and 03:00 o'clock is picked on each run.

