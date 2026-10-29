# LG OLED55BX3LB — Homebrew setup

## Device

- Model: LG OLED55BX3LB
- webOS TV: 5.6.2-21
- Firmware: 04.64.00
- Root status: Homebrew Channel starts successfully.

## Root installation record

The TV was rooted from a Mac on the same trusted local network with SlopBro.
This checkout is pinned to commit `21023d56ca424607bb03d2939609cc658c655fc7`
of `throwaway96/slopbro`.

The successful invocation used the webOS 5 target profile and explicit local
network addresses:

```sh
python3 slopbro.py --debug --webos-version 5 --local-ip <MAC_LAN_IP> <TV_LAN_IP>
```

The actual IP addresses, the SSAP pairing key, and any SSH private keys are
deliberately excluded from version control.

## Network and security baseline

- Keep the TV on a trusted, non-guest local network.
- Do not expose the TV with router port forwarding or UPnP rules.
- Keep Telnet disabled in Homebrew Channel: `Settings` → `Root Configuration`
  → `Telnet` off, then restart the TV.
- Use SSH only with an `ed25519` public key. Do not retain a password-based
  remote login after key access is verified.
- Do not install LG Developer Mode on this rooted TV.
- Evaluate each LG firmware update before installing it. Updates can remove
  root persistence, while declining updates retains known security exposure.

## SSH plan

SSH was enabled through Homebrew Channel, then configured and verified after a
full TV restart.

- Authentication: dedicated `ed25519` key only.
- Fingerprint: `SHA256:Btp7WAKX6yWNpSYs6ypoGJjakhl/1Po2thcTLLSENgY`.
- Local helper configuration: `.secrets/tv_ssh_config`.

Use the helper configuration from this checkout:

```sh
ssh -F .secrets/tv_ssh_config lg-oled55bx
```

Keep Telnet disabled and do not place private keys, pairing files, or host keys
in this repository.

## Verified runtime state (checked 2026-10-29)

Confirmed live over SSH, directly on the TV, by inspecting
`/var/lib/webosbrew/startup.sh` (the webosbrew boot hook) and the flag files it
reads from `/var/luna/preferences/`:

- `webosbrew_telnet_disabled` is present → the startup script skips launching
  `telnetd`, and no telnet process is running. Matches the "Telnet off" setting
  above.
- `webosbrew_sshd_enabled` is present → the startup script starts `dropbear
  -R`. Key-only auth confirmed (no `root` password fallback, since
  `/home/root/.ssh/authorized_keys` already exists).
- `webosbrew_block_updates` is present → the startup script bind-mounts a
  patched `/etc/hosts` that resolves LG's update-check domains (`snu.lge.com`,
  `su.lge.com`, `su-dev.lge.com`, `su-ssl.lge.com`) to `127.0.0.1`/`::1`. This
  is why firmware auto-update hasn't silently removed root yet: the TV cannot
  reach LG's update servers. **This does not replace judgment on manual
  updates** — if the update check is later unblocked (flag removed, firmware
  override, LG changes domains), re-evaluate before accepting any update, per
  the baseline above.

Filesystem layout at the time of this check: `/` is a read-only squashfs
(`/dev/mmcblk0p27`); writable state lives on the ext4 partition mounted at
`/var` (`/dev/mmcblk0p57`, ~722 MB, ~27% used). Homebrew Channel itself is
installed at
`/media/developer/apps/usr/palm/applications/org.webosbrew.hbchannel`.

Also present under `/media/cryptofs/apps/usr/palm/applications/` (group 5000,
i.e. installed as developer-mode apps rather than from the LG Content Store):
`netflix`, `youtube.leanback.v4`, `amazon`, `com.apple.appletv`,
`com.disney.disneyplus-prod`, `com.wbd.stream`, `spotify-beehive`. Not
investigated further yet — revisit if their provenance or update channel
matters later.

## Privacy / telemetry hardening (added 2026-10-29)

On top of the update-domain block above, this TV also blocks LG's ACR
(Live Plus), ad-network, and ThinQ/IoT telemetry domains, independent of
whether the on-screen "Live Plus" toggle is on or off. This matters because
the ACR binary (process name `sdx`) was observed still running even with
Live Plus switched off in Settings — the toggle does not stop the process,
only (allegedly) its reporting.

- **Domain list**: 420 entries from the "safe" base layer of
  [hugobatista/lg-tv-ad-block](https://github.com/hugobatista/lg-tv-ad-block)
  (`list` file — ads, tracking, ACR/telemetry). The repo's separate
  `list-store` ("aggressive") layer is deliberately **not** used, since it
  can break LG Content Store app updates.
- **Stored on the TV** at `/var/lib/webosbrew/lg-privacy-hosts.txt`.
- **Applied by** `/var/lib/webosbrew/init.d/50-lg-privacy-hosts.sh`, which
  webosbrew's `startup.sh` runs on every boot via `run-parts`. It layers its
  entries on top of whatever `/etc/hosts` already is (preserving the
  update-block bind mount above) and is idempotent (marker-guarded, safe to
  re-run).
- **Verify it's active**: `getent hosts ads.lgtvcommon.com` (or
  `alphonso.tv`, `ad.lgsmartad.com`) should resolve to `0.0.0.0`; an
  unrelated domain like `www.google.com` should resolve normally.
- Not touched: the EULA/ad-consent flags in `/var/luna/preferences/eula`
  (`acrAdAllowed`, `adCookie`, `doNotSellMyPersonalInformation`, etc.).
  That store is checksum-protected and editing it directly over SSH risks
  corrupting the EULA flow. Change these from the TV's own remote instead
  (Settings → General → AI Service / Additional Settings → ads & privacy
  toggles — exact menu path varies by firmware).

### `luna-send` gotcha

`luna-send` prints **zero bytes** of output — no error, no JSON, exit code
0 — when its stdout isn't a real TTY. Over SSH this means a plain
`ssh host 'luna-send ...'` silently "succeeds" and tells you nothing. Force
a pty: `ssh -tt host 'luna-send ...'` (and feed it `</dev/null` if run from
a non-interactive script/tool, since `-tt` still needs *a* stdin).

## Installed Homebrew Channel apps

Installed via the Homebrew Channel's own Luna API, scripted over SSH:

```sh
ssh -tt -F .secrets/tv_ssh_config lg-oled55bx \
  "luna-send -n 1 -a webosbrew -f luna://org.webosbrew.hbchannel.service/install \
   '{\"ipkUrl\":\"<url>\",\"ipkHash\":\"<sha256>\"}'"
```

- **Jellyfin** (`org.jellyfin.webos`) v1.2.2 — media server client.
- **Moonlight** (`com.limelight.webos`) v1.6.36 — NVIDIA GameStream/Sunshine
  client.

Both installs were confirmed with
`luna-send ... luna://org.webosbrew.hbchannel.service/getAppInfo '{"id":"<app id>"}'`.

Deliberately skipped: a webosbrew ad-free YouTube client — not needed since
this account has YouTube Premium.

## Repository layout

- `slopbro.py` and `wwwroot/`: upstream SlopBro source used for the setup.
- `MY_TV_SETUP.md`: local ownership and operational notes for this TV.
- `.secrets/`: intentionally ignored storage for local private credentials.

## Recovery notes

Do not overwrite system partitions such as the kernel, root filesystem, or TV
service files. Treat a factory reset or firmware upgrade as a recovery action
only after checking current community guidance for this exact model and
firmware.
