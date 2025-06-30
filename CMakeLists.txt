cmake_minimum_required(VERSION 2.8.3)
project(eigen_catkin)

find_package(catkin_simple REQUIRED)

catkin_simple()

include(ExternalProject)

set(EIGEN_CATKIN_EIGEN_INCLUDE ${CATKIN_DEVEL_PREFIX}/include/eigen3)
set(EIGEN_CATKIN_EIGEN_INCLUDE_SIGNATURE ${EIGEN_CATKIN_EIGEN_INCLUDE}/signature_of_eigen3_matrix_library)

set(EIGEN_MINIMUM_VERSION 3.3.4)

# If set to OFF, will compile the given version of Eigen.
# If set to ON, will use the system version of Eigen.
# If set to AUTO, will determine if the system version is suitable.
set(USE_SYSTEM_EIGEN "OFF" CACHE INTERNAL "Whether to use the system version of Eigen.")

# Check if the version is suitable, otherwise force downloading the newer one.
if(USE_SYSTEM_EIGEN STREQUAL "AUTO")
  # We don't use the find_package version checks here because they fail on
  # Ubuntu 16.04, so we'll do it manually.
  find_package(Eigen3 QUIET)
  if(EIGEN3_FOUND)
    # Necessary for Ubuntu 16.04's god-awful FindEigen3.cmake.
    if((NOT (DEFINED EIGEN3_VERSION)) AND (DEFINED EIGEN3_VERSION_STRING))
      set(EIGEN3_VERSION ${EIGEN3_VERSION_STRING})
    endif()

    # Double check what Eigen3 found, since it doesn't seem to be sufficiently
    # strict at least on OS X. And on Ubuntu 16.04 version is anyway undefined.
    if((${EIGEN3_VERSION} VERSION_GREATER ${EIGEN_MINIMUM_VERSION}) OR (${EIGEN3_VERSION} VERSION_EQUAL ${EIGEN_MINIMUM_VERSION}))
      set(USE_SYSTEM_EIGEN "ON")
      message(STATUS "Suitable Eigen version found. Minimum version: ${EIGEN_MINIMUM_VERSION} Found version: ${EIGEN3_VERSION}")
    else()
      set(USE_SYSTEM_EIGEN "OFF")
      message(STATUS "Suitable Eigen version not found. Minimum version: ${EIGEN_MINIMUM_VERSION} Found version: ${EIGEN3_VERSION}")
    endif()
  else()
    set(USE_SYSTEM_EIGEN "OFF")
    message(STATUS "No suitable Eigen version found. Minimum version: ${EIGEN_MINIMUM_VERSION}")
  endif()
endif()

if(USE_SYSTEM_EIGEN)
  find_package(Eigen3 REQUIRED)  # now guaranteed to exist
  message(STATUS "eigen_catkin: using system Eigen from ${EIGEN3_INCLUDE_DIR}")
  message(STATUS "Eigen version variables:"
                "  Eigen3_VERSION        = '${Eigen3_VERSION}'"
                "  EIGEN3_VERSION_STRING = '${EIGEN3_VERSION_STRING}'"
                "  Final EIGEN3_VERSION  = '${EIGEN3_VERSION}'")

  # Normalize the version into EIGEN3_VERSION (if not already set)
  if(NOT DEFINED EIGEN3_VERSION OR EIGEN3_VERSION STREQUAL "")
    if(DEFINED Eigen3_VERSION)
      set(EIGEN3_VERSION "${Eigen3_VERSION}" CACHE INTERNAL "" FORCE)
    elseif(DEFINED EIGEN3_VERSION_STRING)
      set(EIGEN3_VERSION "${EIGEN3_VERSION_STRING}" CACHE INTERNAL "" FORCE)
    else()
      message(WARNING "Could not determine Eigen version: neither Eigen3_VERSION nor EIGEN3_VERSION_STRING is defined.")
    endif()
  endif()
  # Only patch if it's exactly version 3.3.7
  if(EIGEN3_VERSION VERSION_EQUAL "3.3.7")
    message(STATUS "Eigen ${EIGEN3_VERSION} detected; applying custom patches")
    message(STATUS "CMAKE_CURRENT_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR} EIGEN3_INCLUDE_DIR ${EIGEN3_INCLUDE_DIR}")

    set(_patch_dir     "${EIGEN3_INCLUDE_DIR}")
    set(_patch_files
      SelfAdjointEigenSolver337.patch
      Tridiagonalization337.patch
    )
  
    foreach(patch_file IN LISTS _patch_files)
      set(_patch_path "${CMAKE_CURRENT_SOURCE_DIR}/${patch_file}")
      message(STATUS "  -> Applying ${patch_file} in ${_patch_dir}")
      execute_process(
        COMMAND patch -N -p0 -i "${_patch_path}"
        WORKING_DIRECTORY "${_patch_dir}"
        RESULT_VARIABLE _res
        OUTPUT_VARIABLE  _out
        ERROR_VARIABLE   _err
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_STRIP_TRAILING_WHITESPACE
      )
      if(_res GREATER 1)
        # Serious failure
        message(FATAL_ERROR
          "Failed to apply patch ${patch_file} (exit code ${_res})\n"
          "----- patch stdout -----\n${_out}\n"
          "----- patch stderr -----\n${_err}"
        )
      elseif(_res EQUAL 1)
        # Hunk already applied or skipped
        message(WARNING
          "Patch ${patch_file} was already applied (or reversed). Skipping.\n"
          "----- patch stdout -----\n${_out}\n"
          "----- patch stderr -----\n${_err}"
        )
      else()
        message(STATUS "Patch ${patch_file} applied successfully.")
      endif()
    endforeach()
  else()
    message(WARNING "Skipping Eigen patches: found version ${EIGEN3_VERSION}, expected 3.3.7")
  endif()

  if(EXISTS ${EIGEN_CATKIN_EIGEN_INCLUDE_SIGNATURE})
    file(RENAME ${EIGEN_CATKIN_EIGEN_INCLUDE_SIGNATURE} ${EIGEN_CATKIN_EIGEN_INCLUDE_SIGNATURE}.off)

    if(EIGEN3_INCLUDE_DIR EQUAL EIGEN_CATKIN_EIGEN_INCLUDE)
      unset(EIGEN3_INCLUDE_DIR CACHE)
    endif()
    if(Eigen_INCLUDE_DIRS EQUAL EIGEN_CATKIN_EIGEN_INCLUDE)
      unset(Eigen_INCLUDE_DIRS CACHE)
    endif()
  endif()

  # see https://github.com/ethz-asl/ethzasl_icp_mapping/issues/38 for the discussion behind the following
  find_package(Eigen3)

  # Eigen3 will not be found on Ubuntu 12.04, so we moke it
  if(EIGEN3_FOUND)
    set(EIGEN_CATKIN_EIGEN_INCLUDE ${EIGEN3_INCLUDE_DIR} CACHE INTERNAL "")
    set(EIGEN_NAME "EIGEN3")
  else()
    # Fallback to cmake_modules
    find_package(Eigen REQUIRED)
    if(EIGEN_FOUND)
      set(EIGEN_CATKIN_EIGEN_INCLUDE ${Eigen_INCLUDE_DIRS} CACHE INTERNAL "")
      set(EIGEN_NAME "Eigen")
    endif()
  endif()

  if(EIGEN_NAME)
    # this way ${EIGEN_NAME} will end up in the cs_export call's DEPENDS argument.
    list(APPEND _DEPENDS ${EIGEN_NAME})
    message(STATUS "Found and using system Eigen in ${EIGEN_CATKIN_EIGEN_INCLUDE}")
  else()
    message(FATAL_ERROR "Could not find a system Eigen installation! What about turning on eigen_catkin's own Eigen installation by removing USE_SYSTEM_EIGEN.cmake or switching back to the master branch.")
  endif()
else()
  message(STATUS "eigen_catkin is configured NOT to use system eigen")
  file(MAKE_DIRECTORY ${EIGEN_CATKIN_EIGEN_INCLUDE})
  ExternalProject_Add(eigen_src
    UPDATE_COMMAND ""
    URL https://gitlab.com/libeigen/eigen/-/archive/3.4.0/eigen-3.4.0.tar.bz2
    URL_MD5 132dde48fe2b563211675626d29f1707

    PATCH_COMMAND patch -p0 < ${CMAKE_CURRENT_SOURCE_DIR}/Tridiagonalization340.patch
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX:PATH=${CATKIN_DEVEL_PREFIX} -DCMAKE_BUILD_TYPE:STRING=Release -DBUILD_TESTING=OFF
  )

  if(EXISTS ${EIGEN_CATKIN_EIGEN_INCLUDE_SIGNATURE}.off)
    file(RENAME ${EIGEN_CATKIN_EIGEN_INCLUDE_SIGNATURE}.off ${EIGEN_CATKIN_EIGEN_INCLUDE_SIGNATURE})
  endif()

  message(STATUS "Using eigen_catkin's Eigen in ${EIGEN_CATKIN_EIGEN_INCLUDE}")

  install(DIRECTORY ${EIGEN_CATKIN_EIGEN_INCLUDE}
          DESTINATION ${CATKIN_GLOBAL_INCLUDE_DESTINATION}
  )
endif()

cs_export(INCLUDE_DIRS ${EIGEN_CATKIN_EIGEN_INCLUDE}
          DEPENDS ${_DEPENDS}
          CFG_EXTRAS eigen-extras.cmake)
