# MindAttic.Relay

The message exchange for every MindAttic app, on one machine or across many.

Relay runs a NATS server with JetStream in the background on each machine. Apps send and receive messages through
one client library and one set of shared message definitions, instead of named pipes, shared-database polling or
one-off sockets. Machines join each other by dialling out to a hub, so nobody opens an inbound port.

**Status:** design accepted, nothing built yet. Read [docs/design/relay.md](docs/design/relay.md) for the
architecture, and `docs/USER_STORIES.md` (generated from the ticket database, code `M.RLY`) for the work queue.

## Planned pieces

| Package / tool | What it is |
|---|---|
| Relay server | NATS + JetStream, installed and run as a Windows service |
| `MindAttic.Relay.Contracts` | Protocol Buffers definitions of every message, with generated C# types |
| `MindAttic.Relay.Client` | The one client every app uses: publish, subscribe, request/reply, key-value state |
| `MindAttic.Relay.Outbox` | Saves a database change and its message together, then sends the message |
| `relay` CLI | Install, status, credentials, tail a subject, dead letters |

## First users

1. Orchestrator.Desktop and Tickets.Desktop (today's AppBus over named pipes).
2. KdpPublish announcing a published book and its ASIN.
3. The hub-and-spoke epic: members' machines joined into one Orchestrator.
