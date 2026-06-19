# int2dds_ffi_vendor

ROS 2 vendor package that fetches the **prebuilt int2DDS FFI library** for the
host platform at build time and exposes it to colcon/ament as an imported CMake
target. It is the dependency boundary between the Rust `int2DDS` core (shipped as
prebuilt binaries) and the C++ `rmw_int2dds_cpp` middleware.

## What it does

1. Detects the host OS, architecture, and libc (gnu/musl).
2. Downloads the per-OS release asset from the `int2DDS` GitHub Releases page.
3. Reads the bundled manifest, selects the artifact matching the host, and
   verifies its sha256.
4. Installs `int2dds-ffi.h` + the platform shared library as `include/` + `lib/`.
5. Exports the imported target `int2dds_ffi::int2dds_ffi`.

## Consuming it (downstream)

```cmake
find_package(int2dds_ffi_vendor REQUIRED)
target_link_libraries(my_target int2dds_ffi::int2dds_ffi)
```

```xml
<!-- package.xml -->
<depend>int2dds_ffi_vendor</depend>
```

## Release asset layout

One tarball per OS is published on the `int2DDS` release matching
`INT2DDS_FFI_VERSION` in [CMakeLists.txt](CMakeLists.txt). Each tarball bundles
every architecture for that OS plus a manifest:

```
int2dds-ffi-<version>-linux.tar.gz
├── int2dds-ffi.h                          # C API header
├── int2dds-ffi.manifest.yaml              # per-arch file + sha256 + min_glibc
├── LICENSE
├── linux-x86_64/libint2dds_ffi.so         # amd64, gnu
├── linux-x86_64-musl/libint2dds_ffi.so    # amd64, musl
├── linux-aarch64/libint2dds_ffi.so        # arm64, gnu
├── linux-aarch64-musl/libint2dds_ffi.so   # arm64, musl
└── linux-armhf/libint2dds_ffi.so          # armv7, gnu
```

`int2dds-ffi.manifest.yaml`:

```yaml
name: int2dds-ffi
version: 0.0.1
artifacts:
  - os: linux
    arch: amd64
    triple: x86_64-unknown-linux-gnu
    file: linux-x86_64/libint2dds_ffi.so
    sha256: <hex>
    min_glibc: "2.34"
  ...
```

Because the manifest already carries per-artifact `sha256`, the vendor package
verifies integrity automatically — no SHA values need to be hard-coded here.

### libc selection

The default is glibc (`gnu`), which is what ROS binaries target (Ubuntu 22.04
Jammy ships glibc 2.35 ≥ the artifacts' `min_glibc: 2.34`). For a musl host:

```bash
colcon build --cmake-args -DINT2DDS_FFI_LIBC=musl
```

`armhf` ships only a gnu build.
