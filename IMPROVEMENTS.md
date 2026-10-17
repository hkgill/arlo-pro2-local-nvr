# Improvements and validation plan

Reviewed on 17 October 2026 against repository commit
[`554141b`](https://github.com/hkgill/arlo-pro2-local-nvr/commit/554141b9f6e03c4eb32888bb931fa6a0a43bd546).
This is an additive review of [README.md](README.md), [SPEC.md](docs/SPEC.md), and
[IMPLEMENTATION_PLAN.md](docs/IMPLEMENTATION_PLAN.md); those files are unchanged.
Resolve the differences identified here before turning their examples into deployed configuration.

## Assessment and evidence limits

**There is a credible foundation for local Pro 2 streaming. Reliable continuous recording with this exact stack remains unproven.**
Prove one camera before buying the full kit or retiring the original setup. Keep the original base and a documented way to return the test camera to it. A successful short VLC session or an HTTP success response does not establish a dependable NVR. There is no two-weekend delivery guarantee: the soak and retention gates below alone require longer observation.

The proposed system is two mains-powered Arlo Pro 2 VMC4030P cameras, a replacement base service, isolated Wi-Fi, a ROCK 4C/4C+, Coral USB, a 1 TB SSD, GStreamer → MediaMTX → Frigate, and MQTT/ntfy/Tailscale. This review checks documentation and source; it does not claim any pairing, hardware benchmark, recording test, outage test, or phone test has been performed locally.

Keep three kinds of evidence separate:

- **Verified source facts:** the original project names Pro 2/VMB4000r3; another integration reports Pro 2 support; the chosen fork contains the timed HLS lifecycle described below. These are source observations and upstream reports, not replication on these cameras.
- **Proposed design:** the continuous stream supervisor, container endpoints, isolation controls, rollback process, and acceptance thresholds in this document still need implementation.
- **Unknown local behavior:** camera firmware, timer precedence, fresh pairing, actual stream gaps, ROCK capacity, USB power stability, retention, and background phone notifications remain to be measured.

For the chosen fork, source links below are pinned to inspected commit
[`245defe`](https://github.com/frandallfarmer/arlo-open-base-station/commit/245defef43ce3dec70e1503a43513752860e873b).
Recheck the relevant paths when changing that version.

## Prioritized improvements

P0 items block the proof of concept or safe pairing. P1 items block acceptance as the primary recorder. The execution order is the gate sequence later in this document, with isolation established before pairing.

### 1. P0 — Build and prove continuous stream ownership

**Evidence and rationale.** [Meatballs1/arlo-cam-api][original] explicitly describes reverse engineering a VMB4000r3 with Arlo Pro 2 cameras. [brianschrameck/arlo-local][scrypted] independently reports tested Pro 2 support, but calls itself beta and describes stream, Wi-Fi, and recovery limitations. The selected fork's [README][fork-readme] documents testing the original Pro VMC4030, which is not the Pro 2 VMC4030P. Protocol ancestry makes it a reasonable candidate, not a verified continuous recorder for this installation.

The selected fork's current live-view path is bounded:

- [`api/api.py`][stream-api] calls `stream_manager.start(duration=60)` after status and stream-active requests.
- [`helpers/stream_manager.py`][stream-manager] spawns the helper, schedules cleanup at `duration + 10`, and returns success without waiting for a first frame. Its active check tests process liveness.
- [`helpers/gst_hls_stream.py`][hls-helper] produces HLS segments with `hlssink2` and emits EOS when the requested duration expires. It is not a ready-made RTSP publisher to MediaMTX.

This path provides neither continuous renewal nor recovery after camera re-registration. Simply adding `Restart=always` around a media process does not manage the camera's control state. The registration messages contain both `MaxStreamTimeLimit=1800` and `MaxUserStreamTimeLimit=86400` ([source][messages]). Their precedence, firmware enforcement, and renewal behavior are unproven. Do not describe 30 minutes as a guaranteed hardware cap or 24 hours as guaranteed continuous operation.

**Proposed implementation.** Give each camera one owner responsible for both protocol control and the camera-facing media connection. All Frigate, live-view, and diagnostic readers consume the relay. Disable the upstream on-demand HLS reader, motion-triggered recorders, and one-shot helpers that could compete for the same camera stream. Route their requests through the owner if retained.

Implement explicit states for registration, activation, first-frame wait, streaming, recovery, stopping, and failure. A re-registration must invalidate stale state, restore the chosen settings, and resume through the same owner. Add a real camera-to-MediaMTX publishing pipeline, validating video and optional audio separately; avoid video re-encoding unless measurements require it.

Watch advancing media timestamps and newly playable recorder segments, not only a PID, open RTSP socket, API result, or unchanged playlist. Proposed starting limits are a 20-second first-frame deadline and a 10-second media-progress timeout; tune them against actual GOP and segment lengths before acceptance. Drain child output, handle EOS/errors, and expose readiness only after media arrives. Use a bounded recovery budget, for example three attempts with 5/10/20-second backoff, followed by a visible failed state and alert. Handle a new registration as a fresh recovery opportunity without creating parallel owners. Do not automate camera power cycling or endless restart loops as a substitute for understanding failure.

On stop or restart, attempt RTSP teardown and stream deactivation, terminate and reap children, cancel timers, release locks, and clear stale stream state. Collect per-camera first-frame time, last-progress age, registration count, restart count/reason, session age, and recording-gap start/end/duration. Use anonymous camera labels in public reports.

**Acceptance.** The 48-hour one-camera gate must show advancing, playable media and logged automatic recovery. Test across 1,800 seconds and 86,400 seconds with session-age logs. Include a controlled attempt to keep a session alive beyond each boundary; frequent restarts must not conceal timer behavior. If renewal is needed, establish what actually renews the timer and measure the resulting gaps. Inject a live-but-stalled media process and re-registration; verify detection within the configured timeout, one owner, no orphan children, and recovery within the declared budget. A proposed recovery target is playable recording within 60 seconds after the camera and AP are ready; this is a target, not a measured result.

### 2. P0 — Separate fresh provisioning from the steady-state AP

**Evidence and rationale.** [TP-Link support][tp-link-wps] says Omada EAP does not support WPS. The fork's [Wi-Fi hardware report][wifi-hardware] describes cameras moving from a hostapd adapter to an EAP225; it is evidence about an existing network's operation, not proof of fresh WPS enrollment through an EAP225. Reports of sleeping battery cameras also do not prove mains-powered continuous-stream behavior.

**Proposed implementation.** Specify a temporary AP or hostapd adapter with proven WPS push-button enrollment on the exact chipset, hardware revision, driver, OS, and firmware. The original project describes a TL-WN722N setup, but a product name alone is not a purchase recommendation: revisions may differ. Validate the candidate with one camera before buying a permanent AP or resetting both cameras. Provision on the isolated network, then, if desired, migrate to the stable AP using the same configured SSID/PSK and compatible radio settings. Keep the credentials private. Confirm the destination AP's exact revision/firmware and reconnect behavior; avoid uncontrolled simultaneous APs during migration. Disable WPS after provisioning.

The fork's [installation guide][installation] includes inconsistent prose about custom firmware capability. That line is not evidence that a camera firmware replacement is necessary. This design uses stock camera firmware and normal pairing; do not introduce flashing, extraction, or firmware modification as a prerequisite.

**Acceptance.** Document a reproducible fresh pairing of one test camera, registration, and first playable media with the exact provisioning hardware. Demonstrate migration to the selected AP, reconnection after AP/camera restart, and WPS disabled afterward. Demonstrate the rollback route to the original base before proceeding to the second camera. Any unverified provisioning device remains a purchase blocker.

### 3. P0 — Specify container endpoints and port ownership

**Evidence and rationale.** Under ordinary bridge networking, `127.0.0.1` inside Frigate refers to Frigate's own network namespace. It cannot reach a separate MediaMTX container at that address. [Docker's Compose networking documentation][docker-networking] explains service-name discovery and container versus host ports. The present examples need an explicit topology before use.

**Proposed implementation.** Put Frigate and MediaMTX on a private Docker network and use service DNS, for example `rtsp://mediamtx:8554/front` and `/back`. For a GStreamer publisher running on the host, define a separate host endpoint, such as a loopback-only mapping `127.0.0.1:18554` to MediaMTX container port `8554`, with the corresponding publish path and transport configuration. This is an illustrative mapping, not a tested Compose file. If the publisher is containerized on that network, it can use service DNS instead.

Inventory each listener, namespace, bind address, protocol, consumer, and authentication requirement. Resolve emulator/Frigate port `5000` and MediaMTX/Frigate go2rtc port `8554` collisions at the host publication layer. Keep internal-only ports unpublished. Restrict the authenticated management UI to its intended interfaces and authorized clients. Do not switch every service to host networking to make localhost examples work. Confirm MediaMTX publish permissions and any UDP media-port requirements for the selected transport.

**Acceptance.** From inside Frigate, resolve MediaMTX and receive both playable streams. From the host publisher, publish through the declared endpoint. Check the host listener inventory for collisions and verify camera-side and unauthorized clients cannot reach management or internal media services. Recreate MediaMTX and confirm clients reconnect using service DNS.

### 4. P0 — Establish fail-closed isolation before pairing

**Evidence and rationale.** These cameras retain unpatched firmware. Lockdown cannot wait until the last phase. The fork [binds Flask to `0.0.0.0`][api-bind]; stating that the API is localhost-only does not enforce it. Its [dnsmasq example][dnsmasq] provides `gateway.arlo` and uses `172.14.0.0/24`, which is outside the private `172.16.0.0/12` range.

**Proposed implementation.** Choose a non-overlapping RFC1918 camera subnet, such as `172.16.50.0/24`, after checking LAN, Docker, and VPN routes. Update every dependent DHCP, DNS, service, and firewall setting consistently. Never blindly copy upstream `172.14.0.0/24` values.

Apply a default-deny camera-origin policy before enabling pairing or camera connectivity. Allow only measured necessities on the intended host interface: DHCP, local DNS including `gateway.arlo`, required camera control/upload endpoints, and required media return traffic. Add local time service only if needed. Deny camera forwarding to WAN, home LAN, tailnet, container networks, and unrelated host services. Preserve the local `gateway.arlo` answer while denying upstream DNS forwarding; a test that all DNS must fail would break this requirement.

Separate camera-facing protocol/upload endpoints from management endpoints. Configure the management listener's actual bind address or separate listener/proxy, plus interface firewall restrictions; retain only the specific camera-accessible routes the protocol requires. Prevent broad Flask API exposure to the camera network or home LAN. Authentication and restricted administration must not depend on a descriptive comment in a config file.

Cover IPv6 explicitly, either with equivalent deny rules or disabled camera-interface IPv6. Check Docker-generated forwarding and published-port rules, Tailscale routing/exit-node behavior, and service startup order. A forwarding sysctl alone is insufficient evidence of isolation. Gate AP/DHCP and camera-facing services on successfully loaded firewall policy, including cold boot. Host updates or authorized remote access must not grant cameras a forwarding route.

**Acceptance.** Before pairing, use a test client on the camera segment to prove necessary local DHCP/DNS works and home-LAN/WAN access fails over IPv4 and IPv6. Verify only the allowlisted host ports are reachable. Repeat during WAN-disconnected cold boots and after Docker, Tailscale, and firewall/service restarts; verify startup cannot briefly expose a permissive route. Keep sanitized rules, counters, and results as evidence. These are future tests, not changes performed by this document.

### 5. P0 — Validate the media workaround on the actual firmware

**Evidence and rationale.** The [original project's streaming notes][original] report FFmpeg streams failing around six seconds while VLC/libVLC and live555 work. The chosen fork attributes this to exact five/ten-second RTCP intervals ([dependency notes][dependencies]). However, the inspected [FFmpeg receiver-report implementation][ffmpeg-rtcp] uses byte-count thresholds. That does not establish a universal fixed FFmpeg reporting interval or a universal failure across versions, transports, and firmware.

**Proposed implementation.** Treat GStreamer as a promising compatibility workaround. Pin FFmpeg, GStreamer, plugins, emulator commit, transport, and camera firmware in a private test manifest. Compare actual RTCP receiver reports, RTP progress, re-registration, teardown, and reconnect behavior on one camera. Keep any packet captures private. Use observed behavior to choose the relay; do not change camera firmware to fit a hypothesis.

Do not disable RTCP indiscriminately. The original notes and Scrypted limitations warn about streams left sending after missing teardown, sometimes requiring camera recovery. A workaround that streams until the first disconnect but then needs manual camera recovery does not satisfy automatic continuous recording.

**Acceptance.** Capture sanitized timing summaries explaining which version/transport combination survives, how it stops, and how it recovers. Confirm no stuck stream after client failure or clean stop, and account for timer-boundary behavior separately from RTCP behavior. A manually recovered camera failure must be recorded as a failed reliability case, not omitted from the gap log.

### 6. P1 — Validate dependencies and the exact compute/USB platform

**Evidence and rationale.** The fork's [requirements][requirements] pin Flask 1.1.2, Werkzeug 1.0.1, Jinja2 2.11.2, and other legacy packages. Its installer creates a virtual environment, while StreamManager launches a bare `python3` helper ([installer][installer], [service unit][service-unit], [helper launch][stream-manager]). System-installed `gi`/GStreamer bindings may not be visible in a default venv, and the helper interpreter may differ from the service interpreter. The pins are not a modern ARM64/Python compatibility guarantee.

**Proposed implementation.** Establish a supported Python/OS baseline, update legacy dependencies in a controlled compatibility branch, and validate protocol/API behavior before deployment. Record the exact executable used by both the service and helper; explicitly choose an interpreter strategy with working `gi`, required GStreamer elements, and codecs. Do not assume installing packages into one interpreter fixes another. Pin tested image versions and digests, upstream commits, and configuration schema; retain the previous working set and a tested rollback procedure instead of relying on moving `stable` tags.

Start the one-camera proof on existing Linux hardware if available. Before committing to the full kit, inventory the exact ROCK 4C/4C+ model, RAM, OS/kernel, cooling, USB topology, PSU, SSD enclosure, and Coral USB. Coral accelerates object inference, not full-stream video decoding ([Frigate CPU guidance][frigate-cpu], [decoding documentation][frigate-decode]). Lowering configured detection resolution or FPS can reduce downstream work but does not remove the cost of decoding the incoming full-resolution stream. Measure that stage separately; use a lower source stream or supported hardware decode only if verified on the exact platform.

**Acceptance.** Reproduce the service and helper environment from the pinned manifest, confirm imports/elements and first media, validate Frigate configuration, and demonstrate rollback. During the 72-hour two-camera run, log delivered FPS, recording progress, CPU with its measurement denominator, RAM/swap, inference latency/queue behavior, temperatures/throttling, SSD latency, and USB resets/disconnects. Include simultaneous night IR, busy-scene detection, live viewing, playback, and SSD writes. Proposed capacity targets are sustained host CPU below 80% of total capacity, no throttling/OOM/USB resets, and inference below 20 ms with no persistent backlog; these are acceptance targets to validate, not reported capabilities. A cooler or faster host may be required.

### 7. P1 — Size storage from measurements and make failures safe

**Evidence and rationale.** A combined 4 Mbit/s is `4,000,000 / 8 × 86,400 = 43.2 GB/day` in decimal units, or `604.8 GB` for 14 days. That excludes filesystem/container overhead, audio if not included in the bitrate, database/cache/snapshots, and event segments retained beyond the continuous window. A nominal 1 TB SSD is a candidate, not proof of 14-day continuous plus 30-day event retention.

**Proposed implementation.** Measure combined day/night and busy-scene bitrate, actual usable capacity, event duty cycle, and overlap between continuous and event retention. Account for shared segments according to the pinned Frigate version rather than double-counting all events or assuming they cost nothing. Reserve explicit free-space headroom; the existing under-90%-used target is a proposed minimum margin, not a guarantee against worst-case events.

Require the intended SSD filesystem to be mounted and writable before the recorder starts. Use mount dependencies such as `RequiresMountsFor` plus checks of mount identity and writeability, and handle loss of the mount while running. A plain directory check or Compose bind-directory creation must not allow fallback writes to the OS card. Bound temporary storage and surface degraded recording status.

**Acceptance.** Test missing, full, and read-only SSD states, mount loss, and an unclean power interruption on a backed-up test setup. Verify no fallback recording fills the OS card, failures are visible, existing recordings remain inspectable, and recording recovers after storage returns. Verify playback across interruptions and record any damaged final segment. At day 15, verify the approximately 14-day continuous window; at day 31, verify the approximately 30-day event window and expiry behavior, with actual disk headroom. Test restoring configuration/state and retrieving a known clip from backup.

### 8. P1 — Make alert delivery, privacy, and provenance explicit

**Evidence and rationale.** “No Arlo cloud recording” is compatible with intentional encrypted remote viewing and snapshot delivery. It does not mean video or snapshots never leave the home. Notification providers, phone push services, caches, attachments, and a remotely viewing phone can participate in delivery. A random ntfy topic name alone is not authentication. The [ntfy access-control documentation][ntfy-auth] provides authenticated users/tokens, topic permissions, and a deny-by-default policy.

The chosen fork's [LICENSE][fork-license] also explicitly says the inherited Meatballs1 code has no specified upstream license. The fork's MIT text does not settle rights to all inherited code. Avoid blanket legal assurances about running, fetching, redistributing, or publishing it.

**Proposed implementation.** Document the selected notification route, endpoints receiving content, encryption boundaries, attachment/cache retention, and whether snapshots leave the home. Use authenticated ntfy publishing/subscription with scoped permissions, protected attachment access, and default denial for anonymous access. Keep credentials outside tracked configuration. Define MQTT availability and notification failures independently of recording health. Preserve upstream provenance and obtain licensing clarification before redistributing inherited code; link to upstream rather than copying its code into this repository in the meantime. Recording/audio/privacy obligations depend on the deployment and applicable law; this technical plan supplies no legal guarantee.

**Acceptance.** On each intended phone/OS, test locked-screen/background delivery over mobile data, reconnect after loss of VPN/network, authenticated snapshot access, and remote live/playback. A proposed target is an actionable alert within 10 seconds; measure trigger-to-display latency and failures over repeated day/night trials. Anonymous or unauthorized topic and attachment access must fail. Disconnect WAN and cold-boot the local stack: local recording and detection must continue, while unavailable remote alerts are reported accurately. Do not delete the stock account or alter account settings as part of proving these tests.

## Revised gates and go/no-go decision

All thresholds below are proposed acceptance criteria. Record results and any agreed best-effort limits before evaluating a run; do not turn an untested target into an availability claim.

| Gate | Work and evidence required | Go/no-go condition |
| --- | --- | --- |
| G0 — Inventory, rollback, isolation, single-camera pairing | Record exact hardware/software privately, preserve wanted stock recordings and the stock fallback, document re-pairing rollback, prove the deny policy, then freshly pair only one camera on the isolated network. Start with existing Linux hardware where practical. | One camera registers and delivers playable media; isolation and rollback are demonstrated. No full-kit purchase or second-camera migration until this proof succeeds. |
| G1 — 48-hour single-camera stream | Run the proposed owner/publisher, retain timestamp/segment-progress and gap logs, test both timer boundaries, re-registration, clean stop, and live-but-stalled process recovery. | Continuous progress and playable output throughout normal operation, plus recovery within the declared budget after injected failures. No competing reader, hidden timeout, manual restart, or unexplained gap. |
| G2 — 72-hour two-camera Frigate/Coral run | Use the exact intended ROCK/RAM/OS/PSU/USB/SSD setup. Include day/night, busy scenes, inference, recording, viewing, and playback load; keep resource and gap logs. | Both streams, detection, and recordings meet the selected quality/reliability mode and resource targets. No persistent inference backlog, thermal throttling, storage fallback, or unexplained USB reset. |
| G3 — Failure and phone drills | WAN-disconnected cold boots; camera power/registration loss; AP outage; emulator, relay, and recorder failure; live-but-stalled process; missing Coral; missing/full/read-only disk; test power interruption and restore. Test locked-screen mobile-data alerts with WAN restored. | Each failure has visible health state and documented recovery. Camera/AP recovery target: recording within 60 seconds of readiness; host reboot target: within 5 minutes of local services/storage becoming available. Missing Coral must be visible, with recording preserved or an explicit failure recorded. No silent CPU fallback that overloads the host; isolation survives all startup orders. |
| G4 — Retention and restore | Observe at least 15 days for continuous retention and 31 days for event retention. Verify expiry, headroom, old/new clip playback, and a restore drill using backed-up configuration/state and a known clip. | The required windows and recovery are demonstrated with measured capacity. Only after all gates pass should retirement of the original base/setup be considered; any account changes require a separate deliberate decision. |

Choose the reliability mode explicitly:

- **Gapless target:** no unexplained missing recorded media intervals during normal operation, including any scheduled renewal. State the expected source frame cadence, timestamp uncertainty, and gap-detection resolution. Log deliberately injected outages separately and still include their lost media in the report. A finite successful soak is evidence for that run, not a guarantee of future zero loss.
- **Best-effort alternative:** before accepting results, write down allowed maximum gap duration, total lost time per camera per day, and recovery limit. Report observed gap count, worst gap, total missing time, test duration, and causes. If required renewal causes gaps, describe the system as best effort unless the gapless target is demonstrated. Do not silently weaken the goal after a failed run.

**Go:** the selected mode and every gate pass with reproducible evidence on the intended hardware. **No-go:** fresh pairing is unproven, isolation leaks, stream control needs manual rescue, capacity is insufficient, retention fails, or measured gaps exceed the chosen limits. Keep the stock fallback and investigate; do not retire it based on a short stream demo. No measured availability percentage is available from this documentation review.

## Known unknowns

- Exact camera firmware and its handling of stream-active commands, RTCP, re-registration, and both timer fields; which operation actually renews a session.
- Exact provisioning adapter/AP revision and firmware, fresh-pairing reliability, and migration behavior for these Pro 2 cameras.
- Exact ROCK model/RAM/OS and the sustainable decode, inference, thermal, USB-power, and storage envelope under two-camera load.
- Actual bitrate, event duty cycle, retention overlap, usable SSD capacity, and outage-related segment loss.
- Phone OS/background restrictions, chosen ntfy transport, snapshot exposure/retention, and measured notification latency.
- Permitted redistribution of inherited upstream code; a license label on a fork does not answer every provenance question.

## Implementation and acceptance checklist

- [ ] Preserve inventory, wanted recordings, and rollback instructions privately; verify return to the stock base.
- [ ] Prove fail-closed IPv4/IPv6 isolation and local `gateway.arlo` DNS before pairing.
- [ ] Verify exact WPS provisioning hardware; pair one camera and test AP migration without custom firmware.
- [ ] Implement one control/media owner, an actual MediaMTX publisher, frame/segment watchdogs, bounded recovery, and complete cleanup.
- [ ] Disable competing camera readers and verify session behavior across both timer boundaries.
- [ ] Specify Docker service/host endpoints, listener bindings, permissions, and port ownership.
- [ ] Validate modernized dependency/interpreter compatibility; pin the tested stack and exercise rollback.
- [ ] Pass the 48-hour one-camera and 72-hour two-camera gates with sanitized gap/resource logs.
- [ ] Prove SSD mount enforcement, failure visibility, and storage/Coral/network/power recovery.
- [ ] Test authenticated phone alerts and snapshot access; demonstrate local recording/detection without WAN.
- [ ] Complete 15-day continuous and 31-day event-retention checks and a restore drill.
- [ ] Record the selected gapless or best-effort limits and the measured go/no-go outcome before retiring the stock setup.
- [ ] Preserve provenance; clarify inherited-code licensing before redistribution. Publish no credentials, serial numbers, MAC addresses, packet captures, or footage.

## Sources

These sources support the observations above. Upstream experience reports do not substitute for local acceptance testing. External product documentation can change; recheck it against the versions selected for implementation.

- [Original Arlo Cam API README][original] — Pro 2/VMB4000r3 basis, provisioning approach, and media/RTCP limitations.
- [Arlo Local for Scrypted README][scrypted] — independent Pro 2 testing report, beta status, and recovery limitations.
- [Chosen fork README][fork-readme], [stream API][stream-api], [stream manager][stream-manager], [HLS helper][hls-helper], and [protocol messages][messages] — inspected model support and stream lifecycle.
- [Chosen fork Wi-Fi hardware report][wifi-hardware], [installation guide][installation], [dnsmasq example][dnsmasq], and [API bind][api-bind] — AP migration context, inconsistent prerequisites, addressing, and exposure.
- [TP-Link EAP225/WPS support reply][tp-link-wps] — Omada EAP lacks WPS; confirm exact current revision/firmware before purchase.
- [FFmpeg `rtpdec.c`][ffmpeg-rtcp] and [fork dependency notes][dependencies] — distinguish implementation from timing claims.
- [Docker Compose networking][docker-networking], [Frigate CPU guidance][frigate-cpu], [Frigate video decoding][frigate-decode], and [ntfy access control][ntfy-auth] — integration and validation references.
- [Fork requirements][requirements], [installer][installer], [service unit][service-unit], and [license notice][fork-license] — dependency/interpreter and provenance evidence.

[original]: https://github.com/Meatballs1/arlo-cam-api/blob/1e04c36d3a4afa780238d5e9d8fc0355f2f21373/README.md
[scrypted]: https://github.com/brianschrameck/arlo-local/blob/a8be4c6e8a91a6453c6584b652b2c0dfbf4538a0/README.md
[fork-readme]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/README.md
[stream-api]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/src/arlo-cam-api/api/api.py#L286-L330
[stream-manager]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/src/arlo-cam-api/helpers/stream_manager.py
[hls-helper]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/src/arlo-cam-api/helpers/gst_hls_stream.py
[messages]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/src/arlo-cam-api/arlo/messages.py#L614-L651
[wifi-hardware]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/docs/WIFI-HARDWARE.md
[installation]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/docs/INSTALLATION.md
[dnsmasq]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/config/dnsmasq.conf.example
[api-bind]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/src/arlo-cam-api/api/api.py#L382-L383
[tp-link-wps]: https://community.tp-link.com/en/business/forum/topic/248566
[dependencies]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/docs/DEPENDENCIES.md
[ffmpeg-rtcp]: https://ffmpeg.org/doxygen/trunk/rtpdec_8c_source.html
[docker-networking]: https://docs.docker.com/compose/how-tos/networking/
[frigate-cpu]: https://docs.frigate.video/troubleshooting/cpu/
[frigate-decode]: https://docs.frigate.video/configuration/hardware_acceleration_video/
[ntfy-auth]: https://docs.ntfy.sh/config/#access-control
[requirements]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/src/arlo-cam-api/requirements.txt
[installer]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/scripts/install.sh
[service-unit]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/systemd/arlo.service
[fork-license]: https://github.com/frandallfarmer/arlo-open-base-station/blob/245defef43ce3dec70e1503a43513752860e873b/LICENSE
