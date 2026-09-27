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

**Update:** most calls below now pass an explicit `threads=` matching its task's
`runtime.cpu`. Previously, only `bbmap.sh` did; most others fell back to
BBTools' own default (`threads=auto`, i.e. all cores visible to the JVM), which was
a real oversubscription risk on shared NERSC/Shifter nodes — see
`memory_and_threads.md`. Line numbers below reflect the file after that fix.

| BBTools call | Enclosing task | Threads setting | Memory setting | WDL file | Line |
|---|---|---|---|---|---|
| `bbcms.sh` | `bbcms` | `threads=16` (hardcoded, matches `runtime.cpu: 16`) | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 499 |
| `reformat.sh` | `bbcms` | `threads=16` (hardcoded, matches `runtime.cpu: 16`) | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 514 |
| `readlength.sh` | `bbcms` | no `threads=` flag — this CLI does not accept one | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 522 |
| `reformat.sh` | `stage` | `threads=2` (hardcoded, matches `runtime.cpu: 2`) | task-local `String memory = "4G"` (not a workflow input) → always `-Xmx4G` | `shortReads_assembly.wdl` | 141 |
| `bbmap.sh` | `read_mapping_pairs` | `threads=~{jvm_threads}`, where `jvm_threads = select_first([threads, system_cpu])` and `system_cpu` = core count from `/proc/cpuinfo` | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 315 |
| `reformat.sh` | `read_mapping_pairs` | `threads=~{jvm_threads}` (matches `bbmap.sh` above and `runtime.cpu: 16`) | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 339 |
| `fungalrelease.sh` | `create_agp` | `threads=16` (hardcoded, matches `runtime.cpu: 16`) | `-Xmx~{memory}` if `memory` defined, else `-Xmx105G` | `shortReads_assembly.wdl` | 383 |
| `bbstats.sh` | `create_agp` | `threads=16` (hardcoded, matches `runtime.cpu: 16`) | none passed at all (no `-Xmx` flag) | `shortReads_assembly.wdl` | 399 |

## Notes

- `bbmap.sh` and the second `reformat.sh` in `read_mapping_pairs` share
  `jvm_threads`, which already resolves to `threads` (the workflow input) or the
  `/proc/cpuinfo` core count if unset — see `memory_and_threads.md` for why setting
  the workflow's `threads` input to `16` still matters here despite this fix.
- The other six calls (`bbcms`, `stage`, `create_agp` tasks) now use a hardcoded
  `threads=` literal rather than a workflow input, since none of those tasks
  exposed a `threads` parameter and the fix was scoped to matching each task's
  fixed `runtime.cpu` rather than threading a new input through.
- `bbstats.sh` (line 397) is still the only call in this table with **no** memory
  flag passed at all — not even the `-Xmx105G` default the other calls fall back
  to. Not addressed by this fix; `bbstats.sh` is a lightweight single-pass stats
  computation, so this is lower-priority than the threads gap was.
- All eight calls still run inside tasks with hardcoded `runtime.memory`/
  `runtime.cpu` (`120 GiB`/`16` cpu in `bbcms`, `read_mapping_pairs`, `create_agp`;
  `4 GiB`/`2` cpu in `stage`) — see `memory_and_threads.md` for why the `-Xmx`
  values above rarely matter given those hardcoded ceilings. This fix only
  addresses thread counts, not the oversized/copied-boilerplate memory requests
  (e.g. `create_agp`'s `120 GiB` for comparatively light work).
