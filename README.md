# simmodsuiteCmake
CMake installer for Simmetrix SimModSuite

## Install

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
