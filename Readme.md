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

TODO
