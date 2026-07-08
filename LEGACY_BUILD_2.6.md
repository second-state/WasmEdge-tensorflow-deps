# Legacy build guide — TensorFlow 2.6.0

The build guide for the TF-2.6.0 and TF-2.6.0-CC releases. For the current release see the [README](README.md).

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

### Install Bazelisk 1.7.4 on CentOS 7.6.1810 x86_64 or aarch64

```bash
# In docker
cd /root
curl -sLO https://github.com/bazelbuild/bazelisk/releases/download/v1.16.0/bazelisk-linux-amd64
# For aarch64:
#   curl -sLO https://github.com/bazelbuild/bazelisk/releases/download/v1.16.0/bazelisk-linux-arm64
chmod u+x bazelisk-linux-amd64
mv bazelisk-linux-amd64 /usr/local/bin/bazel
```

### Install the Python packages which are required by the TensorFlow project on CentOS 7.6.1810 x86_64 or aarch64

```bash
# In docker
pip3 install -U --user pip six numpy wheel setuptools mock 'future>=0.17.1'
pip3 install -U --user keras_applications --no-deps
pip3 install -U --user keras_preprocessing --no-deps
```

### Build the TensorFlow and TensorFlow-lite shared library on CentOS 7.6.1810 x86_64 or aarch64

```bash
# In docker
cd /root/tensorflow_src
PYTHON_BIN_PATH=/usr/local/bin/python3.8 USE_DEFAULT_PYTHON_LIB_PATH=1 TF_NEED_CUDA=0 TF_NEED_ROCM=0 TF_DOWNLOAD_CLANG=0 TF_NEED_MPI=0 CC_OPT_FLAGS="-march=native -Wno-sign-compare" TF_SET_ANDROID_WORKSPACE=0 ./configure
bazel build -c opt --cxxopt="-D_GLIBCXX_USE_CXX11_ABI=0" --config=monolithic //tensorflow:libtensorflow_cc.so
bazel build -c opt --cxxopt="-D_GLIBCXX_USE_CXX11_ABI=0" --config=monolithic //tensorflow/lite/c:libtensorflowlite_c.so
bazel build -c opt --cxxopt="-D_GLIBCXX_USE_CXX11_ABI=0" --config=monolithic //tensorflow/lite/delegates/flex:tensorflowlite_flex
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

### Install Bazelisk 1.11.0 on MacOS 10.15 x86_64 or MacOS 12 arm64

```bash
curl -sLO https://github.com/bazelbuild/bazelisk/releases/download/v1.16.0/bazelisk-darwin-amd64
# For apple silicon:
#   curl -sLO https://github.com/bazelbuild/bazelisk/releases/download/v1.16.0/bazelisk-darwin-arm64
chmod u+x bazelisk-darwin-amd64
mv bazelisk-darwin-amd64 /usr/local/bin/bazel
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

### Install Bazel 3.7.2 on Ubuntu 20.04

```bash
# In docker
apt update && apt install -y unzip
cd /root
curl -sLO https://github.com/bazelbuild/bazel/releases/download/3.7.2/bazel-3.7.2-installer-linux-x86_64.sh
chmod u+x bazel-3.7.2-installer-linux-x86_64.sh
./bazel-3.7.2-installer-linux-x86_64.sh
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
bazel build --cxxopt=-std=c++1z --config=android --cpu=arm64-v8a --fat_apk_cpu=arm64-v8a --crosstool_top=//external:android/crosstool --host_crosstool_top=@bazel_tools//tools/cpp:toolchain //tensorflow/lite/c:tensorflowlite_c --verbose_failures --copt=-w
```

The TensorFlow-Lite shared library for Android will be at `bazel-bin/tensorflow/lite/c/libtensorflowlite_c.so`.
