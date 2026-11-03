# Project Moved

**This fork is archived and no longer maintained.**

Active development has moved to **[IPv4 Counter — Optimized](https://github.com/Kukushkasus/ipv4-counter-optimized)**.

The new repository contains an independently maintained implementation with improved performance, an adaptive bitmap, and additional tests.

**Original project:** [alexproskurov/ip-addr-counter](https://github.com/alexproskurov/ip-addr-counter)

---

# IPv4 Counter — Optimized

A performance-focused IPv4 uniqueness counter written in Go. It processes large, newline-delimited text files using two self-contained counting strategies: a concurrent atomic bitmap and a memory-efficient adaptive bitmap.

This is an **independently maintained variant** of [alexproskurov/ip-addr-counter](https://github.com/alexproskurov/ip-addr-counter), developed to explore different parsing, concurrency, and memory tradeoffs.

## Features

- **Custom IPv4 parser:** strict dotted-decimal validation without `net.ParseIP` or third-party parsing libraries.
- **Concurrent `atomic` mode:** bounded workers, reusable read buffers, a shared bitmap with atomic updates, and counting on first insertion.
- **Adaptive `bitset` mode:** allocates 8 KiB pages on demand and switches to a dense bitmap for widely distributed addresses.
- **No third-party counting or bitmap dependencies.**
- Regression tests, race-detector coverage, distribution-based benchmarks, and optional CPU profiling.

Both modes count **unique valid IPv4 addresses**. Invalid and empty lines are skipped, including addresses with out-of-range octets or leading zeros in multi-digit octets. CRLF input and files without a trailing newline are supported.

## Requirements

- Go 1.25.1 or newer, as specified by `go.mod`.

## Usage

From the project directory:

```bash
go run . atomic ips.txt
go run . bitset ips.txt
```

Input: one IPv4 address per line, for example:

```text
192.168.1.1
8.8.8.8
192.168.1.1
```

Output:

```text
Unique IPs: 2
```

Optional CPU profiling:

```bash
go run . -cpuprofile cpu.prof atomic ips.txt
```

## How it works

### Atomic counter

The concurrent implementation reads the file in **4 MiB chunks** and sends buffers to workers, handling lines that cross chunk boundaries. Workers parse each address and set its bit in a shared **512 MiB IPv4 bitmap** using atomic compare-and-swap. A successful first insertion increments a worker-local counter; the final answer is the sum of those counters, without a full bitmap scan.

**Tradeoff:** higher throughput on tested workloads, but the dense bitmap requires substantial memory even when few unique addresses are present.

### Adaptive bitset counter

The single-threaded implementation initially stores bits in **8 KiB pages**, each covering one IPv4 `/16` range. Pages are allocated only for ranges that appear in the input. When addresses span more than **64 pages**, it converts to a dense bitmap while preserving the already-seen addresses.

**Tradeoff:** very low allocations for clustered addresses; widely scattered addresses can trigger a full 512 MiB allocation.

## Performance

Benchmarked on **AMD Ryzen 5 2600, Linux**, using **3 million input addresses per scenario**. Numbers are medians of five runs of the same distribution benchmark against the [original repository](https://github.com/alexproskurov/ip-addr-counter) at commit `be2802d`.

**Atomic mode (original → optimized):**

- 16 repeated addresses: **227.5 ms → 109.6 ms** (2.08× faster)
- Sequential unique addresses: **246.3 ms → 106.3 ms** (2.32× faster)
- Scattered unique addresses: **550.5 ms → 131.8 ms** (4.18× faster)

**Bitset mode (original → optimized):**

- 16 repeated addresses: **184.1 ms → 126.0 ms** (1.46× faster)
- Sequential unique addresses: **218.8 ms → 140.7 ms** (1.56× faster)
- Scattered unique addresses: **504.2 ms → 465.1 ms** (1.08× faster)

For the repeated and sequential inputs, adaptive-bitset allocations dropped from approximately **537 MB/op** to **598 KB/op** and **967 KB/op**, respectively. These are **total bytes allocated per operation (`B/op`)**, not peak RAM usage. Performance depends on hardware, input distribution, and system load.

Reproduce the distribution benchmarks:

```bash
go test -run '^$' -bench '^BenchmarkIPDistributions' -benchtime=10x -count=5 -benchmem ./counter
```

## Tests

```bash
go test -count=1 ./...
go test -race -count=1 ./...
go vet ./...
```

Tests exercise parsing, malformed input, duplicates, buffer boundaries, line endings, concurrent insertions, and adaptive-bitmap transitions.

## Origin and attribution

This project is an independent continuation of ideas and code from **[IP Address Counter by Oleksii Proskurov](https://github.com/alexproskurov/ip-addr-counter)**. The original repository and this optimized variant are maintained separately; neither replaces the other.

The original project is distributed under the **MIT License**. Its original copyright and license notice are retained in [`LICENSE`](LICENSE). See the Git history for the original and subsequent contributions.

## License

MIT — see [LICENSE](LICENSE).
