# Design: Tailscale Services (VIP services / `svc:`) for Headscale

| | |
|---|---|
| **Status** | Draft — for discussion on the issue tracker |
| **Author** | (to be filled in) |
| **Scope** | Add control-plane support for Tailscale "Services" (VIP services advertised with `tailscale serve --service=svc:…`) |
| **Targets** | tailscale client ≥ v1.80 (cap ≥ 113); built against `tailscale.com` v1.98.3 |
| **Related** | Builds on the per-node HTTPS-certificate / `tailscale serve` work (branch `claude/upbeat-hopper-QcjrT`). Distinct from Funnel (unsupported) and from per-node serve (already supported). |

> Per `CONTRIBUTING.md`, new features must start as a design document discussed
> with the maintainers before implementation. This document is intended to seed
> that discussion. **No implementation should begin until the approach and
> maintenance plan are agreed.**

---

## 1. Summary

Tailscale **Services** let a tailnet expose a *named, virtual* service —
`svc:my-service` — that lives at its own pair of virtual IP addresses (a "VIP",
one IPv4 + one IPv6), independent of any single node's IPs. One or more nodes
*host* the service (via `tailscale serve --service=svc:my-service …`), and
consumer nodes reach it by the service's MagicDNS name / VIP, with control
deciding which host(s) currently receive traffic.

This is **not** the same as the per-node `tailscale serve` we already support
(which exposes a port on a node's *own* name). Services add a control-plane
concept — a first-class object with allocated addresses, an ownership/host
authorization model, service discovery pushed to consumers, and optional
high-availability across multiple hosts.

This document describes the protocol, the headscale subsystems it touches, a
proposed data model and architecture, the security model, a phased delivery
plan, and the open questions and risks. The single biggest new primitive is
**control-to-node (c2n) request support**, which — importantly — can be built
on headscale's *existing* `PingRequest` plumbing rather than a brand-new
transport.

## 2. Motivation & use cases

- Run a service (database, dashboard, internal API) addressable by a stable
  name and address that is **decoupled from the node** currently serving it.
- **Migrate / fail over** a service between nodes without consumers changing
  configuration (the VIP and name stay constant).
- **Scale out** a service across multiple hosts with control distributing new
  connections to active hosts (`drain`/`advertise` for graceful rollout).
- Present a discoverable list of services to clients (the desktop UIs show a
  services menu when the tailnet advertises any).

## 3. Background: the protocol primitives

All types below are from `tailscale.com/tailcfg` (v1.98.3). They are the
contract headscale must satisfy.

### 3.1 Service identity and definition

```go
// tailcfg.go
type ServiceName string // "svc:dns-label"

type VIPService struct {
    Name   ServiceName       // svc:foo — unique per tailnet, maps 1:1 to the VIP pair
    Ports  []ProtoPortRange  // which proto/ports this host exposes on the VIP
    Active bool              // should control send new requests for the service to this node?
}
```

A node that runs `tailscale serve --service=svc:foo …` becomes a *host* of
`svc:foo`. It reports the services it currently serves via c2n (below).

### 3.2 How a host advertises (Hostinfo + c2n)

Modern clients do **not** put the service list in the map request. Instead:

```go
// tailcfg.Hostinfo
ServicesHash string // opaque hash; a change signals "fetch my service list via c2n"
```

```text
// Capability-version history (tailcfg.go):
// 108: 2024-11-08: Client sends ServicesHash in Hostinfo, understands c2n GET /vip-services.
```

So when the host's set of advertised services changes, only the **hash**
changes in `Hostinfo`. Control must then fetch the actual list out-of-band:

```go
// tailcfg/c2ntypes.go — response to control's "GET /vip-services" c2n request
type C2NVIPServicesResponse struct {
    VIPServices []*VIPService
    ServicesHash string // matches the latest Hostinfo.ServicesHash
}
```

Because every client headscale supports (≥ v1.80, cap ≥ 113 > 108) uses this
model, **there is no fallback**: supporting services *requires* implementing
the c2n `GET /vip-services` fetch.

### 3.3 What control pushes back to nodes

Two `NodeCapMap` capabilities carry the control→client half:

```go
// To an APPROVED HOST: "you may host these services, and here are their VIPs."
// Value type: ServiceIPMappings
NodeAttrServiceHost NodeCapability = "service-host"
type ServiceIPMappings map[ServiceName][]netip.Addr

// To a CONSUMER, per service it may access ("services/" + serviceName):
//   key:   NodeAttrPrefixServices + "svc:foo"   ("services/svc:foo")
//   value: ServiceDetails
NodeAttrPrefixServices NodeCapabilityPrefix = "services/"
type ServiceDetails struct {
    Name        ServiceName
    DisplayName string
    Addrs       []netip.Addr      // the VIP addresses
    Ports       []ProtoPortRange
    Actions     []ServiceAction   // optional UI hints
}
```

Plus, for desktop UIs to show the services menu, the tailnet-wide capability
`CapabilityServicesInDesktopClients` is advertised.

### 3.4 Routing the VIP

Consumer traffic to a VIP is delivered to an **Active host** node. The natural
mapping onto the existing protocol is: the host node carries the VIP as an
**AllowedIP** (a `/32` + `/128` it "owns", exactly like a subnet route), and the
**packet filter** permits consumers → VIP:ports. The host's `tailscaled` then
maps inbound VIP:port traffic to its local serve config.

## 4. Goals / Non-goals

**Goals**

- Allocate and persist stable VIP address pairs per service.
- Fetch host service advertisements via c2n on `ServicesHash` change.
- Authorize *who may host* and *who may access* a service via policy.
- Push `service-host` caps to hosts and `services/<name>` caps to consumers.
- Route VIP traffic to active host(s); generate the right packet-filter rules.
- Resolve the service's MagicDNS name to its VIP.
- CLI / gRPC to list and inspect services.
- Support multi-host HA with `Active`/drain/advertise (can be a later phase).

**Non-goals (initially)**

- Funnel for services (public ingress) — not possible on headscale.
- Service-level metrics/analytics, the full desktop "actions" UX beyond passing
  through `ServiceDetails`.
- Any change to the per-node `tailscale serve` already implemented.

## 5. End-to-end flow (control-plane perspective)

1. Operator defines, in policy, **who may host** `svc:foo` and **who may
   access** it (§9).
2. A permitted node runs `tailscale serve --service=svc:foo --https=443 3000`.
   Its `Hostinfo.ServicesHash` changes on the next map request.
3. Headscale detects the hash change in `UpdateNodeFromMapRequest` and issues a
   **c2n `GET /vip-services`** to that node (§7). It receives
   `C2NVIPServicesResponse{VIPServices:[{Name:svc:foo,Ports,Active}], …}`.
4. Headscale **allocates** (or looks up) the VIP pair for `svc:foo` and records
   this node as a host (§8, §10).
5. Headscale recomputes maps:
   - The host gets `service-host` cap = `{svc:foo: [VIP4, VIP6]}` and the VIP in
     its AllowedIPs (if Active).
   - Each permitted consumer gets `services/svc:foo` cap = `ServiceDetails{…}`,
     a packet-filter rule allowing it → VIP:ports, and a MagicDNS record for the
     service name → VIP (§11–§13).
6. Consumer resolves `foo.<base_domain>` → VIP, connects; WireGuard routes to the
   active host; the host's `tailscaled` serves it.
7. On host changes (new/removed host, drain/advertise, node offline), headscale
   updates `Active` hosts and re-pushes affected maps (§14).

## 6. Affected subsystems (map)

| Subsystem | File(s) | Change |
|---|---|---|
| c2n transport | `hscontrol/noise.go`, `hscontrol/app.go`, `hscontrol/types/change/change.go`, `hscontrol/mapper/` | Send c2n via `PingRequest`; new POST callback handler; response correlation |
| Hostinfo | `hscontrol/state/state.go` (`UpdateNodeFromMapRequest`) | Detect `ServicesHash` change → trigger c2n fetch |
| Data model | `hscontrol/db/` (new model + migration), `hscontrol/types/` | `Service` table; host membership |
| VIP allocation | `hscontrol/db/ip.go`, `hscontrol/state/` | Allocate/persist VIP pairs |
| Policy | `hscontrol/policy/v2/` | `svc:` target; host/consumer authorization; caps; packet filter |
| State / NodeStore | `hscontrol/state/state.go`, `node_store.go`, `hscontrol/types/change/` | Track services; fan-out changes |
| Mapper | `hscontrol/mapper/builder.go`, `mapper.go` | VIP AllowedIPs, packet filter, service caps |
| MagicDNS | `hscontrol/dns/`, `hscontrol/mapper/mapper.go`, `hscontrol/util/dns.go` | service name → VIP records |
| CLI / API | `cmd/headscale/`, `hscontrol/grpcv1.go`, `proto/headscale/v1/` | `services` command group |

## 7. The c2n primitive (foundational)

This is the only fundamentally new capability and should be built and proven
**first** (Phase 0), independently of services, because it is reusable (e.g.
`/whoami`, `/update-health`, posture, OIDC id-token — all of headscale's other
`NotImplementedHandler` stubs in `noise.go`).

### 7.1 How c2n actually works (verified in tailscale source)

c2n is **not** control dialing the node (nodes are behind NAT). It rides on
`tailcfg.PingRequest`, delivered to the node inside a `MapResponse`:

1. Control puts a `PingRequest` in a `MapResponse.PingRequest` for the target
   node:
   - `Types: "c2n"`
   - `Payload: <raw HTTP/1.1 request>` (e.g. `GET /vip-services HTTP/1.1\r\n…`),
     parsable by `http.ReadRequest`.
   - `URL: <unique control callback URL>` — where the node POSTs the response.
   - `URLIsNoise` is ignored for c2n; the node always replies over Noise.
2. The node (`feature/c2n.answerC2NPing`) parses `Payload`, runs it against its
   local c2n handler (which has `GET /vip-services` registered in
   `ipn/ipnlocal/serve.go`), serializes the HTTP response, and **POSTs it to
   `pr.URL`** over the Noise transport.
3. Control receives the POST, parses the HTTP response body, and gets the
   `C2NVIPServicesResponse` JSON.

### 7.2 What headscale already has

Headscale already sends `PingRequest`s and receives ping replies — this is the
plumbing c2n reuses:

- `hscontrol/types/change/change.go:50` — `Change.PingRequest *tailcfg.PingRequest`,
  always targeted to a specific node (`Change.TargetNode`); `change.PingNode(...)`
  at `change.go:511`.
- The mapper applies `Change.PingRequest` to the outgoing `MapResponse` for that
  node (today used for HEAD connectivity pings, e.g. from `hscontrol/debug.go`
  `doPing` at `:513`).
- `hscontrol/noise.go:319` `PingResponseHandler` + route `r.Head("/machine/ping-response", …)`
  at `app.go:589` — the existing **HEAD** reply path, authenticated by an
  unguessable ping ID in the URL.

### 7.3 What headscale must add

- **Send**: a helper that builds a c2n `PingRequest` (`Types:"c2n"`, `Payload`
  = a serialized `http.Request`, `URL` = a unique callback) and dispatches it to
  a node via the existing `Change.PingRequest` path.
- **Receive**: a new **POST** route (e.g. `POST /machine/c2n-response/{id}`)
  on the Noise router (`noise.go`) that reads the HTTP response the node sends
  back, matches it to the pending request by `id`, and hands the parsed body to
  the caller. Authentication is the same unguessable-id model as the HEAD ping.
- **Correlation**: an in-memory map of `id → pending request` (channel/future)
  with timeouts, since the reply arrives asynchronously on a *different* Noise
  connection (the node's). This must be safe across the batcher/multiple
  headscale goroutines.
- **Helper API**: `state.C2N(ctx, nodeID, httpReq) (*http.Response, error)` so
  callers (the services code, and future features) get a synchronous-looking
  request/response.

### 7.4 c2n risks

- **Correlation lifetime**: the node may be offline, slow, or reconnect on a new
  Noise session between the ping and the reply — needs robust timeout/cleanup.
- **Idempotency**: PingRequests with a repeated `URL` are ignored by the client;
  callback URLs must be unique per attempt.
- **Delivery coupling**: the PingRequest is delivered in the node's map stream,
  so the node must have an active long-poll. Define behavior when it does not
  (defer until reconnect, or fail).

## 8. Data model & persistence

A new GORM model and migration (following the rules in `AGENTS.md` §"Database
Migration Rules": append-only, ID format `YYYYMMDDHHMM-short-description`, never
disable FKs, never rename referenced columns).

Proposed tables:

```text
services
  id            PK
  name          string   // "svc:foo", unique
  vip_v4        string   // allocated IPv4 (nullable until allocated)
  vip_v6        string   // allocated IPv6
  display_name  string
  created_at, updated_at

service_hosts            // many hosts per service (HA)
  service_id    FK -> services.id
  node_id       FK -> nodes.id
  active        bool     // mirrors VIPService.Active
  ports         json     // []ProtoPortRange last reported
  (unique service_id + node_id)
```

Notes:

- `name` and the VIP pair are stable for the life of the service (the VIP must
  not change under consumers). Decide GC policy: when the last host stops
  advertising, keep the row (stable VIP on return) vs. delete after a grace
  period. Recommend **keep** (stable identity) with an explicit
  `headscale services delete`.
- Persisted VIPs must be excluded from node IP allocation (§9).

Reference patterns: `hscontrol/db/node.go`, `users.go`, `preauth_keys.go` for
GORM model + CRUD; `hscontrol/db/db.go` (migrations array near `:962`,
`migrationsRequiringFKDisabled` is frozen — do **not** add to it).

## 9. VIP address allocation

VIP addresses must be real tailnet addresses so existing WireGuard routing and
the packet filter work unchanged.

**Recommendation**: allocate VIPs from the **same `PrefixV4`/`PrefixV6`** space
used for nodes (`hscontrol/db/ip.go`), but persisted in `services` so the node
allocator and the service allocator never hand out the same address.

- Pros: no new routing/filtering rules; VIPs are ordinary tailnet IPs;
  consumers already accept them.
- Cons: must make the node IP allocator and VIP allocator share a single source
  of truth for "address in use" (a combined uniqueness check, or a small
  dedicated sub-range carved out of the prefix for services).

**Alternative**: carve a dedicated sub-prefix (e.g. the top of `PrefixV4`) for
VIPs only. Cleaner separation, but reduces node space and needs care for
operators with custom prefixes. **Open question (§17).**

Whatever the choice, allocation must be **idempotent per service name** (the
same `svc:foo` always resolves to the same VIP) and survive restarts.

## 10. Policy / ACL integration

This is the highest-blast-radius change (policy v2 is load-bearing). Two
distinct authorization questions:

### 10.1 Who may *host* a service

Tailscale models this with service/app owners. Proposed minimal approach: a
policy construct that maps `svc:foo` → a set of host principals (tags / users /
groups), analogous to **tag owners** (`hscontrol/policy/v2`). Only nodes in that
set may advertise `svc:foo`; an advertisement from a non-permitted node is
rejected (the service is simply not created/hosted, and a warning is logged).

When a permitted host advertises, headscale emits the `service-host`
(`ServiceIPMappings`) cap to that node (§11).

### 10.2 Who may *access* a service

ACL/grant rules must accept `svc:foo` as a **destination**. This needs a new
alias/target type in policy v2 (`hscontrol/policy/v2/types.go` — alongside tag,
group, autogroup, host targets) that resolves to the service's **VIP
addresses + ports**. From there, the existing packet-filter generation produces
`FilterRule`s `src → VIP:ports`, and the per-consumer `services/svc:foo`
(`ServiceDetails`) cap is emitted (§11).

### 10.3 Capability compilation

Per-node caps are compiled in `hscontrol/policy/v2/compiled.go`
(`compileNodeAttrs`, ~`:144`) and merged into `NodeCapMap` in the mapper
(`builder.go` ~`:280-293`). Service caps slot in here:

- hosts → `service-host` entries,
- consumers → `services/<name>` entries,
- tailnet-wide → `CapabilityServicesInDesktopClients`.

The current code **strips** unknown/unsupported caps in
`hscontrol/policy/v2/tailnet_state_caps.go`; these new caps must be allowed
through.

### 10.4 Open policy questions

- Exact policy syntax for service owners and for `svc:` in ACLs — should mirror
  Tailscale's grant/ACL syntax for portability of policy files.
- Interaction with **tags-as-identity** (`AGENTS.md`): can a tagged node host a
  service? (Expected: yes; authorization is by the host set.)

## 11. State / NodeStore & change propagation

- Track services and their hosts in `State` (`hscontrol/state/state.go`), the
  central coordinator, with the `NodeStore`
  (`hscontrol/state/node_store.go`) continuing to own per-node snapshots.
- A change to service membership/active-set must fan out **PolicyChange-like**
  recomputation to *all affected consumers and hosts* (not just the advertising
  node) — reuse the `types/change` mechanism and `State.Change()`.
- The trigger chain: `UpdateNodeFromMapRequest` sees a new
  `Hostinfo.ServicesHash` → enqueue a c2n fetch (§7) → on response, diff against
  `service_hosts`, persist, and emit changes.
- Be mindful of the hot path: `NodeStore` writes rebuild a snapshot; service
  membership changes should batch like policy recomputes
  (cf. recent "coalesce duplicate policy recomputes" work).

## 12. Mapper / map generation

In `hscontrol/mapper/`:

- **Host node**: when Active, add the service VIPs to the host's `AllowedIPs`
  (the same place subnet-router/exit routes are expressed on a `tailcfg.Node`),
  and add the `service-host` cap.
- **Consumer**: add `services/<name>` caps and ensure the **packet filter**
  (`tailcfg.PacketFilter`) includes `consumer → VIP:ports` rules generated by
  policy (§10.2).
- **DNS**: include the service-name → VIP record (§13).

The builder already has the seams: `WithDNSConfig` (`builder.go:157`), the
per-peer `CapMap` merge (`builder.go:~280-293`), and the packet-filter
attachment. Services add to each.

## 13. MagicDNS for services

Service names must resolve to their VIPs for consumers. Today MagicDNS records
are generated in `hscontrol/mapper/mapper.go` (`generateDNSConfig`) and
`hscontrol/util/dns.go`. Add per-consumer records mapping the service's FQDN
(`<svc-label>.<base_domain>`) → its VIP `Addrs`, scoped to consumers permitted
to see the service (so service existence isn't leaked tailnet-wide). Confirm
the exact name form Tailscale expects for service DNS during implementation.

## 14. High availability (later phase)

- Multiple hosts per service; control marks a subset `Active` and only Active
  hosts carry the VIP in AllowedIPs and receive new connections.
- `tailscale serve drain svc:foo` / `advertise svc:foo` change which hosts are
  Active; these surface to control as `ServicesHash` changes → c2n refetch →
  `Active` recompute.
- Define the control policy for choosing Active hosts (all healthy hosts?
  primary + standby?) and the failover trigger (host offline in `NodeStore`).
- This is distributed, timing-sensitive logic — the most flake-prone area; keep
  it out of the first functional milestone.

## 15. CLI / gRPC / API

Add a `services` resource mirroring existing ones (`hscontrol/grpcv1.go`,
`proto/headscale/v1/`, `cmd/headscale/`):

- `headscale services list` (name, VIPs, hosts, active).
- `headscale services delete svc:foo` (free the VIP / row).
- Possibly `headscale services hosts svc:foo`.

Proto changes land in a **separate commit** from their consumers and require
`make generate` (`AGENTS.md` §Code Conventions).

## 16. Security considerations

- **Host authorization**: only policy-permitted nodes may host a service;
  reject/ignore unauthorized advertisements (don't allocate a VIP for them).
- **Access authorization**: consumers only learn of (`services/<name>` cap +
  MagicDNS) and can reach (packet filter) services their policy permits — no
  tailnet-wide leakage of service existence.
- **VIP integrity**: VIPs are allocated by control only; a node cannot choose
  its VIP. The `service-host` cap tells the host which addresses are authoritative.
- **c2n authentication**: the callback URL carries an unguessable id (as the
  existing ping path does); the reply arrives over the authenticated Noise
  session bound to the node's machine key. Validate the responding node matches
  the targeted node.
- **Tags-as-identity** invariants (`AGENTS.md`) must be respected when deciding
  host eligibility.

## 17. Open questions / unknowns

1. **VIP range**: shared node prefix vs. dedicated sub-range (§9).
2. **Service DNS name form**: exact FQDN/label Tailscale clients expect for a
   service (`<label>.<base_domain>`?), and whether a record type/format beyond
   A/AAAA is needed (§13).
3. **Policy syntax** for service owners and `svc:` ACL targets — match Tailscale
   grants for policy-file portability (§10.4).
4. **HA semantics**: control's algorithm for selecting Active hosts and failover
   (§14).
5. **c2n delivery when node is offline / mid-reconnect** (§7.4).
6. **GC policy** for services with no hosts (§8).
7. **Protocol completeness**: the control side of VIP services is **not publicly
   specified**; this design is reconstructed from `tailcfg` types and the client
   source. Expect to discover additional requirements (e.g. exact cap merge
   semantics, desktop-UI expectations) during a spike. **This is the largest
   uncertainty.**

## 18. Phased delivery plan

- **Phase 0 — c2n primitive** (`§7`). Build and test control→node c2n on the
  existing ping plumbing, with a throwaway endpoint (e.g. fetch `/whoami` or
  `/vip-services` and log it). De-risks the hardest unknown; reusable for other
  stubs. *Ship/validate before anything else.*
- **Phase 1 — Single-host services, no policy nuance**: data model + VIP
  allocation + `ServicesHash` detection + c2n fetch + host `service-host` cap +
  VIP AllowedIPs + MagicDNS + a permissive access rule. Prove a consumer can
  reach a one-host service end-to-end.
- **Phase 2 — Policy integration**: `svc:` ACL target + host-owner
  authorization + correct per-consumer caps and packet-filter rules.
- **Phase 3 — HA**: multiple hosts, `Active`, drain/advertise, failover.
- **Phase 4 — CLI/gRPC polish + docs.**

Each phase: unit tests + an integration test (Docker + real client) following
`integration/README.md`. Note that, as with the serve cert flow, some
end-to-end behavior may be only partially CI-testable; call those out per phase.

## 19. Effort & risk assessment

Substantially larger and riskier than the per-node serve work (which was one
small subsystem). Services touches the **db (migrations), state, the policy v2
engine, the mapper, DNS, a new c2n primitive, and the CLI/API** at once, against
an **under-specified control protocol**. Rough order: **several weeks** for
someone fluent in the codebase, and it warrants maintainer buy-in and a spike
(Phase 0) before committing to the full build. The policy-engine changes and HA
semantics are the highest-risk areas; the c2n primitive is new but
de-risked by the existing `PingRequest` plumbing.

## 20. Alternatives considered

- **Do nothing / document as unsupported** — current state. Keeps headscale
  simpler; users wanting services stay on SaaS.
- **Per-node serve only** (already shipped) — covers many "expose a service"
  use cases without VIPs, HA, or discovery. For some users this is enough; this
  proposal is for those who need stable, node-independent service identity.
- **A headscale-specific service abstraction** (not protocol-compatible) — would
  avoid the c2n/cap complexity but break client compatibility (clients wouldn't
  see services). Rejected: the value is in working with the stock client.

## Appendix A — Key tailscale types (v1.98.3)

- `tailcfg.ServiceName`, `tailcfg.VIPService`, `tailcfg.ServiceIPMappings`,
  `tailcfg.ServiceDetails`, `tailcfg.ServiceAction` (`tailcfg/tailcfg.go`).
- `tailcfg.Hostinfo.ServicesHash` (`tailcfg/tailcfg.go`).
- `NodeAttrServiceHost = "service-host"`, prefix `NodeAttrPrefixServices = "services/"`,
  `CapabilityServicesInDesktopClients` (`tailcfg/tailcfg.go`).
- `tailcfg.PingRequest` (`Types`, `Payload`, `URL`, `URLIsNoise`)
  (`tailcfg/tailcfg.go`).
- `tailcfg.C2NVIPServicesResponse` (`tailcfg/c2ntypes.go`).
- Node side: `feature/c2n/c2n.go` (`answerC2NPing`), `ipn/ipnlocal/serve.go`
  (`RegisterC2N("GET /vip-services", …)`), `ipn/ipnlocal/c2n.go` (handler mux),
  `control/controlclient/direct.go` (`answerPing`).

## Appendix B — Key headscale anchors

- c2n / ping: `hscontrol/noise.go` (`PingResponseHandler:319`, Noise routes,
  `/c2n` stub `:211`), `hscontrol/app.go:589` (`/machine/ping-response`),
  `hscontrol/types/change/change.go` (`PingRequest:50`, `PingNode:511`),
  `hscontrol/debug.go:513` (`doPing`).
- Hostinfo sync: `hscontrol/state/state.go` (`UpdateNodeFromMapRequest`).
- Policy: `hscontrol/policy/v2/compiled.go` (`compileNodeAttrs:144`),
  `types.go`, `tailnet_state_caps.go`.
- Mapper: `hscontrol/mapper/builder.go` (`WithDNSConfig:157`, CapMap merge
  `~280-293`), `mapper.go` (`generateDNSConfig`).
- IP allocation: `hscontrol/db/ip.go`.
- Migrations/models: `hscontrol/db/db.go` (`~962`), `node.go`, `users.go`.
- DNS: `hscontrol/util/dns.go`, `hscontrol/mapper/mapper.go`.
- API: `hscontrol/grpcv1.go`, `proto/headscale/v1/`, `cmd/headscale/`.
