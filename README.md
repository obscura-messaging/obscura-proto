# Obscura transport protocol

The Protocol Buffer schema shared by [`obscura-server`](https://github.com/obscura-messaging/obscura-server) and [`obscura-native`](https://github.com/obscura-messaging/obscura-native), both of which pin this repository as the `proto` submodule and generate their own code from it. The server routes encrypted bytes and never parses the client content inside them.

| File | Purpose |
|---|---|
| [`obscura/v1/obscura.proto`](obscura/v1/obscura.proto) | Message submission (`POST /v1/messages`) and gateway WebSocket frames. |
| [`TRANSPORT.md`](TRANSPORT.md) | Normative transport behaviour. |
| [`buf.yaml`](buf.yaml) | `STANDARD` lint and `FILE` breaking-change rules. |

Elsewhere:

- client-to-client encrypted content: [`obscura-native/protocol`](https://github.com/obscura-messaging/obscura-native/tree/main/protocol)
- native/app facade: [`obscura-native/docs`](https://github.com/obscura-messaging/obscura-native/tree/main/docs)
- application routing and merge rules: [`obscura-pix/docs/DOMAIN_CONTRACT.md`](https://github.com/obscura-messaging/obscura-pix/blob/main/docs/DOMAIN_CONTRACT.md)

## Changing the schema

CI runs both checks (breaking only on pull requests):

```bash
buf lint
buf breaking --against '.git#branch=main' --path obscura/v1
```

A breaking change needs a coordinated server and native migration. The package is `obscura.v1` for compatibility; it predates the `obscura.<layer>.<version>` convention and is the transport package.
