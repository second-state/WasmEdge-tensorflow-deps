# WasmEdge-Tensorflow Dependencies

This repository provides pre-built dependencies for the [WasmEdge with TensorFlow extension project](https://github.com/second-state/WasmEdge-tensorflow).

## Motivation

The TensorFlow official repository only provides a shared library which is compiled on Ubuntu 18.04. Hence, we cannot use this pre-built shared library on some legacy systems such as CentOS 7.6.1810. The obvious issue is that the GLIBC version is too old, so the pre-built shared library cannot be executed.

But building the dependencies on the legacy system takes lots of time. To reduce the compilation time, we create this project to compile and release the pre-built TensorFlow shared library.

## License

This project is under the Apache-2.0 License as the same as the TensorFlow project.

### Credits

- [TensorFlow](https://github.com/tensorflow/tensorflow)

## How to build on the legacy operating system - CentOS 7.6.1810 x86_64 or aarch64

### Clone the TensorFlow source

```bash
git clone https://github.com/tensorflow/tensorflow.git tensorflow_src
cd tensorflow_src
git checkout v2.12.0
```

### Pull the WasmEdge docker image base on CentOS 7.6.1810 x86_64 or aarch64 and run

```bash
docker run -it --rm -v $(pwd):/root/$(basename $(pwd)) wasmedge/wasmedge:manylinux2014_x86_64
# For aarch64:
#   docker run -it --rm -v $(pwd):/root/$(basename $(pwd)) wasmedge/wasmedge:manylinux2014_aarch64
```

### Build and install Python 3.8.6 on CentOS 7.6.1810 x86_64 or aarch64

```bash
# In docker
cd /root
yum update
yum install -y wget bzip2-devel libffi-devel
curl -sLO https://www.python.org/ftp/python/3.8.6/Python-3.8.6.tgz
tar -zxf Python-3.8.6.tgz
cd Python-3.8.6
./configure --enable-optimizations
make && make install
```

The python installation path will be at `/usr/local/bin/python3.8`,
and the version can be checked with the command `python3.8 --version`.

### Install Bazelisk 1.16.0 on CentOS 7.6.1810 x86_64 or aarch64

```bash
# In docker
cd /root
curl -sLO https://github.com/bazelbuild/bazelisk/releases/download/v1.16.0/bazelisk-linux-amd64
# For aarch64:
#   curl -sLO https://github.com/bazelbuild/bazelisk/releases/download/v1.16.0/bazelisk-linux-arm64
chmod u+x bazelisk-linux-amd64
mv bazelisk-linux-amd64 /usr/local/bin/bazel
# For aarch64:
#   chmod u+x bazelisk-linux-arm64
#   mv bazelisk-linux-arm64 /usr/local/bin/bazel
```

### Install the Python packages which are required by the TensorFlow project on CentOS 7.6.1810 x86_64 or aarch64

```bash
# In docker
pip3 install -U --user pip numpy wheel packaging requests opt_einsum
pip3 install -U --user keras_preprocessing --no-deps
```

### Build the TensorFlow and TensorFlow-lite shared library on CentOS 7.6.1810 x86_64 or aarch64

```bash
# In docker
cd /root/tensorflow_src
PYTHON_BIN_PATH=/usr/local/bin/python3.8 USE_DEFAULT_PYTHON_LIB_PATH=1 TF_NEED_CUDA=0 TF_NEED_ROCM=0 TF_DOWNLOAD_CLANG=0 TF_NEED_MPI=0 CC_OPT_FLAGS="-march=native -Wno-sign-compare" TF_SET_ANDROID_WORKSPACE=0 ./configure
BAZEL_LINKLIBS=-l%:libstdc++.a bazel build -c opt --cxxopt="-D_GLIBCXX_USE_CXX11_ABI=0" //tensorflow:libtensorflow_cc.so
BAZEL_LINKLIBS=-l%:libstdc++.a bazel build -c opt --cxxopt="-D_GLIBCXX_USE_CXX11_ABI=0" //tensorflow/lite/c:libtensorflowlite_c.so
BAZEL_LINKLIBS=-l%:libstdc++.a bazel build -c opt --cxxopt="-D_GLIBCXX_USE_CXX11_ABI=0" --config=monolithic //tensorflow/lite/delegates/flex:tensorflowlite_flex
```

The TensorFlow shared library will be at `bazel-bin/tensorflow/libtensorflow.so.2.12.0`, `bazel-bin/tensorflow/libtensorflow_framework.so.2.12.0`, `bazel-bin/tensorflow/lite/c/libtensorflowlite_c.so`, and `bazel-bin/tensorflow/delegates/flex:tensorflowlite_flex.so`.

## How to build on the MacOS systems - MacOS 10.15 x86_64 or MacOS 12 arm64

### [Clone the TensorFlow source on MacOS](#clone-the-tensorflow-source)

### [Install Homebrew](https://brew.sh/)

Install python with brew:

```bash
brew install python
```

### Install the Python packages which are required by the TensorFlow project on MacOS x86_64 or MacOS 12 arm64

```bash
pip3 install -U --user pip numpy wheel
pip3 install -U --user keras_preprocessing --no-deps
```

### Install Bazelisk 1.16.0 on MacOS 10.15 x86_64 or MacOS 12 arm64

```bash
curl -sLO https://github.com/bazelbuild/bazelisk/releases/download/v1.16.0/bazelisk-darwin-amd64
# For Apple M series:
#   curl -sLO https://github.com/bazelbuild/bazelisk/releases/download/v1.16.0/bazelisk-darwin-arm64
chmod u+x bazelisk-darwin-amd64
mv bazelisk-darwin-amd64 /usr/local/bin/bazel
# For Apple M series:
#   chmod u+x bazelisk-darwin-arm64
#   mv bazelisk-darwin-arm64 /usr/local/bin/bazel
```

### Build the TensorFlow and TensorFlow-lite shared library on MacOS 10.15 x86_64 or MacOS 12 arm64

```bash
# In the tensorflow source directory
export CC=clang
export CXX=clang++
# Beware to use Apple Clang to build TensorFlow.
PYTHON_BIN_PATH=/usr/local/opt/python@3.11/bin/python3.11 USE_DEFAULT_PYTHON_LIB_PATH=1 TF_NEED_CUDA=0 TF_NEED_ROCM=0 TF_DOWNLOAD_CLANG=0 TF_NEED_MPI=0 CC_OPT_FLAGS="-march=native -Wno-sign-compare" TF_SET_ANDROID_WORKSPACE=0 TF_CONFIGURE_IOS=0 ./configure
bazel build -c opt //tensorflow:libtensorflow_cc.dylib
bazel build -c opt //tensorflow/lite/c:libtensorflowlite_c.dylib
bazel build -c opt --config=monolithic //tensorflow/lite/delegates/flex:tensorflowlite_flex
```

The TensorFlow shared library will be at `bazel-bin/tensorflow/libtensorflow.2.12.0.dylib`, `bazel-bin/tensorflow/libtensorflow_framework.2.12.0.dylib`, `bazel-bin/tensorflow/lite/c/libtensorflowlite_c.dylib`, and `bazel-bin/tensorflow/delegates/flex:tensorflowlite_flex.dylib`.

## How to build TensorFlow-Lite shared library for Android platforms

### [Clone the TensorFlow source on host system](#clone-the-tensorflow-source)

### Pull the WasmEdge:latest (based on Ubuntu 20.04) docker image and run

```bash
docker pull wasmedge/wasmedge:latest
docker run -it --rm -v $(pwd):/root/$(basename $(pwd)) wasmedge/wasmedge:latest
```

### Install Bazel 5.3.0 on Ubuntu 20.04

```bash
# In docker
apt update && apt install -y unzip
cd /root
curl -sLO https://github.com/bazelbuild/bazel/releases/download/5.3.0/bazel-5.3.0-installer-linux-x86_64.sh
chmod u+x bazel-5.3.0-installer-linux-x86_64.sh
./bazel-5.3.0-installer-linux-x86_64.sh
```

### Download and extract the Android command line tools on Ubuntu 20.04

```bash
# In docker
cd /root
curl -sLO https://dl.google.com/android/repository/commandlinetools-linux-8092744_latest.zip
unzip -q commandlinetools-linux-8092744_latest.zip
```

### Install the Android SDK and NDK through the sdkmanager on Ubuntu 20.04

```bash
# In docker
apt update && apt install -y openjdk-8-jdk
cd /root
export ANDROID_SDK_HOME=/root/android-sdk
yes | ./cmdline-tools/bin/sdkmanager --sdk_root=$ANDROID_SDK_HOME --licenses
./cmdline-tools/bin/sdkmanager "platform-tools" "platforms;android-23" --sdk_root=$ANDROID_SDK_HOME
./cmdline-tools/bin/sdkmanager "build-tools;30.0.0" --sdk_root=$ANDROID_SDK_HOME
./cmdline-tools/bin/sdkmanager "ndk;21.4.7075529" --sdk_root=$ANDROID_SDK_HOME
```

### Install the numpy on Ubuntu 20.04

```bash
# In docker
apt update && apt install -y pip
ln -s /usr/bin/python3 /usr/bin/python
pip3 install -U --user numpy
```

### Build the TensorFlow-lite shared library for Android

```bash
# In docker
cd /root/tensorflow_src
PYTHON_BIN_PATH=/usr/bin/python3 USE_DEFAULT_PYTHON_LIB_PATH=1 TF_NEED_CUDA=0 TF_NEED_ROCM=0 TF_DOWNLOAD_CLANG=0 TF_NEED_MPI=0 CC_OPT_FLAGS="-march=native -Wno-sign-compare" TF_SET_ANDROID_WORKSPACE=1 ANDROID_NDK_HOME=$ANDROID_SDK_HOME/ndk/21.4.7075529 ANDROID_NDK_API_LEVEL=23 ANDROID_API_LEVEL=23 ANDROID_BUILD_TOOLS_VERSION="30.0.0" ./configure
bazel build --cxxopt=-std=c++1z --config=android --cpu=arm64-v8a --crosstool_top=//external:android/crosstool --host_crosstool_top=@bazel_tools//tools/cpp:toolchain //tensorflow/lite/c:tensorflowlite_c --verbose_failures --copt=-w
```

The TensorFlow-Lite shared library for Android will be at `bazel-bin/tensorflow/lite/c/libtensorflowlite_c.so`.

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
