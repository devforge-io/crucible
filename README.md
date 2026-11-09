# Crucible

Crucible is a self-hosted multistream relay. OBS streams once to a Crucible server on a Linux machine, and the server forwards that stream to Twitch, Kick, YouTube and Rumble. Each destination runs on its own, so if one fails the others keep going.

You set Crucible up from a plugin inside OBS. Apart from the TLS certificate and the one-time install, you don't edit anything on the server.

This repository hosts the release binaries. Downloads are on the [Releases](https://github.com/devforge-io/crucible/releases) page.

> **Status: early development (v0.2.0).** The server installs as a systemd service and serves its control API over HTTPS, and the OBS plugin connects to it. Ingest, forwarding to platforms, pairing and account linking are not built yet.

- [Install the server](#install-the-server)
- [Server commands](#server-commands)
- [Configuration](#configuration)

## Install the server

You need a Linux machine (x86_64 or aarch64) with systemd, a DNS name pointing at it, and a free TCP port (8740 by default).

### 1. Install

```sh
curl -fsSL https://github.com/devforge-io/crucible/releases/latest/download/install.sh | sh
```

The installer:

1. Downloads the latest release from [devforge-io/crucible](https://github.com/devforge-io/crucible/releases) and checks it against `SHA256SUMS`.
2. Installs the binary to `/usr/local/bin/crucible`.
3. Runs `crucible init --service`, which writes `/etc/crucible/crucible.toml`, creates the `crucible` system user and `/var/lib/crucible`, and installs a systemd service that starts at boot.

It doesn't start the server yet, because the server needs a TLS certificate first.

Installer options, set as environment variables:

| Variable | Effect |
|---|---|
| `CRUCIBLE_VERSION=v0.2.0` | Install that release instead of the latest |
| `CRUCIBLE_NO_SERVICE=1` | Install the binary and config only, without the systemd service |

For example: `curl -fsSL …/install.sh | CRUCIBLE_VERSION=v0.2.0 sh`

### 2. Get a TLS certificate

The control API only serves plain HTTP on localhost. Anywhere else it needs a certificate. With [certbot](https://certbot.eff.org/) and port 80 open for the check:

```sh
sudo certbot certonly --standalone -d relay.example.com
```

Copy the certificate where Crucible reads it, readable by the `crucible` user. Put this in `/etc/letsencrypt/renewal-hooks/deploy/crucible.sh` so every renewal is copied too:

```sh
#!/bin/sh
d=/etc/letsencrypt/live/relay.example.com
install -o root -g crucible -m 644 "$d/fullchain.pem" /etc/crucible/tls.crt
install -o root -g crucible -m 640 "$d/privkey.pem"   /etc/crucible/tls.key
```

```sh
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/crucible.sh
sudo /etc/letsencrypt/renewal-hooks/deploy/crucible.sh    # run it once now
```

Crucible checks the certificate files every 30 seconds and picks up a renewal without a restart.

### 3. Open the port and start

```sh
sudo ufw allow 8740/tcp      # or your firewall's equivalent
sudo crucible start
crucible status
```

Check it from another machine:

```sh
curl https://relay.example.com:8740/api/v1/server-info
# {"name":"crucible","version":"0.2.0","api_version":1}
```

### 4. Connect OBS

In OBS, open **Docks → Crucible**, enter `https://relay.example.com:8740`, and click **Check connection**. The plugin is built from source for now; prebuilt downloads will be published here.

### Upgrade

Run the install command again. It replaces the binary, keeps `/etc/crucible` and `/var/lib/crucible`, refreshes the service and restarts it if it was running.

A server installed with v0.1.0 keeps working after the upgrade, on its old loopback-only settings. To move it to TLS:

```sh
sudo crucible stop
sudo crucible init --service     # writes /etc/crucible/crucible.toml and replaces the unit
sudo rm /etc/crucible/crucible.env
```

Then follow steps 2 and 3 above.

### Uninstall

```sh
sudo crucible service uninstall    # stops and removes the service; config, data and user stay
sudo rm /usr/local/bin/crucible
# Also remove the config, data and service user:
sudo rm -rf /etc/crucible /var/lib/crucible
sudo userdel crucible
```

## Server commands

| Command | What it does |
|---|---|
| `crucible init [--force] [--service]` | Write `/etc/crucible/crucible.toml` (TLS on `0.0.0.0:8740`) and print the next steps. `--service` also runs `service install`. |
| `crucible service install` | Create the `crucible` user and data directory, write the systemd unit and enable it at boot. Doesn't start it. |
| `crucible service uninstall` | Stop the service and remove the unit. The config, data and user stay. |
| `crucible service show` | Print the unit `install` would write. |
| `crucible start` | Start the server. |
| `crucible stop` | Stop it. |
| `crucible restart` | Restart it, for example after editing the config. |
| `crucible status` | Show whether it's running (with recent log lines under the service). |
| `crucible serve [--listen ADDR]` | Run in the foreground. This is what the service runs. |

Every command takes `-c/--config PATH` (or `CRUCIBLE_CONFIG`) to use another config file.

Once the service is installed, `start`, `stop`, `restart` and `status` go through systemd and need `sudo` (except `status`). Without the service, `start` runs the server in the background with a PID file and a log in the data directory.

Logs from the service: `journalctl -u crucible -f`

## Configuration

`/etc/crucible/crucible.toml` holds startup settings only. Destinations, linked accounts and stream keys will be configured from the OBS plugin and kept in the server's database.

| Key | Environment variable | Default | Meaning |
|---|---|---|---|
| `listen` | `CRUCIBLE_LISTEN` | `127.0.0.1:8740` | Address the OBS plugin connects to. Anything other than loopback needs TLS. |
| `tls_cert` | `CRUCIBLE_TLS_CERT` | unset | PEM certificate chain. Setting both `tls_cert` and `tls_key` turns on HTTPS. |
| `tls_key` | `CRUCIBLE_TLS_KEY` | unset | PEM private key. |
| `data_dir` | `CRUCIBLE_DATA_DIR` | `data` | Server state. `init` sets `/var/lib/crucible`. |
| `log_file` | `CRUCIBLE_LOG_FILE` | `<data_dir>/crucible.log` | Log for `crucible start` without the service. |

The defaults apply when there is no config file. `crucible init` writes a file with TLS on `0.0.0.0:8740`. An environment variable overrides the file, and unknown keys are an error. Restart after editing: `sudo crucible restart`.

To use port 443, set `listen = "0.0.0.0:443"`. The service is allowed to bind ports below 1024.

Set `RUST_LOG` (for example `RUST_LOG=debug`) to change the log level.

## Licence

Copyright (c) 2026 Devforge Pty Ltd. All rights reserved.

Crucible is proprietary software: see [LICENSE](LICENSE). The OBS plugin ([devforge-io/crucible-obs](https://github.com/devforge-io/crucible-obs)) is also copyright Devforge Pty Ltd, licensed GPL-2.0-or-later.
