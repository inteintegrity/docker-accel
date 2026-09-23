# docker-accel

Configure and use container registry acceleration on a device — one script,
three routes, no per-device setup drift.

When a new board is flashed, the Docker daemon has no proxy and no mirror, so
`docker pull` from `ghcr.io` falls back to the slow direct path. This script
detects which route actually works on the current network, applies it, and can
undo everything again.

| Route | How it works | Typical speed (measured from a reComputer on a CN LAN) | Requirement |
| --- | --- | --- | --- |
| `direct` | plain connection | 30–70 KB/s | — |
| `proxy` | Docker daemon goes through an HTTP proxy (`/etc/systemd/system/docker.service.d/http-proxy.conf`) | ~544 KB/s | the proxy host must be reachable on the LAN (e.g. Clash with `allow-lan`) |
| `mirror` | image references are rewritten to a registry mirror (`ghcr.nju.edu.cn`), then retagged back | ~260 KB/s | none |

Numbers are from real pulls of the Seeed GHCR images; the script re-measures on
the device so you see the numbers for your own network.

## Install

Copy the script to the device and make it executable:

```bash
scp docker-accel <user>@<board>:/tmp/
ssh <user>@<board> 'sudo install -m 0755 /tmp/docker-accel /usr/local/bin/docker-accel'
```

Or from a clone:

```bash
git clone https://github.com/Seeed-Projects/docker-accel.git
sudo install -m 0755 docker-accel/docker-accel /usr/local/bin/docker-accel
```

Requires `bash` and `curl` only (both ship with Raspberry Pi OS).

## Use

```bash
# 1. See what is fast on this network (measures direct, and the proxy if given)
docker-accel probe 192.168.3.114:6789

# 2a. Fastest: route the Docker daemon through the LAN proxy
sudo docker-accel proxy 192.168.3.114:6789
docker pull ghcr.io/seeed-projects/recomputer-hailo10h-cv/yolov8_pose:latest

# 2b. Or without any proxy: pull through the registry mirror
docker-accel mirror ghcr.nju.edu.cn
docker-accel pull ghcr.io/seeed-projects/recomputer-hailo10h-cv/yolov8_pose:latest
```

`docker-accel pull` chooses the route automatically from the last probe, and
falls back to the direct path if the chosen one fails. The image is always
retagged to the original name, so scripts and compose files keep working
without a mirror prefix.

Other commands:

```bash
docker-accel status        # current proxy/mirror config, last probe, last pull
docker-accel proxy off     # remove the daemon proxy
docker-accel off           # restore the previous docker configuration
docker-accel -n proxy ...  # --dry-run: print the plan, change nothing
```

## Safety

- Every file the script touches is copied to `/etc/docker-accel/backup/<timestamp>/`
  before it changes.
- If `systemctl restart docker` or `docker info` fails afterwards, the previous
  configuration is restored automatically.
- `--dry-run` works on any platform (including a laptop) and prints every action
  without executing it.
- Configuration lives on the device only; nothing is sent anywhere.

## Proxy side checklist (Windows/macOS host running Clash)

The proxy route needs the host side to accept LAN connections:

1. Turn on `allow_lan` (Clash for Windows: *Allow LAN*; mihomo API:
   `curl -X PATCH http://127.0.0.1:9097/configs -H "Authorization: Bearer <secret>" -d '{"allow-lan": true}'`).
2. Allow the proxy port through the firewall:
   `New-NetFirewallRule -DisplayName "Allow LAN proxy 6789" -Direction Inbound -Protocol TCP -LocalPort 6789 -Action Allow -Profile Any`.
3. Confirm the host IP did not change: `docker-accel probe <host>:6789` should
   report the proxy route as reachable.

## Notes

- `registry-mirrors` in `/etc/docker/daemon.json` only affects Docker Hub
  (`docker.io`); it does not accelerate `ghcr.io`. That is why the mirror route
  is implemented as a pull wrapper instead of a daemon setting.
- The mirror route only accelerates pulls performed through
  `docker-accel pull`. Plain `docker pull ghcr.io/...` still uses the direct
  path unless the proxy route is configured.
- The throughput probe downloads a fixed 4 MiB range from the Hailo Model Zoo
  S3 bucket as a reference file. It measures the route, not the registry, so use
  the `pull` timing line for the real end-to-end number.