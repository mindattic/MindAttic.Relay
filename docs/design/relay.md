# MindAttic.Relay: design

Status: accepted direction (owner, 2026-10-27), nothing built yet. Ticket M.RLY-T1.

## 1. What Relay is

Relay is the one way MindAttic apps send each other messages, on one machine or across many. It is:

- **a message server** that runs in the background on every machine: NATS with JetStream;
- **a contracts package** (`MindAttic.Relay.Contracts`) that defines every message;
- **a client package** (`MindAttic.Relay.Client`) that every app uses, so no app touches NATS directly;
- **an outbox package** (`MindAttic.Relay.Outbox`) so a database change and its message can never come apart;
- **a CLI** (`relay`) to install, inspect and operate it.

It replaces today's ad hoc paths: AppBus over the Orchestrator daemon's named pipes, apps polling shared databases to
notice changes, and the hand-built control socket, outbox and heartbeat that the hub-and-spoke plan was going to write.

## 2. Why NATS (and not Redis + gRPC)

| Need | NATS + JetStream | Redis + gRPC |
|---|---|---|
| Runs natively on Windows as a service | Yes, one small binary | Redis is not supported on Windows (Docker, WSL or Memurai) |
| Fan-out events, durable queues, per-app read position, replay | Built in (streams, durable consumers) | Redis Streams covers part; acks, retries and dead letters are ours to write |
| Request/reply (commands) | Built in, with service discovery and stats | gRPC, a second system |
| Shared state with compare-and-set | Built in (Key-Value buckets with revisions) | Redis keys; CAS by hand |
| Across the web, machines behind home routers | Leaf nodes dial *out* to a hub; WebSocket on 443 | Redis must not face the internet, so a gRPC gateway is needed in front |
| Auth | Per-app and per-device credentials, per-subject permissions, accounts as hard isolation | Redis ACLs plus our own gateway auth |
| Moving parts | One server per machine | Redis + a gateway we write and run |

gRPC's useful idea is kept: message payloads are **Protocol Buffers** with generated C# types. A gRPC endpoint can be
added later only for an outside system that can't use a NATS client.

## 3. Topology

```
 Machine A (hub)                         Machine B (spoke)            Machine C (spoke)
 +-------------------------------+       +-----------------------+    +----------------+
 | Relay server (hub)            |<------| Relay server (leaf)   |    | Relay server   |
 |  JetStream domain "hub"       | TLS / |  JetStream domain     |    |  (leaf)        |
 |  KV: presence, policy, leases | wss   |  "dev-<deviceId>"     |    |                |
 +-------------------------------+ :443  +-----------------------+    +----------------+
   ^      ^          ^                     ^        ^       ^
 Orch.   Tickets   Prose Hub             Orch.   Tickets  KdpPublish   (apps always talk to
 Desktop Desktop   KdpPublish            Desktop Desktop               their own machine's server)
```

- **Every machine runs its own Relay server.** Apps only ever connect to `localhost`. An app never knows whether the
  machine is alone or joined.
- **Single machine = no remotes.** That is today's setup and the default. Nothing leaves the machine.
- **Joining = one leaf connection dialled out** from the spoke to the hub, over TLS, or WebSocket on 443 where a proxy
  is in the way. The hub is the only listener. No spoke opens an inbound port.
- **Each spoke has its own JetStream domain**, so its streams keep accepting messages while the hub is unreachable.
  The hub *sources* what it needs from spoke streams when the link is up, and spokes *mirror* the hub's KV buckets
  (policy, roster) so the last known value is always local. This is what replaces the planned `hub_outbox` and its
  replay. It is the most intricate part of the design; M.RLY-T10 prototypes it before anything depends on it.
- **The hub machine is also a spoke** for its own apps (today's main PC). It can move to a dedicated box later
  (hub-and-spoke Phase 6).

## 4. Subjects

Accounts give isolation between organisations, so subjects carry no org. The scheme is:

```
<app>.<noun>.<event>[.<key>]        events      prose.book.published.<slug>
<app>.cmd.<verb>                    commands    orch.cmd.open-ticket, tix.cmd.claim
term.<device>.<session>.<channel>   terminals   term.d7f3.s42.out | .in | .ctl
```

- Lowercase, dot-separated, `-` inside a token. No personal data or secrets in a subject (subjects show up in logs and
  monitoring).
- App tokens: `prose`, `kdp`, `orch`, `tix`, `launcher`, `relay` (Relay's own), more as apps join.
- Every subject an app may publish or subscribe to is listed in its permissions (section 7); an app can't publish
  outside its own `<app>.>` space except to command subjects it is allowed to call.

## 5. Four kinds of traffic

| Kind | Mechanism | Delivery | Examples |
|---|---|---|---|
| **Events**: "this happened" | JetStream streams, one per app (`PROSE` = `prose.>`, `TIX` = `tix.>`, ...). Each subscriber app has its own durable consumer | At least once, ordered per subject, replayable, kept for a retention window | `prose.book.published`, `tix.ticket.changed`, `orch.meter.reading` |
| **Commands**: "please do this" | Request/reply on core NATS, served as NATS services (discoverable, with stats) | At most once, with a timeout; the caller decides what a timeout means | `orch.cmd.open-ticket`, `orch.cmd.show-worker`, `tix.cmd.claim` |
| **State**: "what is true now" | JetStream Key-Value buckets; watchers see every change; updates can be conditional on the last revision | Latest value per key, history kept briefly | `presence` (TTL), `policy`, `leases`, `roster` |
| **Byte streams**: terminal frames | Core NATS subjects, not stored; replay comes from TermHost's own ring by request | Live only | `term.<device>.<session>.out` |

## 6. The envelope

Every message carries NATS headers, so nothing has to open the payload to route, deduplicate or trace it:

| Header | Meaning |
|---|---|
| `Nats-Msg-Id` | Unique id (UUIDv7). JetStream drops a duplicate published within the stream's duplicate window |
| `Relay-Type` | Contract name, e.g. `prose.BookPublished` |
| `Relay-Version` | Contract major version (integer) |
| `Relay-Source` | Sending app and machine, e.g. `kdp@d7f3` |
| `Relay-Correlation` | Id that ties a chain of messages together (a request and everything it caused) |
| `Relay-Sent` | UTC timestamp, ISO 8601 |

The payload is the Protocol Buffers encoding of the contract. Contracts live in `MindAttic.Relay.Contracts` as `.proto`
files with generated C# types, versioned with the package.

## 7. Security, from day one

- **Credentials everywhere, including on one machine.** NATS decentralized auth: an **operator** key (kept offline,
  backed up), one **account** per organisation, one **user** per app per machine (and later per device). Each user has
  an NKey and a signed credential that lists exactly which subjects it may publish and subscribe to.
- **TLS on every listener**, loopback included: the Relay server generates a local certificate authority at install;
  across machines the hub uses a real certificate where it can, and spokes pin the hub's key at enrollment.
- **Secrets stay in MindAttic.Vault** (DPAPI). Apps get their credential from the Vault at start.
- **Revocation**: a user or device credential can be revoked at the account; its connection drops.
- **Dynamic, expiring permissions** (who may watch or type into which terminal) use NATS **auth callout**: the hub's
  auth service decides a connection's permissions at connect time, with an expiry. This is how hub-and-spoke grants
  are enforced by the server itself, with the spoke still re-checking as defence in depth.
- **Compression off on links that carry interactive terminal input** (attacker-chosen input next to secrets is the
  CRIME/BREACH shape, already flagged in Orchestrator T934/T935).
- **The monitoring endpoint listens on localhost only.**

## 8. Delivery rules every app follows

1. **The database is the record; a message is the news.** Relay never replaces a table. KdpPublish still writes the
   ASIN to the Prose database; `prose.book.published` only announces it.
2. **At least once, so handlers are idempotent.** A consumer acks only after its handler succeeds, and skips a
   `Nats-Msg-Id` it has already processed (a small `relay_processed` table, kept longer than the stream's duplicate
   window). Exactly-once delivery is not a thing; nobody builds on it.
3. **Outbox for every event that follows a database write.** The app writes its change and an outbox row in one
   transaction; `MindAttic.Relay.Outbox` publishes the row with `Nats-Msg-Id` = the row's id and marks it sent. A crash
   between the two can no longer lose the event, and a retry can't double it.
4. **Commands time out loudly.** A command that gets no reply says so in plain words ("Tickets Desktop didn't answer").
   It never silently retries something that isn't safe to repeat.
5. **Poison messages go to a dead-letter stream** after a set number of failed deliveries, with the failure reason,
   and show up in monitoring.
6. **Nothing depends on Relay to keep working.** An app whose Relay connection is down keeps doing its own job; it
   only stops hearing about others.

## 9. Versioning

- Contracts change by adding fields (Protocol Buffers' rule). A breaking change is a new `Relay-Version`.
- Consumers skip a `Relay-Type` they don't know and a `Relay-Version` newer than they understand, and count it.
- Across machines, the hub supports the current and previous version of every contract (N and N-1).
- The Relay server version is pinned and upgraded deliberately, hub first.

## 10. How the hub-and-spoke plan maps onto Relay

The hub-and-spoke epic (formerly Orchestrator M.ORCH-T1145, now in this repo) keeps its decisions: masking, sharing,
consent and enforcement stay on the spoke; the hub decides; single-machine behaviour is unchanged. What changes is
that its plumbing comes from Relay instead of being written by hand:

| Hub-and-spoke piece (old plan) | With Relay |
|---|---|
| `wss://hub/spoke` control socket, `HubProtocol`, `HubEnvelope` with seq/ack | Leaf connection + Relay envelope headers; JetStream does sequencing and acks |
| `hub_outbox` table and replay on reconnect | The spoke's own JetStream domain; the hub sources from it when the link returns |
| Heartbeat carrying presence | `presence` KV bucket with a TTL, refreshed each daemon tick |
| `welcome` + deltas for members, orgs, policy | `roster` and `policy` KV buckets, mirrored to every spoke; last value survives offline |
| Claims, renewals, epoch fencing | `tix.cmd.claim` command + `leases` KV with revision-conditional updates (the revision is the fence) |
| Ticket replica feed | `TIX` stream of ticket changes; each spoke's durable consumer applies them to its local `Tickets.db` |
| Gateway outbound mode + hub `StreamRelay` | Terminal frames on `term.<device>.<session>.*` subjects; attach/replay by command. No separate relay process |
| Grants with expiry, rechecked | Auth callout issues expiring per-viewer permissions; revoke drops the connection |
| Device key + 15-minute tokens | Device NKey + short-lived user credential from the hub's enrollment service |
| Remote dispatch opt-in | A spoke serves `orch.cmd.dispatch.<device>` only when its owner opted in; no subscriber, no dispatch |

The peer-to-peer gateway keeps working beside all of this, as the epic already required.

## 11. Order of work

The epic is M.RLY-T2; approving it starts the work.

1. Foundation: run the server (M.RLY-T3), contracts (T4), client (T5), security (T6), outbox (T7).
2. First real users: AppBus between Orchestrator.Desktop and Tickets.Desktop (T8), then KdpPublish's
   `prose.book.published` (T9).
3. Across machines: leaf nodes and JetStream domains (T10), device credentials and auth callout (T11),
   monitoring (T12).
4. Then the hub-and-spoke phases, rewritten on top of the above (T13 to T30).

## 12. Non-goals

- Replacing any app's database, or git as the code integration point.
- Exactly-once delivery.
- Browser or phone clients at first (NATS speaks WebSocket, so this stays possible).
- Running anyone's code: Relay carries messages; the apps decide what to do with them.
