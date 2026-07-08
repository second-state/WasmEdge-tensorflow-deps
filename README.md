# WasmEdge-Tensorflow Dependencies

This repository provides pre-built dependencies for the [WasmEdge](https://github.com/WasmEdge/WasmEdge) with TensorFlow plug-in.

## Motivation

The TensorFlow project no longer publishes the standalone `libtensorflow` C/C++ shared library packages — the official pre-built libraries only ship inside the Python pip wheels. Projects which link TensorFlow directly, such as the WasmEdge plug-ins, still need standalone shared libraries that run on the manylinux_2_28 (glibc >= 2.28) platforms.

But building TensorFlow takes lots of time. To reduce the compilation time, we create this project to compile and release the pre-built standalone TensorFlow and TensorFlow-Lite shared libraries for the manylinux_2_28, macOS, and Android platforms.

The headers vendored in this repository (`tensorflow/`, `tsl/`, `xla/` at the repo root) are the include closure of the C API headers used by WasmEdge, extracted from the v2.21.0 source tree.

## License

This project is under the Apache-2.0 License as the same as the TensorFlow project.

### Credits

- [TensorFlow](https://github.com/tensorflow/tensorflow)

## How to build the TensorFlow 2.21.0 dependencies

All libraries are built from the **pristine `v2.21.0` tag — no source patch, no `./configure`, no local Python** (hermetic Python via `--repo_env=HERMETIC_PYTHON_VERSION=3.12`).

The TensorFlow asset is the **two-library pywrap pair** (`libtensorflow_cc` + `libtensorflow_framework`, SONAME-form filenames) — the same libraries the official TensorFlow pip wheels ship, built by the same pipeline. Consumers must link **both**, `libtensorflow_cc` first (some C API symbols are defined in `libtensorflow_framework`).

Source, for every platform:

```bash
git clone https://github.com/tensorflow/tensorflow.git tensorflow_src
cd tensorflow_src && git checkout v2.21.0
```

### Build on macOS

1. Environment settings

   - Full Xcode (the Command Line Tools alone are not enough), selected as the developer directory (`xcode-select -p`); ~50 GB free disk.
   - `brew install bazelisk` — it fetches bazel 7.7.0 from `tensorflow/.bazelversion`; always build with `bazelisk`, never a standalone `bazel`.

2. Build steps on darwin_arm64 (macOS 12+)

   ```bash
   # TensorFlow pair:
   bazelisk build --config=release_macos_arm64 \
     --repo_env=HERMETIC_PYTHON_VERSION=3.12 \
     //tensorflow/python:libtensorflow_cc.2.dylib \
     //tensorflow/python:libtensorflow_framework.2.dylib
   # -> bazel-bin/tensorflow/python/libtensorflow_cc.2.dylib, libtensorflow_framework.2.dylib

   # TensorFlow-Lite C library + flex delegate:
   bazelisk build --config=release_macos_arm64 --config=monolithic \
     --repo_env=HERMETIC_PYTHON_VERSION=3.12 \
     //tensorflow/lite/c:libtensorflowlite_c.dylib \
     //tensorflow/lite/delegates/flex:tensorflowlite_flex
   # -> bazel-bin/tensorflow/lite/c/libtensorflowlite_c.dylib,
   #    bazel-bin/tensorflow/lite/delegates/flex/libtensorflowlite_flex.dylib
   ```

3. Build steps on darwin_x86_64 (macOS 10.15+)

   The flag set below is `release_macos_x86` without its **host** AVX flag, plus three additions: the oneDNN contraction-kernel opt-out and a **target-only** AVX2/FMA re-enable that forces RUY's x86 SIMD kernels on. Two things go wrong otherwise at v2.21.0:

   - The stock `--config=release_macos_x86` **fails to build**: its `--host_copt=-mavx` is applied to the arm64 slice of gRPC's `universal_binary` codegen tools (built even on Intel), where `-mavx` is illegal; and macOS x86 has no real oneDNN, so the `_with_mkl` contraction kernel can't find `dnnl.h`.
   - Even once it builds, RUY (the TF-Lite / TF matmul kernels) **disables all of its x86 SIMD paths on Apple** — `ruy/platform.h` gates `RUY_PLATFORM_X86_ENHANCEMENTS` on `__linux__` clang only. With no AVX2 kernel, RUY falls back to its incomplete SSE4.2 8-bit kernel stub and **segfaults at runtime on quantized (uint8) inference**. `--copt=-DRUY_FORCE_ENABLE_X86_ENHANCEMENTS --copt=-mavx2 --copt=-mfma --copt=-mf16c` restores RUY's real `Kernel8bitAvx2`. These are target-only `--copt`s, so they do not reach the gRPC arm64 slice and do not reintroduce the build failure above.

   This makes the darwin_x86_64 libraries AVX2-baseline — safe on every Intel Mac (all Haswell+).

   ```bash
   # TensorFlow pair:
   bazelisk build --config=release_macos_base --cpu=darwin \
     --macos_minimum_os=10.15 --action_env=MACOSX_DEPLOYMENT_TARGET=10.15 \
     --define tensorflow_mkldnn_contraction_kernel=0 \
     --copt=-DRUY_FORCE_ENABLE_X86_ENHANCEMENTS --copt=-mavx2 --copt=-mfma --copt=-mf16c \
     --repo_env=HERMETIC_PYTHON_VERSION=3.12 \
     //tensorflow/python:libtensorflow_cc.2.dylib \
     //tensorflow/python:libtensorflow_framework.2.dylib
   # -> bazel-bin/tensorflow/python/libtensorflow_cc.2.dylib, libtensorflow_framework.2.dylib

   # TensorFlow-Lite C library + flex delegate:
   bazelisk build --config=release_macos_base --cpu=darwin \
     --macos_minimum_os=10.15 --action_env=MACOSX_DEPLOYMENT_TARGET=10.15 \
     --define tensorflow_mkldnn_contraction_kernel=0 --config=monolithic \
     --copt=-DRUY_FORCE_ENABLE_X86_ENHANCEMENTS --copt=-mavx2 --copt=-mfma --copt=-mf16c \
     --repo_env=HERMETIC_PYTHON_VERSION=3.12 \
     //tensorflow/lite/c:libtensorflowlite_c.dylib \
     //tensorflow/lite/delegates/flex:tensorflowlite_flex
   # -> bazel-bin/tensorflow/lite/c/libtensorflowlite_c.dylib,
   #    bazel-bin/tensorflow/lite/delegates/flex/libtensorflowlite_flex.dylib
   ```

### Build on Linux (manylinux_2_28_x86_64 / manylinux_2_28_aarch64)

Measured floors: GLIBC 2.27, GLIBCXX 3.4.22, CXXABI 1.3.11 — the libraries run on any manylinux_2_28 (glibc ≥ 2.28) system. x86_64 baseline is AVX; aarch64 is armv8-a.

1. Environment settings

   Docker is required; the build runs in a **manylinux_2_34** container (the hermetic clang prebuilts need build-host glibc ≥ 2.29, so manylinux_2_28 images cannot host the build — the artifacts still keep the GLIBC 2.27 floor); ~50 GB free disk. Start the container from the `tensorflow_src` checkout, then install xxd and bazelisk:

   ```bash
   docker run -it --rm -v "$PWD":/root/tensorflow_src -w /root/tensorflow_src \
     quay.io/pypa/manylinux_2_34_x86_64 bash
   # aarch64: quay.io/pypa/manylinux_2_34_aarch64

   # In the container:
   dnf -q -y install vim-common   # provides xxd
   curl -sLo /usr/local/bin/bazel \
     https://github.com/bazelbuild/bazelisk/releases/download/v1.26.0/bazelisk-linux-amd64 \
     && chmod +x /usr/local/bin/bazel
   # aarch64: .../bazelisk-linux-arm64
   ```

2. Build steps

   ```bash
   # x86_64 shown; aarch64: --config=release_arm64_linux
   bazel build --config=release_cpu_linux \
     --repo_env=HERMETIC_PYTHON_VERSION=3.12 \
     //tensorflow/python:libtensorflow_cc.so.2 \
     //tensorflow/python:libtensorflow_framework.so.2
   # -> bazel-bin/tensorflow/python/libtensorflow_cc.so.2, libtensorflow_framework.so.2

   # TensorFlow-Lite C library + flex delegate:
   bazel build --config=release_cpu_linux --config=monolithic \
     --repo_env=HERMETIC_PYTHON_VERSION=3.12 \
     //tensorflow/lite/c:libtensorflowlite_c.so \
     //tensorflow/lite/delegates/flex:tensorflowlite_flex
   # -> bazel-bin/tensorflow/lite/c/libtensorflowlite_c.so,
   #    bazel-bin/tensorflow/lite/delegates/flex/libtensorflowlite_flex.so
   ```

### Build on Android (android_aarch64, TensorFlow-Lite only)

The Android asset contains only `libtensorflowlite_c.so` (arm64-v8a, minSdkVersion 23).

1. Environment settings

   - A macOS or Linux x86_64 host with bazelisk (as above).
   - The Android **NDK r25c** from `https://dl.google.com/android/repository/android-ndk-r25c-<darwin|linux>.zip` — no Android SDK, no Java.

2. Build steps

   ```bash
   bazelisk build \
     --config=android_arm64 \
     --repo_env=ANDROID_NDK_HOME=/path/to/android-ndk-r25c \
     --repo_env=ANDROID_NDK_API_LEVEL=23 \
     --repo_env=ANDROID_NDK_VERSION=25 \
     --repo_env=HERMETIC_PYTHON_VERSION=3.12 \
     //tensorflow/lite/c:tensorflowlite_c
   # -> bazel-bin/tensorflow/lite/c/libtensorflowlite_c.so
   ```

On every platform, copy the libraries out of `bazel-bin` with `cp -L` (dereferences bazel's symlinks) and `chmod 755`.

## Legacy build guides

- [LEGACY_BUILD_2.6.md](LEGACY_BUILD_2.6.md) — TensorFlow 2.6.0 (CentOS 7.6.1810, MacOS, Android)
- [LEGACY_BUILD_2.12.md](LEGACY_BUILD_2.12.md) — TensorFlow 2.12.0 (CentOS 7.6.1810, MacOS, Android)

## Minimum requirements of our pre-built shared libraries

| Pre-built shared library   | Platform              | TensorFlow Tag | GLIBC | GLIBCXX | CXXABI |
| -------------------------- | --------------------- | -------------- | ----- | ------- | ------ |
| libtensorflow.so           | manylinux2014_x86_64  |     2.6.0      | 2.17  | 3.4.19  | 1.3.7  |
| libtensorflow_cc.so        | manylinux2014_x86_64  |     2.6.0      | 2.17  | 3.4.19  | 1.3.7  |
| libtensorflow_framework.so | manylinux2014_x86_64  |     2.6.0      | 2.16  | 3.4.19  | 1.3.7  |
| libtensorflowlite_c.so     | manylinux2014_x86_64  |     2.6.0      | 2.14  | 3.4.19  | 1.3.5  |
| libtensorflow_cc.so        | manylinux2014_x86_64  |     2.12.0     | 2.17  | 3.4.19  | 1.3.7  |
| libtensorflow_framework.so | manylinux2014_x86_64  |     2.12.0     | 2.16  | 3.4.19  | 1.3.7  |
| libtensorflowlite_c.so     | manylinux2014_x86_64  |     2.12.0     | 2.14  | 3.4.19  | 1.3.5  |
| libtensorflowlite_flex.so  | manylinux2014_x86_64  |     2.12.0     | 2.17  | 3.4.19  | 1.3.7  |
| libtensorflowlite_c.so     | manylinux2014_aarch64 |     2.6.0      | 2.17  | None    | None   |
| libtensorflow_cc.so        | manylinux2014_aarch64 |     2.12.0     | 2.17  | 3.4.19  | 1.3.7  |
| libtensorflow_framework.so | manylinux2014_aarch64 |     2.12.0     | 2.16  | 3.4.19  | 1.3.7  |
| libtensorflowlite_c.so     | manylinux2014_aarch64 |     2.12.0     | 2.17  | None    | None   |
| libtensorflowlite_flex.so  | manylinux2014_aarch64 |     2.12.0     | 2.17  | 3.4.19  | 1.3.7  |
| libtensorflowlite_c.so     | android_aarch64       |     2.6.0      | None  | None    | None   |
| libtensorflowlite_c.so     | android_aarch64       |     2.12.0     | None  | None    | None   |
| libtensorflow_cc.so.2        | manylinux_2_28_x86_64  |     2.21.0     | 2.27  | 3.4.22  | 1.3.11 |
| libtensorflow_framework.so.2 | manylinux_2_28_x86_64  |     2.21.0     | 2.27  | 3.4.22  | 1.3.11 |
| libtensorflowlite_c.so       | manylinux_2_28_x86_64  |     2.21.0     | 2.27  | 3.4.22  | 1.3.5  |
| libtensorflowlite_flex.so    | manylinux_2_28_x86_64  |     2.21.0     | 2.27  | 3.4.22  | 1.3.11 |
| libtensorflow_cc.so.2        | manylinux_2_28_aarch64 |     2.21.0     | 2.27  | 3.4.22  | 1.3.11 |
| libtensorflow_framework.so.2 | manylinux_2_28_aarch64 |     2.21.0     | 2.27  | 3.4.22  | 1.3.11 |
| libtensorflowlite_c.so       | manylinux_2_28_aarch64 |     2.21.0     | 2.27  | 3.4.22  | 1.3.5  |
| libtensorflowlite_flex.so    | manylinux_2_28_aarch64 |     2.21.0     | 2.27  | 3.4.22  | 1.3.11 |
| libtensorflowlite_c.so       | android_aarch64        |     2.21.0     | None  | None    | None   |

| Pre-built shared library      | Platform      | TensorFlow Tag | Minimum MacOS version |
| ----------------------------- | ------------- | -------------- | --------------------- |
| libtensorflow.dylib           | darwin_x86_64 |     2.6.0      |          10.15        |
| libtensorflow_cc.dylib        | darwin_x86_64 |     2.6.0      |          10.15        |
| libtensorflow_framework.dylib | darwin_x86_64 |     2.6.0      |          10.15        |
| libtensorflowlite_c.dylib     | darwin_x86_64 |     2.6.0      |          10.15        |
| libtensorflow_cc.dylib        | darwin_x86_64 |     2.12.0     |          10.15        |
| libtensorflow_framework.dylib | darwin_x86_64 |     2.12.0     |          10.15        |
| libtensorflowlite_c.dylib     | darwin_x86_64 |     2.12.0     |          10.15        |
| libtensorflowlite_flex.dylib  | darwin_x86_64 |     2.12.0     |          10.15        |
| libtensorflow_cc.dylib        | darwin_arm64  |     2.12.0     |          12.0         |
| libtensorflow_framework.dylib | darwin_arm64  |     2.12.0     |          12.0         |
| libtensorflowlite_c.dylib     | darwin_arm64  |     2.12.0     |          12.0         |
| libtensorflowlite_flex.dylib  | darwin_arm64  |     2.12.0     |          12.0         |
| libtensorflow_cc.2.dylib        | darwin_x86_64 |     2.21.0     |          10.15        |
| libtensorflow_framework.2.dylib | darwin_x86_64 |     2.21.0     |          10.15        |
| libtensorflowlite_c.dylib       | darwin_x86_64 |     2.21.0     |          10.15        |
| libtensorflowlite_flex.dylib    | darwin_x86_64 |     2.21.0     |          10.15        |
| libtensorflow_cc.2.dylib        | darwin_arm64  |     2.21.0     |          12.0         |
| libtensorflow_framework.2.dylib | darwin_arm64  |     2.21.0     |          12.0         |
| libtensorflowlite_c.dylib       | darwin_arm64  |     2.21.0     |          12.0         |
| libtensorflowlite_flex.dylib    | darwin_arm64  |     2.21.0     |          12.0         |

## Active Releases

- [TF-2.6.0](https://github.com/second-state/WasmEdge-tensorflow-deps/releases/tag/TF-2.6.0): TensorFlow 2.6.0 C library
  - TensorFlow 2.6.0 C shared library for `manylinux2014_x86_64` and `darwin_x86_64`.
  - TensorFlow-Lite 2.6.0 C shared library for `manylinux2014_x86_64`, `manylinux2014_aarch64`, `darwin_x86_64`, and `android_aarch64`.
- [TF-2.6.0-CC](https://github.com/second-state/WasmEdge-tensorflow-deps/releases/tag/TF-2.6.0-CC): TensorFlow 2.6.0 C++ library
  - TensorFlow 2.6.0 C++ shared library for `manylinux2014_x86_64` and `darwin_x86_64`.
  - TensorFlow-Lite 2.6.0 C shared library for `manylinux2014_x86_64`, `manylinux2014_aarch64`, `darwin_x86_64`, and `android_aarch64`.
- [TF-2.12.0-CC](https://github.com/second-state/WasmEdge-tensorflow-deps/releases/tag/TF-2.12.0-CC): TensorFlow 2.12.0 C++ library
  - TensorFlow 2.12.0 C++ shared library for `manylinux2014_x86_64`, `manylinux2014_aarch64`, `darwin_x86_64`, and `darwin_arm64`.
  - TensorFlow-Lite 2.12.0 C shared library with Flex delegate shared library for `manylinux2014_x86_64`, `manylinux2014_aarch64`, `darwin_x86_64`, and `darwin_arm64`.
  - TensorFlow-Lite 2.12.0 C shared library for `android_aarch64`.
- [TF-2.21.0-CC](https://github.com/second-state/WasmEdge-tensorflow-deps/releases/tag/TF-2.21.0-CC): TensorFlow 2.21.0 C++ library
  - TensorFlow 2.21.0 C++ shared library for `manylinux_2_28_x86_64`, `manylinux_2_28_aarch64`, `darwin_x86_64`, and `darwin_arm64`.
  - TensorFlow-Lite 2.21.0 C shared library with Flex delegate shared library for the same platforms.
  - TensorFlow-Lite 2.21.0 C shared library for `android_aarch64`.
