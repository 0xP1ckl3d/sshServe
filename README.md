```text
              .__      _________                          
  ______ _____|  |__  /   _____/ ______________  __ ____  
 /  ___//  ___/  |  \ \_____  \_/ __ \_  __ \  \/ // __ \ 
 \___ \ \___ \|   Y  \/        \  ___/|  | \/\   /\  ___/ 
/____  >____  >___|  /_______  /\___  >__|    \_/  \___  >
     \/     \/     \/        \/     \/                 \/ 
```

## Overview

Ephemeral SSH/SFTP server for quick file sharing or a throwaway shell. Runs a jailed `sshd` with a temporary user, chroots to a minimal environment, and bind‑mounts your current directory to `/shared`. Credentials are random by default. Nothing persists except host keys, optional session keys, and shell history.

## Features

* SFTP‑only mode by default (no command execution)
* Optional SSH shell (`--allow-ssh`) in a chroot
* Password or temporary SSH key authentication
* Files uploaded are owned by your real user (matching UID/GID)
* Persistent host keys to avoid client warnings across runs
* Safe cleanup on Ctrl+C

## Requirements

* Linux with OpenSSH server binary (`sshd`) and `sftp-server`
* `sudo` privileges
* Tools: `mount`, `cp`, `ldd`, `openssl` or `perl`, `ip`

Default directories used:

* `/var/lib/sshserve/hostkeys` — persistent host keys
* `/var/lib/sshserve/keys` — generated session keys (when `--use-key`)
* `/var/lib/sshserve/history` — bash history for chroot shells

## Quick Start

```bash
# Default: SFTP‑only on port 2222, random user/pass
sudo ./sshserve.sh
```

Client examples:

```bash
# SFTP
sftp -P 2222 <user>@<server>

# SCP download
scp -P 2222 <user>@<server>:/shared/file.txt .
```

## Options

```
-P <port>        Port to listen on (default 2222)
-u <username>    Username to create (default random)
-p <password>    Password (default random)
-i <ip>          Listen IP (default 0.0.0.0)
--use-key        Use a temporary SSH key instead of password
--allow-ssh      Enable shell access in chroot (SCP, SFTP, SSH)
-h               Show usage
```

## Security Modes

**SFTP‑only (default)**

* File transfer only via internal‑sftp
* No command execution

**SSH enabled (`--allow-ssh`)**

* Allows SFTP, SCP, and shell
* Commands run inside the chroot
* History saved at `/var/lib/sshserve/history/.bash_history`

## Authentication

**Password (default)**

* Random if `-p` not provided

**SSH key (`--use-key`)**

* Generates an Ed25519 keypair under `/var/lib/sshserve/keys/`
* Publishes public key to `/shared/.ssh/authorized_keys`

## Examples

```bash
# Custom credentials and port
sudo ./sshserve.sh -u admin -p secret123 -P 2200

# Key‑based auth, SFTP only
sudo ./sshserve.sh --use-key

# Key‑based auth with shell access
sudo ./sshserve.sh --allow-ssh --use-key -P 2200 -i 0.0.0.0
```

Client examples:

```bash
# With key
sftp -P 2200 -i /var/lib/sshserve/keys/<keyfile> <user>@<server>
scp  -P 2200 -i /var/lib/sshserve/keys/<keyfile> <user>@<server>:/shared/file.txt .

# With password
sftp -P 2222 <user>@<server>
ssh  -p 2222 <user>@<server>   # if --allow-ssh used
```

## How It Works

* Re‑execs via sudo and stores original user’s UID/GID
* Creates a temporary user with that UID/GID to map ownership
* Builds minimal chroot with required binaries/libs
* Bind‑mounts current directory as `/shared`
* Starts `sshd` with `AllowUsers` restricted to temp user
* Cleans up on exit

## Persistence and Data Retention

* Host keys: `/var/lib/sshserve/hostkeys/` (kept)
* Session keys: `/var/lib/sshserve/keys/` (if `--use-key`)
* Shell history: `/var/lib/sshserve/history/.bash_history`
* Everything else removed during cleanup

## Logging

* `sshd` log shown at runtime and stored under the session directory
* Live activity events printed (connections, auth, open/close sessions)

## Troubleshooting

* “Permission denied (publickey)” — ensure correct key path and perms
* Port conflict — specify another port with `-P`
* Username exists — choose another or omit `-u`
* If `sshd` fails, check printed log path

## Safe Usage Notes

* Use SFTP‑only on untrusted networks
* Chroot reduces risk but is not a full container
* SELinux/AppArmor may restrict mounts or chroot usage

## TODO
* Fix issue with `--use-key` which currently doesnt work due to misplaced public key file
* make temporary keys accessible to the user who started the server

## Remove Persistent Data

```bash
sudo rm -rf /var/lib/sshserve/hostkeys /var/lib/sshserve/keys /var/lib/sshserve/history
```
