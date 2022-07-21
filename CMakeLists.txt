cmake_minimum_required(VERSION 3.15)
project(simmetrix-simmodsuite)

set(libs
  libSimAcis2021
  libSimAbstract
  libSimAdvMeshing
  libSimDiscrete
  libSimField
  libSimMeshing
  libSimMeshTools
  libSimModel
  libSimOpenCascade74
  libSimParasolid320
  libSimParasolid330
  libSimParasolid340
  libSimPartitionedMesh
  libSimPartitionedMesh-mpi
  libSimPartitionWrapper
  libSimPartitionWrapper-mpich3
  libSimPartitionWrapper-openmpi3
  libSimPartitionWrapper-openmpi4
)

foreach(lib ${libs})
  message(STATUS "${lib}")
  add_library(${lib} STATIC IMPORTED GLOBAL)
  set_target_properties(${lib} PROPERTIES IMPORTED_LOCATION
    "${SIMMETRIX_SIMMODSUITE_DIR}/lib/x64_rhel7_gcc48/${lib}.a")
endforeach()

add_library(SimModSuite INTERFACE)
foreach(lib ${libs})
  target_link_libraries(SimModSuite INTERFACE ${lib})
endforeach()
target_include_directories(SimModSuite INTERFACE
  "$<BUILD_INTERFACE:${SIMMETRIX_SIMMODSUITE_DIR}/include>"
  "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
)

if (UNIX AND NOT APPLE)
  find_package(Threads REQUIRED)
  target_link_libraries(SimModSuite INTERFACE Threads::Threads)
endif()

add_executable(testSim testSim.cpp)
target_link_libraries(testSim SimModSuite)
