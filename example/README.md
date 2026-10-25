# NATS mTLS Failure PoC

## Purpose

This PoC demonstrates two distinct breakage patterns in the current library's implentation of mTLS.

## Breaking Scenarios

### Scenario 1: Basic mTLS
Currently the library does not attach the mTLS certificates provided in the `nats_tls_config_t` client fields to the backing TLS session. When mTLS is mandatory on the NATS server the client cannot connect regardless of other client authentication settings.

### Scenario 2: Failed TLS session initialization
During a `client->connect()` call where the backing TLS session fails during the upgrade process (i.e. if there are no mTLS certificates attached to the client session) the client session is not marked as connected and so the backing LWIP socket is never freed when `client->disconnect()` is called - this is a **critical resource starvation bug** in the library.

## Quick Start
1. Generate certs as normal for the TLS server example and ensure the `nats-tls` container is started from a network location reachable by your ESP32
2. Copy the CA, client cert, and client key to `./main/certs`
3. Update your SSID and password and `nats-tls` IP address in `main.cpp`, as well as uncommenting `init_external_antenna()` if using a C6 board with ext. antenna
4. Select your target in the IDF and then build, flash, and monitor the logs as the example progresses through to completion
5. Disable mTLS fixes in `main.cpp` by setting `#define ATTACH_MTLS_CERTS 0` and build, flash, monitor again - note that repeated client TLS connection failures do NOT mention failed socket allocation (`LWIP has no free TCP sockets available (ENOBUFS)`)
6. Disable TLS socket fixes in `main.cpp` by setting `#define RUN_FULL_DISCONNECT 0` and build, flash, monitor again - note that client TLS connection failures DO mention failed socket allocation (`LWIP has no free TCP sockets available (ENOBUFS)`)

## Example Logs

### Finished mTLS Session
Full 16/16 test run with mTLS enforced at the server and the client being compliant, with both fixes in place.

### Failure Scenario 1
Fixing mTLS certificates is skipped - resulting in endless `mbedtls_ssl_handshake failed` errors that appropriately recycle TLS sockets

### Failure Scenario 2
Fixing mTLS certificates is skipped and fixing TLS socket handling is skipped - resulting in a number of `mbedtls_ssl_handshake failed` errors equal to `CONFIG_LWIP_MAX_ACTIVE_TCP` and then the only value returned from `socket()` for the lifecycle of the ESP's stack pointer is `ENOBUFS`.