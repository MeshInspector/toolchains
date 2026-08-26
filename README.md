# toolchains

Prebuilt compiler toolchains for the MeshInspector CI: `-O3` + PGO builds of
LLVM/Clang that are considerably faster at compiling C++ than the stock distro
packages or Homebrew bottles of the same version.

Everything is published as **GitHub release assets** in this repository. There is
no package index and no authentication — every asset is a plain
`https://github.com/MeshInspector/toolchains/releases/download/<tag>/<asset>`
URL that `curl` can fetch anonymously.

## Releases

| Tag | Platform | Archs | Unpacks to | Size | Status |
| --- | --- | --- | --- | --- | --- |
| [`clang-22.1.8-pgo-dylib-linux`](https://github.com/MeshInspector/toolchains/releases/tag/clang-22.1.8-pgo-dylib-linux) | Linux, glibc >= 2.28 | `x86_64`, `aarch64` | `llvm-pgo-dylib-22.1.8/` | ~185 MB | **current** |
| [`clang-emsdk-4.0.19-pgo`](https://github.com/MeshInspector/toolchains/releases/tag/clang-emsdk-4.0.19-pgo) | `emscripten/emsdk:4.0.19` (jammy, glibc >= 2.35) | `x86_64`, `aarch64` | `llvm-emsdk-4.0.19-pgo/` | ~114 MB | **current** |
| [`clang-18.1.8-pgo-dylib-linux`](https://github.com/MeshInspector/toolchains/releases/tag/clang-18.1.8-pgo-dylib-linux) | Linux, glibc >= 2.28 | `x86_64`, `aarch64` | `llvm-pgo-dylib-18.1.8/` | ~178 MB | **current** |
| [`clang-22.1.8-pgo-rockylinux8`](https://github.com/MeshInspector/toolchains/releases/tag/clang-22.1.8-pgo-rockylinux8) | Linux, glibc >= 2.28 | `x64`, `arm64` | `llvm-pgo-22.1.8/` | ~1 GB | superseded |
| [`llvm-pgo-22.1.8_2-arm64`](https://github.com/MeshInspector/toolchains/releases/tag/llvm-pgo-22.1.8_2-arm64) | macOS arm64 | `arm64` | Homebrew keg under `/opt/homebrew` | ~438 MB | **current** |
| [`llvm-pgo-22.1.8_2-x64`](https://github.com/MeshInspector/toolchains/releases/tag/llvm-pgo-22.1.8_2-x64) | macOS Intel | `x86_64` | Homebrew keg under `/usr/local` | ~468 MB | **current** |

To list them from the command line:

    gh release list --repo MeshInspector/toolchains
    gh release view clang-22.1.8-pgo-dylib-linux --repo MeshInspector/toolchains
    gh release download clang-22.1.8-pgo-dylib-linux --repo MeshInspector/toolchains --pattern "*x86_64*"

Asset names are stable once published, but a *tag* can be renamed in place (that
is how `clang-22.1.8-pgo-dylib-rockylinux8` became `clang-22.1.8-pgo-dylib-linux`),
which 404s the old URLs. Pin the tag in exactly one place per consumer so a
rename stays a one-line fix.

## clang for Emscripten, PGO — `clang-emsdk-4.0.19-pgo`

The compiler `emscripten/emsdk:4.0.19` ships, rebuilt with PGO. Upstream builds
that toolchain with ThinLTO and assertions off but **no PGO** at all
(`emscripten-releases` `src/build.py` has no `LLVM_BUILD_INSTRUMENTED`, no
`LLVM_PROFDATA_FILE`, no BOLT), so this is `-O3` + IR-PGO + ThinLTO on top of the
same configuration.

It is the **same llvm-project revision emsdk pins**, `12f392cff` /
`clang version 22.0.0git`, read out of the emscripten-releases `DEPS` for the
4.0.19 tag. That matters: it keeps emcc's `EXPECTED_LLVM_VERSION = 22` check
satisfied and keeps the image's prebuilt wasm sysroot and resource headers
valid, so dropping it in is a compiler swap and nothing else.

Point emscripten at it with `EM_LLVM_ROOT` — emscripten honours `EM_<KEY>` for
any config key, so nothing under `/emsdk` has to be modified:

    TOOLCHAIN=clang-emsdk-4.0.19-pgo
    curl -fsSL --retry 5 --retry-all-errors "https://github.com/MeshInspector/toolchains/releases/download/${TOOLCHAIN}/${TOOLCHAIN}-$(uname -m).tar.gz" | tar -C /opt -xz
    export EM_LLVM_ROOT=/opt/llvm-emsdk-4.0.19-pgo/bin
    emcc --version

Compile time against the stock emsdk compiler, same runner, template-heavy
translation unit, `em++ -O3 -std=c++20 -c`, min of 5 with both caches warm:

| arch | stock emsdk clang | this toolchain | delta |
| --- | --- | --- | --- |
| `x86_64` | 1647 ms | 1336 ms | **-18.9%** |
| `aarch64` | 2159 ms | 1670 ms | **-22.6%** |

sha256 of the tarballs:

    65c70c0d997a9ab6a82c8ea6af5c69f46ac64bc96bd8816a3a29f445c7edc21d  clang-emsdk-4.0.19-pgo-x86_64.tar.gz
    a740be411d2333189f28783af80a35b9efcbcabcdb5ef2ebd5d0650c9d9b764a  clang-emsdk-4.0.19-pgo-aarch64.tar.gz

Two things to know before reusing this recipe for another emsdk version:

* It is built **inside the target image**, not in `rockylinux:8` like the kegs
  above, so the glibc floor is jammy's 2.35 rather than 2.28. That is deliberate:
  the training step drives the image's own `embuilder` to profile the real
  WebAssembly code paths (system libraries for wasm32 *and* wasm64), alongside a
  slice of LLVM itself for the frontend and middle-end.
* The emsdk clang cannot bootstrap it. Its compiler-rt is wasm-only, so it
  cannot link `-fprofile-generate` binaries, and the arm64 one reports its own
  host target as `unknown` because it is cross-built — a native CMake configure
  with it fails outright. The recipe therefore builds a stage1 clang with the
  distro gcc first.

Only the `bin/` tree is needed by emscripten; the tarball also carries
`lib/clang/22` from the same revision, which is why overlaying it onto
`/emsdk/upstream` works too if `EM_LLVM_ROOT` is inconvenient.

## clang 22.1.8 PGO dylib (Linux) — `clang-22.1.8-pgo-dylib-linux`

clang/lld 22.1.8 built with `-O3` + IR-PGO (no LTO), `LLVM_ENABLE_RTTI=ON`, and
distro-style shared `libLLVM.so` / `libclang-cpp.so`, which is what mrbind links
against. Built in a `rockylinux:8` container, so the glibc floor is 2.28 and one
tarball runs on every Linux image we use (RockyLinux 8/9, Ubuntu 22/24).

Assets are named after `uname -m`, so no arch mapping is needed:

    TOOLCHAIN=clang-22.1.8-pgo-dylib-linux
    curl -fsSL "https://github.com/MeshInspector/toolchains/releases/download/${TOOLCHAIN}/${TOOLCHAIN}-$(uname -m).tar.gz" | tar -C /opt -xz
    /opt/llvm-pgo-dylib-22.1.8/bin/clang++ --version

Point the build at it with `LLVM_PREFIX=/opt/llvm-pgo-dylib-22.1.8` (mrbind and
the Python bindings read it), or with
`-DCMAKE_CXX_COMPILER=/opt/llvm-pgo-dylib-22.1.8/bin/clang++`.

sha256 of the tarballs:

    97b40230ff3c65e567b532b3a7cff009b0f3d397b3b5311816743a50f5641727  clang-22.1.8-pgo-dylib-linux-x86_64.tar.gz
    41e405b76822e19b7d6cf00db648d290ad78b274fd03a970b91a0bbbb76af5d7  clang-22.1.8-pgo-dylib-linux-aarch64.tar.gz

Each tarball has a `.sha256` sidecar asset that names the tarball itself, so
fetching both side by side and running `sha256sum -c` just works:

    TOOLCHAIN=clang-22.1.8-pgo-dylib-linux
    ASSET=${TOOLCHAIN}-$(uname -m).tar.gz
    BASE=https://github.com/MeshInspector/toolchains/releases/download/${TOOLCHAIN}
    curl -fsSL -O "${BASE}/${ASSET}" -O "${BASE}/${ASSET}.sha256"
    sha256sum -c "${ASSET}.sha256"
    tar -C /opt -xzf "${ASSET}"

One host requirement: the keg picks its default libstdc++ from `/opt/rh`, and
clang 22 only scans that tree up to `gcc-toolset-13`. Install
`gcc-toolset-13-gcc-c++` — the `-gcc-c++` package, not just `libstdc++-devel`,
because detection keys on `crtbegin.o` — or the C++ standard library is not found.

## clang 18.1.8 PGO dylib (Linux) — `clang-18.1.8-pgo-dylib-linux`

The same recipe, container and flags as the 22.1.8 dylib keg above, built from
llvmorg-18.1.8 (the last 18.x release) for consumers that need clang 18 rather
than 22. Install it the same way — only the version differs:

    TOOLCHAIN=clang-18.1.8-pgo-dylib-linux
    ASSET=${TOOLCHAIN}-$(uname -m).tar.gz
    BASE=https://github.com/MeshInspector/toolchains/releases/download/${TOOLCHAIN}
    curl -fsSL -O "${BASE}/${ASSET}" -O "${BASE}/${ASSET}.sha256"
    sha256sum -c "${ASSET}.sha256"
    tar -C /opt -xzf "${ASSET}"
    /opt/llvm-pgo-dylib-18.1.8/bin/clang++ --version

    eb544c6ebe6db2a5ebc3cb288c021bbbc6242ae324245d74e221d94d539bbcd4  clang-18.1.8-pgo-dylib-linux-x86_64.tar.gz
    684e54fe27645450dd9b63f9cdf7b9c30d09cb69a72e07d5aaa9f9b1f94fc6fc  clang-18.1.8-pgo-dylib-linux-aarch64.tar.gz

This major installs the dylibs as `libLLVM.so.18.1` and `libclang-cpp.so.18.1`,
with `libLLVM-18.so` and the unsuffixed `libLLVM.so` as symlinks beside them.

## clang 22.1.8 PGO static + ThinLTO (Linux) — `clang-22.1.8-pgo-rockylinux8`

Same sources and container, but a static-LLVM + ThinLTO build: ~4-7% faster
compiles than the dylib flavor, at ~1 GB per tarball (+0.8 GiB on a Docker image)
and only usable by mrbind through its static-LLVM machinery. Unused since the
dylib switch; kept for reference and A/B measurements.

    ARCH=x64   # or arm64 -- this release predates the uname-style asset names
    TOOLCHAIN=clang-22.1.8-pgo-rockylinux8
    ASSET=${TOOLCHAIN}-${ARCH}.tar.gz
    BASE=https://github.com/MeshInspector/toolchains/releases/download/${TOOLCHAIN}
    curl -fsSL -O "${BASE}/${ASSET}" -O "${BASE}/${ASSET}.sha256"
    sha256sum -c "${ASSET}.sha256"
    tar -C /opt -xzf "${ASSET}"
    /opt/llvm-pgo-22.1.8/bin/clang++ --version

    3deac7c02e46b7d048e80c78d097622eab7927671d99e4307d1c3a7a9f48a7d2  clang-22.1.8-pgo-rockylinux8-x64.tar.gz
    7867aa7d6603234d336caaf13bb629e3672cfe9aae7117ab43fe47f29c4af2b6  clang-22.1.8-pgo-rockylinux8-arm64.tar.gz

## llvm-pgo 22.1.8_2 Homebrew kegs (macOS)

PGO + ThinLTO LLVM/Clang 22.1.8, built from a local-tap copy of homebrew-core's
`llvm.rb` (llvmorg-22.1.8) with `ENV.O3` and `brew install --build-bottle`, which
is what enables the formula's stage1 -> instrumented stage2 -> train on the
clang/LLVM test suites -> final build pipeline. The formula gates PGO to arm, so
the x64 build removes that gate. z3 excluded; the x64 superenv baseline is
`-march=nehalem`. Roughly 25-30% faster at compiling C++ than a plain build of the
same version.

These are **Homebrew kegs relocated only to the standard prefix** they were built
for: `/opt/homebrew` on arm64, `/usr/local` on Intel. A machine with a custom brew
prefix has to build its own.

arm64 (`/opt/homebrew`):

    curl -fsSL -o /tmp/k.tar.gz https://github.com/MeshInspector/toolchains/releases/download/llvm-pgo-22.1.8_2-arm64/llvm-pgo-22.1.8_2.arm64_opt-homebrew.tar.gz
    echo "94f76fb55ec64490605d0e68115259426a428a78f269f7e9160726d90a22e197  /tmp/k.tar.gz" | shasum -a 256 -c -
    tar -xzf /tmp/k.tar.gz -C /opt/homebrew/Cellar
    ln -sfn ../Cellar/llvm-pgo/22.1.8_2 /opt/homebrew/opt/llvm-pgo
    brew install --quiet zstd xz
    /opt/homebrew/Cellar/llvm-pgo/22.1.8_2/bin/clang++ --version

Intel (`/usr/local`):

    curl -fsSL -o /tmp/k.tar.gz https://github.com/MeshInspector/toolchains/releases/download/llvm-pgo-22.1.8_2-x64/llvm-pgo-22.1.8_2.x86_64_usr-local.tar.gz
    echo "8f1f1d97e4a5391c7f871bc5cf9601a9b4a1262e6e61f1538327bb48d7ef71ed  /tmp/k.tar.gz" | shasum -a 256 -c -
    tar -xzf /tmp/k.tar.gz -C /usr/local/Cellar
    ln -sfn ../Cellar/llvm-pgo/22.1.8_2 /usr/local/opt/llvm-pgo
    brew install --quiet zstd xz
    /usr/local/Cellar/llvm-pgo/22.1.8_2/bin/clang++ --version

`zstd` and `xz` are the runtime deps that actually matter; `python@3.14` is only
needed for lldb. The full set of external brew deps the keg links against is the
`deps.txt` asset of each release. Each release also has a `sha256.txt` naming its
own tarball, so `shasum -a 256 -c sha256.txt` works if you download both into the
same directory instead of pasting the digest.

In MeshLib CI this is wrapped as a composite action,
`.github/actions/install-llvm-pgo-keg`, which no-ops when the keg is already in
the local Cellar (self-hosted macOS runners build it themselves) and otherwise
runs exactly the commands above.

## Who consumes these

- `docker/rockylinux8-vcpkgDockerfile` in both MeshLib and MeshInspectorCode
  fetches the Linux dylib keg into `/opt`. The version is pinned by the single
  `TOOLCHAIN=` line there, plus one `LLVM_PREFIX` env per workflow.
- MeshLib `.github/workflows/build-test-macos.yml` and `pip-build.yml` install the
  macOS kegs on the GitHub-hosted runners through the composite action and use
  them for mrbind and the Python bindings.

## Rebuilding

The Linux recipe is a two-job chain per arch, in a `rockylinux:8` container:
stage1 `gcc-toolset-11` -> IR-instrumented stage2 trained on compiling
LLVMSupport/Core/Analysis -> final build with the profile. It lives on a branch
per version, and pushing to that branch is what triggers the build:

- [`clang-18-linux-pgo`](https://github.com/MeshInspector/toolchains/tree/clang-18-linux-pgo)
  built 18.1.8. `LLVM_VER` is the only knob — it drives the source tarball, the
  install prefix, the release tag and the asset names — so the next version is a
  one-line change on a new branch.
- [`clang-linux-pgo`](https://github.com/MeshInspector/toolchains/tree/clang-linux-pgo)
  built 22.1.8, and its history also holds the static+ThinLTO config.

Budget roughly 3h of hosted-runner wall clock for a dylib build (18.1.8 took
2h56m for all four jobs); the static+ThinLTO flavor took closer to 8h.

The macOS kegs are built on the self-hosted macOS runners by
`brew install --build-bottle` from a local tap, then packed straight out of the
Cellar. The [`Validate llvm-pgo keg`](.github/workflows/validate-llvm-pgo.yml)
workflow on `main` re-downloads the published macOS kegs on four hosted images and
benchmarks them against the `llvm@22` bottle.
