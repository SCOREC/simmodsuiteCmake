cmake_minimum_required(VERSION 3.15)
project(simmetrix-simmodsuite)

set(SIM_ARCH "" CACHE STRING "Architecture/OS string for libraries (e.g., x64_rhel7_gcc48)")
set(SIM_MPI "" CACHE STRING "MPI implementation used for SimPartitionWrapper")
if(SIM_MPI MATCHES "^$")
  message(FATAL_ERROR "SIM_MPI is not defined... libSimPartitionWrapper-$SIM_MPI.a should exist in the SimModSuite lib directory")
endif()

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
    find_library(simlib "${lib}" PATH_SUFFIXES lib/${SIM_ARCH})
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

find_file(simVersion SimModSuiteVersion.h REQUIRED)
message(STATUS "simVersion ${simVersion}")
getVersion(${simVersion})
message(STATUS "minor ${SIMMODSUITE_MINOR_VERSION}")
message(STATUS "major ${SIMMODSUITE_MAJOR_VERSION}")
message(STATUS "dot ${SIMMODSUITE_DOT_VERSION}")

set(SIM_BOOTSTRAP_LIB_NAME SimModel)
simLibCheck("${SIM_BOOTSTRAP_LIB_NAME}" TRUE)

#FIXME pick a way to set these and find the libs - currently require both
#CMAKE_PREFIX_PATH (or some other default search path) and SIMMETRIX_SIMMODSUITE_DIR
set(SIMMODSUITE_INSTALL_DIR ${SIMMETRIX_SIMMODSUITE_DIR})

string(FIND "${SIMMODSUITE_LIBS}" "/lib/" archStart)
string(FIND "${SIMMODSUITE_LIBS}" "/libSim" archEnd)
math(EXPR archStart "${archStart}+5")
math(EXPR len "${archEnd}-${archStart}")
string(SUBSTRING "${SIMMODSUITE_LIBS}" "${archStart}" "${len}" SIM_ARCHOS)
message(STATUS "SIM_ARCHOS ${SIM_ARCHOS}")

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
  "$<BUILD_INTERFACE:${SIMMETRIX_SIMMODSUITE_DIR}/include>"
  "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
)

if (UNIX AND NOT APPLE)
  find_package(Threads REQUIRED)
  target_link_libraries(SimModSuite INTERFACE Threads::Threads)
endif()

add_executable(testSim testSim.cpp)
target_link_libraries(testSim SimModSuite)
