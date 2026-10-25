# Home server management (from the X230)

This laptop (ThinkPad X230, folder `~/x230set`) is used to manage the home network.
Notes below were carried over from Claude Code sessions on the main laptop (Legion Pro 5); dates are when they were last verified.

## Network map (LAN 192.168.10.0/24)

| Host | Address | What | Access |
|---|---|---|---|
| Proxmox host **pve** | 192.168.10.15 | Hypervisor; holds `/mnt/ssd/downloads` and `/mnt/usb-1` media disk | `ssh root@192.168.10.15` (key auth — copy a key from the X230 first) |
| qBittorrent LXC (CT 110) | 192.168.10.104 | gluetun + qBittorrent docker stack, WebUI `:8081` | `ssh` as `sauce3`, or from pve: `pct exec 110 -- bash` |
| Home Assistant (HAOS) | 192.168.10.159:8123 | Home automation, user `mabod` | Web UI / REST + websocket API (no SSH). Files edited via Studio Code Server add-on |
| LinkPlay speaker | 192.168.10.182 | Flaky speaker | — |

## qBittorrent + gluetun stack (LXC 192.168.10.104)

- Compose file: `/home/sauce3/docker/torrent-windscribe/docker-compose.yml`
  - `gluetun-ws` (Windscribe WireGuard VPN) + `qbittorrent-ws` using `network_mode: service:gluetun-ws`.
  - Backups: `docker-compose.yml.bak-20261011`, `docker-compose.yml.bak-20261012`.
- 2026-10-11: a Docker 29.9 upgrade restarted dockerd; gluetun had no restart policy, stayed down, and qBit couldn't join its netns. Fixed with `restart: always` on `gluetun-ws`.
  **If the WebUI is down, check `docker ps -a` for gluetun first.**
- **Port forwarding (Windscribe ephemeral port):** currently **10135** (set 2026-10-12), configured in two places:
  `FIREWALL_VPN_INPUT_PORTS` in the compose file **and** qBit `Session\Port`.
  - Ephemeral ports **expire after 7 days** → renew in Windscribe, update both places, `docker compose down && docker compose up -d` (new port only applies after the VPN reconnects), then verify.
  - **Reliable check** (run inside the LXC):
    `docker run --rm --network container:gluetun-ws curlimages/curl -s https://ifconfig.co/port/<port>` → expect `reachable: true`.
  - Do NOT test from a laptop on the home ISP with `/dev/tcp` — it gives false "connection refused".
- Static-IP option tested and rejected (2026-10-12): Windscribe static IP Toronto 82.24.182.102 with static forward 36115 works but is OpenVPN-only and much slower (23–241 Mbps down vs 430–690 on WireGuard, gigabit line baseline ~940/920). The `.ovpn` is `Windscribe-StaticIP.ovpn` on the main laptop (`~/plexset`).

## Media library — copying finished torrents (on pve)

- `/mnt/ssd/downloads/` (1.8T SSD) = qBit save path (`/downloads` inside the container). Flat, one entry per torrent.
- `/mnt/usb-1/` (7.3T USB): `series/`, `movies/` — also `music/`, `back/` (leave alone).

**Rules**
- **Always COPY, never move/delete/rename the source** — everything is still seeding (private trackers, e.g. oldtoons.world). No `--delete`, no `--remove-source-files`.
- Run rsync as root on pve with `-a` so ownership is kept: library is `1000:1000`, files 644, dirs 755. New dirs → `chown 1000:1000`. Exclude `.DS_Store` (user browses from a Mac via SMB).
- Always `--dry-run` first, then `rsync -a --info=progress2`.
- Don't copy torrents still downloading/checking. qBit preallocates full-size files, so check per-file progress via the qBit API, not size on disk.
- Check for existing copies by **file size** (`find … -printf "%s\t%p\n"`), not by name — library names often differ from download names.
- Don't bulk-sync "everything missing": some downloads were deliberately never copied (e.g. alternate-language duplicates). The user says which items, one at a time.

**Series:** `series/<Show.Name>/S0X/<episodes with ORIGINAL filenames>`
- Show folder dot-separated, normally no year/quality (`The.Expanse`, `Looney.Tunes`). If the folder exists, reuse it exactly as spelled even if odd (`Lego.Jurrasic.World`, French titles stay French).
- Season folder `S01`, `S02`… (some old shows use `Season 01`/`Saison 01` — follow what that show already uses).
- Season pack: copy the folder's *contents* into `S0X/` (rsync `src/` with trailing slash). Skip .nfo/samples/Screens. Don't rename episodes.
- Legacy flat folders (`Bluey`, `Mr.Robot.S01`, …) — don't copy that style.

**Movies:** flat — `movies/<video file>`, no per-movie folder.
- From a release folder copy only the main video file.
- Keep the release filename by default; if it's ugly (spaces/brackets/junk) ask — user sometimes uses `Title (Year).ext` or dotted form.
- Collections → individual files flat in `movies/`.

**Never copy:** `*.iso`, ebooks/PDFs/`*.epub`, `music`, `youtube`, `05`.

**Done as of 2026-10-18:** `series/Transformers.Prime/S01–S03`; movies `Sleeping.with.the.Enemy.1991…`, `The.Age.of.Adaline.2015…`.

## Home Assistant (192.168.10.159:8123)

- HA 2026.10.1 as of 2026-10-11. Access via REST/websocket API with a token from the login flow — **revoke tokens after use**.
- 2026-10-11 cleanup: Mushroom "Custom element doesn't exist" was stale browser cache (hard refresh). Removed unused HACS Bubble Card + mini-climate-card. Mushroom 5.2.3, kiosk-mode 14.2.1 (still throws an uncaught promise error on load — unresolved). Gree runs via the UI integration (legacy YAML removed).
- Cast/Apple TV/Brother/IPP are "ignored" discoveries, not failures.
- 4 unavailable automations (Master SAM…, HVAC off 21.8–22.5, Heat Pump Smart Control _2) are orphans — **user wants them KEPT**.
- Hue bulbs on dumb wall switches going offline is normal.
- Open items: "Chambre maître" Neviweb thermostat reports 0.0 °C (user to check Neviweb app); LinkPlay speakers spam `getMetaInfo` warnings (suggested `logger: logs: linkplay: error`); the .182 speaker is flaky.
