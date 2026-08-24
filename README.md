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

The `.sha256` sidecar assets still spell the pre-rename temp paths inside, so
`sha256sum -c` fed one of them directly fails on a missing file. Compare the
digest against the name you actually downloaded instead:

    echo "97b40230ff3c65e567b532b3a7cff009b0f3d397b3b5311816743a50f5641727  clang.tar.gz" | sha256sum -c -

One host requirement: the keg picks its default libstdc++ from `/opt/rh`, and
clang 22 only scans that tree up to `gcc-toolset-13`. Install
`gcc-toolset-13-gcc-c++` — the `-gcc-c++` package, not just `libstdc++-devel`,
because detection keys on `crtbegin.o` — or the C++ standard library is not found.

## clang 22.1.8 PGO static + ThinLTO (Linux) — `clang-22.1.8-pgo-rockylinux8`

Same sources and container, but a static-LLVM + ThinLTO build: ~4-7% faster
compiles than the dylib flavor, at ~1 GB per tarball (+0.8 GiB on a Docker image)
and only usable by mrbind through its static-LLVM machinery. Unused since the
dylib switch; kept for reference and A/B measurements.

    ARCH=x64   # or arm64 -- this release predates the uname-style asset names
    TOOLCHAIN=clang-22.1.8-pgo-rockylinux8
    curl -fsSL -o /tmp/clang.tar.gz "https://github.com/MeshInspector/toolchains/releases/download/${TOOLCHAIN}/${TOOLCHAIN}-${ARCH}.tar.gz"
    tar -C /opt -xzf /tmp/clang.tar.gz
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
`deps.txt` asset of each release, and `sha256.txt` carries the digest above.

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

The Linux recipe lives on the [`clang-linux-pgo`](https://github.com/MeshInspector/toolchains/tree/clang-linux-pgo)
branch: a two-job chain per arch, in a `rockylinux:8` container, stage1
`gcc-toolset-11` -> IR-instrumented stage2 trained on compiling
LLVMSupport/Core/Analysis -> final build with the profile.

The macOS kegs are built on the self-hosted macOS runners by
`brew install --build-bottle` from a local tap, then packed straight out of the
Cellar. The [`Validate llvm-pgo keg`](.github/workflows/validate-llvm-pgo.yml)
workflow on `main` re-downloads the published macOS kegs on four hosted images and
benchmarks them against the `llvm@22` bottle.
