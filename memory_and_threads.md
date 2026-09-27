# `memory` / `threads` inputs (short-read path)

Notes on what the top-level `jgi_metaAssembly.memory` and `jgi_metaAssembly.threads`
inputs (`String?`, no default) actually do in the short-read (`shortRead=true`) path
through `shortReads_assembly.wdl`. Long-read (Flye) path does not use these inputs at all.

## Where `memory` goes

| Task | Uses `memory` input? | Behavior if unset |
|---|---|---|
| `stage` | No — has its own local `String memory = "4G"`, unrelated to the workflow input | `-Xmx4G` |
| `bbcms` | Yes (`String?`) | `-Xmx105G` for `bbcms.sh`, `reformat.sh`, `readlength.sh` |
| `predict_memory` | No — computes memory/cpu itself from k-mer count | n/a |
| `assy` (metaSPAdes) | No — always called with `memory = predict_memory.resource.request` | n/a |
| `create_agp` | Yes (`String?`) | `-Xmx105G` for `fungalrelease.sh` |
| `read_mapping_pairs` | Yes (`String?`) | `-Xmx105G` for `bbmap.sh`, `reformat.sh` |

In every task where `memory` is consumed, it only sets the JVM `-Xmx` heap flag.
The actual container resource reservation (`runtime.memory`) is **hardcoded to
`120 GiB`** in `bbcms`, `create_agp`, and `read_mapping_pairs` regardless of the
input value. Since the default `-Xmx105G` already sits under that 120 GiB ceiling,
there's no real scenario where changing the input helps: raising it risks exceeding
the 120 GiB container cap, lowering it has no upside.

`predict_memory` derives memory/cpu/runtime independently from a curve fit on
k-mer count, snapping to one of three tiers: `(120 GiB, 16 cpu)`, `(250 GiB, 32 cpu)`,
`(500 GiB, 32 cpu)` (defaults to the last tier if predicted memory exceeds all
thresholds). This result feeds `assy` directly — the workflow's `memory` input never
reaches metaSPAdes.

**Conclusion: `memory` is effectively vestigial.** It changes a JVM heap flag that
never gets close to mattering given the hardcoded container memory caps, and the one
task that needs real dynamic sizing (`assy`) ignores it entirely.

## Where `threads` goes

| Task | Uses `threads` input? | Fallback if unset |
|---|---|---|
| `stage` | No | hardcoded `cpu: 2` |
| `bbcms` | No such parameter | BBTools auto-detects (JVM `availableProcessors()`) |
| `assy` (metaSPAdes) | No — always called with `threads = predict_memory.resource.cpu` | n/a (workflow `threads` input never reaches it) |
| `create_agp` | No such parameter | BBTools auto-detects |
| `read_mapping_pairs` | Yes (`String?`) | `select_first([threads, system_cpu])` where `system_cpu = grep "model name" /proc/cpuinfo \| wc -l` — i.e. **all cores visible in the container**, not the `16` cpu actually requested via `runtime.cpu` |

`read_mapping_pairs` is the one task where an unset `threads` input creates a real
risk: on a shared/multi-core node, `jvm_threads` can end up far higher than the `16`
CPUs Cromwell/Shifter scheduled for it, oversubscribing the node via `bbmap.sh
threads=` and `samtools sort -@`.

**This risk is not unique to `read_mapping_pairs`.** Every BBTools call but one in the
short-read path — `bbcms.sh`, `reformat.sh` (all three instances, in `stage`,
`bbcms`, and `read_mapping_pairs`), `fungalrelease.sh`, and
`bbstats.sh` — has no `threads=` flag passed at all, since none of those tasks
(`stage`, `bbcms`, `create_agp`) even expose a `threads` parameter to set one. Left
unset, BBTools' own default (`threads=auto`) applies, which resolves to
`Runtime.getRuntime().availableProcessors()` inside the JVM — i.e. "use all cores
the JVM can see." Whether that ends up being the `16`/`2` cpu the task's
`runtime.cpu` actually requested, or the full host core count, depends on whether
the container runtime (Shifter, per `shifter.conf`) properly scopes the visible
cpuset/cgroup down to the Cromwell allocation. On a shared NERSC node this is a real
oversubscription risk across the board, not just in `bbmap.sh`/`read_mapping_pairs`
— it's just more consequential for `bbcms`, since that's the heaviest, most
CPU-intensive task in the pipeline (k-mer counting over the full raw read set). See
`bbtools_shortreadss_instances.md` for the full call-by-call breakdown.

## Does setting `threads=16` fix everything?

Partially:

- **Fixes `read_mapping_pairs`** — `jvm_threads` becomes `16`, matching its
  hardcoded `runtime.cpu: 16`. This was the only task with an actual all-cores
  risk.
- **Doesn't touch `assy`** — cpu/threads come from `predict_memory.resource.cpu`
  (16 or 32 depending on k-mer tier), independent of the workflow's `threads` input.
  Internally consistent, just not controlled by you.
- **Doesn't touch `bbcms`/`create_agp`** — no `threads` parameter exists on these
  tasks; concurrency is whatever BBTools decides on its own (should track
  `runtime.cpu: 16` if cgroup limits are respected by the JVM, but not guaranteed
  by anything in the WDL).
- **Doesn't touch `stage`** — no `threads` parameter; hardcoded `cpu: 2`.

So `threads=16` is worth setting (removes the one real oversubscription risk in
`read_mapping_pairs`), but it is not a blanket guarantee that every task's actual
thread usage matches its resource request.

## Implication for an API that only supports short reads

If a wrapping API only needs to support the short-read path, it should only need to
accept `proj` and `input_files` from the caller. The API layer would then construct
the Cromwell `input.json` itself, adding the fields the caller shouldn't need to
think about:

```json
{
  "jgi_metaAssembly.proj": "<proj>",
  "jgi_metaAssembly.input_files": ["<file1>", "..."],
  "jgi_metaAssembly.shortRead": true,
  "jgi_metaAssembly.threads": "16"
}
```

- `shortRead: true` — fixed, since the API only supports short reads.
- `threads: "16"` — matches `read_mapping_pairs`'s hardcoded `runtime.cpu: 16`,
  closing off the one real oversubscription risk described above. Must be a JSON
  string (`"16"`), not a number, since the WDL type is `String?`.
- `memory` — leave unset. As established, it's vestigial for the short-read path.
- `prefix` — no need to supply; defaults to `sub(proj, ":", "_")`.

Everything else in `jgi_metaAssembly`'s `input {}` block (container images, tool
parameters) has a workflow-level default and is irrelevant once `shortRead=true`,
since the long-read branch (`if (!shortRead)`) never executes — Cromwell doesn't
require inputs that have defaults, so they can simply be omitted from the generated
JSON.
