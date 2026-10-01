# AOD Tune agent releases

Signed releases of the **AOD Tune agent** (`tune-agent`), the read-only
collector for [AOD Tune](https://aodtune.com) by Admin on Demand, LLC. The
agent reads host and MySQL/MariaDB status and settings and sends them to AOD
Tune. It changes nothing on the server and never reads table data. Source is
not published here; this repository holds the release key and the signed
releases.

**Release key fingerprint (OpenPGP, RSA-4096, sign only):**

    1F32 6385 FC71 4E73 3AEF  04AE 0F31 3567 0B13 386A

The same fingerprint is shown in your AOD Tune dashboard under *Install
agent*. If they differ, stop and contact AOD.

## Install

As root on the server. Set `VER` to the version you want (the latest is at
the top of [Releases](https://github.com/aodsupport/tune-agent-releases/releases))
and `ARCH` to `amd64` (x86_64) or `arm64`. You need `curl`, `gnupg2` and
`tar`. Every step must succeed; the chain stops at the first failure.

```sh
FPR=1F326385FC714E733AEF04AE0F3135670B13386A
VER=0.1.0; ARCH=amd64
D=https://github.com/aodsupport/tune-agent-releases/releases/download/agent-v$VER
if command -v gpg2 >/dev/null 2>&1; then GPG=gpg2; elif command -v gpg >/dev/null 2>&1; then GPG=gpg; else echo "install gnupg2" >&2; false; fi &&
T=$(mktemp -d) && cd "$T" && export GNUPGHOME="$T/gpg" && mkdir -m 700 "$GNUPGHOME" &&
printf '%s\n' "$FPR" | grep -qx '[0-9A-F]\{40\}' &&
curl -fsSLO "$D/tune-agent-$VER-linux-$ARCH.tar.gz" -O "$D/SHA256SUMS" -O "$D/SHA256SUMS.asc" &&
curl -fsSL https://raw.githubusercontent.com/aodsupport/tune-agent-releases/main/aod-tune-release.asc | "$GPG" --batch --import &&
"$GPG" --batch --status-fd 3 --verify SHA256SUMS.asc SHA256SUMS 3>"$T/gpg.status" &&
grep -q "^\[GNUPG:\] VALIDSIG $FPR " "$T/gpg.status" &&
grep " tune-agent-$VER-linux-$ARCH.tar.gz\$" SHA256SUMS | sha256sum -c - &&
tar xzf "tune-agent-$VER-linux-$ARCH.tar.gz" && "./tune-agent-$VER/install.sh"
```

The installer puts the agent in `/usr/bin/tune-agent`, the systemd unit in
`/etc/systemd/system/tune-agent.service`, pins this release key in
`/etc/aod-tune-release` and installs `tune-agent-upgrade` for later
upgrades. It does not start anything yet. Keep the extracted directory if
you may want `uninstall.sh` later.

## Database user

The agent needs only the replica-status privilege; it reads server status
and variables.

MariaDB 10.5.9 or newer (socket authentication, no password, no config
file needed):

```sql
CREATE USER 'tune-agent'@'localhost' IDENTIFIED VIA unix_socket;
GRANT SLAVE MONITOR ON *.* TO 'tune-agent'@'localhost';
```

MySQL or Percona 8.x, and older MariaDB (password account over the local
socket):

```sql
CREATE USER 'aod_tune'@'localhost' IDENTIFIED BY '<a random 32-character password>';
GRANT REPLICATION CLIENT ON *.* TO 'aod_tune'@'localhost';
```

then put the credentials in `/etc/aod-tune/mysql.json` (owner root, group
tune-agent, mode 0640):

```json
{"user": "aod_tune", "password": "<the password>"}
```

## Enroll and start

Get a one-time enrollment token from AOD (it expires after one hour), then:

```sh
tune-agent enroll
```

paste the token and press Enter (never put the token on the command line),
and start the agent:

```sh
systemctl enable --now tune-agent
```

The server appears in your AOD Tune dashboard after its first report.

## Upgrade

Use the program installed on the server; it only accepts releases signed by
the key pinned at first install:

```sh
tune-agent-upgrade --help
tune-agent-upgrade X.Y.Z
```

## Remove

From the extracted release directory: `./uninstall.sh` (keeps
`/etc/aod-tune` and the pinned key) or `./uninstall.sh --purge`. Then ask
AOD to revoke the host, and drop the database user.

## Supported systems

- x86_64: AlmaLinux, Rocky Linux, RHEL and CloudLinux 8 to 10; Ubuntu 22.04
  and 24.04; Debian 12. systemd is required.
- Best effort: CentOS/CloudLinux 7 and arm64.
- Databases: MariaDB 10.5.9 and newer, MySQL and Percona 8.x; older MariaDB
  best effort.

## Releases

| Version | Date | Notes |
|---|---|---|
| 0.1.0 | 2026-10-01 | First release (early access). |

Security issues: open a ticket at https://my.aod.net/ (Admin on Demand support).
