# simmodsuiteCmake
CMake installer for Simmetrix SimModSuite

This assumes that the user has extracted the SimModSuite tarballs.  That
directory is referred to below as the SimModSuite install dir.

## Install

The CMake config file may be installed alongside the extracted SimModSuite
libraries and headers by passing the path to the SimModSuite install dir to
`-DCMAKE_INSTALL_PREFIX`.
A different location can also be used if you do not have write access to
the SimModSuite install dir.
The example commands below use the SimModSuite install dir for the install
prefix.

```
cmake \
-S simmodsuiteCmake \
-B buildSimModSuite \
-DCMAKE_PREFIX_PATH=/path/to/simmodsuite/install/dir \
-DSIM_MPI=mpiSuffix \
-DSIM_ARCHOS=[x64_rhel7_gcc48|x64_rhel8_gcc83] \
-DCMAKE_INSTALL_PREFIX=/path/to/simmodsuite/install/dir

cmake --build buildSimModSuite --target install
```

## Example CMake package depending on SimModSuite

```
cmake \
-S simmodsuiteCmake/example/build_cmake_installed/ \
-B buildTestSim \
-DCMAKE_PREFIX_PATH=/path/to/simmodsuite/install/dir

cmake --build buildTestSim
```
