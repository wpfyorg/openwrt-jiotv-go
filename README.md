# openwrt-jiotv-go

> [!IMPORTANT]
> **This project has been sunset and is no longer maintained.** Please move to the actively maintained Rust version: [wpfyorg/better-jiotv-go](https://github.com/wpfyorg/better-jiotv-go).

Run [JioTV Go](https://jiotv_go.rabil.me) as a proper OpenWrt service: procd-managed, configured through UCI, starts at boot, restarts if it crashes, and keeps your Jio login across reboots and sysupgrades.

![JioTV Go web UI running on an OpenWrt router](docs/web-ui.png)

## TL;DR

📺 Watch JioTV live channels in a browser or any IPTV app (TiviMate, OTT Navigator, VLC, Kodi), with the TV guide included.

- ⚡ One-command install over SSH, as a procd/UCI service
- 💾 Flash-friendly: logs and the TV guide live in RAM
- 📡 Runs on your main router or a dumb AP
- 🌍 Optional Cloudflare Tunnel to watch from anywhere, even outside India, with no open ports
- 🔑 Needs a Jio number (one-time OTP) and a router with an Indian IP. Your ISP doesn't have to be Jio.

```sh
git clone https://github.com/wpfyorg/openwrt-jiotv-go && cd openwrt-jiotv-go && ./deploy.sh root@<router-ip>
```

## Requirements

- OpenWrt with `wget` able to reach HTTPS (the default `uclient-fetch` + `ca-bundle` work)
- CPU: aarch64, armv5–7, x86_64, i386 or riscv64. MIPS has no upstream build.
- About 18 MB of flash for the binary, or none in RAM mode (see below)
- About 40 MB RAM while running
- An Indian IP on the router's WAN, because Jio geo-blocks playback. Viewers can be anywhere: through the tunnel, Jio only sees the router.

## Install

One-liner from your computer (macOS or Linux, needs `git` and `ssh`):

```sh
git clone https://github.com/wpfyorg/openwrt-jiotv-go && cd openwrt-jiotv-go && ./deploy.sh root@192.168.1.100
```

Replace `192.168.1.100` with the LAN IP of **the router that should run JioTV Go**. It doesn't have to be your main router: a dumb AP or any other OpenWrt device on the LAN works, and is often the better choice because it keeps the main router's flash and RAM free. Streams still go out through your main router's internet connection.

That copies the repo over SSH (tar, so Dropbear without sftp is fine), downloads the latest release for the router's CPU, installs the service and starts it. Re-run the same command to upgrade; your config and login are kept.

Options go after the target:

```sh
./deploy.sh root@192.168.1.100 -v v3.22.0   # pin a release
./deploy.sh root@192.168.1.100 -m ram       # force RAM mode
./deploy.sh root@192.168.1.100 -p 8080      # different port
./deploy.sh root@192.168.1.100 --uninstall          # remove, keep login
./deploy.sh root@192.168.1.100 --uninstall --purge  # remove everything
```

If the repo is already on the router, run `sh install.sh [options]` there instead.

### Flash vs RAM mode

`-m auto` (the default) picks flash when there is room for the binary plus 10 MB headroom, and RAM otherwise.

| Mode | Binary lives in | After a reboot |
|---|---|---|
| flash | `/usr/bin/jiotv_go` | starts immediately |
| ram | `/tmp/jiotv_go` | re-downloaded at boot (retries every 30 s until WAN is up), then starts |

In both modes, the login is stored on flash in `/etc/jiotv_go`. It is only a few hundred bytes.

## First login

Open `http://<router-ip>:5001` and sign in with your Jio number and OTP. You only do this once.

## Use

| What | URL |
|---|---|
| Web player | `http://<router-ip>:5001` |
| M3U playlist (VLC, TiviMate, OTT Navigator, Kodi…) | `http://<router-ip>:5001/playlist.m3u` |

## Configuration

Edit `/etc/config/jiotv_go`, then `service jiotv_go restart`:

| Option | Default | Meaning |
|---|---|---|
| `enabled` | `1` | start the service |
| `host` | `0.0.0.0` | listen address |
| `port` | `5001` | listen port |
| `path_prefix` | `/etc/jiotv_go` | login and settings (flash) |
| `log_path` | `/tmp/log/jiotv_go` | request log (RAM, because it logs every request) |
| `epg` | `0` | build the TV guide (`/epg.xml.gz`, rebuilt daily) |
| `epg_path` | `/tmp/jiotv_go-epg` | where the guide is kept (RAM; rebuilt after reboot). Empty = flash |
| `memory_limit` | `128MiB` | soft memory cap (`GOMEMLIMIT`); guide builds otherwise balloon past 200 MB |
| `drm` | `1` | DRM channels |
| `disable_logout` | `1` | hide the logout button so a viewer can't log you out |
| `title` | `JioTV Go` | page title |
| `bin`, `url` | set by installer | binary path and its download URL |

Logs: `cat /tmp/log/jiotv_go/jiotv_go.log` or `logread -e jiotv_go`.

## Watching away from home

JioTV Go has **no password**. Anyone who can reach it can stream on your Jio account, and Jio can block an account that streams from many places. Never port-forward it.

### Option A: Tailscale (only your own devices)

```sh
tailscale up --advertise-routes=192.168.1.0/24   # add --advertise-exit-node to also use home as your internet exit
```

Approve the route in the Tailscale admin console under **Machines → router → Edit route settings**. Then any device on your tailnet can open `http://<router-ip>:5001` from anywhere.

### Option B: Cloudflare Tunnel (share with someone who has no Tailscale)

Nothing is opened on the router; cloudflared dials out to Cloudflare. JioTV Go builds every stream link from the request's hostname, so the protection is a **secret hostname** like `tv-3a7c1e9f0b2d4c68.example.com`. There is no path or password to rewrite, so it works in every IPTV app, DRM channels included.

It stays secret because Cloudflare's wildcard certificate means the name never appears in public certificate logs, and Cloudflare DNS can't be listed. Treat the URL like a password: if it leaks, delete the hostname and make a new one.

1. Generate a name: `echo tv-$(openssl rand -hex 8)`
2. In [Cloudflare Zero Trust](https://one.dash.cloudflare.com) go to **Networks → Tunnels → Create a tunnel → Cloudflared**, name it, and copy the token: the long string after `--token` in the install command it shows.
3. Add a **Public hostname**: subdomain = the generated name, your domain, service `HTTP` → `localhost:5001`.
4. Connect the router:
   ```sh
   ./deploy.sh root@192.168.1.100 --tunnel <token>
   ```
5. Share `https://<name>.<domain>/playlist.m3u`.

Optional hardening: add a WAF custom rule that blocks the hostname for countries other than the viewer's.

Turn it off with `./deploy.sh root@192.168.1.100 --tunnel --off`.

> Cloudflare's free-plan terms discourage serving video through its network. Light personal use is usually fine, but heavy use risks the account being flagged. Each viewer also uses about 3–8 Mbps of your home upload.

## License

[GPL-3.0](LICENSE). JioTV Go itself is a separate project with its own license; this repo only installs and manages it.
