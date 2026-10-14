# LoongArch64 ABI2 development

This branch targets the mainline `loongarch64-unknown-linux-gnu` Rust target and
the LoongArch ABI2 userspace. It does not target the legacy ABI1 userspace.
Rust's target requires a compatible Linux 5.19 kernel interface, glibc 2.36 or
newer, and an LSX-capable CPU. Check the target distribution and run
`rpm --eval '%{_arch}'` before packaging; the Rust target triple, Node's
`loong64` name, the Mihomo asset name, and the RPM architecture name are
different strings.

## Changes in this branch

- Prebuild downloads the `mihomo-linux-loong64-abi2` stable and Alpha assets.
  Runtime core upgrades use the same asset variant.
- `CLASH_VERGE_SERVICE_IPC_SOURCE` lets prebuild compile the three service
  executables from a local `clash-verge-service-ipc` checkout. This avoids the
  missing LoongArch64 asset in the service's v2.7.6 release without changing the
  application's pinned service dependency. The existing Cargo dependency path
  and published-archive paths still work when this variable is unset.

## Build the application binary

Use Rust 1.99.0 and the repository's pnpm 12.10.0. The target needs a working
LoongArch64 C/C++ linker, `pkg-config`, and development libraries for WebKitGTK
4.1, GTK 3, Ayatana AppIndicator, librsvg, and OpenSSL. Cross builds also need
a target sysroot and target-aware `CC`, `CXX`,
`CARGO_TARGET_LOONGARCH64_UNKNOWN_LINUX_GNU_LINKER`, and `PKG_CONFIG_*`
settings. The distro's package names may differ from the existing Linux CI.

The service source must match the application's pinned v2.7.6 dependency:

```sh
git clone --branch v2.7.6 --depth 1 \
  https://github.com/clash-verge-rev/clash-verge-service-ipc.git \
  ../clash-verge-service-ipc
export CLASH_VERGE_SERVICE_IPC_SOURCE="$(realpath ../clash-verge-service-ipc)"
rustup target add loongarch64-unknown-linux-gnu --toolchain 1.99.0
pnpm install --frozen-lockfile
pnpm run prebuild loongarch64-unknown-linux-gnu
```

Prebuild compiles `clash-verge-service`, `clash-verge-service-install`, and
`clash-verge-service-uninstall` with `standalone,client` and places their
target-suffixed copies alongside the two Mihomo sidecars in `src-tauri/sidecar`.
It also downloads architecture-independent data into `src-tauri/resources`.

The frontend must exist in `dist/` before a direct Cargo build. On a supported
x86_64 Linux build host, `pnpm run web:build` produces it. A native LoongArch64
frontend build needs a compatible Node.js and a Rolldown WASM binding or a
locally built Rolldown binding; the lockfile does not currently contain a
LoongArch64 native Rolldown binding. Once `dist/` is available, build without
the Tauri CLI:

```sh
cargo build --package clash-verge --release \
  --target loongarch64-unknown-linux-gnu
```

On an x86_64 Linux host with the target sysroot configured, the existing Tauri
CLI can instead run:

```sh
pnpm build --target loongarch64-unknown-linux-gnu --no-bundle
```

The main executable is expected at
`target/loongarch64-unknown-linux-gnu/release/clash-verge`. The application also
needs the five sidecars and its resources at the paths expected by Tauri and
`src-tauri/src/core/service.rs`; the main ELF alone is not a complete install.

## Work remaining on LoongArch64

- Compile and run on an ABI2 machine. Check the Rust dependency graph and the
  service's native dependencies against the target toolchain, then verify the
  GUI, Mihomo start/upgrade, service install/uninstall, and resource lookup.
- For a native frontend build, provide Node.js and the matching Rolldown WASM
  binding (or native bindings), then refresh the pnpm lockfile with the pinned
  pnpm version. The Oxc parser/resolver bindings in the lockfile also need
  validation on LoongArch64. The Tauri npm CLI has no LoongArch64 native
  package in the current lockfile; direct Cargo builds avoid that CLI.
- The stock Tauri bundler does not recognize `loongarch64` for RPM packaging.
  Patch its architecture detection and RPM mapping, or add an RPM spec that
  packages the main binary, five sidecars, resources, desktop entry, icons,
  runtime dependencies, and the scripts/configuration in
  `src-tauri/tauri.linux.conf.json`. The RPM architecture name supported by
  upstream RPM is `loongarch64`; verify the target distribution's value. A
  native source build of the Tauri CLI also needs its `tauri-utils` target
  triple detection to recognize LoongArch64.
- Tauri's updater plugin v2.13.1 returns `UnsupportedArch` when checking for
  app updates on LoongArch64. It does not block startup. Supporting app updates
  requires a plugin architecture mapping, LoongArch64 release assets, and
  corresponding keys in `scripts/updater.mjs`. The current CI release jobs also
  need a LoongArch64 toolchain/sysroot or a native runner before they can publish
  this target; their Ubuntu 22.04 ARM sysroots cannot be reused as-is.

The changes here have not been compiled or run on LoongArch64. The source
asset names and service build command were checked against their upstream
release layouts; target-specific compilation and runtime behavior remain to be
verified on the machine that will ship the package.
