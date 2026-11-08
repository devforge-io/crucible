# Crucible

Crucible is a self-hosted multistream relay. OBS streams once to your Crucible server, and the server forwards that stream to Twitch, Kick, YouTube and Rumble. Each destination runs on its own, so if one fails the others keep going.

You set up Crucible from the Crucible plugin for OBS. You don't need to edit config files on the server.

This repository hosts the release binaries. Downloads are on the [Releases](https://github.com/devforge-io/crucible/releases) page.

> **Status:** early development. The current server answers health and status checks, and the OBS plugin can connect to it. Ingest and forwarding to platforms are still being built.

## Install

You need Linux (x86_64 or aarch64) with systemd. Install or upgrade with:

```sh
curl -fsSL https://github.com/devforge-io/crucible/releases/latest/download/install.sh | sh
```

The installer:

- downloads the latest static binary and checks it against `SHA256SUMS`
- installs it to `/usr/local/bin/crucible`
- creates a `crucible` system user, `/etc/crucible/crucible.env` (startup settings only) and `/var/lib/crucible`
- installs and starts a hardened `crucible` systemd service

Options, set as environment variables before `sh`:

| Variable | Effect |
|---|---|
| `CRUCIBLE_VERSION=v0.1.0` | Install that release instead of the latest |
| `CRUCIBLE_NO_START=1` | Install without enabling or starting the service |

Running the installer again upgrades the server. It keeps your settings and data and restarts the service.

## After installing

```sh
systemctl status crucible     # service state
journalctl -u crucible        # logs
```

The server listens on `127.0.0.1:8740` by default. Everything else is set up from the Crucible plugin in OBS.

## Uninstall

```sh
sudo systemctl disable --now crucible
sudo rm /etc/systemd/system/crucible.service /usr/local/bin/crucible
sudo systemctl daemon-reload
# Also remove settings, data and the service user:
sudo rm -rf /etc/crucible /var/lib/crucible
sudo userdel crucible
```
