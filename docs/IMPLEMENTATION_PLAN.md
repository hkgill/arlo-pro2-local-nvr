# Implementation plan

The plan has six phases. Each one ends with a gate that must pass before moving on. The Arlo base station stays plugged in until Phase 2 passes, so a working fallback is always available. Expect about two weekends in total.

See [SPEC.md](SPEC.md) for the reasoning behind each choice.

## Phase 0: Prepare (1–2 hours)

- [ ] Note the exact ROCK model, RAM, Coral type and base station model
- [ ] Download any clips you want to keep from the Arlo app and USB drive
- [ ] Record each camera's MAC address and serial number (Arlo app, device info). Keep them private
- [ ] Buy the SSD, the access point or Wi-Fi adapter, and a USB Ethernet adapter if using option A
- [ ] Clone this repo onto the host. Keep real secrets in an untracked `.env`

**Gate:** all hardware on the desk; clips backed up.

## Phase 1: Host build (2–3 hours)

1. Flash Armbian minimal to eMMC/SD, boot, update, set hostname `rock4c`.
2. Create a user, install SSH keys, disable password login.
3. Install Docker and the Compose plugin.
4. Format and mount the SSD at `/srv/nvr` with an fstab entry that uses its UUID.
5. Plug in the Coral and confirm `lsusb` shows `Global Unichip Corp` (first run) or `Google Inc` (after initialisation).
6. Install Tailscale and log in.

**Gate:** the reboot test passes, the SSD mounts by itself, and the ROCK is reachable over Tailscale from the phone.

## Phase 2: Camera network and pairing (3–4 hours, highest risk)

1. Bring up the camera interface (`camnet`) with a static IP on the camera subnet (for example `172.16.50.1/24`, or whatever the emulator's setup script expects).
2. Configure the AP (option A) or hostapd (option B) with the SSID, WPA2 key and WPS push-button.
3. Configure dnsmasq for DHCP on `camnet` only, with fixed leases per camera MAC and no upstream DNS.
4. Install [arlo-open-base-station](https://github.com/frandallfarmer/arlo-open-base-station) following its INSTALLATION.md, skipping the VPS and bore tunnel steps.
5. Factory-reset camera 1 per Arlo's reset instructions. Then trigger WPS on the AP and press Sync on the camera.
6. Confirm the camera gets a DHCP lease and registers with the emulator (the emulator log shows it).
7. Open its stream in VLC from the ROCK and leave it running for 30 minutes.
8. Repeat for camera 2.
9. Unplug the Arlo base station.

**Gate:** both cameras are registered, both streams play in VLC for 30 minutes without dropping, and the base station is unplugged.

**If pairing fails:** try a maintained [arlo-cam-api](https://github.com/Meatballs1/arlo-cam-api) fork directly ([mmornati](https://github.com/mmornati/arlo-cam-api), [greiner-petter](https://github.com/greiner-petter/arlo-cam-api)), which was built against the Pro 2. Another option is the Scrypted Arlo Local plugin.

## Phase 3: Stream relay and Frigate (3–4 hours)

1. Deploy MediaMTX in Docker, bound to localhost.
2. Create one GStreamer systemd unit per camera that relays the camera's RTSP into MediaMTX without re-encoding, with `Restart=always` and `RestartSec=10`.
3. Deploy Frigate (`ghcr.io/blakeblackshear/frigate:stable`, arm64) with the Coral (`detectors: coral: type: edgetpu, device: usb`, and `/dev/bus/usb` passed through), pointed at `rtsp://127.0.0.1:8554/front` and `/back`.
4. Set detect to 5 fps at 720p (or 640×360), continuous recording to 14 days and events to 30 days. Set `shm_size` to 256 MB and leave hardware decode off.
5. Draw zones and masks for each camera.
6. Measure CPU load (`htop`), temperature and real bitrate. If CPU stays above 80%, lower the detect resolution.

**Gate:** 24 hours of unbroken recording on both cameras, Coral inference under 20 ms on Frigate's System page, and CPU below 80%.

## Phase 4: Alerts and remote access (1–2 hours)

1. Deploy Mosquitto and enable MQTT in Frigate.
2. Deploy ntfy (self-hosted) and frigate-notify, then subscribe the phone to the topic.
3. Limit alerts to `person` in chosen zones, and add a cooldown so one visit gives one alert.
4. Check the Frigate UI (port 8971, authenticated) and ntfy on the phone over mobile data via Tailscale.

**Gate:** walking in front of each camera sends a phone alert with a snapshot in under 10 seconds, with the phone away from home Wi-Fi.

## Phase 5: Lockdown (1–2 hours)

1. Apply the nftables rules (below).
2. Work through the hardening checklist.
3. Disable WPS on the camera AP.
4. Remove the devices from the Arlo account.

**Gate:** all tests in the test plan pass.

### Firewall rules (nftables)

`eth0` is the home LAN and `camnet` is the camera network.

1. **Input from camnet:** allow DHCP (UDP 67), the emulator's camera ports, and RTSP/RTP replies only. Drop and log everything else.
2. **Forward from camnet to anywhere:** drop and log. No NAT or masquerade on camnet.
3. **Forward from eth0 to camnet:** drop.
4. **Input from eth0:** SSH, plus Frigate 8971, from the LAN only.
5. **Input from tailscale0:** Frigate 8971, ntfy, SSH.
6. **Outbound from the host itself:** allowed (updates, Tailscale, NTP).
7. Keep `net.ipv4.ip_forward` at 0 unless Tailscale features need it, and disable IPv6 on camnet.

### Hardening checklist

- [ ] Unplug the Arlo base station and remove it from the router's DHCP list
- [ ] Remove the cameras and base station from the Arlo account
- [ ] Change the Arlo account password and enable 2FA, or close the account
- [ ] SSH key-only login, root login disabled, fail2ban optional
- [ ] unattended-upgrades enabled for security updates
- [ ] Frigate authentication on, with a strong admin password
- [ ] Emulator API, MediaMTX and MQTT bound to localhost
- [ ] WPS disabled on the camera AP after pairing
- [ ] Weekly check of firewall drop logs from camnet (any outbound attempt is a red flag)

## Phase 6: Publish configs (optional)

1. Strip secrets: Wi-Fi keys, ntfy topics, Tailscale keys, MAC addresses, serials.
2. Commit Compose files, systemd units, nftables rules and the Frigate config with placeholders.
3. Add an `.env.example`.

## Test plan

The system is accepted when all twelve tests pass. Re-run T5–T9 after any update to Frigate, the emulator or the OS.

| ID | Test | Method | Pass criteria |
| --- | --- | --- | --- |
| T1 | Pairing | Factory-reset a camera and pair it | Registers with the emulator within 2 minutes |
| T2 | Stream stability | Relay running, watch stream | No gaps over 24 hours in Frigate's recordings |
| T3 | Detection | Walk past each camera, drive a car past | Person and car events appear with correct labels |
| T4 | False positives | Review 48 hours of events | Under 5 false person alerts per day after masks |
| T5 | Alert latency | Walk into a zone, time it until the phone alert | Under 10 seconds |
| T6 | Remote access | Phone on mobile data, open Frigate + playback | Live and recorded video load |
| T7 | Power cut | Pull the ROCK's power plug, restore | Recording resumes within 5 minutes, no manual steps |
| T8 | Camera drop | Unplug a camera's power for 2 minutes | Stream back within 60 seconds of the camera returning |
| T9 | Egress blocked | From a device on camnet: `ping 1.1.1.1`, a DNS lookup, curl any site | All fail; drops logged |
| T10 | LAN isolation | From camnet, try to reach the home router and a LAN PC | All fail |
| T11 | No open ports | External port scan of the home IP | No new open ports |
| T12 | Retention | After 15 days, check the oldest continuous recording | About 14 days old, disk under 90% |

For T9 and T10, a laptop joined to the camera Wi-Fi works as the test device. Also run `tcpdump -i camnet` for an hour. Any attempt to reach beyond the host should appear as a drop in the firewall log.
