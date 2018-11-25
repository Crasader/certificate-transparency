### Build Dependencies

The following tools are needed to build the CT software and its dependencies.
- [depot_tools](https://www.chromium.org/developers/how-tos/install-depot-tools)
- autoconf/automake etc.
- libtool
- shtool
- clang++ (>=3.4)
- cmake (>=v3.1.2)
- git
- GNU make
- Tcl
- pkg-config
- Python 2.7


### Build Instructions
```
$ export CXX=clang++ CC=clang
$ make -C certificate-transparency check  # build the CT software & self-test
```
There are two directories named `certificate-transparency`: one is nested inside the other. Make sure you are calling `make` from the parent-level `certificate-transparency` (the one with all the other folders like `protobuf`) in it.
