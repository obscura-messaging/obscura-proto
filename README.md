# Obscura transport protocol

The Protocol Buffer schema shared by [`obscura-server`](https://github.com/obscura-messaging/obscura-server) and the Kotlin and Swift kits in [`obscura-app`](https://github.com/obscura-messaging/obscura-app). Both pin this repository as a submodule (`proto/` in the server, `server-proto/` in the app) and generate their own code from it. The server routes encrypted bytes and never parses the client content inside them.

| File | Purpose |
|---|---|
| [`obscura/v1/obscura.proto`](obscura/v1/obscura.proto) | Message submission (`POST /v1/messages`) and gateway WebSocket frames. |
| [`TRANSPORT.md`](TRANSPORT.md) | Normative transport behaviour. |
| [`buf.yaml`](buf.yaml) | `STANDARD` lint and `FILE` breaking-change rules. |

Elsewhere:

- client-to-client encrypted content: [`obscura-app/client-protocol`](https://github.com/obscura-messaging/obscura-app/tree/main/client-protocol)
- kit behaviour contract: [`obscura-app/docs/KIT_API.md`](https://github.com/obscura-messaging/obscura-app/blob/main/docs/KIT_API.md)
- application routing and merge rules: [`obscura-app/docs/DOMAIN_CONTRACT.md`](https://github.com/obscura-messaging/obscura-app/blob/main/docs/DOMAIN_CONTRACT.md)

## Changing the schema

CI runs both checks (breaking only on pull requests):

```bash
buf lint
buf breaking --against '.git#branch=main' --path obscura/v1
```

A breaking change needs a coordinated server and native migration. The package is `obscura.v1` for compatibility; it predates the `obscura.<layer>.<version>` convention and is the transport package.
