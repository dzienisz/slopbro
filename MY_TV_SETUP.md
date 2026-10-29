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

1. Enable SSH in Homebrew Channel: `Settings` → `Root Configuration` → `SSH`
   on, then select `System reboot`.
2. Generate a dedicated local `ed25519` key in `.secrets/` with permissions
   restricted to the local user.
3. Use the temporary local-only SSH access once to install the public key in
   `/home/root/.ssh/authorized_keys` on the TV.
4. Reboot the TV and verify key-only login.
5. Keep Telnet disabled and do not place private keys, pairing files, or host
   keys in this repository.

## Repository layout

- `slopbro.py` and `wwwroot/`: upstream SlopBro source used for the setup.
- `MY_TV_SETUP.md`: local ownership and operational notes for this TV.
- `.secrets/`: intentionally ignored storage for local private credentials.

## Recovery notes

Do not overwrite system partitions such as the kernel, root filesystem, or TV
service files. Treat a factory reset or firmware upgrade as a recovery action
only after checking current community guidance for this exact model and
firmware.
