# Build recipes

The exact recipe behind each published release, copied here so they can be read
without digging through branches. Nothing in this directory runs: GitHub only
picks up workflows under `.github/workflows/`. To rebuild, copy a file back
there on a new branch (see [Rebuilding](../README.md#rebuilding)).

| Release | Recipe | Source |
| --- | --- | --- |
| `clang-22.1.8-pgo-dylib-linux` | [`clang-22.1.8-pgo-dylib-linux/`](clang-22.1.8-pgo-dylib-linux/build-clang-linux-pgo.yml) | `clang-linux-pgo` at `e10991e` |
| `clang-22.1.8-pgo-rockylinux8` | [`clang-22.1.8-pgo-rockylinux8/`](clang-22.1.8-pgo-rockylinux8/build-clang-linux-pgo.yml) | `clang-linux-pgo` at `369b3b6` |
| `clang-18.1.8-pgo-dylib-linux` | [`clang-18.1.8-pgo-dylib-linux/`](clang-18.1.8-pgo-dylib-linux/build-clang-linux-pgo.yml) | `clang-18-linux-pgo` at `f9a0d21` |
| `clang-emsdk-4.0.19-pgo` | [`clang-emsdk-4.0.19-pgo/`](clang-emsdk-4.0.19-pgo/build-clang-emsdk-pgo.yml) | `clang-emsdk-4.0.19-pgo` at `13acd69` |
| `clang-emsdk-5.0.7-pgo` | [`clang-emsdk-5.0.7-pgo/`](clang-emsdk-5.0.7-pgo/build-clang-emsdk-pgo.yml) | `clang-emsdk-5.0.7-pgo` at `dbf451e` |
| `clang-emsdk-6.0.9-pgo` | [`clang-emsdk-6.0.9-pgo/`](clang-emsdk-6.0.9-pgo/build-clang-emsdk-pgo.yml) | `clang-emsdk-6.0.9-pgo` at `73e81a4` |
| `llvm-pgo-22.1.8_2-arm64`, `llvm-pgo-22.1.8_2-x64` | [`llvm-pgo-22.1.8_2-macos/`](llvm-pgo-22.1.8_2-macos/) | throwaway MeshInspectorCode branches, since deleted |
| `lld-22.1.8-pgo-macos` | [`build-lld-macos.yml`](../.github/workflows/build-lld-macos.yml), [`install-lld-macos.yml`](../.github/workflows/install-lld-macos.yml) | already on `main` |

`clang-linux-pgo` has since moved on (its head also PGO-trains lld), so the
22.1.8 rows are pinned to the commits whose runs uploaded the assets.

## macOS Homebrew kegs

These were built on the self-hosted runners by throwaway workflows in
MeshInspectorCode, reproduced verbatim:

- `build-arm64.yml` (daniil, `macos-13` label) and `build-x86_64.yml`
  (MACPRO2013, `macos-12-intel`) write a local tap `local/pgo` whose
  `llvm-pgo.rb` is homebrew-core's `llvm.rb` with the class renamed (own Cellar
  rack, nothing else touched) and `ENV.O3` as the first line of `install`
  (defeats superenv's `-Os`), then `brew install --build-bottle`, which is what
  turns on the formula's PGO pipeline. The x86_64 build also drops
  `Hardware::CPU.arm? && ` from the `pgo_build` gate. Both end with a compile
  benchmark against the other kegs on the machine.
- `package-arm64.yml` and `package-x86_64.yml` tar the keg straight out of the
  Cellar and write `deps.txt` (the brew libraries its Mach-O files link) and
  `sha256.txt`; the artifact was then uploaded to the release by hand.

The build workflows fetch `llvm.rb` from homebrew-core `master`, which no longer
is 22.1.8. [`llvm-pgo.rb`](llvm-pgo-22.1.8_2-macos/llvm-pgo.rb) is the exact
formula Homebrew embedded in the published x86_64 keg (`.brew/llvm-pgo.rb`);
the arm64 keg's copy differs only by still having the arm-only gate, which is a
no-op there, so this one file reproduces both. To rebuild 22.1.8_2, put it
into the tap instead of the `curl` + `sed`.

Wall clock: about 2h on the M-series runner, 9h35m on the 2013 Xeon.
