# CS-021 — Simulation Specification

## Goal

Demonstrate online/offline delivery, ack/read state, duplicate prevention, group fan-out, media traffic, presence, key inventory, device revocation, calls, and regional failover.

## Controls

| Key | Label | Type | Default | Min | Max |
|---|---|---|---:|---:|---:|
| active_devices | Active devices | Slider | 100000000 | 1000 | 1000000000 |
| messages_per_second | Messages/s | Slider | 92000 | 1 | 1000000 |
| average_group_size | Average group size | Slider | 20 | 2 | 10000 |
| offline_probability | Offline probability | Slider | 0.20 | 0 | 1 |
| duplicate_probability | Duplicate probability | Slider | 0.01 | 0 | 0.5 |
| media_share | Media share | Slider | 0.10 | 0 | 1 |
| queue_workers | Queue workers | Slider | 10000 | 1 | 100000 |
| ws_connections_per_gateway | WS connections/gateway | Slider | 100000 | 100 | 1000000 |
| random_seed | Seed | Integer | 21 | 0 | 2147483647 |

## Derived values

- Text delivery tasks = messages × average recipient-device fan-out.
- Offline tasks = delivery tasks × offline_probability.
- Message metadata daily volume = messages × 86,400 × record size.
- Media daily volume = messages × 86,400 × media_share × average_media_size.
- Group fan-out writes = message × member count × write distribution.
- Active gateways = active_devices / connections_per_gateway.
- Queue age = offered tasks / worker service capacity.

## Required scenarios

1. Send/accept/deliver/read one-to-one message.
2. Disconnect recipient and deliver after reconnect.
3. Deduplicate duplicate message IDs.
4. Fan-out write/read for small and large groups.
5. Exhaust and rotate one-time prekeys.
6. Revoke a device and stop synchronization.
7. Upload/download/decrypt media.
8. Typing/presence expiration.
9. Push without plaintext.
10. Gateway and regional failure.
11. Queue/database outage.
12. Direct versus TURN call.

## Acceptance checks

- Per-conversation sequence is deduplicated and monotonic.
- Server accepted, device delivered, and read are distinct.
- Offline ciphertext survives reconnection.
- Revoked device gets no new ciphertext.
- Push omits plaintext by default.
- Media bytes do not overload message database.
- Call signaling and media paths are distinct.
