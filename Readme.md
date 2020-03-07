# OptSched - Project

This repository is a central location to set up OptSched.

# Getting Source Code

## LLVM + OptSched

Clone this repository, then run:

```
git submodule update --init --recursive llvm-project
```

LLVM will be cloned to llvm-project and OptSched will be cloned to llvm-project/llvm/projects/OptSched.

## LLVM + OptSched + Flang

Clone this repository with the `--recursive` flag.

## Changing configurations

Want to add flang after the fact?

```
git submodule update --init --recursive
```

That command can also fix cloning without the `--recursive` flag.

# Building

```
$ mkdir build && cd build
$ cmake -GNinja -DCMAKE_BUILD_TYPE=Debug -DLLVM_PARALLEL_LINK_JOBS=1 -DOPTSCHEDPRJ_FLANG_INSTALL_PREFIX=$PWD/flang-install ..
$ ninja
```
