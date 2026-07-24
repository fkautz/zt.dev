---
title: "Every experiment behind the shared-base gVisor memory work"
date: 2026-07-23T08:00:00-08:00
draft: false
series:
 - zero-trust
---

*Author's note: this post's text was drafted with assistance from Fable 5.*

This is the lab-notebook companion to
[Sharing gVisor guest memory worked on KVM. Snapshot restore mattered more.](/posts/gvisor-shared-memory-snapshot-restore/)
That post tells the story; this one shows the experiments. Each test below gets its own
section: what it does, the code that matters, what it measured, and how to run it yourself.

All of the code is committed on the
[`shared-base-experiments` branch](https://github.com/fkautz/substrate/tree/shared-base-experiments)
of [fkautz/substrate](https://github.com/fkautz/substrate) under
[`benchmarking/`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/). The gVisor changes are a single combined patch,
[`benchmarking/shared-base/gvisor-shared-base.patch`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/gvisor-shared-base.patch),
which applies to upstream gVisor `928199eb9`;
[`benchmarking/shared-base/make-gvisor-fork.sh`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/make-gvisor-fork.sh)
clones gVisor at that commit and applies it for you, and the resulting branch is also
published prebuilt at
[fkautz/gvisor `shared-base-density`](https://github.com/fkautz/gvisor/tree/shared-base-density). The full reproduction ladder, including
toolchain setup, is
[`benchmarking/shared-base/REPRODUCE.md`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/REPRODUCE.md).

Every experiment below was re-run from these instructions on 2026-07-23 (local tests on an
aarch64 Linux VM, KVM tests on the GCE environment of record) and the published results
reproduced, including the combined gVisor patch applying and testing cleanly against a
fresh upstream clone; the flatten table in Test 8 reproduced digit for digit.

Two environment notes up front:

- Tests 1 through 10 run on any Linux with userfaultfd enabled
  (`sysctl vm.unprivileged_userfaultfd=1`, or run as root). No gVisor build needed for
  tests 1, 2, and 4 through 6 beyond a stock `runsc`.
- Tests 11 through 17 need the patched gVisor and, for the density results, a host with a
  real `/dev/kvm`. The environment of record is a GCE `n2-standard-4` (4 vCPU, ~15 GiB,
  Ubuntu 24.04, nested virtualization enabled). Apple-silicon and VMware nesting both
  failed under gVisor's KVM backend; GCE's Linux-KVM-backed nesting worked.

## Test 1: Does Linux even do this? (`density_smoke.c`)

The whole design rests on one kernel behavior: if N processes map the same file
`MAP_PRIVATE`, the kernel should keep one physical copy of every clean page and give a
writer a private copy only on first store. Before touching gVisor, a standalone C program
proves that, plus the second primitive the design needs: userfaultfd can guarantee that an
absent page is never silently read as zero, and that a known-zero page can be installed
without fetching anything.

Part one forks N=8 children that each map one 256 MiB memfd `MAP_PRIVATE`, read every
page, and then child 0 writes half of them:

```c
// Each child maps the ONE base file MAP_PRIVATE and reads every page.
char *m = mmap(NULL, base, PROT_READ | PROT_WRITE, MAP_PRIVATE, fd, 0);
volatile uint64_t acc = 0;
for (size_t o = 0; o < base; o += PAGE) acc += (unsigned char)m[o];
// Writer (child 0) touches the first half -> private copy for IT only.
if (i == 0)
    for (size_t o = 0; o < base / 2; o += PAGE) m[o] = 0x5A;
```

The measurement is `/proc/self/smaps_rollup`. Rss counts a shared page in every mapper;
Pss divides it by the number of sharers. If sharing works, every child reports
`rss ~= base` but `pss ~= base/N`, and only the writer shows `private_dirty`.

Part two registers a region with userfaultfd in MISSING mode and serves faults from a
handler thread. Even-indexed pages are populated from the base, odd-indexed pages are
declared known-zero:

```c
if (idx % 2 == 0) {
    // DATA: populate from the verified base -- never a silent zero.
    struct uffdio_copy c = { .dst = addr,
        .src = (unsigned long)(ubase + idx * PAGE), .len = PAGE };
    ioctl(uffd, UFFDIO_COPY, &c);
} else {
    // KNOWN-ZERO: install a verified zero page, no base fetch.
    struct uffdio_zeropage z = { .range = { .start = addr, .len = PAGE } };
    ioctl(uffd, UFFDIO_ZEROPAGE, &z);
}
```

This matters because the LLIFS rule is "absent is never zero": a page that has not been
verified and populated yet must fault and be filled from the verified base, and a page the
manifest says is zero must be installable without any fetch.

**Results** (256 MiB base, N=8):

```
non-writer x7        rss=256 MiB   pss= 34 MiB   private_dirty=  0 MiB
writer (wrote half)  rss=256 MiB   pss=144 MiB   private_dirty=128 MiB
data pages faulted=32 -> populated from base (0xC3), never zero
zero pages faulted=32 -> UFFDIO_ZEROPAGE, no base fetch
RESULT: PASS
```

Pss is 34 MiB, almost exactly 256/8, so resident base pages are physically shared. The
writer's 128 MiB of private dirty pages appear in the writer only; nothing bleeds into the
other seven. The same test at a 1 GiB base and N=64 gives pss=16 MiB (base/64), so the
behavior holds at production scale.

**Run it** ([`benchmarking/density_smoke.c`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/density_smoke.c)):

```
gcc -O2 -pthread -o density_smoke density_smoke.c
./density_smoke
```

## Test 2: A real workload survives checkpoint/restore, and the baseline cost (`cr_workload.c`)

Base/delta restore is pointless if gVisor's checkpoint/restore does not already preserve a
real, stateful workload. This test also measures the "before" picture the whole project
attacks: what N restored clones cost with no sharing.

The workload is deliberately simple and unforgiving: fill 1 GiB of heap with a
deterministic per-page pattern, then once a second increment an in-memory counter and
re-checksum a sample of the heap:

```c
uint64_t tick = 0;
for (;;) {
    sleep(1);
    tick++;
    uint64_t sum = 0;
    for (size_t p = 0; p < pages; p += 64)
        for (size_t o = 0; o < 64; o++) sum += buf[p * PAGE + o];
    printf("tick=%llu warm=%zuMiB checksum=%s\n",
           (unsigned long long)tick, mb, sum == ref ? "ok" : "BAD");
}
```

The two output fields are the whole test. If restore resets execution state, `tick` goes
back to 1. If restore corrupts or zeroes any sampled page, `checksum` prints `BAD`. A
restored clone that prints `tick=9 checksum=ok` after being checkpointed at tick 8 has
provably kept both its memory and its execution state.

**Results:**

```
wl1 before checkpoint:  tick=8  checksum=ok
checkpoint:             0.84 s, image 1.1 GiB
wl1r after restore:     tick=9, 10, 11 ... checksum=ok   (continues, not reset)

3 clones restored from the same checkpoint:
  sentry rss 1064 + 1066 + 1065 MiB = 3196 MiB  (~N x base)
```

Checkpoint/restore is correct for a non-cooperating GiB-scale workload, and the baseline
is stark: every restored clone materializes its own full copy of guest RAM, so N clones
cost N times the base. This 3196 MiB number is the one every later flatten is measured
against.

**Run it** ([`benchmarking/cr_workload.c`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/cr_workload.c)): build static,
put it in an OCI bundle, and drive it with stock runsc:

```
gcc -O2 -static -o cr_workload cr_workload.c
# rootfs containing ./cr, config.json running ["/cr"], env WARM_MB=1024
sudo runsc --network=none run -bundle bundle wl1          # watch ticks
sudo runsc checkpoint --image-path=ckpt wl1
sudo runsc restore --image-path=ckpt -bundle bundle wl1r  # tick continues
```

## Test 3: Stateful HTTP server clones (`hsrv.go`)

The same question as Test 2, but for a real networked Go service: does live server state
survive restore, and are clones actually independent of each other afterward?

`hsrv.go` warms 512 MiB of checksummed heap and serves `/healthz`, reporting an in-memory
request counter, the heap checksum, and uptime. The interesting design choice is how it is
health-checked. Host-to-sandbox networking under gVisor needs CNI plumbing that is
orthogonal to this work, so the binary doubles as its own client, exercising a real
TCP/HTTP round trip over gVisor's netstack loopback from inside the sandbox:

```go
// Client mode: `hsrv check` performs a real HTTP GET against the local
// server over netstack loopback. Used via `runsc exec <container> /hsrv check`.
if len(os.Args) > 1 && os.Args[1] == "check" {
    resp, err := http.Get("http://127.0.0.1:" + port + "/healthz")
    ...
}
```

The request counter is the state probe: it must continue across restore (not reset), and
after cloning it must increment independently in each clone.

**Results:**

```
c1 health x3:   reqs=1, reqs=2, reqs=3   checksum=ok    (stateful server)
checkpoint:     0.39 s, 515 MiB
3 clones from 1 checkpoint:
  round 1:  h_a reqs=4   h_b reqs=4   h_c reqs=4   checksum=ok
  round 2:  h_a reqs=5   h_b reqs=5   h_c reqs=5
memory:   3 x ~557 MiB sentry rss = 1674 MiB total  (~N x base)
```

The counter continues from 3 to 4 in every clone (state preserved), then each clone counts
5 on its own (clones independent). And the per-clone memory baseline from Test 2 holds for
a real Go service: 1674 MiB for three clones of a 512 MiB workload.

**Run it** ([`benchmarking/hsrv.go`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/hsrv.go)):

```
CGO_ENABLED=0 go build -o hsrv hsrv.go
# bundle runs ["/hsrv"], env WARM_MB=512
sudo runsc run -bundle bundle c1 &
sudo runsc exec c1 /hsrv check           # reqs=1 checksum=ok
sudo runsc checkpoint --image-path=ckpt c1
sudo runsc restore --image-path=ckpt -bundle bundle h_a   # etc.
sudo runsc exec h_a /hsrv check          # reqs continue, then diverge
```

## Test 4: What state is dangerously identical across clones? (`rhz.go`)

Restoring N clones from one checkpoint means N processes wake up with byte-identical
memory. For some state that is fine; for randomness and identity it can be a correctness
or security hazard. This probe finds out empirically which sources gVisor refreshes per
restore and which it clones.

The server samples every suspect source fresh on each `/probe`:

```go
prng := rand.New(rand.NewSource(time.Now().UnixNano())) // seeded once, at startup
...
fmt.Fprintf(w,
    "getrandom=%s urandom=%s kuuid=%s bootid=%s prng=%016x mono=%dms wall=%d heapobj=%p pid=%d\n",
    hex.EncodeToString(cr), readN("/dev/urandom", 8),
    proc("/proc/sys/kernel/random/uuid"), proc("/proc/sys/kernel/random/boot_id"),
    pv, time.Since(start).Milliseconds(), time.Now().Unix(), heapObj, os.Getpid())
```

The analysis rule is the entire method: restore N clones from one checkpoint, probe each,
and diff the fields. A field that comes back identical across clones is frozen shared
state (a hazard); a field that differs is something gVisor already refreshes.

**Results:**

```
source                 across clones    verdict
getrandom (kernel)     all different    SAFE  (fresh kernel entropy per clone)
/dev/urandom           all different    SAFE
monotonic + wall clock advance normally SAFE
userspace PRNG         IDENTICAL        HAZARD (math/rand state cloned -> same sequence)
boot_id                IDENTICAL        HAZARD (cloned boot identity)
ASLR / heap address    IDENTICAL        WEAKENING (same layout in every clone)
```

Kernel randomness and clocks are safe to clone; gVisor refreshes them. Userspace PRNG
state and boot_id are not, and no runtime can fix a userspace PRNG transparently, so
clone-safety is conditional: RNG-sensitive agents need `getrandom`, a reseed hook, or a
fresh start instead of a clone. ASLR uniformity across a clone set is a residual risk that
only trust boundaries mitigate.

**Run it** ([`benchmarking/rhz.go`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/rhz.go)):

```
CGO_ENABLED=0 go build -o rhz rhz.go
# run under runsc, checkpoint, restore 3 clones, then:
sudo runsc exec <clone> /rhz probe    # diff the output across clones
```

## Test 5: What does verify-before-expose cost on the cold path? (`uffd_latency.c`)

The security rule is that no base byte reaches a guest before its 2 MiB block is hash
verified. The worry is latency: does verification make cold faults slow? This measures the
two halves separately, the unavoidable userfaultfd populate floor and the added hash cost.

The harness registers a region in MISSING mode and times each first touch, which traps to
a handler that installs the whole 2 MiB block:

```c
struct timespec t0, t1;
clock_gettime(CLOCK_MONOTONIC, &t0);
sink ^= region[b * BLOCK];   // cold touch -> MISSING fault -> UFFDIO_COPY 2 MiB
clock_gettime(CLOCK_MONOTONIC, &t1);
```

**Results** (aarch64 VM, local base):

```
verify (sha256 over 2 MiB)              ~0.90 ms/block   2.33 GB/s
populate floor (UFFDIO_COPY 2 MiB)      ~1.18 ms/block   1.76 GB/s   (p99 1.80 ms)
cold fault + verify, serial              ~2.1 ms/block
```

Hashing is cheaper than the page-fault populate it rides on, so verification is not the
bottleneck. A full 1 GiB base verifies in about 0.43 s, paid once per node, and a ~100 MiB
resume working set adds roughly 105 ms to the first cold restore on a node and near zero
to every clone after it.

**Run it** ([`benchmarking/uffd_latency.c`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/uffd_latency.c)):

```
gcc -O2 -pthread -o uffd_latency uffd_latency.c
./uffd_latency 512        # region size in MiB
```

## Test 6: How big is the per-clone delta, really? (`memsnap.go` + `pagediff.go`)

The economics of base sharing are `1 x base + N x delta`. Test 1 established the base
term; this test measures the delta term on a real running sandbox, with no gVisor rebuild.

The trick is a gVisor layout fact discovered during this work: the sentry's guest-RAM
memfd (`/memfd:runsc-memory`) is offset-linear, meaning file offset equals guest memory
offset. So guest memory can be snapshotted from outside by sparse-copying the memfd's
committed extents:

```go
// gVisor backs guest RAM in a memfd named "runsc-memory", mapped offset-linearly.
// Sparse-copy its committed extents (SEEK_DATA / SEEK_HOLE), preserving offsets,
// so two snapshots are page-comparable.
data, err := syscall.Seek(infd, off, seekData)
hole, err := syscall.Seek(infd, data, seekHole)
```

`pagediff.go` then compares two snapshots page by page; the differing pages are exactly
the delta a base/delta checkpoint would have to carry. The workload (`hsrv.go` from Test
3) grew two control endpoints to keep the measurement honest: `load <N>` drives N requests
from a single process so transient client processes do not pollute the diff, and
`dirty <MiB>` writes a known number of pages so the harness can be validated against
ground truth.

**Results** (512 MiB warmed Go HTTP server, 514 MiB committed):

```
after 5000 real requests:  2205 / 262144 pages = 8.6 MiB changed  (1.7% of base)
  the 448 MiB warm read-only region: ZERO changed pages
ground-truth check: asked the server to dirty 100 MiB -> measured 100.4 MiB (0.4% error)
```

A read-mostly service dirties 1.7% of its RAM over 5000 requests, and the harness is
trustworthy to within half a percent. Plugging the measured numbers into the cost model:
at N=64, no sharing costs about 32.9 GiB while base+delta costs about 1.06 GiB, a ~31x
flatten of the guest-memory plane, and node density goes from ~14 to ~148 clones on an
8 GiB node. This was the go/no-go result that justified building the gVisor integration.

**Run it** ([`benchmarking/memsnap.go`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/memsnap.go),
[`benchmarking/pagediff.go`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/pagediff.go)):

```
go build -o memsnap memsnap.go
go build -o pagediff pagediff.go
sudo ./memsnap <sentry-pid> mem0            # snapshot 1
sudo runsc exec c1 /hsrv load 5000          # drive load
sudo ./memsnap <sentry-pid> mem1            # snapshot 2
./pagediff mem0 mem1                        # the delta + region histogram
```

## Test 7: The mechanism, built into gVisor's MemoryFile (in-tree pgalloc tests)

This is where the design stops being a C prototype and becomes gVisor code. The combined
patch teaches `pkg/sentry/pgalloc` four things: map a shared base under a MemoryFile
(`MAP_PRIVATE`, copy-on-write), export a live MemoryFile as a linear base image
(`ExportLinearBase`), save only the pages that differ from a base (delta-only `SaveTo`),
and restore by overlaying the base and loading only the delta (`LoadFrom`, sync and async
paths). Each piece has an in-tree test.

The mapping hook is the Test 1 primitive placed inside gVisor:

```go
// GVISOR-3 (S1): back base-range chunks with a MAP_PRIVATE mapping of the
// shared base file so resident base pages are shared copy-on-write across
// MemoryFiles and writes fault to private copies.
if _, _, errno := unix.Syscall6(unix.SYS_MMAP,
    newChunks[i].mapping, uintptr(hi-off),
    unix.PROT_READ|unix.PROT_WRITE,
    unix.MAP_PRIVATE|unix.MAP_FIXED,
    f.opts.SharedBaseFile.Fd(), uintptr(off)); errno != 0 {
```

The save side folds a base comparison into the per-page scan `SaveTo` already does (it
already reads every page to drop zero pages), so delta detection costs no extra pass.
Pages identical to the base are excluded from the pages file and recorded in the
checkpoint metadata:

```go
// GVISOR-3 (A6): committed pages whose contents equal the base are excluded
// from the pages file and recorded in baseBacked (start->end).
recordBaseBacked := func(fr memmap.FileRange) {
    if fr.Length() != 0 {
        baseBacked[fr.Start] = fr.End
    }
}
```

The restore side overlays the base before any delta is applied, which makes the rest fall
out for free: delta writes go through the mapping and copy-on-write the overlay, on both
the synchronous path and gVisor's async page loader:

```go
// GVISOR-3 (B1/F8): overlay [0, SharedBaseBytes) of the restore mapping
// MAP_PRIVATE from the shared base, so base-backed pages are served by the
// shared base (copy-on-write) instead of the per-sandbox memfd.
if _, _, errno := unix.Syscall6(unix.SYS_MMAP,
    mapStart, uintptr(hi),
    unix.PROT_READ|unix.PROT_WRITE,
    unix.MAP_PRIVATE|unix.MAP_FIXED,
    opts.SharedBaseFile.Fd(), 0); errno != 0 {
```

One symmetry constraint governs the whole split: the pages file is packed in scan order,
so save and load must skip exactly the same `baseBacked` set or every later offset shifts.
Recording the set once in the metadata keeps both sides agreeing, and an empty set makes
the wire format byte-identical to upstream.

**Results:** nine tests, all passing under
`bazel test //pkg/sentry/pgalloc:pgalloc_test`, on both aarch64 and x86_64:

- `TestSharedBaseCOW`: base-backed reads, one base shared across two MemoryFiles, writes
  stay private, base file untouched.
- `TestExportLinearBaseAndShare`: export a live MemoryFile, use the export as another
  MemoryFile's base, read it back.
- `TestBasePlusDeltaRestore`: base + delta compose with isolation; a no-delta clone reads
  pure base.
- `TestBaseBackedRangesDelta`: of 8 pages matching a base, modifying 2 yields exactly 2
  delta and 6 base-backed.
- `TestSaveRestoreRoundTrip`: the real `SaveTo` to `LoadFrom` path restores 16 pages,
  including a zero page, byte for byte.
- `TestSaveWithBaseExcludesDelta`: with a base, save writes 3 pages instead of 16 and
  records 13 as base-backed.
- `TestRestoreOverBase` and `TestRestoreOverBaseAsync`: the full delta-only restore over
  an overlaid base, through both the sync and the runsc-style async loader.
- `TestBaseDecommitRevertsToBase`: decommitting a dirtied base page reverts it to shared
  base content instead of punching a hole.

**Run it:**

```
cd benchmarking/shared-base
./make-gvisor-fork.sh          # clones gVisor 928199eb9, applies the patch
cd gvisor-shared-base
bazel test //pkg/sentry/pgalloc:pgalloc_test
```

## Test 8: The flatten through the real save/restore path (`flatten_test.go`)

With the mechanism in-tree, the payoff question: restore N MemoryFiles from one delta-only
checkpoint over one shared base, through the real `SaveTo`/`LoadFrom` code, and measure
the physical memory. The elegance of the measurement is that one `smaps_rollup` read gives
both sides of the comparison at once:

```go
// Rss counts the base once per mapping (the would-be no-sharing footprint)
// while Pss counts it once physically (the actual shared footprint),
// so Rss/Pss is the flatten ratio.
rss := rollupKB(t, "Rss")
pss := rollupKB(t, "Pss")
...
t.Logf("  FLATTEN (Rss/Pss) = %.1fx", float64(rss)/float64(pss))
```

The test builds a warmed MemoryFile, exports it with `ExportLinearBase`, dirties a small
delta, saves a delta-only checkpoint, then restores N clones over the one base and touches
every base page so the sharing is resident and visible.

**Results** (identical on aarch64 and x86_64):

```
config                       measured   ideal    Pss(shared)  Rss(no-share)
N=8,  base=128MiB, delta=8     4.9x      5.7x      215 MiB      1057 MiB
N=16, base=128MiB, delta=4    10.1x     11.0x      206 MiB      2068 MiB
N=32, base=64MiB,  delta=2    15.0x     16.5x      137 MiB      2061 MiB
```

Consistently ~90% of the ideal `N(base+delta)/(base+N*delta)`; the gap is a fixed 15 to 30
MiB of Go runtime and page tables that amortizes as the base grows. Private_Dirty tracks
N times delta, confirming each clone pays only for its own copy-on-write pages. One honest
scope note: this measures the sentry's mapping of guest memory, which is exactly what the
KVM platform feeds to the guest (Test 11 is where that distinction bites).

**Run it** (the test ships in the patch; also standalone at
[`benchmarking/flatten_test.go`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/flatten_test.go)):

```
cd gvisor-shared-base
bazel test //pkg/sentry/pgalloc:pgalloc_test \
  --test_filter=TestNCloneFlatten --test_output=all --test_arg=-test.v
# knobs: --test_env=FLATTEN_N=32 --test_env=FLATTEN_BASE_MB=64 --test_env=FLATTEN_DELTA_MB=2
```

## Test 9: Verify-before-expose composed with the flatten (`verify_share/`)

Tests 1 through 8 trusted the base. A shared base fetched from a cache or a peer must not
be trusted; it must be verified before any sandbox maps it. This test shows the integrity
layer and the density flatten hold together, using the real
[terrapin-go v0.3.0](https://github.com/fkautz/terrapin-go) content addressing (2 MiB
blocks, GitOID SHA-256, a manifest committing to block size, length, and tree root).

The verification gate is a full re-hash of every block against the manifest before
anything maps the base:

```go
// verifyBase re-hashes every block against the manifest (verify-before-expose).
for b := 0; b < nblocks; b++ {
    f.ReadAt(buf, int64(b)*blockSize)
    if terrapin.Identifier(buf) != manifest[b] {
        return false, b, time.Since(start) // mismatch -> reject this block
    }
}
```

Then N sandboxes `MAP_PRIVATE` the verified base (the flatten), and finally the test flips
a single byte and re-verifies, expecting rejection.

The architectural point this test settled: verification must live at the node level, not
in a per-sandbox fault handler. A per-sandbox `UFFDIO_COPY` would install a private page
per faulter and silently destroy the sharing. Verify once per node, then let every sandbox
`MAP_PRIVATE` the same verified backing; that is what keeps the "once per node"
amortization honest.

**Results:**

```
N=16, base=64MiB:  FLATTEN=8.0x    verify: 35ms (1.90 GB/s), once per node
N=32, base=64MiB:  FLATTEN=15.6x   verify: 35ms (same, independent of N)
tamper: flipped 1 byte -> verify REJECTED at that block, never exposed
```

The flatten with verification in front matches the integrity-free numbers at the same
configs, verification cost does not grow with the sandbox count, and a corrupted block is
caught. Integrity is free at the density layer.

**Run it** ([`benchmarking/verify_share/`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/verify_share/)):

```
cd benchmarking/verify_share
go build -o verifyshare .
./verifyshare                  # env: VS_BASE_MB=64 VS_N=16 VS_DELTA_MB=4
```

## Test 10: Lazy remote fetch, verified at the fault (`lazy_verify/`)

Test 9 verified eagerly. In the transport case the base lives in a content-addressed store
and should be fetched lazily: a block is pulled, verified, and installed only when
something first touches it. This test builds that whole path with a hand-rolled
userfaultfd handler over a node-shared memfd.

The fault handler is the heart of it: map the faulting address to a 2 MiB block, look the
block up in the CAS by its Terrapin ID, recompute the ID over the fetched bytes, and only
then install:

```go
off := (msg.PFAddress - nodeAddr) &^ (blockSize - 1)
id := manifest[off/blockSize]
if id == zeroMarker {
    ioctl(uffd, ioctlUffdioZeropage, ...)   // known-zero: no fetch at all
    continue
}
content := cas[id]
// VERIFY-BEFORE-EXPOSE: recompute the Terrapin id of the fetched bytes.
if terrapin.Identifier(content) != id {
    rejects.Add(1)                           // tampered CAS entry: never exposed
    continue
}
ioctl(uffd, ioctlUffdioCopy, ...)            // verified: install into shared backing
```

The test seeds the CAS with one deliberately tampered entry and one known-zero block, then
touches every block twice. The second pass is the amortization check: already-installed
blocks must add zero fetches.

**Results** (32 MiB base, 16 blocks, one zero, one tampered):

```
fetches=15  verifies=15  zero-installs=1  rejects=1
re-touching every block added 0 fetches      (once per node, independent of N)
tamper:     block 8 REJECTED by verify, not exposed
known-zero: block 4 installed via UFFDIO_ZEROPAGE, no fetch
share N=16: Rss=575MiB  Pss=63MiB   FLATTEN= 9.1x
share N=32: Rss=1087MiB Pss=63MiB   FLATTEN=17.3x
```

Fetch and verification counts are independent of the sandbox count, tampered content never
reaches a mapping, zero blocks cost nothing on the wire, and Pss stays flat as N doubles,
so the lazy transport composes with the flatten.

**Run it** ([`benchmarking/lazy_verify/`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/lazy_verify/)); note the uffd
ioctl and syscall numbers are hardcoded for linux/arm64:

```
cd benchmarking/lazy_verify
go build -o lazyverify .
./lazyverify                   # env: LV_BASE_MB=32 LV_N=16
```

## Test 11: Restore over a base through runsc, and the crash that mapped the platform boundary

With everything proven at the pgalloc layer, the base file was threaded through the real
`runsc restore` path: an optional `base.img` next to the checkpoint, passed as an FD
through the sandbox and boot controller down to `LoadFrom`. Then the Test 2 workload was
restored two ways on systrap:

```
restore WITHOUT base.img:  tick 5,6,7,8,9  checksum=ok   (regression clean)
restore WITH    base.img:  container stopped, no output   (workload crashed)
```

Restore completed cleanly by every log line, and the guest was dead. The control run
without a base ruled out an ordinary restore regression, pinning the failure on the base
path. The root cause is the most important finding in the project: the guest does not
execute from the mapping the overlay modifies. On systrap, the platform maps guest memory
into a separate stub process straight from the per-sandbox memfd; on KVM, it maps the
guest from the sentry's own mapping, exactly where the overlay lives:

```go
// systrap subprocess.MapFile: guest RAM comes from the per-sandbox memfd,
// mapped into a separate stub process. The sentry-side overlay is invisible here.
mmap(MAP_SHARED, f.DataFD(fr), fr.Start)

// KVM address_space.MapFile: guest RAM comes from the sentry's own mapping,
// the exact chunk the base overlay modified copy-on-write.
bs, err := f.MapInternal(fr, ...)
```

So the delta had been copy-on-written into the sentry's overlay, the memfd was left holey
for the base range, and the systrap stub executed zeros. The design was predicted correct
for KVM and wrong for systrap by code reading; proving it needed real hardware, which no
Apple-silicon or VMware setup could provide (`/dev/kvm` absent or host crashes). A GCE
instance with nested virtualization finally supplied a working Linux KVM:

```
restore WITHOUT base.img (--platform=kvm):  tick continues, checksum=ok
restore WITH    base.img (--platform=kvm):  tick continues, checksum=ok   (confirmed)
```

The identical restore that killed the guest on systrap resumed cleanly on KVM with memory
intact. The platform boundary is real, measured, and matches the code: base sharing via
this overlay is a KVM feature; systrap would need KSM or a deeper coherent mechanism.

**Run it:** apply the combined patch (Test 7), build runsc, and use the Test 2 bundle on a
host with `/dev/kvm`:

```
bazel build //runsc:runsc
R=bazel-bin/runsc/runsc_/runsc
sudo $R --platform=kvm run -bundle bundle s1 &
sudo $R --platform=kvm checkpoint --image-path=ckpt s1
sudo $R --platform=kvm restore --image-path=ckpt -bundle bundle s1r   # with/without ckpt/base.img
```

## Test 12: The end-to-end flatten through the runsc CLI on live KVM (`runs/measure.sh`)

Test 11 proved correctness; this run measures the magnitude an operator would actually
see, using the checkpoint-side plumbing (`runsc checkpoint --shared-base`) that exports
the base image and writes a delta-only checkpoint in one step. The workload touches all
256 MiB every second, so the entire base faults resident and the sharing has to show up as
physics, not accounting.

The orchestration is deliberately plain shell; the measurement loop sums Rss and Pss over
every live sentry:

```bash
$R $ROOT checkpoint --shared-base --image-path=$D/ckpt s2
for i in $(seq 1 8); do $R $ROOT restore --image-path=$D/ckpt -bundle $B k$i & done
for pid in $(pgrep -f "runsc-sandbox.*root=/run/c1b2"); do
  r=$(awk "/^Rss:/{print \$2}" /proc/$pid/smaps_rollup)
  p=$(awk "/^Pss:/{print \$2}" /proc/$pid/smaps_rollup)
  trss=$((trss+r)); tpss=$((tpss+p))
done
```

**Results** (GCE n2-standard-4, `--platform=kvm`, N=8, 256 MiB touch-all base):

```
checkpoint --shared-base:  base.img=257 MiB   pages.img=0 bytes (delta=0)
all 8 clones: tick continues, checksum=ok
sum sentry Rss = 2398 MiB (would-be no-sharing)
sum sentry Pss =  421 MiB (shared)
FLATTEN = 5.69x through the real runsc CLI on a live KVM guest
```

A freshly warmed clone's delta really is zero (`pages.img` is empty), and 2.4 GiB of
would-be private guest RAM collapses to 0.42 GiB. The number is lower than the pgalloc
layer's ~10x because each sandbox carries 20 to 45 MiB of unshareable sentry overhead,
which is the subject of Test 14.

**Run it** ([`benchmarking/shared-base/runs/measure.sh`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/runs/measure.sh)):
follow [`REPRODUCE.md`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/REPRODUCE.md) sections 1 to 3, then run
the script (edit `H=` at the top for your paths). It writes `flatten_result.txt`.

## Test 13: How many sandboxes fit on a box? (`runs/ceiling.sh`)

Density is the product claim, so this run escalates N real runsc KVM sandboxes over one
shared 128 MiB base until the machine gives out, measuring memory and load at every step.
A guard stops before the OOM killer would.

**Results** (n2-standard-4: 4 vCPU, 15987 MiB):

```
N    sumRss    sumPss   MemAvail   load1
32   2414 MiB   659 MiB  13934 MiB   0.5
64   4835 MiB  1289 MiB  12396 MiB   0.3
128  9682 MiB  2514 MiB   9357 MiB   7.1
200 15125 MiB  3884 MiB   5936 MiB  16.6
300 22698 MiB  5797 MiB    983 MiB  29     <- memory floor; all still correct
```

Every clone ran correctly at every N. The memory ceiling is ~300 sandboxes on 15 GiB;
without sharing, each clone would need base plus overhead (~175 MiB), so only ~85 would
fit: about 3.5x more sandboxes from sharing even with this deliberately small base. The
marginal cost decomposes into ~19 MiB/clone of sentry Pss plus ~28 MiB of gofer and
monitor processes, so total density follows `N ~= (RAM - base) / ~47 MiB`. CPU is the
softer limit: load stays under 1 up to N=64 and reaches ~29 at N=300. A larger shared base
improves both the density and the flatten, since the fixed ~47 MiB floor stops dominating.

**Run it** ([`benchmarking/shared-base/runs/ceiling.sh`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/runs/ceiling.sh))
on the Test 12 setup. It escalates N in steps and records a table per step.

## Test 14: The per-sandbox floor (`runs/gomem.sh`, `runs/gotune.sh`)

The ceiling run said ~47 MiB per sandbox is unshareable. This experiment decomposes that
floor with a tiny guest, so nearly everything measured is overhead, and then attacks the
obvious suspect (Go's garbage collector) to see if tuning reclaims any of it.

**Results:**

```
sentry: Pss=24M   (Go heap/stacks anon=8M, private marginal=17M, binary=19M shared)
gofer:  Pss=10M   (Go anon=3M, private=3M, binary=18M shared)
-> marginal per sandbox ~20M (17 sentry + 3 gofer); ~11M is live Go runtime
   across two Go processes; the ~37M runsc binary is mapped once, not marginal
GOGC/GOMEMLIMIT tuning: no change (the idle sentry has no garbage; the memory is
   threads, goroutine stacks, and runtime arenas -- structural)
```

The floor is structural, not garbage: wiring `debug.SetGCPercent(20)` into the sentry boot
changed nothing, because there is nothing to collect. The sentry is a ~277k LOC userspace
kernel implementing 645 syscalls; that working set is irreducible. The one tractable piece
is the gofer, a second Go process that directfs already sidelines at runtime but which
still runs; reaping it is worth ~3 MiB and is the realistic floor optimization. This floor
is why small agents flatten at ~2.9x while synthetic large-base tests flatten at 10x+.

**Run it** ([`benchmarking/shared-base/runs/gomem.sh`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/runs/gomem.sh),
[`runs/gotune.sh`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/runs/gotune.sh)) on the Test 12 setup.

## Test 15: A real agent, and the latency result (`runs/lat2.sh`, `agent/`)

Everything so far used workloads built to be measurable. This test uses a real agent on
Google's Agent Development Kit (google-adk 2.3.0, Python 3.12) with the LLM endpoint
mocked out (a custom `BaseLlm`, no network), so it exercises the full framework: sessions,
the runner, asyncio, gRPC. Two questions: does a live Python agent checkpoint/restore
correctly, and what do density and latency look like for something real?

The timing harness measures wall and CPU from `runsc run` (or `restore`) to the first
served response, polling the container log:

```bash
t0=$(date +%s.%N)
$R $ROOT run -bundle $B cold > $H/cold.log 2>&1 &
until grep -q "tick=" $H/cold.log; do sleep 0.05; done
t1=$(date +%s.%N)     # cold start: run -> serving
...
$R $ROOT restore --image-path=$D/ckpt -bundle $B warm &
until grep -q "tick=" $H/warm.log; do sleep 0.02; done   # restore -> serving
```

**Results:**

```
correctness:  live Python + asyncio + grpc checkpoint/restores correctly
density:      base.img=80M, delta~0; N=8: sumRss=721M sumPss=250M -> 2.88x
              (floor-limited, per Test 14; per-clone Python COW delta only ~4M,
               thanks to CPython 3.12 immortal objects)
latency:      COLD START  (run -> serving):     wall=11.4s  cpu=7.1s
              RESTORE     (restore -> serving):  wall=0.47s  cpu=0.51s
              RESTORE 8 concurrent: all serving within 1.87s
```

The density flatten is modest for a small agent, and padding the agent with a 1.1 GiB
allocation did not help: memory the clones never touch is never resident, so there is
nothing to share. The latency result is the standout: restore is ~24x faster in wall time
and ~14x cheaper in CPU than a cold start, because the cold start is ~1,400 Python module
imports and restore is just mapping an already-initialized image. Eight agents serve in
under two seconds instead of ~57 seconds of cumulative cold-start CPU.

**Run it**
([`benchmarking/shared-base/runs/lat2.sh`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/runs/lat2.sh),
agent in [`benchmarking/shared-base/agent/`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/agent/)): build
the agent rootfs with `runs/buildimg.sh` (uses `agent/Dockerfile`), then run `lat2.sh`.
The ADK flatten run is `runs/adkexp2.sh`. Note `lat.sh` (without the 2) is the buggy first
cut, kept for honesty; its ready-marker poll never matched.

## Test 16: Trying to break the latency claim (`runs/expA.sh`, `runs/expB.sh`)

"Sub-second restore" is exactly the kind of claim that hides a fault-in stall or a
correctness bug, so this experiment attacks it three ways.

**Is it actually warm?** Yes. First post-restore response at 0.6 s on the external clock,
second request at 29 ms, steady state 20 ms. One in-agent timer did report 3 seconds for
the first request, which turned out to be self-inflicted: the timer was started before the
checkpoint and dutifully counted the time the process spent frozen. The external clock and
the 29 ms second request disprove a real stall.

**Is it correct?** Bit-exact. Every post-restore response was semantically verified
through the full ADK path (session, runner, agent, model; zero bad responses), and an
in-memory running accumulator came back with exactly the value it would have had without
the stop: tick 34 to 35, sum 595 to 630.

**Does it hold at burst?** This is the real caveat:

```
N=8 concurrent restores:    all serving < 2s
N=50:                       p50=18.2s  p99=19.4s  (all served)
N=100:                      p50=41.9s  p99=49.2s  (5 stalled past 150s)
```

Each restore plus first request costs ~1.5 s of CPU (mostly working-set fault-in), so
burst wall time is roughly `N * 1.5s / cores`. Cold start costs 7.1 s CPU each, so restore
is ~4x cheaper at every scale, but it is not magically instant: the defensible claim is
"about 4x more starts per core", not "sub-2s bursts at any N". Restoring 100 agents on 4
vCPUs takes ~42 s versus ~178 s to cold-start them.

**Run it**
([`benchmarking/shared-base/runs/expA.sh`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/runs/expA.sh) for
warmth/correctness,
[`runs/expB.sh`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/runs/expB.sh) for burst) on the Test 15
setup.

## Test 17: Parked-agent storage, and the part that works everywhere (`runs/expC.sh`)

Two final questions. What does a parked (checkpointed) agent cost on disk? And how much of
all this survives on systrap, the platform without `/dev/kvm`?

**Storage results** (KVM, ADK agent, warm rss 116M):

```
checkpoint --shared-base:  wall 0.11s
base.img:  81M actual (sparse), shared once across the whole pool
per-agent parked cost:  state 240K + delta(pages.img) 0 + metadata 8K  = ~248K
vs normal checkpoint:   ~81M per agent
pool of 10,000 parked agents:  ~2.5G shared-base  vs  ~810G normal  (~300x)
```

Parking an agent takes a tenth of a second and a quarter megabyte. The storage flatten
mirrors and exceeds the memory flatten because a fresh clone's delta is zero; only the
small kernel-state file is per-agent. The caveat: the delta grows as an agent diverges,
but state plus delta stays far below a full checkpoint.

**Systrap results** (normal checkpoint, since base sharing is KVM-only per Test 11):

```
systrap cold start:  wall 4.05s  cpu 1.99s
systrap restore:     wall 0.18s  cpu 0.26s   (all responses correct)
```

The restore-skips-cold-start win holds on systrap, and is even larger there in this
environment (the GCE host makes gVisor-KVM pay nested-virtualization overhead; on bare
metal the gap narrows). This is the clean split the whole project lands on: the density
flatten needs KVM and a large resident base; the latency win needs neither, and applies to
essentially every gVisor deployment.

**Run it**
([`benchmarking/shared-base/runs/expC.sh`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/runs/expC.sh)) on
the Test 15 setup; the systrap half runs on any Linux host, no `/dev/kvm` required.

## The scoreboard

| # | Experiment | Question | Result |
|---|---|---|---|
| 1 | `density_smoke.c` | Does Linux share MAP_PRIVATE pages COW? | Yes; pss = base/N, writes private |
| 2 | `cr_workload.c` | Does C/R preserve real state? Baseline cost? | Yes; N clones = N x base |
| 3 | `hsrv.go` | Does a live server survive and diverge? | State continues; clones independent |
| 4 | `rhz.go` | What state is dangerously cloned? | Kernel RNG safe; userspace PRNG, boot_id cloned |
| 5 | `uffd_latency.c` | Is verify-before-expose affordable? | Verify (0.9ms) < populate (1.2ms) per 2 MiB |
| 6 | `memsnap` + `pagediff` | How big is the real delta? | 8.6 MiB per 514 MiB base (1.7%) |
| 7 | pgalloc tests | Does the base/delta split work in gVisor? | 9/9 tests pass, both arches |
| 8 | `flatten_test.go` | Does it flatten through real save/restore? | 4.9x to 15.0x, ~90% of ideal |
| 9 | `verify_share` | Does integrity cost density? | No; same flatten, tamper rejected |
| 10 | `lazy_verify` | Does lazy CAS fetch compose? | Yes; once per node, 9.1x to 17.3x |
| 11 | runsc restore over base | Does the overlay reach the guest? | systrap no (crash), KVM yes (confirmed) |
| 12 | `runs/measure.sh` | Operator-visible flatten? | 5.69x, N=8, live KVM, delta=0 |
| 13 | `runs/ceiling.sh` | How many sandboxes per box? | ~300 on 15 GiB (~3.5x more than unshared) |
| 14 | `runs/gomem.sh` | What is the floor? | ~20 MiB/sandbox, structural, not GC-tunable |
| 15 | `runs/lat2.sh` | Real agent: density and latency? | 2.9x flatten; restore 24x faster than cold |
| 16 | `runs/expA/B.sh` | Does the latency claim survive attack? | Warm, bit-exact; burst is CPU-bound (~4x/core) |
| 17 | `runs/expC.sh` | Storage cost? Systrap reach? | ~248K/parked agent; latency win holds on systrap |

Seventeen experiments, three layers (raw kernel, gVisor internals, real runsc on real
KVM), and two results worth building on: shared-base memory is a genuine density win on
KVM when the resident base is large, and snapshot restore instead of cold start is a large
win everywhere. The code for every row is in
[`benchmarking/`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/), and
[`benchmarking/shared-base/REPRODUCE.md`](https://github.com/fkautz/substrate/blob/shared-base-experiments/benchmarking/shared-base/REPRODUCE.md) is the
ladder from a bare Ubuntu host to every number above.
