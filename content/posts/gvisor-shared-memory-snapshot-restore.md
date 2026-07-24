---
title: "Sharing gVisor guest memory worked on KVM. Snapshot restore mattered more."
date: 2026-07-06T08:00:00-08:00
draft: false
series:
 - zero-trust
---

*Author's note: the patches and benchmark code are now published: the harness on the
[`shared-base-experiments` branch of fkautz/substrate](https://github.com/fkautz/substrate/tree/shared-base-experiments/benchmarking)
and the gVisor changes on the
[`shared-base-density` branch of fkautz/gvisor](https://github.com/fkautz/gvisor/tree/shared-base-density).
A companion post,
[Every experiment behind the shared-base gVisor memory work](/posts/gvisor-shared-base-experiments/),
walks each experiment with code, results, and run instructions. This post's text was drafted with
assistance from Opus 4.8.*

**TL;DR**

- KVM shared-base memory works: roughly 5×-17× flatten in controlled tests; ~2.9× on a real small ADK agent because of a ~20 MiB/sandbox floor.
- Systrap overlay does not compose because the stub maps the per-sandbox memfd, not the sentry's shared overlay.
- Portable win: snapshot restore (~11 s cold start → ~0.6 s on KVM; ~0.18 s on systrap) skips framework init on both platforms.
- Verified bases ([Terrapin](https://github.com/fkautz/terrapin-go)) compose without materially hurting flatten; tampered blocks are rejected before guest exposure.

The starting question was about memory density: can thousands of restored gVisor sandboxes share
the same warmed guest RAM copy-on-write, instead of each clone materializing a private copy?
The answer is yes on KVM. Through gVisor's real `MemoryFile` save/restore path, shared-base
restore cut physical memory by roughly 5× to 17× in controlled tests, and a live KVM guest
restored and ran correctly over the shared base.

But the more useful result was not the one this work set out to find. A real agent showed that small
workloads quickly hit a fixed per-sandbox memory floor, which limits the density win. The
bigger, portable win was snapshot restore itself: skipping Python and framework
cold start made agents resume several times faster and cheaper on both KVM and systrap. The
memory-sharing work was real; restore was the prize.

## The problem: density is a memory wall

[Substrate](https://github.com/agent-substrate/substrate) runs many agent workloads as
gVisor sandboxes, and the workload shape is unusual. Hundreds or thousands of sandboxes
start out *almost identical* (same runtime, same loaded model code, same warmed heap), and
each then diverges a little as it does its own work. The mental model is "fork a warmed
process a thousand times," except each fork is a full gVisor sandbox restored from a
checkpoint.

gVisor (runsc) checkpoint/restore handles this well: warm a sandbox, checkpoint it, restore
N clones. But restore is a *full copy*. Each restored sandbox materializes its own private
copy of guest RAM. The baseline, measured directly, was three clones of a 512 MiB warmed
HTTP server restored from one checkpoint:

```
3 clones = 1674 MiB sentry RSS   (~558 MiB each)
```

Each number is the resident memory (RSS) of a runsc *sentry*, the per-sandbox userspace
kernel process. That is ~1× base *per clone*, and at a thousand clones RAM is the wall long
before CPU is.

The hypothesis is the obvious one: most of that 558 MiB is *identical* across clones (the
warmed base), and only a small slice differs per clone (the delta). If the shared portion is
large and the per-clone delta is small, the economics change from one full RAM copy per
clone to one base plus N deltas. Keep one physical copy of the base, share it copy-on-write,
and density becomes **delta-bound, not base-bound**.

Call the improvement factor the **flatten** (a sharing factor): the ratio of would-be private resident
memory (every clone a full copy) to shared physical memory (one base plus the per-clone
deltas). A 10× flatten means the clones consume roughly one-tenth the physical RAM they
would as full copies.

The rest of the post walks the evidence in order:

| Layer | Question | Result |
|---|---|---|
| Linux `MAP_PRIVATE` smoke test | Will the kernel physically share clean mapped file pages? | Yes. PSS scales roughly as base/N; writes copy-on-write only in the writer. |
| Live gVisor memory diff | Is the per-clone delta small? | 5000 requests to a warmed Go server changed 8.6 MiB, about 1.7% of base. |
| gVisor `MemoryFile` tests | Does base/delta save/restore preserve memory? | Yes, through both the sync and async load paths. |
| runsc on systrap | Does the shared-base *overlay* reach the running guest? | No. On systrap the guest maps from the memfd in a separate stub, so the overlay is invisible to it. Ordinary checkpoint/restore, and the latency win, still work on systrap. |
| runsc on KVM | Does the same design run end-to-end? | Yes. The guest resumed correctly over the shared base. |
| Real ADK agent | Is density the main product win? | Not for small agents. Restore latency was the larger, cross-platform win. |

## Does the OS even do this?

The cheapest version of the question comes first, before any gVisor change. If N processes
`MAP_PRIVATE` the *same* file, does Linux keep one physical copy of the unwritten pages and
copy-on-write only on the first store?

A standalone C harness (`density_smoke.c`, no gVisor involved) answers it: one base file, N
processes each mapping it `MAP_PRIVATE`, reading all of it, and one writer scribbling over
half. Reading `/proc/self/smaps` for PSS (proportional set size, where shared pages count
as size/num_sharers):

```
256 MiB base, N=8:
  non-writers:  rss=256 MiB   pss= 34 MiB   private_dirty=  0 MiB
  writer (½):   rss=256 MiB   pss=144 MiB   private_dirty=128 MiB
```

RSS stays high because each process maps the full file, but PSS shows the physical reality:
clean pages are shared until a writer dirties them. Writes create private pages for the
writer only. The kernel primitive is therefore exactly what is needed: **resident base pages
are physically shared, and writes go copy-on-write.** The remaining work is to route gVisor's
guest memory through this primitive, not to invent a mechanism.

## How big is the delta, really?

Base sharing only matters if the per-clone delta is small. If a restored agent dirties most
of its pages in the first few seconds, sharing buys little and the effort is wasted. The
measurement had to come through the *real* gVisor memory, not a model.

gVisor backs guest RAM with a per-sandbox `memfd` (the `MemoryFile` in
`pkg/sentry/pgalloc`), and that memfd turns out to be laid out linearly in guest-physical
order. So a running sandbox's guest memory can be snapshotted just by reading
`/proc/<sentry>/fd/<memfd>` at two points in time and diffing it page by page, with no
gVisor rebuild. This leans on an internal gVisor layout invariant, the guest-physical-linear
memfd, rather than a stable interface, but it holds for the measurement.

A 512 MiB warmed Go HTTP server under runsc, snapshotted, driven through **5000 real
requests**, and snapshotted again:

```
base committed:                       514 MiB
delta after 5000 requests:   2205 pages = 8.6 MiB   (1.7% of base)
```

The intentionally warmed read-mostly region showed **zero** changed pages after the request
run; the measured delta came from runtime and request state, not the warmed working set. As
a sanity check on the harness, the server was made to dirty a *known* 100 MiB and re-measured
at 100.4 MiB, within 0.4%.

This is the *checkpoint delta*: pages whose final contents differ from the base. It is not
total write traffic or churn (a page dirtied and then restored to its original bytes is not
delta), which would matter for a different design but not for base/delta restore size. For a
read-mostly service the per-clone delta is ~1.7%, so the flatten is worth building. The
number is workload-dependent: a write-heavy agent has a larger delta, and the sharing
ceiling falls as delta grows. For N clones, the ideal flatten is
`N·(base+delta)/(base+N·delta)`; at large N it approaches `(base+delta)/delta`.

## Building it into gVisor: the base/delta split

The plan: a restored sandbox loads **only its delta** and gets the base from a shared,
read-only base image mapped copy-on-write. That requires changing both halves of gVisor's
save/restore. The invariant is simple: for every committed page, restore must get bytes from
exactly one source, the base overlay if the page is marked `baseBacked`, otherwise the
checkpoint pages file. Save and restore must therefore agree on the same walk order and the
same skip set.

**The save side.** `MemoryFile.SaveTo` walks its memory-accounting tree and writes every
committed non-zero page into a "pages file." It now takes a base image and, for each
committed page, compares it to the base; pages identical to the base are *excluded* from the
pages file and recorded in a `baseBacked` set in the checkpoint metadata. The comparison
folds into the existing per-page scan, so there is no second walk of guest memory. With no
base supplied, behavior is byte-for-byte identical to upstream.

Zero pages need one subtlety. Upstream omits all-zero committed pages, since restore would
see zero there anyway. A base overlay breaks that reasoning: an all-zero page sitting over a
*non-zero* base page is a real delta, and omitting it would make restore wrongly fall through
to the base. So inside the base range the deciding rule is "differs from the base," not "is
non-zero."

**The save/restore symmetry.** One property forces the skip to be symmetric: the pages file
is *packed in walk order*. A page's position in the file is its scan order, not its memory
offset, so the restore side has to walk the same structure in the same order to line the
offsets back up. Both save and load skip exactly the `baseBacked` set, recorded once in the
metadata so both sides agree.

**The load side.** `LoadFrom` maps the chunks and overlays `[0, baseBytes)` `MAP_PRIVATE`
from the base image, which is the `density_smoke` primitive now living inside gVisor. The
delta pages are applied *over* that overlay (copy-on-write), and the `baseBacked` pages are
never loaded, because the overlay already shows them.

One worry evaporated on inspection. gVisor's async page loader reads the pages file into
guest memory, and the concern was that it wrote to the memfd file descriptor directly,
bypassing the `MAP_PRIVATE` overlay. The code (`FDReader`) reads straight into the *mapping
memory* via the iovecs the loader builds from the chunk mapping. Because the overlay is
established before those iovecs are built, an async delta load writes into the private
overlay and copy-on-writes it, exactly as a userspace write would. No loader rewrite was
needed.

Each piece has an in-tree test: a base-backed page reads base content, a delta-written page
reads delta content, the base file stays immutable, and a full `SaveTo → LoadFrom` round-trip
over both the synchronous and the async (runsc-style) paths reproduces memory byte-for-byte.

## The flatten, measured through the real code

With the split working, the payoff is measurable through the *actual* `SaveTo`/`LoadFrom`
code: build one base, save a delta-only checkpoint against it, restore N `MemoryFile`s over
the one shared base, and read `smaps`. One measurement gives both terms. Summed `Rss` counts
the same shared-clean base pages once per clone (the would-be no-sharing cost); summed `Pss`
divides those pages across sharers and is the better proxy for physical resident memory.

One caveat on what the number includes. This is not "total machine memory saved" in the
abstract; it is the ratio measured over the mapped regions in this experiment, and depending
on the measurement point it may exclude gofer RSS, page tables, unrelated sentry runtime
memory, or host file-cache effects. That matters later, when a real agent's flatten comes in
lower.

| N | base / delta | flatten (Rss/Pss) | ideal | Pss (shared) | Rss (no-share) |
|---|---|---|---|---|---|
| 8 | 128 / 8 MiB | **4.9×** | 5.7× | 215 MiB | 1057 MiB |
| 16 | 128 / 4 MiB | **10.1×** | 11.0× | 206 MiB | 2068 MiB |
| 32 | 64 / 2 MiB | **15.0×** | 16.5× | 137 MiB | 2061 MiB |

That is about 90% of the theoretical `N·(base+delta) / (base + N·delta)`; the gap is a fixed
Go-runtime and page-table overhead that amortizes as the base grows. Density is delta-bound,
as hoped, and now through gVisor's real memory path rather than a toy.

## Can you trust the base?

A shared base becomes a security question the moment it comes from anywhere other than the
sandbox itself: a node-local cache, a peer, an object store. A base page should not be mapped
into a sandbox unless it is certain to be the page it claims to be. The rule to enforce is
**verify-before-expose**: no byte reaches the guest without being checked first.

The check is content-addressing, using [Terrapin v0.3](https://github.com/fkautz/terrapin-go/blob/main/README.md).
Terrapin gives the whole base image a single dataset identity, `terrapin-sha256:<digest>`.
The scheme is cross-confirmed by two independent v0.3 implementations,
[terrapin-go](https://github.com/fkautz/terrapin-go) and
[terrapin-rs](https://github.com/fkautz/terrapin-rs).

The base is split into 2 MiB blocks, exactly 2,097,152 bytes each, with the final block
allowed to be smaller. Each leaf is hashed with GitOID SHA-256, the Git blob
construction `sha256("blob " + len + "\0" + data)`. The leaf hashes are recursively reduced
to a tree root, and that root is wrapped in a canonical manifest that commits the algorithm,
block size, total length, and tree root. The Terrapin identifier is the GitOID of that
canonical manifest, not the bare tree root, so the identity is unambiguous about size and
tree height and cannot be reinterpreted at a different block size.

The trust model follows from that. The control plane only has to authenticate the expected
32-byte Terrapin ID, for example by pinning it in signed checkpoint metadata. The manifest
and the data blocks can come from an untrusted cache or object store. Before exposing any
base block, the node verifies the manifest against the trusted ID, derives the exact tree
shape from the committed length and block size, fetches the target data block plus the
hash-file blocks needed to recompute upward, and accepts the block only if the recomputed
root matches the manifest. A changed byte, a wrong length, a wrong block size, or a
non-canonical manifest all change the identity and are rejected before the block reaches the
guest. Only the 32-byte ID needs a trusted channel; everything else is self-verifying
against it.

The placement is what makes this cheap. The naive version, a per-sandbox userfaultfd handler
that fetches and verifies each page as the sandbox faults it, defeats the purpose: each
sandbox would fault and populate its *own* private copy, and nothing would be shared.
Verification instead happens **once per node**, into a sealed node-local backing, before any
sandbox maps it. In the lazy/remote case the userfaultfd handler is attached to that shared
backing's population path, not to each sandbox's private guest mapping. Once a block is
fetched, verified, and installed into the shared backing, later sandboxes map the same
already-populated backing `MAP_PRIVATE` and do not fetch, verify, or allocate it again. That
is what keeps "once per node" honest.

Two things still sit outside the hash. First, the expected Terrapin ID must be authenticated
by the control plane; Terrapin proves the bytes match that ID, not that the ID is the one the
workload intended to run. Second, the verified node-local base must stay immutable after
verification: sealed, held by an immutable content-addressed cache entry, or otherwise
guaranteed that the exact verified file descriptor is the one sandboxes later map. Trust
terminates at the authenticated ID and the sealed base, not at the bytes alone.

Both paths were built and tested: an eager path (verify the whole base up front, then share)
and a lazy/remote path (fetch, verify, and install each absent block on first access, with
known-zero blocks installed via `UFFDIO_ZEROPAGE` with no fetch). Crucially, "known-zero" is
not a hint from side metadata, which would reopen the exact tamper hole verify-before-expose
closes; a block is known zero because its *verified* leaf hash equals the fixed constant for a
2 MiB block of zeros, so the zero-ness is authenticated by the same tree as every other block.
Verification did not
materially change the measured resident-memory flatten in these tests, because it gates trust
rather than adding resident pages. And it holds under attack: with one block deliberately
corrupted in the store, the flipped byte changes the leaf hash, so recomputation no longer
reaches the manifest root; the block is rejected at the fault and never exposed. Throughput
numbers are in the measurement details at the end.

At this point every layer was proven in-tree: share a base, copy-on-write a delta over it,
compute the delta, exclude it on save, overlay it on restore, verify it eagerly or lazily,
and a measured ~10× flatten at N=16. The next step was to wire it into runsc and restore a
real workload over a base.

## Then it crashed

The restore-side plumbing came first. `runsc restore` picks up a `base.img` from the image
directory and threads it as a file descriptor through the sandbox, then the boot controller,
then `kernel_restore`, down to the main `MemoryFile`'s `LoadFrom`. runsc built. A small
stateful workload (a counter plus a checksummed heap) checkpointed at tick 4, then restored
two ways:

```
restore WITHOUT base.img:  tick 5,6,7,8,9   checksum=ok      ✅  (regression clean)
restore WITH    base.img:  container stopped, no output       ❌  (workload crashed)
```

The debug log showed the base image threaded through, the restore completing cleanly, every
timer running to "end Restore," and then nothing. No tick. Restore had succeeded flawlessly,
and the guest was dead. The clean restore-without-base was the useful control here: it ruled
out an ordinary restore regression and pinned the failure squarely on the base-overlay path.

## Who actually maps the guest's memory?

The unit tests all passed, and the flatten was real. So what was different about a *running
guest*?

The answer: **the guest does not execute against the mapping being shared.** The overlay,
the flatten, and the unit tests all operate on the *sentry's* view of guest memory,
`MemoryFile.MapInternal`, the chunk mapping the sentry uses to read and write guest pages for
syscalls. But the guest *runs* against memory the **platform** sets up, which is a different
code path.

`platform.AddressSpace.MapFile`, the call that installs guest memory into the guest's address
space, has two very different implementations. On **systrap** (`subprocess.go`), `MapFile`
does `mmap(MAP_SHARED, f.DataFD(fr), fr.Start)`: it maps the guest straight from the
per-sandbox **memfd** file descriptor, into a **separate stub process** (`clone`d with
`CLONE_FILES | SIGCHLD` and *no* `CLONE_VM`, so the sentry and the stub are distinct address
spaces that share guest RAM only through `MAP_SHARED` of the memfd).

The overlay was on the sentry's chunk mapping. The async loader had copied the delta into
*that* mapping (copy-on-write), which left the memfd itself holey for the base range. The
stub, mapping the memfd, saw zeros where the program's code and heap should be, and the
workload crashed on the first instruction it tried to run from a base page. The unit tests
passed precisely because they read through `MapInternal`, the sentry view, which *did* have
the data.

The failure was not in the base/delta memory composition itself. It was in assuming that
every platform executes from the same composed view that the sentry uses. Every layer was
individually correct; the layering was wrong, and only an end-to-end test could surface it.

## The platform boundary: systrap versus KVM

The natural next thought, "move the overlay down to `MapFile` or `DataFD`," runs into
systrap. The sentry and stub are separate address spaces, and both need a coherent view of
guest RAM, which is why systrap maps the per-sandbox memfd `MAP_SHARED` into the stub. If
both processes instead mapped a shared base `MAP_PRIVATE`, the first write to a base-backed
page could create *different* private copies in the sentry and the stub. The two paths, and
where systrap breaks:

```
KVM: shared base reaches the guest
  Guest execution
      │
      ▼
  KVM maps pages from sentry MapInternal
      │
      ▼
  MAP_PRIVATE base overlay + COW delta
      │
      ▼
  Shared physical base pages across sandboxes          ✅

systrap: overlay invisible to the guest
  Guest execution in stub process
      │
      ▼
  Stub MAP_SHARED maps per-sandbox memfd
      │
      ▼
  memfd has holes/zeros where base pages were skipped
      │
      ▼
  Sentry overlay is correct; guest never executes from it   ❌
```

The prototype was not generically broken; it targeted the platform where this overlay design
composes naturally. On KVM, the guest consumes the sentry's `MapInternal` view, so the shared
base reaches the running guest. On systrap, the guest runs from a separate stub process
mapping the memfd directly, so the overlay is invisible to it. Cross-sandbox base sharing,
intra-sandbox sentry/stub coherence, and per-page base/delta composition are not all
available from plain mmap on systrap; the realistic options there are KSM (let the host dedup
identical pages across sandboxes, which is simple but opportunistic and content-blind) or a
deeper coherent shared-base plus delta-overlay mechanism. KSM also carries a security cost the
explicit design avoids: cross-tenant page deduplication is a documented side channel, since an
attacker can infer another tenant's memory contents from merge timing, whereas sharing a
known, public base leaks nothing, because there is nothing secret to discover.

KVM is different, and the two platforms' `MapFile` implementations are what decide it:

```go
// systrap: the guest runs in a separate stub process that maps the
// per-sandbox memfd. The base overlay lives on the sentry's mapping, not here,
// so the stub sees holes where the base pages were skipped.
mmap(MAP_SHARED, f.DataFD(fr), fr.Start)

// KVM: the guest runs from the sentry's own mapping, the exact chunk
// the base overlay modified copy-on-write, so the guest sees the shared base.
bs, err := f.MapInternal(fr, ...)
```

There is no separate stub with an independent `MAP_SHARED` memfd view, and there is one
process per sandbox, so `MAP_PRIVATE` base plus copy-on-write is coherent and shares across
sandboxes via the page cache. The design lines up with KVM's memory path, and the measured
~10× flatten is at the layer KVM consumes. What remained was end-to-end validation on real
`/dev/kvm`, which the Apple-silicon test setup could not provide, leaving systrap as the only
platform on hand. The pgalloc unit tests read through `MapInternal`, exactly the path KVM
uses for guest memory, so they are valid coverage of the real target.

## Getting to real KVM

That is where the story sat: the design lines up with KVM, but no `/dev/kvm` was anywhere in
reach to prove it on a running guest. Leaving a load-bearing claim resting on a code read is
an uncomfortable place to stop, so the next move was to go find hardware.

The answer was Google Cloud. A GCE instance with nested virtualization enabled exposes
`/dev/kvm` backed by Linux KVM, which is the environment gVisor's KVM backend is built to
use. It runs there without the host-resetting crashes that nesting KVM under Apple's or
VMware's hypervisors produced. The same patched runsc, built on that instance, could finally run the test the
earlier setup could not.

First, `runsc --platform=kvm` ran a sandbox at all, and the host stayed up. Then the test
that mattered: the same small stateful workload, checkpointed and restored two ways, this
time on KVM:

```
restore WITHOUT base.img (--platform=kvm):  ticks continue, checksum=ok   (regression clean)
restore WITH    base.img (--platform=kvm):  ticks continue, checksum=ok   ✅
```

The workload that crashed on systrap's shared-base restore resumed cleanly on KVM, memory intact, with the debug
log confirming the base image was threaded in and the overlay applied. The platform boundary
was not a hedge. KVM maps the guest from `MapInternal`, exactly the mapping the overlay
modifies, so the guest sees the shared base and the copy-on-write delta, while the same
restore on systrap still dies because its stub reads a different memfd. Predicted from the
code, confirmed on hardware.

The checkpoint side, missing until now, also came together. `runsc checkpoint --shared-base`
exports the base and writes a delta-only checkpoint. For a freshly warmed clone the delta is
zero, so the pages file comes out empty and all of guest RAM lands in the shared base image.
The lifecycle composes: because `SaveTo` reads the composed `MapInternal` view (base overlay
plus private delta), a sandbox restored over a base and then dirtied can be checkpointed again
into a fresh delta against the same base. Repeated generations are not exercised end to end
yet, but nothing in the design requires re-basing to do it. With both sides in place, the
flatten is measurable through the real runsc CLI on a live KVM guest rather than through the
pgalloc tests: eight real sandboxes restored over one shared 256 MiB base gave a 5.7× flatten
in summed sentry PSS.

That is lower than the ~10× the pgalloc tests reported, and the gap is the subject of the
next section.

## A real agent, and a humbler flatten

Everything so far used workloads built to be measurable: a Go HTTP server, a C program with a
checksummed heap. A real one was needed to trust the result. The choice was an agent on
Google's Agent Development Kit (google-adk) with the LLM endpoint mocked out, so it drives the
full framework (sessions, the runner, the model interface) with no network. Checkpointed and
restored under KVM, it came back correctly: live Python, asyncio, and gRPC all intact.

The density flatten for the real agent was modest: about 2.9× across eight clones over an 80
MiB base. Two honest reasons, and both matter more than the number.

First, the flatten only ever applies to memory that is actually resident. Inflating the base
by having the agent allocate a gigabyte at startup did nothing: memory the clones never touch
after restore is never faulted in, so it costs no physical RAM and there is nothing to share.
Density scales with the shared resident working set, not with allocated size.

Second, each sandbox carries a fixed floor of about 20 MiB that cannot be shared: roughly 17
MiB of sentry and 3 MiB of gofer, about half of it live Go runtime across those two Go
processes. It is not garbage the collector can reclaim (capping the GC moved nothing); it is
threads, goroutine stacks, and runtime structure, on top of a full guest kernel (the sentry
is hundreds of thousands of lines of Go implementing hundreds of syscalls, which is not a
thing you shrink). With an 80 MiB resident base and a 20 MiB unshareable floor, the best
possible eight-clone sharing factor is about `(80 + 20) / (80/8 + 20) = 3.3×`, before any
other overhead. The measured 2.9× is not a surprise; it is close to the floor-limited
ceiling. On a 15 GiB, four-core box about 300 of these sandboxes fit before memory ran out.

For small agents, this changes the density story: base sharing is real, but the fixed sandbox
floor dominates before the base-sharing mechanism does. The same agent had a better result to
offer, and it had nothing to do with memory.

## The win I was not looking for

Cold-starting the agent under gVisor takes about eleven seconds, almost all of it CPU:
importing roughly 1,400 Python modules, most of the time spent inside the Gemini SDK
constructing its Pydantic type hierarchy as the modules load. It is not fetching anything
(the sandbox has no network and the packages are pre-installed) and not compiling bytecode
(every `.pyc` is already present); it is executing that much initialization code, amplified
by gVisor's per-syscall cost on a file-heavy import.

Restoring from a snapshot skips all of it, mapping the already-initialized image and
resuming:

```
                 cold start (import + init)      restore from snapshot
 wall            11.4 s                          0.6 s
 CPU              7.1 s                          ~0.5 s
```

"Sub-second restore" is exactly the kind of claim that hides a fault-in stall, so it was
worth trying to break. Three checks.

*Is it actually warm?* Yes. The restored agent serves its first real request in about 0.6
seconds and its second in 29 milliseconds, steady at 20. There is no multi-second thrash.
(One instrument read three seconds for the first request, which was alarming right up until
the cause turned out to be self-inflicted: a wall-clock timer left running across the
checkpoint freeze, dutifully counting the time the process spent stopped. The external clock
and the 29 ms second request set the record straight.)

*Is it correct?* Bit-exact. Every post-restore response verified through the full agent path,
and an in-memory accumulator the agent kept came back with precisely the value it would have
had if it had never stopped.

*Is it cheap to park?* Checkpointing takes 0.11 seconds, and with a shared base each parked
agent costs about 248 KiB (its kernel state plus a near-zero delta) instead of the ~81 MiB of
a full checkpoint. A pool of ten thousand parked agents is about 2.5 GiB of checkpoint storage
at ~248 KiB each (plus the one-time ~81 MiB base image), instead of about 790 GiB at ~81 MiB
each: a couple of gigabytes rather than most of a terabyte.

The caveat is burst. A single restore is sub-second, but fifty at once on four cores take
about 18 seconds and a hundred take forty-plus, because each restore plus its first request is
roughly 1.5 seconds of CPU and they contend for the cores. Restore is not instant scale-up.
The two CPU figures are not in tension: a single uncontended restore is about 0.5 s of CPU,
roughly fourteen times less than a cold start's 7.1 s, but the number that matters under load
counts each start's first-request fault-in too (about 1.5 s of CPU), and that is where the
roughly four-times-cheaper figure comes from. It is the narrower and more defensible claim:
provision cores for the burst rate, and each start costs a fraction of what it did.

## The part that runs anywhere

The density flatten is KVM-only, because it depends on the base overlay reaching the guest,
and that only happens on KVM. Skipping the cold start does not use the overlay at all; it
needs only ordinary checkpoint and restore. In these tests, that made the latency win work on
both KVM and systrap. The same cold-versus-restore test on systrap, the platform used where
there is no `/dev/kvm`:

```
 systrap    cold 4.05 s / 1.99 s CPU   →   restore 0.18 s / 0.26 s CPU   (correct)
```

Faster than KVM here, in fact, because gVisor's KVM backend pays nested-virtualization
overhead on the rented cloud host; on bare metal that gap narrows. The exact number is not the
point. The point is that the conditional, KVM-only feature this work set out to build sits
right next to a broader result that turned up alongside it: restore
an agent instead of cold-starting it, and each start is warm, correct, and several times
cheaper on both platforms tested.

## What the work showed

- **The kernel primitive and the economics check out.** Linux shares `MAP_PRIVATE` file pages
  copy-on-write (PSS ≈ base/N), and a read-mostly agent's per-clone delta is ~1.7% of its RAM.
  Density is delta-bound.
- **The flatten is real, and it was the smaller prize.** Through gVisor's save/restore path it
  scales 5× to 17× with clone count at the pgalloc layer; on a live KVM guest through the
  runsc CLI it is lower (about 5.7× at eight clones, 2.9× for a real 80 MiB agent) because of a
  per-sandbox floor the pgalloc tests never see.
- **The floor is the density ceiling.** About 20 MiB per sandbox is not shareable, so base
  sharing only wins big when the shared resident base is large next to that floor.
- **Verify-before-expose composes without hurting density.** Terrapin verification of the
  shared base, once per node against an authenticated dataset ID, did not materially change the
  measured resident-memory flatten; tampering is caught before exposure.
- **The platform's guest-mapping call hides a load-bearing difference.** KVM maps
  the guest from the sentry's own mapping (`MapInternal`); systrap maps it from the per-sandbox
  memfd into a separate stub. A memory feature living in `pgalloc` is implicitly betting on
  which one the guest actually executes from.
  This one was right for KVM (confirmed on hardware) and wrong for systrap, exactly as the code
  predicted.
- **End-to-end tests, and real workloads, find what unit tests cannot.** Nine green unit tests
  found a measured flatten. One real checkpoint/restore found the platform bug. And one real
  agent showed that the memory density this work set out to find was a conditional bonus, while
  the startup saving it was not looking for was the robust, universal result.

## What's next

- **Density on a large active base.** The flatten is modest for a small agent and should be
  large for one that keeps a big model or index resident and reads it every turn. That is the
  measurement that would make the density case, and it has not been run end to end.
- **A systrap density story.** The latency result already works on systrap; the memory result
  does not. Kernel-samepage-merging on the per-sandbox memfds is the pragmatic,
  verification-free option; a coherent shared-base-plus-delta across the sentry and stub is the
  explicit one.
- **The floor itself.** The one tractable piece of the 20 MiB floor is the gofer, which
  directfs already sidelines at runtime but still runs as a whole second Go process. Reaping it
  is worth more than any GC tuning.
- **Base-aware accounting.** Resident base pages should be counted once per node, not once per
  sandbox.

The gVisor patches (the base/delta `pgalloc` change plus the checkpoint and restore plumbing)
and the full measurement harness live in the
[Substrate repository](https://github.com/agent-substrate/substrate).

Base sharing works where the platform consumes the sentry's composed memory view: KVM does,
systrap does not. That makes memory sharing a real density bonus on KVM when the active
resident base is large enough.

The result to build around first is broader: do not cold-start near-identical agents.
Snapshot them after initialization and restore them when needed. Skipping startup is the portable win; shared guest memory is the KVM bonus.

There is one more door this opens. The property that makes a shared base safe,
verify-before-expose, is also what lets the base travel. A content-addressed,
cryptographically verified block can be fetched from any untrusted peer, mirror, or cache and
re-checked before the guest sees it, so nothing on the wire has to be trusted. The local win
(one verified base resident per node) and the fleet win (distributing that base over
untrusted transport) are the same mechanism at two radii, and that distributed verified store
is the next thing to build.

## Measurement details

Audit material for the experiments above, kept out of the main narrative.

**Linux smoke test, larger base.** The same `density_smoke.c`, 1 GiB base: N=8 gives
pss=137 MiB, N=64 gives pss=16 MiB (= base/64). PSS tracks base/N as expected.

**Eager verification.** Verifying the whole base against the manifest, then sharing: 35 ms per
64 MiB (~1.9 GB/s), paid once per node, independent of clone count. Flatten with verification
in the loop: N=16 → 8.0×, N=32 → 15.6×. (These verification-path runs use a different
base/delta mix than the flatten table in the main text, so their per-N figures are not
directly comparable to it; the same applies to the lazy numbers below.)

**Lazy / remote verification.** A userfaultfd MISSING handler over a node-shared backing,
16 blocks with one known-zero (recognized by its verified zero-constant leaf hash) and one deliberately tampered:

```
fetches=15  verifies=15  zero-installs=1  rejects=1
re-touching every block added 0 fetches  → once per node
tampered block → rejected by verify, never exposed
flatten: N=16 → 9.1×,  N=32 → 17.3×
```

**Block-level costs.** Verifying a 2 MiB block is ~0.9 ms (2.33 GB/s), cheaper than the
page-fault populate it rides on (1.18 ms), so a full 1 GiB base verifies in ~0.43 s, once per
node. Capturing a base from a live sandbox runs at ~3.7 GB/s.

**Environment.** Density and latency runs were on a GCE `n2-standard-4` (four vCPUs, ~15 GiB,
nested virtualization enabled for real `/dev/kvm`); gVisor built from source with the base/delta
patches. The nested-KVM host mattered: an Apple-silicon setup exposed no usable `/dev/kvm` at all,
VMware nesting crashed when the KVM backend ran, and only GCE's Linux KVM-backed nested
virtualization ran it cleanly.
