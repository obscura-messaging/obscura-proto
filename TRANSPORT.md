# Transport contract

Normative for `obscura-server` and `obscura-native`. [`obscura.proto`](obscura/v1/obscura.proto) defines the shapes; this file defines behaviour. The server queues, routes, timestamps and deletes encrypted client bytes, and MUST NOT parse their content. REST details are in the server's `openapi.yaml`.

## Sending: `POST /v1/messages`

Body `SendMessageRequest`, response `SendMessageResponse`, both protobuf. Requires a device-scoped JWT.

- Each submission is encrypted separately for one target device. `submission_id` and `device_id` are 16-byte UUIDs; `message` must be non-empty.
- `Idempotency-Key` (a UUID header) is required. A repeat within the server's idempotency TTL (default 24 h) returns the cached response.
- Too many submissions (default limit 100) rejects the whole request with 413.
- Otherwise failures are per submission in `failed_submissions`; an empty list means every submission was queued.

### Complete delivery

The users checked are those listed in `recipient_user_ids` together with the owners of the targeted devices. A client lists every user it intends to reach, including its own user for messages to its other devices, so that a user whose devices it does not know yet is still checked. For each checked user, the targeted devices MUST equal that user's current devices, excluding the sending device when the user is the sender. Otherwise none of that user's submissions are queued: each fails with `ERROR_CODE_DEVICE_MISMATCH`, and `device_mismatches` holds one entry for that user listing the current devices that were not targeted. Submissions to other users in the same request are unaffected. A submission to an unknown device, or to the sending device itself, fails individually with `ERROR_CODE_INVALID_DEVICE` and does not affect any user's group.

The check and the queueing are atomic: a device added concurrently either is part of the checked set or is added after the submissions were queued.

A response is cached by `Idempotency-Key`, a mismatch included. A client that retries after a mismatch, with the device set corrected, MUST use a new `Idempotency-Key`.

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
