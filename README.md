# SSVM-Tensorflow Dependencies

This repository provides pre-built dependencies for the [Second State VM (SSVM) with TensorFlow extension project](https://github.com/second-state/ssvm-tensorflow).

## Motivation

The TensorFlow official repository only provides a shared library which is compiled on Ubuntu 18.04. Hence, we cannot use this pre-built shared library on some legacy systems such as CentOS 7.6.1810. The obvious issue is that the GLIBC version is too old, so the pre-built shared library cannot be executed.

But building the dependencies on the legacy system takes lots of time. To reduce the compilation time, we create this project to compile and release the pre-built TensorFlow shared library.

## License

This project is under the Apache-2.0 License as the same as the TensorFlow project.

### Credits

- [TensorFlow](https://github.com/tensorflow/tensorflow)

## How to build on the legacy operating system - CentOS 7.6.1810

### Clone the TensorFlow source

```bash
$ git clone https://github.com/tensorflow/tensorflow.git tensorflow_src
$ cd tensorflow_src
$ git checkout v2.4.0
```

### Pull the CentOS docker image and run

```bash
$ docker pull centos:7.6.1810
$ docker run -it --rm -v $(pwd):/root/$(basename $(pwd)) centos:7.6.1810
```

### Update and install GCC-7.3.1

```bash
(docker) $ yum update
(docker) $ yum install -y centos-release-scl
(docker) $ yum install -y devtoolset-7-gcc*
(docker) $ scl enable devtoolset-7 bash
```

The gcc installation path will be at `/opt/rh/devtoolset-7/root/usr/bin/`,
and the gcc version can be checked with the command `gcc -v`.

### Build and install Python 3.8.6

```bash
(docker) $ cd /root
(docker) $ yum install -y wget make openssl-devel bzip2-devel libffi-devel
(docker) $ wget https://www.python.org/ftp/python/3.8.6/Python-3.8.6.tgz
(docker) $ tar -zxvf Python-3.8.6.tgz
(docker) $ cd Python-3.8.6
(docker) $ CXX=/opt/rh/devtoolset-7/root/usr/bin/g++ ./configure --enable-optimizations
(docker) $ make && make install
```

The python installation path will be at `/usr/local/bin/python3.8`,
and the version can be checked with the command `python3.8 --version`.

### Install Bazel 1.7.4

```bash
(docker) $ cd /root
(docker) $ wget https://github.com/bazelbuild/bazelisk/releases/download/v1.7.4/bazelisk-linux-amd64
(docker) $ chmod u+x bazelisk-linux-amd64
(docker) $ mv bazelisk-linux-amd64 /usr/local/bin/bazel
```

### Install the Python packages which are required by the TensorFlow project

```bash
(docker) $ pip3 install -U --user pip six numpy wheel setuptools mock 'future>=0.17.1'
(docker) $ pip3 install -U --user keras_applications --no-deps
(docker) $ pip3 install -U --user keras_preprocessing --no-deps
```

### Build the TensorFlow shared library

```bash
(docker) $ yum install -y git which
(docker) $ cd /root/tensorflow_src
(docker) $ ./configure
(docker) $ BAZEL_LINKLIBS=-l%:libstdc++.a bazel build -c opt --cxxopt="-D_GLIBCXX_USE_CXX11_ABI=0" //tensorflow:libtensorflow.so
(docker) $ BAZEL_LINKLIBS=-l%:libstdc++.a bazel build -c opt --cxxopt="-D_GLIBCXX_USE_CXX11_ABI=0" //tensorflow/lite/c:libtensorflowlite_c.so
```

The TensorFlow shared library will be at `bazel-bin/tensorflow/libtensorflow.so.2.4.0`, `bazel-bin/tensorflow/libtensorflow_framework.so.2.4.0`, and `bazel-bin/tensorflow/lite/c/libtensorflowlite_c.so`.

## Minimum requirements of our pre-built shared libraries

| Pre-built shared library          | GLIBC          | GLIBCXX       | CXXABI          |
| --------------------------        | -------------- | ------------- | --------------- |
| libtensorflow.so.2.4.0            | 2.17           | 3.4.19        | 1.3.7           |
| libtensorflow\_framework.so.2.4.0 | 2.16           | 3.4.19        | 1.3.7           |
| libtensorflowlite\_c.so           | 2.14           | 3.4.19        | 1.3.5           |
