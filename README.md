# helium-profile-sync

Automatically sync [Helium browser](https://github.com/nicholasgasior/helium) profiles to AWS S3 using [rclone](https://rclone.org/) and systemd user timers.

Keep your bookmarks, extensions, settings, and other profile data backed up to S3 — or synced across multiple machines — without any manual effort.

## Features

- **Automatic background sync** via systemd user timer (configurable interval)
- **Three sync modes:** upload-only, download-only, or bidirectional (via `rclone bisync`)
- **Multi-profile support:** sync one or more Helium browser profiles
- **Safe restore:** backs up local data before overwriting from S3
- **Merge restore:** pull S3 data into local without deleting local files
- **Dry-run mode:** preview what would change before committing
- **Configurable exclude patterns:** skip Chromium cache/temp files by default
- **File locking:** prevents overlapping syncs
- **Interactive setup wizard** for easy first-time configuration
- **systemd hardening:** `PrivateTmp`, `NoNewPrivileges`, `ProtectSystem=strict`

## Prerequisites

- **Arch Linux** (or any systemd-based distro)
- **[rclone](https://archlinux.org/packages/extra/x86_64/rclone/)** — the sync engine
- **[aws-cli-v2](https://archlinux.org/packages/extra/x86_64/aws-cli-v2/)** — used by the setup wizard to detect profiles and validate buckets
- **[jq](https://archlinux.org/packages/extra/x86_64/jq/)** — JSON processing
- **bash** ≥ 4
- AWS credentials configured (`aws configure --profile <name>`)
- An existing S3 bucket

On Arch Linux:

```bash
sudo pacman -S --needed rclone aws-cli-v2 jq
```

## Installation

### From the AUR

```bash
# Using an AUR helper (e.g. yay, paru)
yay -S helium-profile-sync
```

### Build from GitHub manually

```bash
git clone https://github.com/stonespren/helium-profile-sync.git
cd helium-profile-sync
makepkg -si
```

for subsequent updates, pull the latest changes and run `makepkg -Ccfsi`.

### Install from GitHub manually without building a package

If you've cloned the repo directly from GitHub, you can install the files manually:

```bash
git clone https://github.com/stonespren/helium-profile-sync.git
cd helium-profile-sync

# Install the scripts
sudo install -Dm755 helium-profile-sync      /usr/bin/helium-profile-sync
sudo install -Dm755 helium-profile-sync-setup /usr/bin/helium-profile-sync-setup

# Install systemd user units
sudo install -Dm644 helium-profile-sync.service /usr/lib/systemd/user/helium-profile-sync.service
sudo install -Dm644 helium-profile-sync.timer   /usr/lib/systemd/user/helium-profile-sync.timer

# (Optional) Install the example config and license
sudo install -Dm644 helium-profile-sync.conf.example /usr/share/doc/helium-profile-sync/helium-profile-sync.conf.example
sudo install -Dm644 LICENSE /usr/share/licenses/helium-profile-sync/LICENSE
```

To **uninstall** a manual GitHub install:

```bash
sudo rm /usr/bin/helium-profile-sync \
       /usr/bin/helium-profile-sync-setup \
       /usr/lib/systemd/user/helium-profile-sync.service \
       /usr/lib/systemd/user/helium-profile-sync.timer
sudo rm -rf /usr/share/doc/helium-profile-sync \
            /usr/share/licenses/helium-profile-sync
```

### Development (symlink install)

For active development, symlink instead of copying so changes take effect immediately:

```bash
cd helium-profile-sync
chmod +x helium-profile-sync helium-profile-sync-setup

sudo ln -sf "$PWD/helium-profile-sync"       /usr/bin/helium-profile-sync
sudo ln -sf "$PWD/helium-profile-sync-setup" /usr/bin/helium-profile-sync-setup

mkdir -p ~/.config/systemd/user
ln -sf "$PWD/helium-profile-sync.service" ~/.config/systemd/user/
ln -sf "$PWD/helium-profile-sync.timer"   ~/.config/systemd/user/

systemctl --user daemon-reload
```

## Setup

Run the interactive setup wizard:

```bash
helium-profile-sync --setup
```

The wizard will:

1. Detect your AWS CLI profiles and prompt you to choose one
2. Auto-detect the AWS region from that profile
3. Prompt for an S3 bucket name and validate access
4. Scan `~/.config/net.imput.helium/` for existing Helium profiles
5. Ask for sync interval, direction, and exclude patterns
6. Write the config and rclone remote to `~/.config/helium-profile-sync/`
7. Create a systemd timer drop-in for your custom interval
8. Enable and start the timer

| Setting             | Default                    | Description                                             |
| ------------------- | -------------------------- | ------------------------------------------------------- |
| AWS profile name    | `default`                  | AWS CLI profile to use for credentials                  |
| AWS region          | Auto-detected from profile | S3 bucket region                                        |
| S3 bucket name      | _(required)_               | Must already exist                                      |
| S3 key prefix       | `helium-profiles`          | Subfolder inside the bucket                             |
| Profile directories | Auto-discovered            | Comma-separated folder names (e.g. `Default`)           |
| Sync interval       | `15` minutes               | How often the timer fires                               |
| Sync direction      | `upload-only`              | One of: `upload-only`, `download-only`, `bidirectional` |
| Exclude patterns    | Chromium cache/temp files  | Comma-separated rclone exclude globs                    |

## Usage

```
helium-profile-sync [OPTION]
```

### Flags

| Flag              | Description                                                                                                                                                                                    |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `--setup`         | Run the interactive setup wizard. Creates config, rclone remote, and enables the systemd timer.                                                                                                |
| `--sync-now`      | Trigger an immediate sync using the configured direction.                                                                                                                                      |
| `--dry-run`       | Perform a sync dry-run — shows what would be transferred without making changes.                                                                                                               |
| `--restore`       | Download profiles from S3, **overwriting** local data. Existing local profiles are backed up to `~/.config/helium-profile-sync/backups/<timestamp>/` first. Requires interactive confirmation. |
| `--restore-merge` | Download profiles from S3, **merging** into local. No local files are deleted; S3 versions win on conflicts (newer overwrites older).                                                          |
| `--register`      | Register synced profile folders in Helium's `Local State` file so the browser recognizes them.                                                                                                 |
| `--status`        | Show the current state of the systemd timer and service.                                                                                                                                       |
| `--logs`          | Show journal logs from the last 24 hours for the sync service.                                                                                                                                 |
| `--enable`        | Enable and start the systemd user timer.                                                                                                                                                       |
| `--disable`       | Disable and stop the systemd user timer.                                                                                                                                                       |
| `-h`, `--help`    | Show the help message.                                                                                                                                                                         |
| _(no arguments)_  | Run a sync immediately. This is what the systemd service calls.                                                                                                                                |

### Sync Directions

- **`upload-only`** (default) — Local profiles are synced **to** S3. S3 becomes a mirror of local. Files deleted locally are deleted from S3.
- **`download-only`** — S3 profiles are synced **to** local. Local becomes a mirror of S3. Files deleted on S3 are deleted locally.
- **`bidirectional`** — Uses `rclone bisync` to keep both sides in sync. The first run for each profile automatically uses `--resync` to establish the baseline. If a bisync fails, the state file is removed so the next run re-establishes with `--resync`.

### Restore vs Restore-Merge

|                      | `--restore`                    | `--restore-merge`                            |
| -------------------- | ------------------------------ | -------------------------------------------- |
| **Mechanism**        | `rclone sync` (S3 → local)     | `rclone copy` (S3 → local)                   |
| **Local-only files** | Deleted                        | Kept                                         |
| **Backup first**     | Yes (timestamped copy)         | No                                           |
| **Confirmation**     | Required (`yes`)               | Not required                                 |
| **Use case**         | Clean restore on a new machine | Pull in S3 changes without losing local data |

### Profile Registration (Local State)

Chromium-based browsers (including Helium) maintain a `Local State` JSON file at the root of the browser data directory. This file contains `profile.info_cache` — a registry of known profile folders. If a profile folder exists on disk but has no entry in `Local State`, **the browser won't show it in the profile picker**.

`--restore`, `--restore-merge`, and download-only / bidirectional syncs automatically register any new profile folders in `Local State` after syncing. You can also run it manually:

```bash
helium-profile-sync --register
```

The registration reads each profile's `Preferences` file to extract the display name and avatar, then writes a minimal entry into `profile.info_cache`. Profiles that are already registered are left untouched.

> **Note:** Close Helium before running `--register` (or any restore). The browser overwrites `Local State` on exit, which would discard your changes.

## Configuration

All configuration is stored under `~/.config/helium-profile-sync/`:

| File            | Description                                                     |
| --------------- | --------------------------------------------------------------- |
| `config`        | Sourced by the sync script. Contains all settings. `chmod 600`. |
| `rclone.conf`   | Auto-generated rclone remote (`helium-s3`). `chmod 600`.        |
| `bisync-state/` | State files for bidirectional sync tracking.                    |
| `backups/`      | Timestamped backups created by `--restore`.                     |

### Example Config

```bash
AWS_PROFILE="default"
AWS_REGION="us-east-1"
S3_BUCKET="my-helium-backups"
S3_PREFIX="helium-profiles"
HELIUM_DATA_DIR="/home/user/.config/net.imput.helium"
PROFILE_DIRS="Default,Profile 1"
SYNC_INTERVAL="15"
SYNC_DIRECTION="upload-only"
EXCLUDE_PATTERNS="Service Worker/**,GPUCache/**,Cache/**,Code Cache/**,DawnCache/**,*.lock,*.log,LOCK,Cookies-journal,databases/Databases.db-journal"
```

You can edit this file manually instead of re-running `--setup`. Changes take effect on the next sync.

> **Note:** If you change `SYNC_INTERVAL` manually, you also need to update the systemd timer drop-in at `~/.config/systemd/user/helium-profile-sync.timer.d/interval.conf` and run `systemctl --user daemon-reload`.

### Default Exclude Patterns

The following Chromium temp/cache directories and files are excluded by default:

- `Service Worker/**`
- `GPUCache/**`
- `Cache/**`
- `Code Cache/**`
- `DawnCache/**`
- `*.lock`, `*.log`, `LOCK`
- `Cookies-journal`
- `databases/Databases.db-journal`

## systemd Units

The package installs two systemd **user** units:

- **`helium-profile-sync.service`** — A `Type=oneshot` service that runs the sync script. Includes security hardening (`ProtectSystem=strict`, `NoNewPrivileges`, `PrivateTmp`).
- **`helium-profile-sync.timer`** — Starts the service 2 minutes after login, then every 15 minutes (customizable via drop-in).

Useful systemd commands:

```bash
# Check timer status
systemctl --user status helium-profile-sync.timer

# Check last sync result
systemctl --user status helium-profile-sync.service

# View logs
journalctl --user -u helium-profile-sync.service -f

# Manually trigger a sync via systemd
systemctl --user start helium-profile-sync.service
```

## S3 Bucket Structure

Synced profiles are stored under `s3://<bucket>/<prefix>/<profile-name>/`:

```
s3://my-helium-backups/
└── helium-profiles/
    ├── Default/
    │   ├── Bookmarks
    │   ├── Preferences
    │   ├── Extensions/
    │   └── ...
    └── Profile 1/
        ├── Bookmarks
        ├── Preferences
        └── ...
```

## Troubleshooting

**"Config not found" error**
Run `helium-profile-sync --setup` to create the initial configuration.

**"Another sync is already running"**
The script uses a lock file at `/tmp/helium-profile-sync-<uid>.lock` to prevent overlapping runs. If a previous sync crashed, you can remove the lock file manually.

**Bisync keeps re-running with `--resync`**
This happens when bisync fails — the state file is removed to force a clean resync next time. Check `helium-profile-sync --logs` for the underlying rclone error.

**Bucket access denied**
Ensure your AWS profile has `s3:ListBucket`, `s3:GetObject`, `s3:PutObject`, and `s3:DeleteObject` permissions on the target bucket.

**Restored profiles don't appear in Helium**
Helium tracks known profiles in `~/.config/net.imput.helium/Local State`. If you synced profile folders manually or the automatic registration was skipped, run `helium-profile-sync --register`. Make sure Helium is **closed** first — the browser writes `Local State` on exit and would overwrite your changes.

## License

[MIT](LICENSE)
