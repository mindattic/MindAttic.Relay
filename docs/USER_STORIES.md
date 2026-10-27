---
codex: 1
project: MindAttic.Relay
code: M.RLY
layer: stories
status: living
updated: 2026-10-27
---
<!-- GENERATED from MindAttic.Tickets/data/Tickets.db by `tickets db render`. Do not edit: this file is rewritten from the database; change stories and tickets with the tickets CLI. -->

# MindAttic.Relay — User Stories
> ✅ done (shipped & tested) · 🟡 partial · ⬜ planned. Every ✅ cites the test.

## Priority backlog
_Nothing recorded yet._

## Tickets
> Permanent log of every change and fix — this repo's ticket system. Rows are append-only: never
> delete or reword one; only move its Status (open → closed / wont-do) and fill in Ref. Status is
> a word, not a glyph, so tickets don't count toward the story index. Keep this section last.

| ID | Date | Type | Status | Summary | Ref |
|---|---|---|---|---|---|
| M.RLY-T1 | 2026-10-27 | docs | closed | Scaffold the MindAttic.Relay repo (agent standard files, ticket code M.RLY) and write docs/design/relay.md: the architecture every MindAttic app and machine uses to message each other (NATS + JetStream, subjects, envelope headers, security, delivery rules, outbox, versioning) and how the hub-and-spoke plan maps onto it | ba9744e |
| M.RLY-T2 | 2026-10-27 | feature | open | EPIC: MindAttic.Relay, one message exchange for every MindAttic app and machine (NATS + JetStream), and hub and spoke built on it: members' machines joined into one Orchestrator, everyone's CLIs visible and shareable, single-machine mode unchanged | — |
| M.RLY-T3 | 2026-10-27 | feature | open | Relay server: install a pinned NATS server with JetStream as a Windows service on each machine (loopback-only, TLS, local CA, credentials required), with `relay install\|status\|uninstall` | — |
| M.RLY-T4 | 2026-10-27 | feature | open | MindAttic.Relay.Contracts: Protocol Buffers message definitions with generated C# types, the envelope header names (Nats-Msg-Id, Relay-Type, Relay-Version, Relay-Source, Relay-Correlation, Relay-Sent) and the subject naming rules, as a NuGet package | — |
| M.RLY-T5 | 2026-10-27 | feature | open | MindAttic.Relay.Client: the one client every app uses (PublishAsync, SubscribeAsync with durable consumers and ack-after-handler, RequestAsync, ServeAsync, key-value access), DI via AddMindAtticRelay(), tested against a real embedded nats-server | — |
| M.RLY-T6 | 2026-10-27 | feature | open | Relay security: operator/account/user credentials with per-app subject permissions, credentials stored in MindAttic.Vault, `relay creds issue\|revoke\|list`, TLS everywhere, operator key backup | — |
| M.RLY-T7 | 2026-10-27 | feature | open | MindAttic.Relay.Outbox: save a database change and its outgoing message in one EF Core transaction, then a background dispatcher publishes it with Nats-Msg-Id = the outbox row id and marks it sent | — |
| M.RLY-T8 | 2026-10-27 | feature | open | First users: move AppBus between Orchestrator.Desktop and Tickets.Desktop onto Relay (ticket changed as an event, open-ticket and show-worker as commands), named pipes kept only as a fallback until both apps ship | — |
| M.RLY-T9 | 2026-10-27 | feature | open | Second user: KdpPublish announces `prose.book.published` (slug, ASIN, Amazon URL, KDP title id) through the outbox in the same transaction that writes the ASIN to the book record; Prose Hub and Writer refresh on it instead of on re-read | — |
| M.RLY-T10 | 2026-10-27 | feature | open | Across machines: a hub Relay server accepts leaf connections that spokes dial out (TLS, or WebSocket on 443), each spoke keeps its own JetStream domain so it works offline, the hub sources spoke streams and spokes mirror hub key-value buckets; prototype first, then `relay join\|leave` | — |
| M.RLY-T11 | 2026-10-27 | feature | open | Device credentials and auth callout: a hub service issues each enrolled device a short-lived NATS user credential, and decides per-connection permissions with an expiry (the mechanism hub-and-spoke grants use) | — |
| M.RLY-T12 | 2026-10-27 | feature | open | Relay monitoring: `relay tail <subject>`, `relay dead-letters [--replay]`, stream and consumer lag, and a Relay page in Orchestrator.Desktop showing servers, links, streams, lag and dead letters | — |
| M.RLY-T13 | 2026-10-27 | feature | open | Hub 0.1: hub seams in Orchestrator Core (IClaims, IPolicySource, IMeterSink, IOrgDirectory, ISessionDirectory) with Local implementations over today's code and OrchestratorSettings.Hub (Mode Off by default); no behaviour change | — |
| M.RLY-T14 | 2026-10-27 | feature | open | Hub 0.2: Orchestrator and Tickets hub messages defined in MindAttic.Relay.Contracts (presence, meter reading, session info, lease, policy snapshot, attach) instead of a separate Hub.Contracts library with its own protocol and envelope | — |
| M.RLY-T15 | 2026-10-27 | feature | open | Hub 1.3: MindAttic.Orchestrator.Hub service on the hub machine: MindAttic.Authentication member sign-in, device enrollment page using Relay device credentials, a devices table, served as Relay commands (no custom WebSocket control socket) | — |
| M.RLY-T16 | 2026-10-27 | feature | open | Hub 1.4: spoke side of joining: the daemon supervises the machine's Relay leaf link (Relay join), publishes presence each tick, and its state survives offline in the spoke's JetStream domain (replaces HubLink, its control socket and hub_outbox) | — |
| M.RLY-T17 | 2026-10-27 | feature | open | Hub 1.5: `orchestrator hub host\|join\|status\|leave\|devices` as thin wrappers over `relay join\|leave\|status` plus the hub service, and a dashboard hub chip (Off, Connected, Offline since) | — |
| M.RLY-T18 | 2026-10-27 | feature | open | Hub 2.6: presence and meters over Relay: spokes publish CLI roster, masked sessions, meter readings and agent summaries; the hub merges meters per login and serves org presence and usage as Relay commands | — |
| M.RLY-T19 | 2026-10-27 | feature | open | Hub 2.7: org view in the dashboard and `/org usage` across machines, remote people read-only; members and orgs live at the hub and reach spokes through the mirrored `roster` KV bucket (`hub import-org` seeds it) | — |
| M.RLY-T20 | 2026-10-27 | feature | open | Hub 3.8: terminal streams over Relay: the spoke publishes masked TermHost frames on `term.<device>.<session>.out`, viewers attach by command (replay from TermHost's ring), input on `.in` only for interact; no gateway outbound mode and no hub StreamRelay process | — |
| M.RLY-T21 | 2026-10-27 | feature | open | Hub 3.9: grants and audit at the hub, enforced by Relay auth callout (expiring per-viewer permissions on term subjects), with the spoke re-checking the hub's signed decision; `gateway grant\|revoke\|audit` go to the hub when joined | — |
| M.RLY-T22 | 2026-10-27 | feature | open | Hub 3.10: owner consent for interact: a first-use toast per grant on the owner's machine, watch-only on 30 s timeout; the spoke only reads `.in` for a consented grant | — |
| M.RLY-T23 | 2026-10-27 | feature | open | Hub 3.11: Team Wall and TermViewer through Relay (`--hub`), listing org sessions from the presence bucket beside peer-to-peer remotes | — |
| M.RLY-T24 | 2026-10-27 | feature | open | Hub 4.12: Tickets over Relay: the hub's TicketsDb is the store of record, every change is a `tix.ticket.changed` event on the TIX stream, each spoke's durable consumer keeps `data/Tickets.db` as a replica for the hooks; ticket commands (notes, status, assign, approve) are Relay commands; DB git sync only on the hub | — |
| M.RLY-T25 | 2026-10-27 | feature | open | Hub 4.13: ticket leases on Relay: `tix.cmd.claim` runs Allocate on the hub; leases live in the `leases` KV bucket where each renewal and completion is a revision-conditional update (the revision is the fence); offline grace before requeue | — |
| M.RLY-T26 | 2026-10-27 | feature | open | Hub 4.14 (MindAttic.Tickets): the tickets CLI and Tickets Desktop send writes as Relay commands to the hub when the machine has joined; read-only with a clear message when the hub can't be reached | — |
| M.RLY-T27 | 2026-10-27 | feature | open | Hub 5.15: central pacing policy in the hub's `policy` KV bucket (cutoffs, allowances, pace target, routing, weekly reserve), mirrored to every spoke so the last policy holds offline, then the defaults | — |
| M.RLY-T28 | 2026-10-27 | feature | open | Hub 5.16: per-login allowance split across devices when one Claude login is live on several machines, published as `AllowanceShare` entries in the policy bucket | — |
| M.RLY-T29 | 2026-10-27 | feature | open | Hub 5.17: opt-in remote dispatch (a spoke serves `orch.cmd.dispatch.<device>` only when its owner set AcceptDispatch) plus cross-machine rebalance and API-key failover to key-holder spokes only | — |
| M.RLY-T30 | 2026-10-27 | feature | open | Hub 6.18: operations: deploy the hub service and the hub Relay server, JetStream and hub DB backups, operator and certificate key rotation, device revocation UI, rate limits on sign-in and enrollment, optional cloud relay for members off the tailnet, version-skew and chaos tests | — |
| M.RLY-T31 | 2026-10-27 | docs | closed | docs/design/relay.md cites the foundation tickets one number low (written before the epic took M.RLY-T2): server is T3, contracts T4, client T5, security T6, outbox T7, AppBus T8, KdpPublish T9, leaf nodes T10, device credentials T11, monitoring T12 | 3891ecd |
