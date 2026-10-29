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

EULA/ad-consent flags in `/var/luna/preferences/eula` were withdrawn from the
TV's own remote (Settings → User Agreements), not by editing the file over
SSH — that store is checksum-protected and a direct edit risks corrupting the
EULA flow. **Withdrawing every agreement also withdraws the mandatory ones**
(Terms of Service) and blocks the TV until at least that one is re-accepted;
the optional ones (marketing, ad display, interest-based ads, voice) stay
declined. Current state, confirmed via `cat /var/luna/preferences/eula`: only
`generalTermsAllowed`/`networkAllowed`/`chpAllowed` are `true`; every
`acr*`/`marketing*`/`voice*`/`*ads*` flag is `false`.

A first attempt at host-level ACR/ads/telemetry blocking (420 domains from
hugobatista/lg-tv-ad-block, applied via a hand-written
`/var/lib/webosbrew/init.d/` script) was superseded by installing Glasshouse
(below), which manages `/etc/hosts` itself and overwrites anything else
living there. The hand-written script was removed rather than left to fight
Glasshouse for the same file. If Glasshouse is ever removed, that approach is
still valid as a fallback — see git history for the script.

### `luna-send` gotcha

`luna-send` prints **zero bytes** of output — no error, no JSON, exit code
0 — when its stdout isn't a real TTY. Over SSH this means a plain
`ssh host 'luna-send ...'` silently "succeeds" and tells you nothing. Force
a pty: `ssh -tt host 'luna-send ...'` (and feed it `</dev/null` if run from
a non-interactive script/tool, since `-tt` still needs *a* stdin).

### busybox `run-parts` gotcha

`run-parts` (as webosbrew's `startup.sh` calls it on `/var/lib/webosbrew/init.d/`)
silently skips any filename containing a dot — so a script named
`50-foo.sh` never runs, with no error anywhere. Name init.d scripts without
an extension (`50-foo`). Glasshouse's own installer independently works
around the same thing (writes `.50-tvweb.new`, then renames to `50-tvweb`).

## Glasshouse dashboard (added 2026-10-29)

[Glasshouse](https://github.com/rorygallagher2024/glasshouse) — a community
dashboard/debloat/privacy server for rooted webOS TVs — is installed.

- **Why**: a maintained alternative to hand-rolled scripts for exactly the
  things this file was tracking manually — ad/ACR blocking (with the correct,
  firmware-version-dependent EULA flag name, which differs by model and is
  risky to guess at by hand), background-service debloat, OLED panel care,
  and live CPU/RAM/temperature monitoring.
- **Deployed** from a separate local clone (not vendored into this repo) via
  its own `server/deploy.sh 192.168.0.161`, over SSH using the existing root
  key (loaded into a scoped `ssh-agent` for that one command, since
  `deploy.sh` shells out to plain `ssh root@<tv-ip>` rather than our
  `.secrets/tv_ssh_config`).
- **No-auth risk, mitigated at deploy time**: Glasshouse's dashboard has no
  authentication by default and binds to `0.0.0.0:8080` — anyone on the LAN
  can reach every control. A random 48-char token was generated
  (`openssl rand -hex 24`) and pre-seeded into `server/config.192.168.0.161.json`
  *before* the first deploy, so the server came up with `auth: token` from
  its very first log line — never an unauthenticated window. The token lives
  in `.secrets/glasshouse_token.txt` (gitignored). Open
  `http://192.168.0.161:8080/?k=<token>` once; the browser remembers it after
  that.
- **Still true regardless of Glasshouse**: telnet stays off, SSH stays
  key-only. Per Glasshouse's own `docs/SECURITY.md`, an open root telnet is a
  far bigger exposure than anything its dashboard adds — already handled on
  this TV.
- Boot hook: `/var/lib/webosbrew/init.d/50-tvweb` (correctly named, no dot).
- Use the dashboard's own **Privacy** tab to turn on its "Ads & telemetry"
  blocking and confirm the ACR/Live Plus switch — do this from the dashboard,
  not by scripting its API blind, since the exact EULA flag it toggles is
  firmware-version-dependent (see its `docs/ACR.md`).

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
