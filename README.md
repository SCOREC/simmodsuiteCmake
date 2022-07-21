# simmodsuiteCmake
CMake installer for Simmetrix SimModSuite

## Install

```
cmake \
-S simmodsuiteCmake \
-B buildSimModSuite \
-DSIMMETRIX_SIMMODSUITE_DIR=/path/to/dir/where/simmodsuite/tarballs/were/extracted \
-DCMAKE_PREFIX_PATH=/path/to/simmodsuite/dir/with/libs \
-DSIM_MPI=mpiSuffix

cmake --build buildSimModSuite
```

