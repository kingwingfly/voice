# Repository guidance

## Start with the existing work

Read [CLAUDE.md](CLAUDE.md) before making implementation changes. It is the
shared repository guidance for all coding agents, despite its filename, and
records the architecture, porting traps, verification recipes, and reasons for
past decisions. Consult the sections relevant to the task rather than
rediscovering or reversing those decisions. Keep detailed rationale there;
this file is a short entry point and the local environment override.

## Local build environment

LibTorch is already installed at `/opt/libtorch` (2.9.0+cu130, matching
`tch 0.22`). Use it; do not download another copy or disable the `tch` backend
to work around a missing library path.

```sh
export LIBTORCH=/opt/libtorch
export LD_LIBRARY_PATH="/opt/libtorch/lib${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
```

Set these in each shell that builds, tests, or runs the project. `LIBTORCH`
selects the build-time installation; `LD_LIBRARY_PATH` lets executables and
tests load its shared libraries. These paths supersede the illustrative
`$PWD/libtorch` paths in the existing documentation for this machine. Do not
bake `/opt/libtorch` into the binaries' portable RUNPATH.

See [docs/setup.md](docs/setup.md) for ffmpeg development libraries, ONNX
Runtime, backend features, and device selection. `ORT_DYLIB_PATH` is needed
when using a non-system ONNX Runtime installation; it is separate from
LibTorch configuration.

## Resource limits on this machine

Resources are very limited. Keep the machine responsive and avoid GPU OOMs or
driver crashes; this takes priority over running a broad validation workload.

- Run builds and model workloads sequentially. Default to one Cargo build job,
  one test thread, and one CPU math-library thread. Start with focused checks
  before the workspace gate; do not run benchmarks or training unless the task
  calls for them.
- Prefer explicit `--backend tch --device cpu` for model smoke checks that do
  not require a GPU. Avoid `auto` for these checks, since it may select the GPU.
  Inspect tests/examples for hardcoded GPU selection before running them.
- Before a necessary GPU run, check available RAM and GPU memory/load (for
  example, `free -h` and `nvidia-smi`). Run only one GPU workload at a time,
  start with a short clip and the smallest useful batch/model, leave memory
  headroom, and monitor usage. Do not launch when existing workloads leave
  insufficient headroom. Stop the task's own workload on memory pressure or
  driver errors; do not reset the GPU or terminate unrelated processes.
- If a required check cannot fit safely, use a smaller or CPU check where
  meaningful and report exactly which validation remains unrun.
- `/tmp` is **tmpfs**: large temporary files consume RAM. Put scratch data,
  downloads, generated audio, and temporary build files under the agent's
  disk-backed home (`/home/ai`). Reuse the configured Cargo target directory
  (`~/target`) for ordinary sequential builds. Do not put large assets or
  build outputs in `/tmp` or `/dev/shm`.

Use these defaults in development shells, alongside the LibTorch exports:

```sh
mkdir -p "$HOME/.cache/voice-dev/tmp"
export TMPDIR="$HOME/.cache/voice-dev/tmp"
export CARGO_BUILD_JOBS=1
export RUST_TEST_THREADS=1
export OMP_NUM_THREADS=1
export MKL_NUM_THREADS=1
export OPENBLAS_NUM_THREADS=1
```

## Build and verification

Cargo's target directory is already configured as `~/target`, outside this
repository. A missing workspace-local `target/` is expected. Reuse the existing
configuration and cache; do not create a replacement or run `cargo clean`
because artifacts are absent from the workspace. When locating binaries,
resolve the actual path from Cargo configuration or `cargo metadata`.

Run Cargo from the workspace root. Prefer the dev profile for development and
model checks: dependencies are already optimized. Use release builds for
benchmarking or distribution.

```sh
cargo build                                      # all six binaries
cargo build -p rvc-cli                            # one binary
cargo check -p rvc-core --features cuda,tch
cargo check -p rvc-train --features cuda,tch
cargo test -p text-kit                            # example focused suite

# Workspace gate for code changes (there is no CI):
cargo fmt --all -- --check
cargo clippy --workspace
cargo test --workspace
```

Choose focused checks for the affected crates as well. Model changes may need
explicit `--features tch` tests and real-weight/audio examples; the default
workspace tests do not cover every backend, and weight coverage alone does
not verify inference arithmetic. Use the recipes and expected results in
`CLAUDE.md` and the examples' module documentation. Do not run ignored tests
without their documented model/audio prerequisites.

Concurrent build processes, including those in other worktrees, need distinct
`CARGO_TARGET_DIR` values. Shared example/test outputs have previously caused
plausible results from the wrong checkout. Verify the combined tree after
integrating parallel work.

## Constraints to preserve

- This is a Rust 2024 speech toolkit. No Python in setup, user workflows,
  tests, or training. The maintainer-only ONNX exporters under `export/` are
  the documented exception; upstream reference repositories are read-only
  porting references, not programs to run.
- Shared infrastructure belongs in `*-kit`; model implementations belong in
  `burn-*`; engine behavior belongs in `*-core`, `*-train`, and `*-cli`.
  Engines do not depend on other engines. Only the integration is `voice-*`.
- Each `*-cli` exposes its clap arguments as a library for `voice-cli` to nest.
  Do not duplicate command definitions. Bare engine invocations are streaming
  filters; `convert` is the batch form. `preprocess` requires a stage subcommand.
- Keep stdout for pipeline data and stderr for logging. Audio boundaries use
  mono `f32` streams, with the documented stereo separation path as an exception.
- Keep Burn networks backend-generic and checkpoint-compatible. Module field
  names determine parameter paths; changing them can break pretrained loading,
  warm-start, and export. Reuse shared backend/device resolution.
- Downloaded assets belong in the shared cache; output directories contain
  only run products. Preserve the overwrite protection for trained weights.
- Keep `README.md` an index, CLI usage in `crates/<name>-cli/README.md`, and
  shared setup/training material in `docs/`. Architecture papers are Typst;
  update their rendered PDFs together with their sources.
