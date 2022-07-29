cmake_minimum_required(VERSION 3.20)
project(simmetrix-simmodsuite)

set(SIM_ARCHOS "" CACHE STRING "Architecture/OS string for libraries (e.g., x64_rhel7_gcc48)")
if(SIM_ARCHOS MATCHES "^$")
  message(FATAL_ERROR "SIM_ARCHOS is not defined... $SIM_ARCHOS should be a "
  "sub-directory of the the SimModSuite 'lib' directory")
endif()
message(STATUS "SIM_ARCHOS ${SIM_ARCHOS}")
set(SIM_MPI "" CACHE STRING "MPI implementation used for SimPartitionWrapper")
if(SIM_MPI MATCHES "^$")
  message(FATAL_ERROR "SIM_MPI is not defined... libSimPartitionWrapper-$SIM_MPI.a "
  "should exist in the SimModSuite lib directory")
endif()
message(STATUS "SIM_MPI ${SIM_MPI}")

function(getVersion simVersionFile)
  file(STRINGS ${simVersionFile} lines)

  set(majorLine ${lines})
  list(FILTER majorLine INCLUDE REGEX ".*SIMMODSUITE_MAJOR_VERSION.*")
  string(REPLACE " " ";" majorList ${majorLine})
  list(GET majorList 2 major)
  set(SIMMODSUITE_MAJOR_VERSION "${major}" PARENT_SCOPE)

  set(minorLine ${lines})
  list(FILTER minorLine INCLUDE REGEX ".*SIMMODSUITE_MINOR_VERSION.*")
  string(REPLACE " " ";" minorList ${minorLine})
  list(GET minorList 2 minor)
  set(SIMMODSUITE_MINOR_VERSION "${minor}" PARENT_SCOPE)

  string(JOIN "." dot ${major} ${minor})
  set(SIMMODSUITE_DOT_VERSION "${dot}" PARENT_SCOPE)
endfunction()

#FIXME change the name
macro(simLibCheck libs isRequired)
  foreach(lib ${libs})
    unset(simlib CACHE)
    find_library(simlib "${lib}" PATH_SUFFIXES lib/${SIM_ARCHOS})
    if(simlib MATCHES "^simlib-NOTFOUND$")
      if(${isRequired})
        message(FATAL_ERROR "simmetrix library ${lib} not found")
      else()
        message("simmetrix library ${lib} not found")
      endif()
    else()
      set("SIMMODSUITE_${lib}_FOUND" TRUE CACHE INTERNAL "SimModSuite library present")
      set(SIMMODSUITE_LIBS ${SIMMODSUITE_LIBS} ${simlib})
      message(STATUS "lib ${lib}")
      add_library(${lib} STATIC IMPORTED GLOBAL)
      set_target_properties(${lib} PROPERTIES IMPORTED_LOCATION "${simlib}")
    endif()
  endforeach()
endmacro(simLibCheck)

macro(getSimCadLib searchPath libName lib check)
  file(GLOB cadLib
    RELATIVE ${searchPath}/
    ${searchPath}/lib${libName}*)
  if( check AND NOT cadLib )
    message(FATAL_ERROR "lib${libName} not found")
  endif()
  set(${lib} "${cadLib}")
endmacro(getSimCadLib)

set(SIMMODSUITE_LIBS "")

find_file(simVersion SimModSuiteVersion.h PATH_SUFFIXES include REQUIRED)
cmake_path(REMOVE_FILENAME simVersion OUTPUT_VARIABLE SIMMODSUITE_INCLUDE_DIR)
message(STATUS "SIMMODSUITE_INCLUDE_DIR ${SIMMODSUITE_INCLUDE_DIR}")
message(STATUS "simVersion ${simVersion}")
getVersion(${simVersion})
message(STATUS "minor ${SIMMODSUITE_MINOR_VERSION}")
message(STATUS "major ${SIMMODSUITE_MAJOR_VERSION}")
message(STATUS "dot ${SIMMODSUITE_DOT_VERSION}")

set(SIM_BOOTSTRAP_LIB_NAME SimModel)
simLibCheck("${SIM_BOOTSTRAP_LIB_NAME}" TRUE)

#FIXME - getSimCadLib is likely broken due to its use of SIMMODSUITE_INSTALL_DIR
option(SIM_PARASOLID "Use Parasolid through Simmetrix" OFF)
if (SIM_PARASOLID)
  set(MIN_SIM_PARASOLID_VERSION 290)
  set(MAX_SIM_PARASOLID_VERSION 330) #FIXME
  foreach(version RANGE
      ${MAX_SIM_PARASOLID_VERSION}
      ${MIN_SIM_PARASOLID_VERSION} -10)
    set(SIM_PARASOLID_VERSION ${version})
    getSimCadLib("${SIMMODSUITE_INSTALL_DIR}/lib/${SIM_ARCHOS}"
      SimParasolid${SIM_PARASOLID_VERSION} simParaLib FALSE)
    if(simParaLib)
      break()
    endif()
  endforeach()
  if(NOT simParaLib)
    message(FATAL_ERROR "libSimParasolid<#>.a "
      "${MIN_SIM_PARASOLID_VERSION}-${MAX_SIM_PARASOLID_VERSION} "
      "not found - check the version installed with SimModSuite")
  endif()
  set(SIM_CAD_LIB_NAMES
    ${simParaLib}
    pskernel)
endif()

option(SIM_ACIS "Use Acis through Simmetrix" OFF)
if (SIM_ACIS)
  getSimCadLib("${SIMMODSUITE_INSTALL_DIR}/lib/${SIM_ARCHOS}"
    SimAcis simAcisLib TRUE)
  set(SIM_CAD_LIB_NAMES
      ${simAcisLib}
      ${SIM_CAD_LIB_NAMES}
      SpaACIS)
endif()

option(SIM_DISCRETE "Use Simmetrix discrete modeling" ON)
if (SIM_DISCRETE)
  set(SIM_CAD_LIB_NAMES SimDiscrete ${SIM_CAD_LIB_NAMES})
endif()

simLibCheck("${SIM_CAD_LIB_NAMES}" TRUE)

set(SIM_OPT_LIB_NAMES
  SimField
  SimAdvMeshing)

simLibCheck("${SIM_OPT_LIB_NAMES}" FALSE)

set(SIM_CORE_LIB_NAMES
  SimPartitionedMesh-mpi
  SimMeshing
  SimMeshTools
  SimPartitionWrapper-${SIM_MPI})

simLibCheck("${SIM_CORE_LIB_NAMES}" TRUE)

add_library(SimModSuite INTERFACE)
foreach(lib ${SIMMODSUITE_LIBS})
  target_link_libraries(SimModSuite INTERFACE ${lib})
endforeach()
target_include_directories(SimModSuite INTERFACE
  "$<BUILD_INTERFACE:${SIMMODSUITE_INCLUDE_DIR}>"
  "$<INSTALL_INTERFACE:${SIMMODSUITE_INCLUDE_DIR}>"
)

if (UNIX AND NOT APPLE)
  find_package(Threads REQUIRED)
  target_link_libraries(SimModSuite INTERFACE Threads::Threads)
endif()

find_library(XDR_LIB tirpc)
if(XDR_LIB)
  message(STATUS "Found XDR_LIB ${XDR_LIB}")
  target_link_libraries(SimModSuite INTERFACE ${XDR_LIB})
else()
  message(STATUS "The libtirpc library was not found.  It defines xdr symbols "
    "(e.g., xdrmem_create) that are need by SimModSuite on systems using "
    "glibc newer than 2.32.  Note, glibc starting with 2.26 could optionally "
    "have been built without the xdr symbols.")
endif()

## install
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

install(
  TARGETS SimModSuite
  EXPORT simmetrix-simmodsuite-targets
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

configure_package_config_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/config.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/simmetrix-simmodsuite-config.cmake"
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/simmetrix-simmodsuite
)
write_basic_package_version_file(
  "${PROJECT_BINARY_DIR}/simmetrix-simmodsuite-config-version.cmake"
  VERSION ${SIMMODSUITE_DOT_VERSION}
  COMPATIBILITY AnyNewerVersion)

install(FILES
  "${PROJECT_BINARY_DIR}/simmetrix-simmodsuite-config.cmake"
  "${PROJECT_BINARY_DIR}/simmetrix-simmodsuite-config-version.cmake"
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/simmetrix-simmodsuite)

install(
  EXPORT simmetrix-simmodsuite-targets
  NAMESPACE simmetrix-simmodsuite::
  DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/simmetrix-simmodsuite)

