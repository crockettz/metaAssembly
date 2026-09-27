# BBTools invocations in the short-read path

Every place a BBTools binary is actually invoked (`*.sh`) in the short-read
(`shortRead=true`) pipeline, with the enclosing WDL task, whatever `threads=`/`-Xmx`
settings apply, the WDL file, and the line number of the call.

Scope: `shortReads_assembly.wdl` (the `jgi_metaASM` workflow) plus
`make_interleaved_reads.wdl` (short-read preprocessing). `jgi_assembly.wdl`'s
`finish_lrasm` task also calls `bbstats.sh`, but that's on the long-read path and is
out of scope here. `finish_asm` (short-read finishing task, `shortReads_assembly.wdl`)
calls `jq`/`python3`/`sed`/`samtools` only — no BBTools binaries, not listed below.
`make_interleaved_reads.wdl`'s `interleave_reads` task is given a `bbtools_container`
but never actually calls a BBTools binary (uses `paste`/`gunzip`/`tr`) — also not
listed below.

| BBTools call | Enclosing task | Threads setting | Memory setting | WDL file | Line |
|---|---|---|---|---|---|
| `bbcms.sh` | `bbcms` | none passed — BBTools default (`threads=auto`) | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 490 |
| `reformat.sh` | `bbcms` | none passed — BBTools default (`threads=auto`) | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 504 |
| `readlength.sh` | `bbcms` | none passed — BBTools default (`threads=auto`) | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 511 |
| `reformat.sh` | `stage` | none passed — BBTools default (`threads=auto`) | task-local `String memory = "4G"` (not a workflow input) → always `-Xmx4G` | `shortReads_assembly.wdl` | 138 |
| `bbmap.sh` | `read_mapping_pairs` | `threads=~{jvm_threads}`, where `jvm_threads = select_first([threads, system_cpu])` and `system_cpu` = core count from `/proc/cpuinfo` | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 311 |
| `reformat.sh` | `read_mapping_pairs` | none passed — BBTools default (`threads=auto`) | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 332 |
| `fungalrelease.sh` | `create_agp` | none passed — BBTools default (`threads=auto`) | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 374 |
| `bbstats.sh` | `create_agp` | none passed — BBTools default (`threads=auto`) | none passed at all (no `-Xmx` flag) | `shortReads_assembly.wdl` | 391 |

## Notes

- `bbmap.sh` in `read_mapping_pairs` is the **only** BBTools call in the short-read
  path that receives an explicit `threads=` value from the workflow's `threads`
  input — see `memory_and_threads.md` for the oversubscription risk when `threads` is
  left unset.
- Every other BBTools call — `bbcms.sh`, `reformat.sh` (both instances),
  `readlength.sh`, `fungalrelease.sh`, `bbstats.sh` — has no `threads=` flag at all
  and falls back to whatever BBTools' own default thread count resolves to in the
  container (BBTools documents this default as `threads=auto`, i.e. "use all
  detected cores"; whether the JVM sees the container's actual cpuset limit or the
  full host core count is an environment question we haven't independently
  verified for this Shifter setup).
- `bbstats.sh` (line 391) is the only call in this table with **no** memory flag
  passed at all — not even the `-Xmx105G` default the other calls fall back to.
- All eight calls run inside tasks with hardcoded `runtime.memory`/`runtime.cpu`
  (`120 GiB`/`16` cpu in `bbcms`, `read_mapping_pairs`, `create_agp`; `4 GiB`/`2`
  cpu in `stage`) — see `memory_and_threads.md` for why the `-Xmx` values above
  rarely matter given those hardcoded ceilings.
