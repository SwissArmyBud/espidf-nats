# Security Advisory: LWIP Socket Leak After Failed TLS Upgrade

| | |
|---|---|
| **Component** | espidf-nats (`include/espidf_nats/nats_client.h`) |
| **Affected versions** | v1.4.0 and earlier (verified on v1.4.0, ESP-IDF v6.0.1, ESP32-C6) |
| **Weakness classes** | CWE-772 (Missing Release of Resource after Effective Lifetime), CWE-404 (Improper Resource Shutdown) |
| **CVSS 3.1** | **5.9 Medium**: `CVSS:3.1/AV:N/AC:H/PR:N/UI:N/S:U/C:N/I:N/A:H` |
| **Operational severity** | **Critical** (unrecoverable at runtime and device-wide; see [Severity](#severity)) |
| **Tracking** | Issue: [docs/GITHUB_ISSUE.md](docs/GITHUB_ISSUE.md). Fix: [docs/PR_TLS_SOCKET.md](docs/PR_TLS_SOCKET.md) |

## Summary

When the TLS upgrade fails during `connect()`, `disconnect()` returns early without closing the socket. Each failed attempt permanently consumes one available `CONFIG_LWIP_MAX_ACTIVE_TCP` (default 16) LWIP socket. After that, `socket()` returns `ENOBUFS` for every caller in the firmware, not only NATS. Only a reboot recovers.

The library's own reconnect loop repeats the failing sequence automatically, so a server-side fault that lasts for 16 reconnection attempts permanently disables networking on the device. **No attacker interaction is required.**

## Scope

The confirmed trigger is a **server certificate host verification failure** with `tls_config.skip_cert_verification = false` (mbedTLS `X509 - Certificate verification failed`, flag `0x2700`). For example, the server certificate's SAN does not match the host or IP the client connects to, or the server presents a different certificate than expected after a failed rotation.

Other TLS error types are not covered by this advisory and have not been tested.

## Severity

### CVSS 3.1 base score: 5.9 (Medium)

| Metric | Value | Reason |
|---|---|---|
| Attack Vector | Network (N) | The trigger is delivered over the network, by the server or by a party that can influence the certificate it presents. |
| Attack Complexity | High (H) | Needs a condition outside the attacker's control: a certificate that fails host verification must be presented, and the client must have `skip_cert_verification = false`. Without a natural fault such as a bad rotation, the attacker needs an on-path position. |
| Privileges Required | None (N) | No credentials needed. |
| User Interaction | None (N) | The reconnect loop repeats the failure on its own. |
| Scope | Unchanged (U) | The impact stays within the firmware image. |
| Confidentiality / Integrity | None (N) | The flaw is a resource leak. |
| Availability | High (H) | Total, persistent loss of networking until reboot. |

Computed: Impact 3.6, Exploitability 2.2, base score 5.9. If you rate the trigger as easy (a routine certificate-rotation mistake on the server), `AC:L` gives 7.5 (High).

### Operational severity: Critical

The formula score undersells the real-world effect, so this advisory also rates it **Critical**:

- The failure is **unrecoverable at runtime**. Every retry consumes another socket.
- The blast radius is the **whole device**. Once LWIP is out of TCP sockets, OTA updates, MQTT, HTTP, and any other networked component fail too, which can leave an unattended field device **unreachable and unpatchable**.
- Triggering needs **no credentials** and no red interactions. Sixteen failed TLS upgrades in a row are enough, even inside an otherwise trusted and secure network.

## Technical Details

### Socket leak on failed TLS upgrade

The failure path is:

1. `connect()` opens a socket, receives INFO, and calls `upgrade_to_tls()` (STARTTLS flow).
2. The handshake fails host verification. `upgrade_to_tls()` frees the mbedTLS state and returns `false`.
3. The caller invokes `disconnect()`.
4. `disconnect()` begins with `if (!connected && (transport == NULL || !transport->is_connected())) return;`. The `connected` flag is only set after a successful INFO and CONNECT exchange, so on this path it is still `false`. The function returns **before reaching the `::close(sockfd)` call**.
5. The next `connect()` call opens a new socket and the old file descriptor is orphaned.

The socket's lifetime is tied to a protocol-level flag, even though the raw file descriptor has been open since the earlier `connect()`.

### Fix

Remove the early return so cleanup always runs, and gate only the `on_disconnect` callback and the state transitions on `connected`. See [docs/PR_TLS_SOCKET.md](docs/PR_TLS_SOCKET.md).

## Reproduction

The PoC in this directory exercises the defect against the Docker `nats-tls` server.

1. Generate certs as normal for the TLS server example and ensure the `nats-tls` container is started from a network location reachable by your ESP32.
2. Copy the CA, client cert, and client key to `./main/certs`. The example's `CMakeLists.txt` embeds them, so the build needs the files even though the leak does not depend on them.
3. Update your SSID and password and `nats-tls` IP address in `main.cpp`. Uncomment `init_external_antenna()` if using a C6 board with an external antenna.
4. Confirm `tls_config.skip_cert_verification` is `false`. In the PoC this holds when `#define ATTACH_MTLS_CERTS 0`, because the flag is only set to `true` inside that block.
5. Set `#define RUN_FULL_DISCONNECT 1` (fix on), then build, flash, and monitor. Every attempt fails host verification and the socket number stays the same.
6. Set `#define RUN_FULL_DISCONNECT 0` (current library behavior), then build, flash, and monitor again. The log reports `socket() failed: LWIP has no free TCP sockets available (ENOBUFS)` once the socket pool is exhausted.

`RUN_FULL_DISCONNECT` is a PoC toggle: `0` is the current library behavior, `1` is the fixed behavior.

## Evidence

All logs are in [logs/](logs/). Server address is redacted.

| Log | `skip_cert_verification` | Socket fix | Result |
|---|---|---|---|
| [mtls-break-socket-break.txt](logs/mtls-break-socket-break.txt) | `false` | off | **Vulnerable.** Host verification fails 16 times (equal to `CONFIG_LWIP_MAX_ACTIVE_TCP`). The TLS socket number climbs from 48 to 63. Attempts 17 to 20 fail in `socket()` with `ENOBUFS`. |
| [mtls-break-socket-fix.txt](logs/mtls-break-socket-fix.txt) | `false` | on | **Fixed.** Host verification keeps failing, and every attempt reuses socket 48, so sockets are recycled. No `ENOBUFS`. |
| [mtls-fix-socket-fix.txt](logs/mtls-fix-socket-fix.txt) | `true` | on | **Control.** The connection succeeds and 16/16 tests pass (`ALL TESTS PASSED`). |

## Impact

- Any ESP-IDF device using espidf-nats with TLS and the production-correct `skip_cert_verification = false` flag can exhaust its TCP sockets if the server certificate fails host verification, for example through a certificate/host mismatch after a failed certificate rotation on the server.
- With the default `CONFIG_LWIP_MAX_ACTIVE_TCP=16`, exhaustion takes 16 failed attempts, which is **about 16 seconds** at the PoC's 1s retry delay.
- Raising the LWIP socket limit **only delays exhaustion** by an equal number of additional reconnection attempts.
- The library's built-in reconnect logic repeats the failing sequence automatically, so **no red/attacker interaction is needed**.
- _Once LWIP is out of TCP sockets, **OTA updates, MQTT, HTTP, and any other networked components cannot allocate additional sockets until the device is reset** (for example by power-cycling it)._

## Remediation

Upgrade to a release containing the fix in [docs/PR_TLS_SOCKET.md](docs/PR_TLS_SOCKET.md). Until then:

- Make sure the server certificate matches the host or IP that clients connect to, and test certificate rotations before deploying them.
- Watch free socket counts in firmware with a `socket()` canary, and restart the device after repeated consecutive failures.

## Timeline

| Date | Event |
|---|---|
| 2026-10-25 | PoC and fix committed (`6391fc1`). Advisory, issue, and PR drafts written. |
| 2026-10-26 | Advisory narrowed to the host verification trigger and rescored. |
