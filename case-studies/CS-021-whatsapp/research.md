# CS-021 — Design WhatsApp

> **Content status:** Educational reference design. This case proposes a secure, globally scalable messaging architecture. It does not claim to describe WhatsApp's proprietary implementation. Product capabilities, workload figures, and private infrastructure choices are illustrative unless explicitly verified.
>
> **Source alignment:** The catalogue identifies this as Case 21 in Category B: Communication & Collaboration. The required structure follows the repository's 18-section case-study template.

## A. Basic information

| Field | Value |
|---|---|
| Case Study ID | CS-021 |
| System name | WhatsApp-like real-time messaging platform |
| Real-world reference | WhatsApp capabilities are product examples, not asserted internal architecture |
| Category | Communication & Collaboration |
| Category number | B |
| Case number | 21 |
| Difficulty | Intermediate |
| Short description | Design global one-to-one and group messaging with persistent connections, durable offline delivery, end-to-end encryption, media storage, presence, push notifications, multi-device synchronization, and calls. |
| Primary users | Mobile/web clients, end users, group administrators, operations, trust and safety |
| Key concepts | WebSocket, message sequencing, acknowledgments, offline queues, fan-out, key agreement, forward secrecy, media encryption, presence, WebRTC |
| Related component IDs | CMP-001 (Load Balancer) |
| Research status | Draft reference content |
| Researcher | To be assigned |
| Reviewer | To be assigned |
| Last updated | 2026-11-20 |

### Scope and assumptions

- The platform supports phone-number registration, one-to-one/group text and media, delivery/read status, typing/presence, push, multi-device synchronization, and voice/video calls.
- Messages are encrypted on the originating device. The backend routes ciphertext and should not require plaintext message content.
- Persistent connections improve real-time delivery. A queue/database stores messages for offline devices.
- Per-conversation ordering is designed; global total ordering is not assumed.
- A service that is delivered or accepted is not necessarily read. Status is tracked per device.
- Capacity figures are illustrative teaching assumptions.

---

## 01 — Story

A user opens a mobile chat and sends a short text to a friend. The client assigns a message ID, encrypts the payload, sends it over a persistent connection, and waits for server and recipient-device acknowledgments.

The messaging backend authenticates the device, validates conversation membership, assigns a per-conversation sequence, stores ciphertext durably, and routes the message to recipient devices. If the recipient is offline, the platform stores or references the ciphertext, pushes a privacy-preserving notification, and delivers it after reconnection.

### Actors

1. **Mobile/web client:** Encrypts, sends, receives, acknowledges, synchronizes, and displays messages.
2. **API gateway/load balancer:** Terminates HTTPS, authenticates REST traffic, and routes API calls.
3. **Connection gateway:** Maintains WebSocket/TCP sessions and device presence.
4. **Authentication/user service:** Registers identity and device sessions.
5. **Messaging service:** Validates and sequences encrypted messages.
6. **Routing service:** Resolves recipient devices and conversation membership.
7. **Message queue:** Buffers durable delivery work.
8. **Message database:** Stores conversation metadata and encrypted message records.
9. **Presence service:** Handles heartbeat, online/typing/last-seen signals.
10. **Media service/storage/CDN:** Separates media bytes from message metadata.
11. **Push service:** Alerts offline devices through APNs/FCM.
12. **Calling/signaling service:** Coordinates WebRTC and ICE/STUN/TURN.

### Core user journey

1. User registers and verifies a phone number.
2. Device obtains identity/session credentials and shares public key material.
3. Client synchronizes contacts and conversations.
4. Sender creates a message ID and encrypts content for recipient devices.
5. Client sends encrypted envelope over WebSocket/TCP.
6. Connection gateway authenticates device and forwards it.
7. Messaging service validates sender, conversation, membership, schema, size, and anti-abuse policy.
8. Service assigns a per-conversation sequence and stores the encrypted payload.
9. Queue routes ciphertext to recipient devices.
10. Online recipient gets it, acknowledges delivery, and later acknowledges read.
11. Offline recipient obtains it after reconnection; push alerts the device.
12. Multi-device synchronization publishes the necessary ciphertext or session state to authorized devices.

### Problem boundaries

This case designs registration, contacts, messaging, media, presence, calls, synchronization, and operations. It does not reproduce any named private infrastructure or implement a complete E2EE library, mobile OS, carrier network, or media codec.

---

## 02 — Requirements

### Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Register and authenticate phone-number/device identities | Must |
| FR-02 | Synchronize contacts without overexposing address books | Must |
| FR-03 | Send/receive one-to-one encrypted messages | Must |
| FR-04 | Create/manage groups and participants | Must |
| FR-05 | Deliver text/media with sent/accepted/delivered/read status | Must |
| FR-06 | Maintain per-conversation ordering and duplicate protection | Must |
| FR-07 | Deliver messages while recipient devices are offline | Must |
| FR-08 | Synchronize authorized messages across devices | Must |
| FR-09 | Provide typing, presence, and privacy controls | Should |
| FR-10 | Support replies, reactions, forwarding, deletion, and search | Should |
| FR-11 | Upload/process/deliver encrypted media through object storage/CDN | Must |
| FR-12 | Support push notifications without plaintext content | Must |
| FR-13 | Establish one-to-one/group voice/video calls | Should |
| FR-14 | Support blocking/reporting, spam/rate controls, and abuse review | Must |

### Non-functional requirements

| ID | Requirement | Initial target |
|---|---|---|
| NFR-01 | Availability | Illustrative 99.95% for messaging ingestion and 99.99% for existing connection tier where economically justified |
| NFR-02 | Message latency | Illustrative p95 < 1 second for online regional delivery |
| NFR-03 | Durability | Accepted messages survive service and single-AZ failures according to RPO |
| NFR-04 | Consistency | Per-conversation ordering and effective-once processing; global ordering not required |
| NFR-05 | Scale | Billions of conversations and millions of concurrent connections |
| NFR-06 | Security | E2EE, device keys, TLS transport, authorization, abuse and secret protections |
| NFR-07 | Bandwidth | Ciphertext/media efficient and suitable for mobile networks |
| NFR-08 | Battery | Heartbeats, batching, backoff, and push preserve battery |
| NFR-09 | Offline | Pending records survive reconnection and device restoration |
| NFR-10 | Privacy | Minimal server knowledge of relationship/content/presence, policy permitting |

### Out of scope

Carrier implementation, proprietary key-protocol internals, real-time codec internals, complete spam ML, and verified private WhatsApp infrastructure.

---

## 03 — Scale

### Illustrative workload assumptions

| Input | Assumption |
|---|---:|
| Registered users | 500 million |
| Daily active users | 200 million |
| Concurrent users | 100 million |
| Messages/user/day | 40 |
| Text messages/day | 8 billion |
| Peak/average factor | 5× |
| Average text payload | 1 KiB |
| Media-active share | 10% of messages |
| Average media | 500 KiB |
| Metadata record | 1 KiB |

Average text throughput:

- `8 billion / 86,400 ≈ 92,593 text messages/second`.
- 5× peak is ≈ `463,000` text messages/second.
- Average text body ingress at 1 KiB is ≈ `92.6 MiB/s` or approximately `740 Mbit/s`, before protocol, metadata, retries, fan-out, and acknowledgments.
- 10% media share creates ≈ `800 million media objects/day`; 500 KiB average is ≈ `381 TiB/day` before thumbnails, retries, retention, and replication.
- 100 million concurrent devices require connection gateway capacity, heartbeats, presence state, and reconnection bandwidth independent of message RPS.

Fan-out multiplies traffic. A 100-member group can create 100 recipient-device records or 100 encrypted deliveries depending on distribution strategy. Large groups require bounded fan-out and long-poll/push/sync design.

### Storage and bandwidth notes

- Message database growth: 8 billion records/day × metadata/replication factor × retention.
- 1 KiB metadata × 8 billion ≈ 7.45 TiB/day before indexes, replicas, ciphertext, and state.
- Media object storage scales with originals, derivatives, replicas, encryption overhead, and retention.
- Push events can be millions/second and must avoid exposing plaintext.
- Per-conversation partitioning is preferable to global sequence storage.

### Production sizing questions

- Registered versus active devices and multi-device ratio.
- Text/media ratio and message-size distribution.
- Fan-out distribution by group size.
- Delivery and RPO targets.
- Region/tenant data residency.
- Presence precision and retention.
- Search and retention requirements.

---

## 04 — Naive Architecture

### Simplest viable design

One application server maintains a TCP/WebSocket connection table, stores messages in a relational database, and forwards each message directly to recipient users.

**Components**
- Mobile client
- One connection server
- Message database
- Direct recipient lookup

**Flow**
1. Sender sends plaintext/message.
2. Server saves it.
3. Server forwards it to recipient.
4. Recipient acknowledges.

### Naive architecture diagram

See `architecture.png`.

### Why begin here?

This design explains connection, routing, and acknowledgment but cannot scale concurrent devices, survive server failure, deliver offline messages, isolate large groups, protect plaintext, or support media and multi-device synchronization.

---

## 05 — Problems

| Problem | Impact | Improvement |
|---|---|---|
| Server-side plaintext | Privacy trust risk | Encrypt on device; route ciphertext |
| Single connection host | Reconnection and failover issue | Distributed connection gateways |
| Direct push | Connection/server coupling | Queue, durable delivery, push fallback |
| Global ordering | Bottleneck and clock complexity | Per-conversation sequence |
| Offline loss | Messages unavailable | Durable queue/database and sync cursor |
| Group fan-out explosion | Queue/storage amplification | Fan-out strategy and membership index |
| Key/device changes | Undelivered/decrypt failure | Per-device prekeys, rekey, and recovery |
| Duplicate delivery | UI duplicates | Message ID deduplication |
| Presence hot keys | Redis/load pressure | Device-level TTL and coarse heartbeat |
| Media in message DB | Expensive queries/storage | Object storage/CDN separated by reference |
| Push plaintext | Privacy risk | Minimal notification content |
| Clock skew | Incorrect visible order | Server sequence and Lamport-style ordering |
| Hot conversation | Database/queue load | Partition and cache hot metadata |
| Multi-region partition | Delivery divergence | Regional routing and reconciliation |
| Account takeover | Access to sessions/messages | Device auth, key verification, revocation |

---

## 06 — Architecture Evolution

### V1 — Single connection server and relational database

A server authenticates and forwards messages. It becomes a state and capacity bottleneck.

### V2 — Distributed connection gateways

Gateways maintain WebSockets. Routing service locates recipient gateway/device sessions.

### V3 — Durable message queue and inbox

Messages are stored and queued before delivery. Offline recipients synchronize from durable state.

### V4 — Device-specific encryption and acknowledgments

Each authorized device receives ciphertext appropriate to its key/session. Delivery/read receipts become device-level.

### V5 — Media and CDN separation

Encrypted media is stored in object storage. Messages carry references and metadata. CDN delivers approved bytes.

### V6 — Multi-region presence, group, and calling

Regional gateways, queues, databases, and key/media services provide regional routing and failover.

### Proposed evolved architecture

1. Mobile/web client opens persistent WebSocket/TCP connection through CMP-001/L4/L7 routing.
2. Gateway authenticates device and registers it with routing/presence.
3. Sender encrypts payload and sends envelope with stable message ID.
4. Messaging service validates tenant/conversation/device and obtains server sequence.
5. Encrypted message and outbox are stored durably.
6. Router resolves recipient online devices and group membership.
7. Queue schedules one delivery task per device or a fan-out inbox update.
8. Connection gateway pushes ciphertext to online devices.
9. Recipient sends accepted/delivered/read acknowledgments.
10. Offline devices sync pending ciphertext after reconnection and receive privacy-preserving push.
11. Media is uploaded to object storage; message carries encrypted media reference.
12. Calling uses signaling and WebRTC; TURN relays protected media when direct paths fail.

### Real-time and offline flows

**Online**
1. Sender sends.
2. Gateway forwards.
3. Server accepts and sequences.
4. Recipient device receives.
5. Server records device delivery and read separately.

**Offline**
1. Routing finds no active session.
2. Queue writes pending device delivery.
3. Push service sends generic alert.
4. Device reconnects/syncs.
5. Server marks delivered only after device acknowledgment.

### Group distribution

**Fan-out on write**
- Store a per-recipient-device delivery record when message is sent.
- Benefits: fast recipient reads and predictable offline sync.
- Costs: storage and write amplification.

**Fan-out on read**
- Store one conversation message and resolve member devices when read.
- Benefits: lower initial write cost.
- Costs: read-time fan-out and possible stale membership.

Large groups use bounded fan-out, membership snapshots, digest/sync cursors, or hybrid delivery.

### Encryption architecture

- Identity, signed prekeys, and one-time prekeys are registered per device.
- X3DH-like agreement establishes a shared session under a documented protocol.
- Double-Ratchet-like state rotates keys per message/session to provide forward secrecy and post-compromise security.
- Sender encrypts individually for recipient devices as required by the selected protocol.
- Server does not decrypt message payload in the normal design.
- Transport TLS protects connection traffic; E2EE protects message content from service compromise.
- Group encryption uses sender keys or a continuously rekeyed group state. Membership changes revoke/rotate membership keys.
- Device verification and secure key storage remain client responsibilities.

### Message envelope example

```json
{
  "message_id": "msg_01J...",
  "conversation_id": "conv_01J...",
  "sender_device_id": "dev_a",
  "recipient_device_ids": ["dev_b"],
  "conversation_sequence_hint": 842,
  "message_type": "TEXT",
  "encrypted_payload": "base64-ciphertext",
  "created_at": "2026-11-20T13:14:15Z",
  "idempotency_key": "client-message-key"
}
```

The backend stores ciphertext, envelope metadata, routing, receipt state, and retention fields. It does not store a protocol-private plaintext payload.

### Database model

| Table | Key fields |
|---|---|
| users | user_id, phone_number_hash, display_name, account_status |
| devices | device_id, user_id, public_key_reference, push_token, last_active, status |
| conversations | conversation_id, conversation_type, created_at, updated_at |
| conversation_participants | conversation_id, user_id, joined_at, role, status |
| messages | message_id, conversation_id, sender_id, encrypted_payload_reference, sequence, message_type, created_at |
| receipts | receipt_id, message_id, device_id, status, acknowledged_at |
| media | media_id, uploader_id, storage_reference, encrypted_size, media_type |
| groups | group_id, name, created_by, participant_count, updated_at |

Use indexes on `(conversation_id, sequence)`, `(recipient_device_id, pending_sequence)`, `(sender_id, created_at)`, and `(device_id, status)`. Shard by conversation or user/device routing key. Replicate metadata separately from large media/object storage.

---

## 07 — Components

| Component ID | Component | Responsibility |
|---|---|---|
| CMP-001 | Load Balancer | Distributes API and connection gateway traffic |
| TBD | API Gateway | HTTPS REST, auth, rate, routing |
| TBD | Authentication Service | Registration, OTP, session, device lifecycle |
| TBD | User/Contact Service | Profiles and privacy-preserving contact matching |
| TBD | Connection Gateway | Persistent WebSocket/TCP sessions |
| TBD | Messaging Service | Validation, sequence, ciphertext persistence |
| TBD | Routing Service | Recipient/group/device session lookup |
| TBD | Queue | Durable delivery work and per-conversation partitions |
| TBD | Delivery Workers | Pending/sync/push/retry processing |
| TBD | Presence Service | Online/typing/heartbeat state |
| TBD | Media Service | Upload authorization and metadata |
| TBD | Object Storage/CDN | Encrypted media bytes and delivery |
| TBD | Search | Optional indexed user/group/approved message metadata |
| TBD | Push Service | APNs/FCM delivery |
| TBD | Calling Signaling | ICE/session/call state |
| TBD | STUN/TURN | NAT traversal and relay |
| TBD | Monitoring/Audit | Health, delivery, presence, calls, security |

### Component interactions

- Client → CMP-001/API gateway → auth/user/device services.
- Client ↔ connection gateway for persistent communication.
- Connection gateway → messaging/routing → queue.
- Messaging → message DB/outbox → queue.
- Queue → online gateway/recipient inbox/device sync.
- Offline worker → push service.
- Media client → media API → object storage/CDN.
- Calling client ↔ signaling/STUN/TURN and direct/relay media.

---

## 08 — Data Flow

### Flow A: One-to-one online send

1. Sender creates message ID and encrypts.
2. Client sends envelope and idempotency key.
3. Gateway authenticates sender and device.
4. Messaging service validates conversation and membership.
5. Database assigns sequence and stores ciphertext/outbox.
6. Router resolves recipient online devices.
7. Queue pushes delivery envelope to gateways.
8. Recipient sends accepted/delivered acknowledgment.
9. Sender receives server acceptance and delivery/read status.

### Flow B: Offline and reconnection

1. No active recipient gateway/session exists.
2. System stores pending device delivery.
3. Push notification contains no plaintext or only privacy-safe metadata.
4. Device reconnects and restores session cursor.
5. Sync service lists missing per-conversation sequences.
6. Device acknowledges accepted/delivered.
7. Server updates receipt and expunges according to policy.

### Flow C: Multi-device

1. Sender has multiple authorized sender devices.
2. Message is routed to the sender's own devices and recipient devices.
3. Each device receives ciphertext for its protocol session.
4. Per-device receipts are separate.
5. Revoked devices stop receiving and synchronize no later state.

### Flow D: Group fan-out

1. Sender encrypts/publishes to the selected group distribution mechanism.
2. Membership snapshot validates active participants.
3. Fan-out on write creates per-device tasks or fan-out on read stores conversation state.
4. Online and offline routing apply device-specific push/sync.
5. Large-member transitions update membership and group state separately.

### Flow E: Media

1. Client encrypts media or receives provider-specific media encryption metadata.
2. Media API validates type/size and returns upload grant.
3. Object storage receives ciphertext/encoded media.
4. Worker generates thumbnails/transcodes and stores derivatives.
5. Messaging service sends encrypted media reference.
6. Recipient downloads via CDN/object URL and decrypts locally.

### REST examples

**Register device**

`POST /v1/devices`

```json
{"phone_token":"...","device_type":"android","public_key":"...","push_token":"..."}
```

**Send message**

`POST /v1/conversations/{conversation_id}/messages`

```json
{"message_id":"msg_01J...","ciphertext":"...","message_type":"TEXT","idempotency_key":"client-key"}
```

**Sync**

`GET /v1/sync?conversation_id=conv_01J&cursor=841`

**Message status**

`200 OK`, `202 Accepted`, `400 invalid envelope`, `401/403 auth/session`, `404 hidden conversation`, `409 duplicate/conflict`, `429 abuse/rate limit`, `503 temporary backend`.

---

## 09 — Failure Scenarios

| Failure | Detection | Recovery |
|---|---|---|
| Connection gateway crash | Heartbeat/socket close | Reconnect, route to healthy gateway, sync cursor |
| Queue outage | Publish/lag/consumer errors | Durable outbox and replay |
| Message DB failure | Transaction/replica health | Failover/replicas, reject unsafe write |
| Offline backlog growth | Pending queue and oldest cursor | Scale workers, backpressure, retention |
| Duplicate message | Stable message ID | Idempotent DB/consumer dedup |
| Recipient offline | Presence/session lookup | Store pending and privacy push |
| Gateway split/region partition | Regional route health | Quarantine stale routing, reconcile |
| Redis presence loss | TTL/health | Reconnect heartbeat rebuilds presence |
| Push invalid token | APNs/FCM error | Re-register/remove token |
| Media upload failure | Storage status | Resume/retry encrypted upload |
| CDN/origin outage | Download/TTFB errors | Object origin fallback; cache policy |
| Key/prekey exhausted | Key inventory | Fetch bundle, rotate keys |
| Device compromise | Security report/admin revoke | Revoke device/session, rotate keys |
| Group membership race | Membership version mismatch | Versioned membership snapshot/rekey |
| Call failure | ICE/TURN/RTT events | Candidate refresh, TURN relay, graceful end |

---

## 10 — Scaling

### Connection and routing

- Run stateless gateways across zones behind CMP-001 and persistent-session-aware routing.
- Partition device routing by user/device hash.
- Scale gateway by active connections, CPU, memory, and heartbeat.
- Use queues and per-conversation/device partitions.

### Message and group scaling

- Assign per-conversation server sequence and partition by conversation.
- Index receipts by device/message.
- Use fan-out on write for low-latency recipient inboxes; fan-out on read for very large groups.
- Apply hybrid/hot-conversation strategies.
- Cache membership and conversation metadata with invalidation.

### Storage and media

- Shard message metadata by conversation/user.
- Use replicas for reads and durable quorum/consensus storage for accepted writes.
- Store media in object storage and use CDN.
- Archive old content according to retention and legal policy.

### Regional

- Route users to nearest healthy region.
- Replicate identity/config and cross-region message state according to RPO/RTO.
- Keep calling relay/media regional until calls require inter-region route.
- Reconcile pending delivery after failover.

---

## 11 — Trade-offs

| Choice | Benefit | Limitation |
|---|---|---|
| WebSocket | Low-latency bidirectional | Stateful gateway and reconnect complexity |
| TCP persistent | Efficient for mobile | Platform-specific implementation |
| Push polling fallback | Battery/coverage | Increased sync and privacy controls |
| At-least-once | Reliable offline delivery | Duplicate UI unless deduplicated |
| Per-conversation sequence | Predictable local order | Cross-conversation global order absent |
| Fan-out write | Fast recipient read | Write/storage amplification |
| Fan-out read | Lower write cost | Read-time work and stale membership |
| E2EE | Strong payload privacy | Server search/moderation/recovery complexity |
| CDN media | Low latency | Cache/revocation and privacy policy |
| Presence strong consistency | Accurate status | Expensive and poor at global scale |

---

## 12 — Cost

### Cost categories

Connection gateway compute/socket memory, message queue/storage, metadata database, object media, CDN, push delivery, call signaling/relay, encryption-key services, Redis, replication, logs/traces, and regional networking.

### Estimation worksheet

Let `M` = messages/day, `P` = text payload bytes, `F` = average recipient-device fan-out, `Media` = media bytes/day.

`text_ingress_bytes/day = M × P`

`delivery_payload_bytes/day = M × P × F × retransmission_factor`

`metadata_bytes/day ≈ M × metadata_record_size × index/replication factors`

`monthly_media_storage = retained_media_GiB × provider_storage_rate`

Use queue, database, region, CDN, call relay, and push provider calculators. Public product traffic values must not be claimed as measured internal statistics.

---

## 13 — Security

### Authentication and authorization

- Verify phone number through OTP or approved identity provider.
- Issue short-lived device/session credentials and support revocation.
- Authenticate every WebSocket and REST action.
- Validate conversation membership and recipient device authorization.
- Separate account, device, group administration, and content permissions.

### E2EE

- Protect identity/prekeys in secure client storage.
- Use public-key key agreement and ratcheted sessions.
- Ensure forward secrecy/post-compromise security at protocol level.
- Verify safety numbers/device keys through independent channels where supported.
- Keep ciphertext in server; do not expose plaintext as a normal backend state.
- Transport encryption remains required even for E2EE payloads.

### Additional controls

- Encrypt media and object storage; use expiring scoped download grants.
- Rate-limit registration, message, media, and calling abuse.
- Detect spam, account takeover, and abusive contact sync.
- Hash or tokenized phone numbers where address-book matching permits.
- Redact content, push payload, logs, and secret material.
- Audit key/device/account/group/security changes.

---

## 14 — Observability

### Metrics

Messages sent/accepted/delivered/read, p50/p95/p99 delivery latency, active connections, reconnect rate, queue depth, offline backlog, media success/latency, push success, invalid tokens, call setup/drop, database/cache latency, replication lag, and regional route health.

### Logs and traces

Use message/conversation/device/connection/tenant IDs. Never log plaintext or private key material. Propagate trace IDs across gateway, routing, queue, push, and sync. Use bounded event content and sampling.

### Alerts

- Delivery latency and failure-rate increase.
- Offline backlog or queue oldest age exceeds target.
- Connection gateway health/socket saturation.
- Redis/DB/queue region degradation.
- Push invalid-token spike.
- Media/CDN origin outage.
- Authentication/key inventory anomalies.

---

## 15 — Simulation

### Learning objective

Demonstrate message routing, acknowledgments, offline sync, group fan-out, media, presence, key inventory, device revocation, call signaling, and regional failover.

### Controls

| Key | Type | Default | Range |
|---|---|---:|---|
| active_devices | Slider | 100000000 | 1000–1000000000 |
| messages_per_second | Slider | 92000 | 1–1000000 |
| average_group_size | Slider | 20 | 2–10000 |
| offline_probability | Slider | 0.20 | 0–1 |
| duplicate_probability | Slider | 0.01 | 0–0.5 |
| media_share | Slider | 0.10 | 0–1 |
| queue_workers | Slider | 10000 | 1–100000 |
| ws_connections_per_gateway | Slider | 100000 | 100–1000000 |
| random_seed | Integer | 21 | 0–2147483647 |

### Required scenarios

1. Send/accept/deliver/read one-to-one message.
2. Disconnect recipient and deliver on reconnection.
3. Deduplicate duplicate message IDs.
4. Fan-out write versus read for small/large groups.
5. Exhaust and rotate one-time prekeys.
6. Revoke a device and stop synchronization.
7. Upload, thumbnail, download, and decrypt media.
8. Typing/presence expiration.
9. Push notification without plaintext.
10. Fail connection gateway and region.
11. Queue/database outage and outbox recovery.
12. Direct vs TURN call.

### Acceptance checks

- Per-conversation sequence is monotonic and deduplicated.
- Server acceptance does not equal device delivery/read.
- Offline ciphertext is retained and synchronized once.
- Revoked device receives no new state.
- Push contains no plaintext by default.
- Media and message metadata are separable.
- Call media and signaling are distinguished.
- Outputs are labeled estimates.

---

## 16 — Design Challenge

Design a messaging platform for 200 million daily active users with one-to-one/group text and media, offline delivery, multi-device, E2EE, calls, presence, and push.

**Requirements**
- Device registration and account security.
- Persistent connection gateways and offline queues.
- Per-conversation sequencing and effective-once processing.
- Per-device encryption/session state.
- Fan-out on write/read/hybrid group strategy.
- Media object storage/CDN and encrypted references.
- Push APNs/FCM with privacy-safe payload.
- Presence/typing TTL and privacy.
- Regional failover and disaster recovery.
- RPO/RTO, observability, abuse, and cost model.

**Learner deliverables**
1. Capacity and requirements.
2. Naive/evolved diagrams.
3. Sequence for send/offline/group/media/call.
4. Database/schema and index plan.
5. Encryption and key lifecycle.
6. Queue/ack/dedup model.
7. Media and push design.
8. Failure/HA/DR plan.
9. Technology trade-offs.
10. Simulation acceptance checks.

---

## 17 — AI Interview

| ID | Difficulty | Question | Strong answer |
|---|---|---|---|
| Q01 | Beginner | How does a persistent connection help? | Low-latency bidirectional real-time delivery |
| Q02 | Beginner | Sent vs delivered vs read? | Server accept, device receipt, user read |
| Q03 | Intermediate | How are offline messages delivered? | Durable queue/inbox, push, sync cursor |
| Q04 | Intermediate | At-least-once vs effectively once? | Network retries with message-ID deduplication |
| Q05 | Intermediate | Why not global message ordering? | Cross-conversation cost and clock skew |
| Q06 | Advanced | Why E2EE matters? | Payload unreadable without recipient device keys |
| Q07 | Advanced | How does per-conversation sequencing work? | Server sequence plus ack/cursor within conversation |
| Q08 | Advanced | Fan-out write vs read? | Write amplification vs read-time work |
| Q09 | Advanced | How do group key changes work? | Membership snapshot and rekey/revoke |
| Q10 | Advanced | How do multi-device keys differ? | Individual devices have distinct sessions/receipts |
| Q11 | Advanced | How does signaling differ from media? | Control session vs WebRTC encrypted media |
| Q12 | Advanced | What privacy-safe push includes? | Generic alert, not plaintext body |
| Q13 | Advanced | How do presence TTLs improve scale? | Coarse state and heartbeat expiry |
| Q14 | Advanced | How does media stay separate? | Object storage/CDN reference and metadata |
| Q15 | Advanced | How does regional failover handle delivery? | Replicated state, route reconciliation, device sync |

---

## 18 — Architecture Score

| Dimension | Weight | Evidence |
|---|---:|---|
| Requirements and scale | 10 | Users, messages, media, concurrency |
| Architecture and data flow | 15 | Gateway, queue, routing, sync, media |
| Real-time reliability | 15 | Ack, retries, offline, dedup, ordering |
| Security/E2EE | 15 | Keys, ratchet, devices, push, abuse |
| Group and media | 15 | Fan-out, membership, object/CDN |
| Calls/presence | 10 | Signaling, WebRTC, TTL, privacy |
| Observability/HA | 10 | Metrics, logs, region, RPO/RTO |
| Cost/trade-offs | 10 | Connection, queue, storage, relay |
| **Total** | **100** | |

Weighted score = sum of (dimension score / 4 × dimension weight).
