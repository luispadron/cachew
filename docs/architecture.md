# Architecture: the request path

How an HTTP request flows through Cachew, from client to upstream. For how
cached objects are stored across multiple backends, see
[tiering.md](tiering.md).

```
┌────────┐   ┌──────┐   ┌──────────────┐   ┌──────────────────┐   ┌───────┐
│ Client │──▶│ OPA  │──▶│ Interceptors │──▶│ Strategy (route) │──▶│ Cache │
└────────┘   └──────┘   └──────────────┘   └────────┬─────────┘   └───────┘
                                                    │ miss
                                                    ▼
                                              ┌──────────┐
                                              │ Upstream │
                                              └──────────┘
```

- **OPA** authorizes every request (`internal/opa`).
- **Strategies** are protocol-aware handlers (Git, GitHub releases, Go
  modules, Hermit, Artifactory, host, proxy). Each is registered against the
  mux at config load (`internal/config/config.go`); strategies implementing
  `strategy.Interceptor` wrap the mux instead, so they can inspect the raw
  request line.
- Each strategy receives a **namespaced view** of the cache
  (`cache.Namespace`), so strategies never collide on keys.
- Most strategies use the shared handler (`internal/strategy/handler`), which
  implements the cache-or-fetch loop: look up the key, serve from cache on a
  hit, and on a miss stream the upstream response to the client and the cache
  simultaneously. Strategies are not limited to the Cache — e.g. the Git
  strategy also maintains bare clones and serves packs directly.
- The **api-v1 strategy** (`internal/strategy/apiv1.go`) exposes the composed
  cache itself over HTTP (`/api/v1/object/{namespace}/{key}`, …). This is the
  API that `client/` and the `Remote` cache implementation speak, and it is
  what makes instance-to-instance tiering possible.

Strategies see a single `Cache`; whether it is one backend or several tiers
composed together is invisible to them. That composition is described in
[tiering.md](tiering.md).

## Ranged and parallel downloads

Large objects — git snapshots and delta bundles in particular — are downloaded
as many concurrent byte-range requests rather than one stream
(`client/parallel_get.go`):

- **Discovery**: the first chunk is requested with a `Range` header. The
  response reveals the object's total size and ETag. Servers that ignore the
  range, objects that fit in one chunk, objects without an ETag, or
  concurrency 1 all fall back to a single full read.
- **Pinning**: every subsequent chunk carries the discovery ETag, so an object
  rewritten mid-download is rejected rather than spliced together from two
  revisions.
- **Streaming**: `ParallelGetReader` reassembles out-of-order chunks into a
  sequential stream with bounded buffering, so consumers (e.g. `tar`
  extraction during `cachew git restore`) overlap with the transfer.

The server side cooperates: the git strategy advertises the ETag and snapshot
metadata headers on both full (200) and ranged (206) responses, so a parallel
client learns everything it needs from its discovery chunk
(`internal/strategy/git/snapshot.go`).

The same mechanism is reused inside the S3 backend
(`internal/cache/s3_parallel_get.go`): a single S3 stream is limited to a
fraction of the available bandwidth, so whole-object and large ranged reads
fan out into parallel sub-range requests against the pinned object revision.

Ranged reads interact with cache tiering — a partial body must never be
backfilled into a lower tier as if it were the whole object; see
[tiering.md](tiering.md).

## The Cache contract

Strategies and cache backends all program against the same interface
(`internal/cache/api.go`), whose key guarantees are:

- Expired objects are never returned.
- Objects are invisible until completely written and closed.
- `Delete` is atomic; missing objects invalidate successfully.
- Conditional options (`If-None-Match`, `If-Match`, `If-Range`, `Range`) are
  evaluated against the stored ETag with RFC 9110 semantics.
