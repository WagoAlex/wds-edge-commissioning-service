# wds-edge-commissioning-service <-> edge-wda-rest-api

How the two device-side services on a WAGO Edge Computer (752-9xxx, x86-64)
fit together to make the edge behave like a PFC/CC in WAGO Device Sphere (WDS).

This document is the integration contract. Each section covers one concern and
one concern only; together they cover the whole seam.

---

## 1. The two components

| | `wds-edge-commissioning-service` | `edge-wda-rest-api` |
|---|---|---|
| Faces | **North** - the WDS server | **South** - the device itself |
| Speaks | WAGO commissioning protocol (`/api/v1/commissioning/*`), the `wds-ca` + `wds-device-agent` contract | WDA JSON:API (`/wda/parameters/...`, `/wda/methods/.../runs`) |
| Direction | Outbound only, client-initiated | Inbound only, server, HTTPS + Basic auth |
| Owns | Device identity, RSA-4096 key + CSR, signed client cert, twin id, enrollment state, heartbeat | Device state: firmware slots, network, users, system info |
| Persists | `/etc/wds/{priv.key,signed.crt,wds-ca.state}` | Host config (systemd-networkd, RAUC slots) |
| Lifetime | One-shot onboarding, then a long-lived heartbeat | Long-lived service |
| Ships from | this repo | `WagoAlex/edge-computer-fw-update` (`MODE=server`) |

Boundary rule: **the commissioning service never touches the device directly,
and the WDA API never talks to the WDS server.** Everything that crosses that
line crosses through the three integration points in section 3.

---

## 2. Runtime topology

```
        WDS / Device Sphere server (1.3.1)
                    |
                    |  HTTPS  /api/v1/commissioning/*        (outbound, mTLS after enroll)
                    v
  +------------------------------------------------------------------+
  |  WAGO Edge Computer 752-9xxx (x86-64, Debian + RAUC A/B)          |
  |                                                                   |
  |   wds-edge-commissioning-service        edge-wda-rest-api         |
  |   (container, network_mode: host)  ---> (container, :443/:8080)   |
  |        |                     WDA JSON:API over localhost          |
  |        |                                       |                  |
  |        | identity files                        | host control     |
  |        v                                       v                  |
  |   /etc/wds/*                       rauc.service (D-Bus),          |
  |                                    systemd-networkd, /etc         |
  +------------------------------------------------------------------+
```

The Device Sphere UI reaches the WDA API **directly** too, once the device is
`Connected`: that is how `Configuration/Device Sphere/*` and `Network/*` are
read and written. The commissioning service is not in that path.

---

## 3. Integration points

Exactly three, and nothing else crosses the boundary.

### 3.1 Firmware update (WDS request -> WDA state machine)

The only point where the commissioning service *drives* the WDA API.

```
Device Sphere UI: Configuration/Device Sphere/Target Firmware = <ver>
   -> GET /api/v1/commissioning/setup?targetFirmware=<ver>   (mTLS, commissioning service)
   -> bundle verified (CMS + sha256), wupload.cfg workflow parsed
   -> firmware workflow dispatched to the WDA firmware-update state machine:

      POST /wda/methods/0-0-firmwareupdate-activate/runs
      POST /wda/methods/0-0-firmwareupdate-start/runs
      GET  /wda/parameters/0-0-firmwareupdate-status      (poll: 0..4)
      GET  /wda/parameters/0-0-firmwareupdate-progress
      GET  /wda/parameters/0-0-firmwareupdate-errorcause
      -> status 4 (Unconfirmed): RAUC wrote the inactive slot; reboot activates
```

No RAUC logic lives in the commissioning service. It owns *when* an update
happens; the WDA API owns *how*.

Two dispatch modes, same state machine underneath:

- `FW_UPDATE_IMAGE` one-shot container run (default, no REST hop) - `agent/fwupdate.py`
- WDA REST calls against a running `edge-wda-rest-api` - use when the API is
  already deployed in `MODE=server`

### 3.2 Device model (WDS reads/writes -> WDA parameters)

Device Sphere expects the PFC/CC parameter tree. `edge-wda-rest-api` serves it;
the commissioning service only guarantees the device reaches a state where the
server is allowed to call it.

| Device Sphere node | WDA parameter |
|---|---|
| `Configuration/Device Sphere/Target Firmware` | `0-0-firmwareupdate-*` (see 3.1) |
| `Network/Bridge/EthernetPort/*` | `0-0-networking-ethernetports[-N-current*]` |
| Hostname / domain | `0-0-networking-hostname-*`, `0-0-networking-domain-*` |
| Routing, DNS | `0-0-networking-routing-*`, `0-0-networking-dns-*` |
| Identity, version | `0-0-identity-*`, `0-0-version-*` |

This repo contributes one writable method the read-only surface lacked:

```
POST /wda/methods/0-0-networking-configure/runs
     {"port": 1, "ip": "192.168.2.17", "prefix": 24, "gateway": "192.168.2.1"}
```

It writes a systemd-networkd drop-in (live `ip` apply as fallback). Reads mirror
live kernel state, so a config written here is visible at
`0-0-networking-ethernetports-N-currentIpaddr` immediately.

### 3.3 Identity consistency (WDA parameters -> WDS enrollment payload)

The values presented at enrollment must be the values the WDA API reports, or
Device Sphere shows a device whose twin disagrees with its own API.

| Enrollment field (`deviceIdentInfos`) | Source of truth | WDA parameter it must match |
|---|---|---|
| `orderNumber` | `WDS_ORDER_NUMBER` | `0-0-identity-ordernumber` |
| `firmwareVersion` | `/etc/REVISIONS` | `0-0-version-firmwareversion` |
| `uii` | type label / `WDS_UII` | `0-0-identity-serialnumber` |
| `hostname` | `gethostname(2)` | `0-0-networking-hostname-currentname` |
| `macAdress` (sic) | `SIOCGIFHWADDR` | `0-0-networking-ethernetports-N-currentMacAddress` |

Set both sides from the same env vars: `WDS_ORDER_NUMBER` / `ORDER_NUMBER`,
`WDS_FIRMWARE_VERSION` / `FIRMWARE_VERSION`.

---

## 4. End-to-end sequence

```
1  discovery      GET   /api/v1                     -> {"name":"wds"}          [commissioning]
2  enroll         POST  /api/v1/commissioning/enroll -> 201 deviceTwinId       [commissioning]
3  operator accepts in the Device Sphere UI                                    [server]
4  poll           POST  /api/v1/commissioning/status -> Prepared + signed cert [commissioning]
5  bundle         GET   /api/v1/commissioning/setup  -> appload (mTLS)         [commissioning]
6  firmware       POST  /wda/methods/0-0-firmwareupdate-{activate,start}/runs  [WDA]
7  confirm        PATCH /api/v1/commissioning/confirm -> 200, twin Connected   [commissioning]
8  heartbeat      POST  /api/v1/... every 30s, answers delete-confirmation     [commissioning]
9  managed        Device Sphere reads/writes /wda/* directly                   [WDA]
```

Steps 1-5, 7, 8 are the commissioning service. Steps 6 and 9 are the WDA API.
Step 3 is the operator. Nothing else exists in this flow.

Two non-obvious requirements, both proven against a real WDS 1.3.1 server:

- **`confirm` must be `PATCH` with `Content-Type: application/json`.** `POST` or
  `text/json` returns HTTP 403. This matches WAGO's own `wds-device-agent`.
- **The server's accept resolves the device by item number.** A genuine edge item
  (`0752-9401`) currently returns `accept: Not Found`; a recognized controller
  item (`0750-8302`) succeeds. See section 7.

---

## 5. Deployment

Both services run as containers on the edge, `network_mode: host`.

```yaml
services:
  commissioning:
    image: wago-edge-commissioning:latest
    network_mode: host
    environment:
      MODE: both                       # onboard | wda | fwupdate | both
      WDS_SERVERS: https://wdsserver:443
      WDS_MODE: unsecure               # TOFU the server cert
      WDS_ORDER_NUMBER: "0750-8302"    # see section 7
      FW_UPDATE_IMAGE: wagoalex/wago-fw-update-edge-computer:bundle-latest
      FW_DRY_RUN: "false"
    volumes:
      - wds-identity:/etc/wds
      - /var/run/docker.sock:/var/run/docker.sock   # to launch the fw executor

  wda:
    image: wagoalex/wago-fw-update-edge-computer:bundle-latest
    restart: unless-stopped
    environment:
      MODE: server
      WDA_PASSWORD: "..."
      ORDER_NUMBER: "0752-9xxx"
      FIRMWARE_VERSION: "04.09.01(31)"
    ports: ["443:8443"]
    volumes:
      - /run/dbus/system_bus_socket:/run/dbus/system_bus_socket
      - /etc/rauc/keyring.pem:/etc/rauc/keyring.pem:ro
      - /docker/rauc-stage:/docker/rauc-stage

volumes: { wds-identity: {} }
```

Required of the host, once: a RAUC keyring that trusts the bundle signature, and
`loop` + `dm-verity` modules loaded. See `edge-computer-fw-update`.

---

## 6. Configuration ownership

Every setting belongs to exactly one side.

| Concern | Owner | Where |
|---|---|---|
| Server discovery, TLS mode, poll rate, timeout | commissioning | `/etc/wds/wds.cfg`, `WDS_*` env |
| Device key, CSR, signed cert, twin id, state | commissioning | `/etc/wds/` volume |
| Which firmware version to install | commissioning (from the server) | `?targetFirmware=` |
| How firmware is installed, slot selection, rollback | WDA API | RAUC over host D-Bus |
| Network interface config | WDA API | systemd-networkd drop-ins |
| API auth (admin/wago Basic) | WDA API | `WDA_PASSWORD` |
| Reported identity strings | both, must agree | section 3.3 |

---

## 7. Known limits

| Limit | Owner | Status |
|---|---|---|
| `commissioning/accept` returns 404 for genuine edge items `0752-9xxx`; succeeds for a known controller item (`0750-8302`) | WDS server (WAGO) | Workaround: set `WDS_ORDER_NUMBER=0750-8302`. Real fix is server-side catalog registration of the edge item, mapped to a Docker-mode profile. |
| No `amd64` build of WAGO's `wds-device-agent` (registry ships `ptxdist-*` arm/arm64 only) | WAGO registry | This repo substitutes an x86 `device_agent.py`; an amd64 image equivalent was built (`Dockerfile.device-agent`). |
| ARM `.ipk` / arm64 agent tarball in the setup bundle cannot be applied on x86 | protocol/bundle | Skipped deliberately; the x86 agent runs in its place. |
| Firmware activation requires a reboot | operator | RAUC writes the inactive slot; `reboot` activates, bad boot auto-falls-back. |
| WDA tree is partial (firmware, networking, identity, users, system) | WDA API | Full `Configuration/Device Sphere/*` round-trip is optional future work. |

---

## 8. Out of scope

Explicitly not part of this integration, so nobody looks for it here:

- RAUC bundle building and signing - `WagoAlex/edge-computer-fw-update`
- The WDS server itself, its catalog, and pairing mode - WAGO product
- CODESYS, BACnet, and the rest of the PFC application stack
- Any inbound path into the commissioning service; it has no listening port
