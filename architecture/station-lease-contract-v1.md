# Station Lease Contract v1

## Purpose

A station is exclusive physical hardware. A `StationLease` prevents multiple
executors, retries, or recovery workers from controlling the same DUT at the
same time.

Jenkins labels and process-local locks are insufficient because executors may
crash, restart, or lose network connectivity.

## Lease record

```yaml
lease_id: lease_01JEXAMPLE
station_id: station_pi_01
run_id: run_01JEXAMPLE
owner_id: worker_7f13
status: active
fencing_token: 42
acquired_at: "2026-10-01T10:30:00Z"
heartbeat_at: "2026-10-01T10:30:20Z"
expires_at: "2026-10-01T10:31:00Z"
released_at: null
reason: validation_run
```

## Required semantics

### Acquire

`acquire(station_id, run_id, owner_id, ttl)` is atomic.

It succeeds only when:

- station state permits leasing;
- no active unexpired lease exists;
- requested run is eligible;
- owner identity is authorized.

On success:

- create a new lease ID;
- increment and persist the station fencing token;
- set station state to `leased`;
- return the complete lease record.

### Renew

`renew(lease_id, owner_id, fencing_token, ttl)` is atomic.

It succeeds only when:

- lease is active and unexpired;
- owner and token match;
- station has not been quarantined or administratively revoked.

Renew updates heartbeat and expiry but never changes the fencing token.

### Release

`release(lease_id, owner_id, fencing_token)` is idempotent.

It:

- marks the lease released;
- records release time;
- moves the station to `idle` only after station cleanup succeeds;
- otherwise moves the station to `recovery` or `quarantine`.

### Expire

A lease becomes expired when the authoritative clock is past `expires_at`.

Expiry permits a new lease only after a recovery worker establishes that:

- previous executor actions have stopped or are fenced;
- attached tools are closed;
- DUT and switching hardware are in a defined safe state.

## Fencing token

The fencing token is a monotonically increasing integer scoped to one station.

Every destructive or state-changing adapter operation must include the current
token:

- flash/write media;
- power on/off/cycle;
- reset;
- relay or multiplexer switching;
- recovery provisioning.

The station-side controller rejects a token lower than the highest token it has
accepted. This prevents a stale executor from resuming after its lease expired
and a new run acquired the station.

TTL alone is not sufficient protection.

## Station states

```text
offline
idle
leased
running
recovery
quarantine
```

Allowed transitions:

```text
offline -> idle
idle -> leased
leased -> running
running -> recovery
running -> idle
recovery -> idle
recovery -> quarantine
quarantine -> idle   # manual or approved remediation only
any -> offline
```

State transitions and lease updates must be committed atomically when they are
part of the same operation.

## Heartbeat and TTL defaults

Initial MVP defaults:

- lease TTL: 60 seconds;
- heartbeat interval: 20 seconds;
- renewal failure budget: two attempts within remaining TTL;
- maximum lease duration: run timeout plus explicit recovery budget.

Long-running tests continue renewing. They must not acquire an unbounded lease.

## Recovery ownership

Recovery uses either:

1. the still-valid run lease; or
2. a new recovery lease with a newer fencing token after expiry/revocation.

A normal validation executor must stop when:

- renew fails definitively;
- its token is fenced;
- the station enters quarantine;
- run cancellation is confirmed.

## Retry behavior

A retry creates a new `ValidationRun`. It may reuse the same release and
station, but it acquires a new lease and fencing token.

The previous run remains immutable and records:

- failure category;
- lease termination reason;
- partial evidence;
- recovery actions.

## Failure handling

Lease-related failures are infrastructure errors, not DUT test failures:

- `lease_conflict`;
- `lease_expired`;
- `lease_revoked`;
- `fencing_rejected`;
- `heartbeat_failed`;
- `station_quarantined`.

The report must distinguish these from deterministic validation `FAIL`.

## MVP implementation options

Preferred:

- relational database transaction with unique active-lease constraint;
- authoritative server time;
- optimistic version or row lock;
- station controller token validation.

Acceptable single-station prototype:

- SQLite transaction on the controller;
- OS file lock only as an additional local guard;
- persisted fencing token.

Not acceptable:

- Jenkins label alone;
- in-memory boolean;
- lease file without atomic compare-and-set;
- expiry without fencing.

## Audit events

Append an audit event for:

- acquire success/failure;
- renew failure;
- release;
- expiry;
- revocation;
- token rejection;
- state transition;
- manual quarantine clear.

Audit entries include station, run, lease, owner, token, timestamp, and reason,
but never credentials.
