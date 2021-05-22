---
layout: post
title: Setting up development environment for the barvinok library
author: Ruizhe Zhao
comments: true
---

`barvinok` is an excellent library for research and development on polyhedral model. It is currently being actively maintained by people of INRIA.

I'm still learning about it so this post I won't talk about how `barvinok` works. Instead, I would like to introduce how I managed to successfully install this library. I have a subtle complaint about the installation instructions listed in `barvinok`'s `README`. Hopefully, this post can help others who try install `barvinok` on their machine and easily begin their polyhedral journey.

The installation instructions are mainly from [https://repo.or.cz/w/barvinok.git/blob/HEAD:/README](https://repo.or.cz/w/barvinok.git/blob/HEAD:/README).

# Environment

What I'm using is a CentOS 7 Linux distribution. But it does not matter too much since in the following I would try to build `barvinok`, as well as other dependencies from source.

The `barvinok` version I'm installing is `0.41.3`, which you can get from [http://barvinok.gforge.inria.fr/](http://barvinok.gforge.inria.fr/). 

We want to install all packages, including `barvinok` and its prerequisites, under `PREFIX`. `PREFIX` can be `/opt` or somewhere else that is accessible without root priviledge. Do remember:

```bash
export PATH=${PREFIX}/bin:${PATH}
export LD_LIBRARY_PATH=${PREFIX}/lib:${LD_LIBRARY_PATH}
```

# Prerequisites

You can build `barvinok` for different purposes, but here in this post, I'm building `barvinok` for all of its functionalities, including its core utilies, `pet` related (specifically, `parse_file`), and the Python interface.

## CMake

We will need CMake to install LLVM. The version we are currently using is `3.13.3`.

I think any version larger than `3.4.3` should work, based on [this page](https://releases.llvm.org/9.0.0/docs/CMake.html). 

I won't list how CMake is installed here since it is rather easy and you can possibly find pre-built binaries. 

## LLVM

LLVM will be required when you try to build external interfaces, e.g., Python. `pet` needs LLVM as well.

The exact version of LLVM I'm using is `9.0.1`. I assume some earlier versions may work too. LLVM-10 can work, but you need to apply some post-release updates to some `barvinok`'s submodules, which is not desirable.

To build LLVM 9.0.1 - 

```bash
# Download llvm-9.0.1 from GitHub
wget https://github.com/llvm/llvm-project/releases/download/llvmorg-9.0.1/llvm-project-9.0.1.tar.xz
# Untar
tar xvf llvm-project-9.0.1.tar.xz
# CMake
cd llvm-project-9.0.1 && mkdir build && cd build
cmake -DLLVM_ENABLE_PROJECTS="clang;clang-tools-extra;lld" \
			-DCMAKE_INSTALL_PREFIX=${PREFIX} \
      -DLLVM_BUILD_LLVM_DYLIB=ON ../llvm
# Build
make 
# Install
make install
```

Note that here we should enable `clang` and make sure LLVM is built with shared library.

## GMP

Here I choose version `6.2.0`

```bash
wget https://gmplib.org/download/gmp/gmp-6.2.0.tar.xz
tar xvf gmp-6.2.0.tar.xz
cd gmp-6.2.0
./configure --prefix=${PREFIX}
make
make check
make install
```

## NTL

I'm using NTL version `11.4.3` and installing it based on the instructions by `barvinok`.

```bash
wget https://shoup.net/ntl/ntl-11.4.3.tar.gz
tar xvf ntl-11.4.3.tar.gz
cd ntl-11.4.3/src
./configure NTL_GMP_LIP=on PREFIX=${PREFIX} SHARED=on GMP_PREFIX=${PREFIX}
make
# This may take a while ...
make check
make install
```

## Python

The bundled Python version is too old on CentOS 7. Therefore, I decide to manually install Python `3.8.3` from source.

```bash
wget 
./configure --prefix=${PREFIX} --enable-shared --enable-optimization
make
make test
make install
```

# Barvinok Install

Now all the prerequisites should have been successfully installed. We can move on to installing `barvinok`.

First of all, we grab the `barvinok` package by `git`:

```bash
git clone --recurse-submodules git://repo.or.cz/barvinok.git
cd barvinok
git checkout barvinok-0.41.3
```

Setup install scripts

```bash
sh autogen.sh
```

Configuration & install:

```bash
./configure \
	--prefix=${PREFIX} \
	--with-ntl-prefix=${PREFIX} \
  --with-pet=bundled \
  --with-isl=bundled \
  --with-clang-prefix=${PREFIX} \
  --enable-shared-barvinok
make 
make check
make install
```

Note that here we use all the bundled packages.

# Verification

That's it! If you follow the steps above, the installation should work properly.

Now you can verify the installation.

First checkout the version of `iscc` , which shows the versions of both `barvinok` and all the dependencies.

```bash
$ iscc --version
isl-0.22.1-347-g7ef6ed0c-IMath-32
barvinok-0.41.3
 -INCREMENTAL
 +PET -OMEGA -CDDLIB -GLPK -TOPCOM +ZSOLVE -PARKER
clang version 9.0.1
pet-0.11.3
```

You can also launch `iscc` and enter command `parse_file` to see whether it is available or not.

Go to the `barvinok` directory and go under `isl/interface`, run `python3` to check the `isl` Python interface.

```bash
python3 -c 'import isl'
```

This command should work well.

If everything is fine, I believe you can now start your amazing polyhedral research!
