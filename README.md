<div align="center">
  <img src="assets/logo.JPG" alt="Kioshun Logo" width="200"/>

  # Kioshun - In-Memory Cache for Go

  *"kee-oh-shoon" /kiːoʊʃuːn/*

  [![Go Version](https://img.shields.io/badge/Go-1.24+-blue.svg)](https://golang.org)
  [![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
  [![CI](https://github.com/unkn0wn-root/kioshun/actions/workflows/test.yml/badge.svg)](https://github.com/unkn0wn-root/kioshun/actions)


  *Thread-safe, sharded in-memory cache for Go - with an optional peer-to-peer cluster backend*
</div>

## Index

- [What is Kioshun?](#what-is-kioshun)
- [Cluster (Overview)](#cluster-overview)
- [Cache Architecture](ARCHITECTURE.md)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [API Reference](#api-reference)
- [HTTP Middleware](MIDDLEWARE.md)
- [Benchmark Results](#benchmark-results)

## What is Kioshun?

Kioshun is a thread-safe in-memory cache for Go. You can run it as a simple local cache or turn on the peer-to-peer cluster when you want replicas across hosts, but the core idea stays the same: keep hot data right next to your code and make lookups predictable.

Under the hood the cache is split into shards (by default four per CPU, capped at 256). Each shard owns its own map, intrusive LRU list, and `sync.RWMutex`, so hot keys never contend with the entire cache. Routing is done through a tiny, type-aware hasher - short strings stick with allocation-free FNV-1a, longer strings flow through xxHash64, and integers get a quick avalanche mix, so finding the shard is basically a bitmask.

Eviction policies are picked per cache, not per build. You can choose LRU, LFU, FIFO, or the default AdmissionLFU. AdmissionLFU samples a handful of candidates, weighs their age and frequency, and only swaps entries when its Bloom-filter doorkeeper, tiny Count-Min Sketch, and scan detector agree that the new item is actually worth keeping. When pressure increases, the cache automatically adjusts how aggressively it admits new keys so the hottest data keeps winning.

## Cluster Overview

> [!NOTE]
> Clustering is fully **optional**. If you don’t enable the cluster, Kioshun runs as a standalone, in‑process in‑memory cache.

Kioshun’s cluster turns each service instance (your application process) into a **small, self-managing, peer-to-peer cache node**. It shards keys via weighted rendezvous, replicates writes with configurable RF/WC, heals via hinted handoff and backfill, and serves hot data from local memory whenever your instance owns it. In practice the cache runs in-process, so every service node embeds a cluster member that discovers peers (*Seeds*), forms a weighted rendezvous ring, and keeps membership fresh through gossip. Point each node at a few reachable seeds; gossip distributes the full peer list from there. Reads route to the primary owner, and read-through population uses single-flight leases.

```
┌─────────────┐      Gossip + Weights      ┌─────────────┐
│  Service A  │◀──────────────────────────▶│  Service B  │
│  + Node     │◀───────────▶◀───────────▶  │  + Node     │
└──────┬──────┘                            └──────┬──────┘
       │   Owner‑routed Get/Set (RF)              │
       └──────────────▶◀──────────────────────────┘
                  Service C + Node
```

Small multinode example:

```
# on each server
CACHE_BIND=:4443
CACHE_PUBLIC=srv-a:4443   # srv-b / srv-c on others
CACHE_SEEDS=srv-a:4443,srv-b:4443,srv-c:4443
CACHE_AUTH=supersecret

// in code
local := cache.NewWithDefaults[string, []byte]()
cfg := cluster.Default()
cfg.BindAddr = os.Getenv("CACHE_BIND")
cfg.PublicURL = os.Getenv("CACHE_PUBLIC")
cfg.Seeds = strings.Split(os.Getenv("CACHE_SEEDS"), ",")
cfg.ReplicationFactor = 3; cfg.WriteConcern = 2
cfg.Sec.AuthToken = os.Getenv("CACHE_AUTH")
node := cluster.NewNode[string, []byte](cfg, cluster.StringKeyCodec[string]{}, local, cluster.BytesCodec{})
if err := node.Start(); err != nil { panic(err) }
dc := cluster.NewDistributedCache[string, []byte](node)
```

Only a subset of nodes need to appear in `CACHE_SEEDS`. The list is purely for bootstrap - include a few stable peers so new processes can reach at least one live seed, then gossip distributes the rest of the membership automatically, whether you run three caches or twenty.

> See **CLUSTER.md** for more details.

## Installation

```bash
go get github.com/unkn0wn-root/kioshun
```

## Quick Start

```go
package main

import (
    "fmt"
    "time"

    cache "github.com/unkn0wn-root/kioshun"
)

func main() {
    // Create cache with default configuration
    c := cache.NewWithDefaults[string, string]()
    defer c.Close()

    // Set with default TTL (30 min)
    c.Set("user:123", "David Nice 1", cache.DefaultExpiration)

    // Set with no expiration
    c.Set("user:123", "David Nice 2", cache.NoExpiration)

    // Set value with custom TTL
    c.Set("user:123", "David Nice 3", 5*time.Minute)

    // Get value
    if value, found := c.Get("user:123"); found {
        fmt.Printf("User: %s\n", value)
    }

    // Get cache statistics
    stats := c.Stats()
    fmt.Printf("Hit ratio: %.2f%%\n", stats.HitRatio*100)
}
```

## Configuration

### Basic Configuration

```go
config := cache.Config{
    MaxSize:         100000,             // Maximum number of items
    ShardCount:      16,                 // Number of shards (0 = auto-detect)
    CleanupInterval: 5 * time.Minute,    // Cleanup frequency
    DefaultTTL:      30 * time.Minute,   // Default expiration time
    EvictionPolicy:  cache.AdmissionLFU, // Eviction algorithm (default)
    StatsEnabled:    true,               // Enable statistics collection
}

cache := cache.New[string, any](config)
```

## API Reference

```go
cache.Set(key, value, ttl time.Duration) error
cache.SetWithCallback(key, value, ttl, callback func(key, value)) error
cache.Get(key) (value, found bool)
cache.GetWithTTL(key) (value, ttl time.Duration, found bool)
cache.Keys() []K
cache.Clear()
cache.Delete(key) bool
cache.Exists(key) bool
cache.Size() int64
cache.Stats() Stats
cache.TriggerCleanup()
cache.Close() error
```

### Statistics

```go
type Stats struct {
    Hits        int64   // Cache hits
    Misses      int64   // Cache misses
    Evictions   int64   // Evictions (all policies)
    Expirations int64   // TTL expirations
    Size        int64   // Current items
    Capacity    int64   // Maximum items
    HitRatio    float64 // Hit ratio (0.0-1.0)
    Shards      int     // Number of shards
}
```

## HTTP Middleware

Kioshun provides HTTP middleware out-of-the-box.

```go
config := cache.DefaultMiddlewareConfig()
config.DefaultTTL = 5 * time.Minute
config.MaxSize = 100000

middleware := cache.NewHTTPCacheMiddleware(config)
defer middleware.Close()

http.Handle("/api/users", middleware.Middleware(usersHandler))
```

**Features:**
- Framework agnostic (net/http, Gin, Echo, Chi, Gorilla Mux)
- Defaults (Cache-Control + Expires support)
- Pattern-based cache invalidation (enable via `SetPathExtractor` + a path-derived key)
- Multiple eviction strategies (LRU, LFU, FIFO, AdmissionLFU)
- Monitoring with callbacks

> See **[MIDDLEWARE.md](MIDDLEWARE.md)** for complete documentation, examples, and advanced configuration.

## Benchmark Results

Latest benchmark run (Apple M4 Max, Go 1.24.7):
- `SET`: 100,000,000 ops/sec · 75.55 ns/op · 41 B/op · 3 allocs/op
- `GET`: 231,967,180 ops/sec · 25.87 ns/op · 31 B/op · 2 allocs/op
- `Real-World`: 52,742,550 ops/sec · 65.25 ns/op · 48 B/op · 3 allocs/op

Full suite and methodology: [_benchmarks/README.md](_benchmarks/README.md)
