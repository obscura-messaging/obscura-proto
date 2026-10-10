# Transport contract

Normative for `obscura-server` and `obscura-native`. [`obscura.proto`](obscura/v1/obscura.proto) defines the shapes; this file defines behaviour. The server queues, routes, timestamps and deletes encrypted client bytes, and MUST NOT parse their content. REST details are in the server's `openapi.yaml`.

## Sending: `POST /v1/messages`

Body `SendMessageRequest`, response `SendMessageResponse`, both protobuf. Requires a device-scoped JWT.

- Each submission is encrypted separately for one target device. `submission_id` and `device_id` are 16-byte UUIDs; `message` must be non-empty.
- `Idempotency-Key` (a UUID header) is required. A repeat within the server's idempotency TTL (default 24 h) returns the cached response.
- Too many submissions (default limit 100) rejects the whole request with 413.
- Otherwise failures are per submission in `failed_submissions`; an empty list means every submission was queued.

## Gateway: `/v1/gateway`

Open with a single-use ticket from `POST /v1/gateway/ticket`: `GET /v1/gateway?ticket=<ticket>`. Every frame is a binary `WebSocketFrame`:

| Payload | Direction | Meaning |
|---|---|---|
| `EnvelopeBatch` | server → client | Queued messages, oldest first. |
| `AckMessage` | client → server | Delete these messages. |
| `PreKeyStatus` | server → client | Advisory: one-time prekeys fell below `min_threshold` after a bundle fetch. |

The server ignores text frames, undecodable frames, server-to-client payloads sent by a client, and malformed IDs within an ack; it logs them and keeps the connection open. It pings periodically and closes a socket that has sent nothing for the ping interval plus timeout (defaults 30 s + 10 s).

## Acknowledgement is deletion

An accepted ack deletes the message; there is no tombstone and no other way to get it back. Messages also disappear unacknowledged when they expire (default 30 days) or overflow the device's queue limit (default 1000, oldest first), and all of a device's queued messages are deleted when it uploads a new identity key.

The native client therefore MUST:

1. not acknowledge a decryption failure;
2. not acknowledge deferred processing;
3. finish durable handling first: `decrypt -> persist/handle -> optional wake -> ack`.

Deletion is batched and asynchronous, and the server drops acks when its per-socket buffer is full. Unacknowledged messages are redelivered on the next connection, so the client MUST deduplicate by `Envelope.id`. A duplicate already in durable storage counts as handled and may be acknowledged.

## Envelope fields

| Field | Meaning |
|---|---|
| `id` | 16-byte UUID; the ack and deduplication key. |
| `timestamp` | Server receipt time, epoch milliseconds. Client-content timestamps are outside this contract. |
| `sender_id` | Sending user UUID, stamped by the server from the sender's JWT. |
| `sender_device_id` | Sending device UUID, likewise; selects the inbound Signal session. |

Neither sender field is a trust root; successful Signal decryption proves the sending device. The native client:

- MUST NOT guess a missing `sender_device_id` or fall back to `registrationId`;
- takes display names from local trusted state, never from transport or payload claims;
- SHOULD report a mismatch when local state already knows the owner of `sender_device_id` and it differs from `sender_id`.

## Compatibility

Field numbers and wire types are stable within `obscura.v1`; removing or renumbering a field is breaking. Do not rely on an added field until the server, Kotlin and Swift bindings are regenerated.
