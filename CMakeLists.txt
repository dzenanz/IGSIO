CMAKE_MINIMUM_REQUIRED(VERSION 3.3.0)
PROJECT(IGSIO)

SET(IGSIO_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}")
SET(IGSIO_BINARY_DIR "${CMAKE_BINARY_DIR}")

# ------------------------------------------------------------------------
# Set project version number
SET(IGSIO_VERSION_MAJOR "1")
SET(IGSIO_VERSION_MINOR "0")
SET(IGSIO_VERSION_PATCH "0")
SET(IGSIO_VERSION ${IGSIO_VERSION_MAJOR}.${IGSIO_VERSION_MINOR}.${IGSIO_VERSION_PATCH})

SET(CMAKE_MODULE_PATH
  ${CMAKE_CURRENT_SOURCE_DIR}/Modules
  ${CMAKE_MODULE_PATH}
  )

# --------------------------------------------------------------------------
# System Settings
IF(WIN32)
  # This method adds the necessary compiler flag
  SET(RUNTIME_MINIMUM_WINDOWS_VERSION "0x0602")
  ADD_DEFINITIONS(-D_MSC_PLATFORM_TOOLSET_$(PlatformToolset))
  IF((NOT CMAKE_GENERATOR_TOOLSET OR NOT ${CMAKE_GENERATOR_TOOLSET} MATCHES .*xp) AND
     (NOT CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION OR CMAKE_VS_WINDOWS_TARGET_PLATFORM_VERSION VERSION_GREATER_EQUAL 8.1))
    ADD_DEFINITIONS(-DIGSIO_USE_VERSION_HELPER)
  ENDIF()
ENDIF()

# --------------------------------------------------------------------------
# Configure output paths for libraries and executables.
IF(NOT DEFINED CMAKE_RUNTIME_OUTPUT_DIRECTORY)
  SET(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
ENDIF()

IF(NOT DEFINED CMAKE_LIBRARY_OUTPUT_DIRECTORY)
  SET(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")
ENDIF()

IF(NOT DEFINED CMAKE_ARCHIVE_OUTPUT_DIRECTORY)
  SET(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
ENDIF()

IF(NOT IGSIO_INSTALL_BIN_DIR)
  SET(IGSIO_INSTALL_BIN_DIR "bin")
ENDIF()

IF(NOT IGSIO_INSTALL_LIB_DIR)
  SET(IGSIO_INSTALL_LIB_DIR "lib")
ENDIF()

IF(NOT IGSIO_INSTALL_DATA_DIR)
  SET(IGSIO_INSTALL_DATA_DIR "share")
ENDIF()

IF(NOT DEFINED BUILD_SHARED_LIBS)
  SET(BUILD_SHARED_LIBS ON)
ENDIF()
OPTION(BUILD_SHARED_LIBS "Build shared libraries" ${BUILD_SHARED_LIBS})

SET(IGSIO_EXECUTABLE_OUTPUT_PATH "${CMAKE_RUNTIME_OUTPUT_DIRECTORY}")
IF(MSVC OR ${CMAKE_GENERATOR} MATCHES "Xcode")
  SET(TEST_OUTPUT_PATH "${IGSIO_EXECUTABLE_OUTPUT_PATH}/Release/Output")
ELSE()
  SET(TEST_OUTPUT_PATH "${IGSIO_EXECUTABLE_OUTPUT_PATH}/Output")
ENDIF()

OPTION(COPY_EXTERNAL_LIBS "Copy external libraries to output directory." OFF)

#-----------------------------------------------------------------------------
# Specify common external project properties
#-----------------------------------------------------------------------------
INCLUDE(${CMAKE_ROOT}/Modules/GenerateExportHeader.cmake)
INCLUDE(${CMAKE_CURRENT_SOURCE_DIR}/CMake/IGSIOMacros.cmake)

# --------------------------------------------------------------------------
OPTION(IGSIO_USE_3DSlicer "" OFF)
IF (IGSIO_USE_3DSlicer)
  FIND_PACKAGE(Slicer REQUIRED)
  SET(IGSIO_Slicer_DIR ${Slicer_DIR} CACHE STRING "")
ENDIF()

# --------------------------------------------------------------------------
# Try to find VTK and include its settings (otherwise complain)
FIND_PACKAGE(VTK NO_MODULE REQUIRED PATHS ${VTK_DIR} NO_DEFAULT_PATH)
IF(VTK_FOUND AND VTK_VERSION VERSION_LESS 8.9.0)
  INCLUDE(${VTK_USE_FILE})
ENDIF()

IF(TARGET VTK::CommonCore)
  SET(IGSIO_VTK_PREFIX VTK::)
ELSE()
  SET(IGSIO_VTK_PREFIX vtk)
ENDIF()

IF (COPY_EXTERNAL_LIBS)
  CopyLibrariesToDirectory(${CMAKE_RUNTIME_OUTPUT_DIRECTORY} ${VTK_LIBRARIES})
ENDIF()

OPTION(IGSIO_USE_SYSTEM_ZLIB "Use system zlib" OFF)
OPTION(BUILD_TESTING "Enable tests" ON)
IF(BUILD_TESTING)
  INCLUDE(CTest)
ENDIF()

OPTION(IGSIO_BUILD_SEQUENCEIO "Build classes for reading/writing sequence files" ON)
OPTION(IGSIO_BUILD_VOLUMERECONSTRUCTION "Build classes for volume reconstruction" OFF)
OPTION(IGSIO_SEQUENCEIO_ENABLE_MKV "Enable MKV reading/writing" OFF)
OPTION(IGSIO_BUILD_CODECS "Build classes for Video codecs" OFF)
OPTION(IGSIO_USE_VP9 "Enable VP9 codec" OFF)
IF(IGSIO_USE_VP9)
  SET(IGSIO_BUILD_CODECS ON CACHE BOOL "Build classes for reading/writing sequence files" FORCE)
ENDIF()

IF(IGSIO_SEQUENCEIO_ENABLE_MKV)
  SET(IGSIO_BUILD_SEQUENCEIO ON CACHE BOOL "Build classes for reading/writing sequence files" FORCE)
ENDIF()

SET(IGSIO_ZLIB_LIBRARY ${ZLIB_LIBRARY} CACHE STRING "")
SET(IGSIO_ZLIB_INCLUDE_DIR CACHE STRING "")
IF(IGSIO_USE_SYSTEM_ZLIB) # setting IGSIOZLib needs to come before Superbuild
  SET(IGSIO_ZLIB_LIBRARY ${ZLIB_LIBRARY})
  SET(IGSIO_ZLIB_INCLUDE_DIR ${ZLIB_INCLUDE_DIR})
ELSE()
  SET(IGSIO_ZLIB_LIBRARY vtkzlib)
  SET(IGSIO_ZLIB_INCLUDE_DIR "")
ENDIF()

# Enable GPU support. Requires OpenCL to be installed
option(IGSIO_USE_GPU "GPU acceleration via OpenCL" OFF)
mark_as_advanced(IGSIO_USE_GPU)

if(IGSIO_USE_GPU)
  FIND_PACKAGE(OpenCL REQUIRED)
endif()

#-----------------------------------------------------------------------------
# Run superbuild script instead of library script
IF (${CMAKE_VERSION} VERSION_GREATER 3.4)
  OPTION(IGSIO_SUPERBUILD "Build IGSIO as Superbuild." OFF)
ENDIF()

IF(IGSIO_SUPERBUILD)
  INCLUDE(SuperBuild/Superbuild.cmake)
ELSE()

  # --------------------------------------------------------------------------
  # Configure include file
  CONFIGURE_FILE(
    ${CMAKE_CURRENT_SOURCE_DIR}/igsioConfigure.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/igsioConfigure.h
    )

  ADD_SUBDIRECTORY(Source/Codecs)
  ADD_SUBDIRECTORY(Source/IGSIOCommon)
  IF(IGSIO_BUILD_SEQUENCEIO)
    ADD_SUBDIRECTORY(Source/SequenceIO)
  ENDIF()
  IF (IGSIO_BUILD_VOLUMERECONSTRUCTION)
    ADD_SUBDIRECTORY(Source/VolumeReconstruction)
  ENDIF()

  #-----------------------------------------------------------------------------
  # Export all targets at once from the build tree in their final configuration.
  get_property(_igsio_targets GLOBAL PROPERTY IGSIO_TARGETS)
  if (_igsio_targets)
    list(REMOVE_DUPLICATES _igsio_targets)
    EXPORT(TARGETS ${_igsio_targets} FILE ${IGSIO_BINARY_DIR}/IGSIOTargets.cmake)
  endif()
  unset(_igsio_targets)

  # Create an IGSIOConfig.cmake and IGSIOConfig Version.cmake file for the use from the build tree
  #-----------------------------------------------------------------------------
  INCLUDE(CMakePackageConfigHelpers)
  write_basic_package_version_file(
    "${CMAKE_CURRENT_BINARY_DIR}/IGSIOConfigVersion.cmake"
    VERSION ${IGSIO_VERSION}
    COMPATIBILITY ExactVersion)
  LIST(APPEND IGSIO_INSTALL_CMAKE_FILES} "${CMAKE_CURRENT_BINARY_DIR}/IGSIOConfigVersion.cmake")
  INCLUDE(${CMAKE_SOURCE_DIR}/CMake/GenerateIGSIOConfig.cmake)

ENDIF()
