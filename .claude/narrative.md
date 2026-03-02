# Project Narrative: kernel-builder

## Summary
A Nix-based build infrastructure for creating optimized computational kernels across multiple hardware backends (CUDA, ROCm, Metal, Intel XPU). The project provides build2cmake, a Rust tool that converts `build.toml` specifications into CMake files, while Nix packages manage the labyrinthine dependency ecosystem (Torch, CUTLASS, cuDNN, SYCL, etc.). This repository is now archived as the project has migrated to the `kernels` repository.

## Current Foci
This repository is now **archived and read-only** as of commit 078351d, with active kernel development moved to the `kernels` repository. Occasional follow-up work (CI improvements, dependency updates) continues. This repo serves as:
- Reference implementation for build2cmake and Nix infrastructure patterns
- Historical record of the Torch binary migration (source → binary packages)
- Documentation of multi-backend (CUDA, ROCm, Metal, XPU) build patterns
- CI/testing infrastructure for Metal and multi-backend builds

## How It Works
The system has three main layers:

1. **build2cmake** (Rust tool): Parses `build.toml` kernel specifications and generates CMake build files using minijinja templating. Handles versioning, backend selection, and dependency resolution.

2. **Nix Package Layer** (`pkgs/`, `lib/`, `overlay.nix`, `versions.nix`): Manages complex interdependencies:
   - Binary Torch packages (via `torch-versions.json` with hash verification)
   - Backend toolchains: CUDA, ROCm, Metal, SYCL
   - Specialized packages: CUTLASS, cuDNN, aotriton, MAGMA
   - Backend-specific Python dependencies (einops, nvidia-cutlass-dsl)

3. **Output Generation** (`lib/gen-flake-outputs.nix`): Produces build variants (CPU, CUDA, ROCm, Metal, XPU combinations) and exposes them as Nix flake outputs.

Kernels are defined by authors in `build.toml`, which specifies C++/CUDA sources, dependencies, backend targets, and Torch version constraints. build2cmake converts this to CMake, which Nix then builds in isolated sandboxes with the correct toolchain.

## The Story So Far

**Early Era** (pre-2.10): Started as a local kernel build system with Torch source builds. This worked but was slow, fragile, and version-locked.

**Simplification Phase** (~commits da0367c-0c49047): Realized Torch binaries are more reliable than source builds. Removed 3000+ lines of Torch build code and replaced with a `torch-versions.json` manifest. This was a major architectural win—vastly simpler, faster, and more maintainable.

**Polyglot Backends** (2024-2025): Expanded beyond CUDA. Added ROCm support, Metal (Apple Silicon) support, and Intel XPU (Windows). Each backend needs different toolchain setup, which Nix abstracts cleanly.

**Version Proliferation**: Supporting multiple Torch release cycles simultaneously (2.8, 2.9, 2.10 RCs). The team settled on `torch-versions.json` for both binaries and metadata—much cleaner than hardcoded version matrices.

**Recent Transition** (commit 078351d): Kernel definitions and builders moved to the `kernels` repo, leaving this as a reference implementation / archived state. This reflects the desire to co-locate kernel code with builder infrastructure.

**Post-Migration Polish** (f7fe17e onwards): After the main migration, the team added Metal test job to macOS CI (f7fe17e), showing that even though this repo is archived, CI/testing infrastructure continues to mature to catch edge cases across backends.

## Dragons & Gotchas

- **Torch Source Builds**: Removed recently but were a significant maintenance burden. If ever needed again, note the patches are gone.
- **CUDA Capability Handling**: Subtle. Need `-static-global-template-stub` flags conditionally based on CUDA < 12.8. Easy to miss when adding new templates.
- **CMake Dependency Lookup Warnings**: Can get spurious "dependency not found" warnings; build2cmake's C++ dependency resolver had to be refined.
- **Windows CUDA/ROCm Guards**: PyTorch has conditional guards around CUDA/ROCm; Windows build needed `USE_CUDA`/`USE_ROCM` forced to prevent guards from being bypassed.
- **Torch Overlays**: The project overlays a local Torch package into the Nix set; this can shadow or conflict with global Torch packages if not careful.
- **Backend Library Linking Order**: Static libraries must be linked in dependency order (reverse of includes). Easy to get "undefined reference" errors if order is wrong.
- **LD_LIBRARY_PATH Pollution**: Devshell had to explicitly unset `LD_LIBRARY_PATH` to avoid runtime errors when mixing host and target libs.

## Open Questions

- **Maintenance of Archive**: The `kernel-builder` repo is now archived and serves read-only. If users find bugs in build patterns documented here, fixes go into the `kernels` repo.
- **Binary Torch Longevity**: With Torch source builds removed, we depend entirely on Torch's official binary releases. If a critical issue is found only in source builds post-2.10 stable, we have no fallback.
- **Metal and XPU Coverage**: Metal profiling and Intel XPU Windows support were relatively new when the migration happened. Real-world edge cases may exist in the `kernels` repo implementation.
