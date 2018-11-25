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
$ mkdir ct  # or whatever directory you prefer
$ cd ct
$ gclient config --name="certificate-transparency" https://github.com/google/certificate-transparency.git
$ gclient sync --disable-syntax-validation  # retrieve and build dependencies
$ make -C certificate-transparency check  # build the CT software & self-test
```
