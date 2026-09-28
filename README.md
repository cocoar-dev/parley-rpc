# ParleyRPC

> **The Parley Protocol** — the negotiated realtime conversation between services and clients.

A *parley* is historically the conversation under the white flag: two parties grant each other
safe conduct, and then they talk — demands, messages, answers, in both directions, for as long as
the flag stands. This protocol takes that structure literally: a connection **negotiates** its
transport, format and capabilities, receives **safe conduct** through a full credential lifecycle,
and then both sides **converse** — typed calls in both directions, item streams, byte channels.

## Status: pre-spec

**There is no code here yet, and nothing is stable.** This repository currently holds the
foundations document — terminology, axioms, the capability model, the invariants, and thirteen
open design questions that are being discussed before the actual specification is written.

- [`docs/foundations-de.md`](docs/foundations-de.md) — the foundations, version 0.1 *(German; the
  specification itself will be English)*
- [`spec/`](spec/) — reserved for the specification
- [`conformance/`](conformance/) — reserved for the conformance suite

## What it will be

- **Spec-first.** The protocol is a document plus a conformance suite. Every implementation is
  equal; none is "the reference".
- **Polyglot by design.** Target languages: C#, TypeScript, Rust, Swift, Kotlin, Go. Only what is
  implementable in *all* of them may become protocol semantics — SDK surfaces stay idiomatic.
- **Bidirectional.** Both peers offer contracts, both call, both stream. The server is not an
  answering machine.
- **Negotiated transports.** WebSocket → Server-Sent Events → Long Polling as the web family, with
  IPC (named pipes, unix domain sockets) and TCP/TLS as HTTP-free families. Session negotiation is
  in-band — any ordered, framed pipe can carry a parley.
- **A full credential lifecycle** (challenge, revalidation, expiry, revocation) and **byte
  channels** for large payloads — first-class citizens, not bolt-ons.
- **Forward means beside.** Replacement is not an operation the protocol knows: an acceptor serves
  the current and previous protocol versions concurrently, so servers can always move first
  without breaking the installed fleet.

## Non-goals

No peer-to-peer media (that is WebRTC), no message bus, no REST replacement, no business logic in
the protocol.

## Steward

[Cocoar](https://cocoar.dev) — future home: `parley-rpc.cocoar.dev`.

Licensed under the [Apache License 2.0](LICENSE).
