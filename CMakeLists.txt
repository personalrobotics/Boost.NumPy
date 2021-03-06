cmake_minimum_required(VERSION 2.8.12)

project( boost.numpy )

set(BUILD_TESTS ON CACHE BOOL "Build Boost.NumPy Tests")
set(BUILD_EXAMPLES ON CACHE BOOL "Build Boost.NumPy Examples")

# put our local cmake find scripts at the beginning of the cmake
# module search path
set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/libs/numpy/cmake ${CMAKE_MODULE_PATH})

IF (NOT DEFINED LIB_SUFFIX)
	EXECUTE_PROCESS(
		COMMAND uname -m
		COMMAND tr -d '\n'
		OUTPUT_VARIABLE ARCH
	)
	message(STATUS "Detected architecture '"  ${ARCH}  "'")
	IF (ARCH STREQUAL x86_64)
		SET(LIB_SUFFIX 64)
	ENDIF (ARCH STREQUAL x86_64)
ENDIF (NOT DEFINED LIB_SUFFIX)

# configure output folders so artifacts are built into a single set of
# top-level folders rather than the default cmake build structure that
# matches the source tree.
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib 
    CACHE PATH "Output directory for static libraries.")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib
    CACHE PATH "Output directory for shared libraries.")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
    CACHE PATH "Output directory for executables and DLL's.")

if(NOT BOOST_NUMPY_PYTHON_VERSION)
  set(BOOST_NUMPY_PYTHON_VERSION 3.4 CACHE STRING "Choose the target Python version (e.g., 3.4, 2.7)" FORCE)
endif()

# Find required python packages.
find_package(PythonInterp ${BOOST_NUMPY_PYTHON_VERSION} REQUIRED)
find_package(PythonLibs ${BOOST_NUMPY_PYTHON_VERSION} REQUIRED)
find_package(NumPy ${BOOST_NUMPY_PYTHON_VERSION} REQUIRED)

# Find Boost.
# 
# set(Boost_USE_STATIC_LIBS ON)
# set(Boost_USE_MULTITHREADED ON) 
# set(Boost_USE_STATIC_RUNTIME ON)
set(CMAKE_MACOSX_RPATH 1)

if(${PYTHON_VERSION_STRING} GREATER 3.0)
  message(STATUS "Using Python3")
  find_package(Boost COMPONENTS python-py${PYTHON_VERSION_MAJOR}${PYTHON_VERSION_MINOR})
  if (NOT Boost_FOUND)
    find_package(Boost COMPONENTS python3 REQUIRED)
  endif()
else()
  message(STATUS "Using Python2")
  find_package(Boost COMPONENTS python REQUIRED)
endif()

message( STATUS "Boost Paths:")
message( STATUS "Include  : " ${Boost_INCLUDE_DIRS})
message( STATUS "Libraries: " ${Boost_LIBRARIES})

if(MSVC)
  # With Visual Studio remove the automatic linking
  # feature
  add_definitions( -DBOOST_ALL_NO_LIB )
endif()

if(Boost_USE_STATIC_LIBS)
  # if the user specified he wants to link to static libraries
  # force linking to static boost python
  add_definitions( -DBOOST_PYTHON_STATIC_LIB )
endif()
# In some cases, you may need to explicitly specify that a dynamic Boost is used; if so use:
# add_definitions( -DBOOST_ALL_DYN_LINK )

# We build Boost.NumPy as different package per different Python major version
# (2 or 3) to enable to install two Boost.NumPy packages of Python 2 and 3.
if(${PYTHON_VERSION_STRING} GREATER 3.0)
  set(boost_numpy_target boost_python3_numpy)
  set(boost_numpy_package BoostPython3Numpy)
else()
  set(boost_numpy_target boost_python2_numpy)
  set(boost_numpy_package BoostPython2Numpy)
endif()

# Variable controlling whether the ${boost_numpy_target} is a shared or static library.
if (WIN32)
  set(LIBRARY_TYPE STATIC CACHE STRING "type of library to make for ${boost_numpy_target}")
else()
  set(LIBRARY_TYPE SHARED CACHE STRING "type of library to make for ${boost_numpy_target}")
endif()

# Variable controlling building of documentation.
set(BUILD_DOCS OFF CACHE BOOL "Build Boost.NumPy Documentation")

# Logic for configuring documentation generation.
if(BUILD_DOCS)
  # Find sphinx executable.
  execute_process(COMMAND ${PYTHON_EXECUTABLE} -c "import sphinx; print sphinx.__version__"
    RESULT_VARIABLE SPHINX_PROCESS
    OUTPUT_VARIABLE SPHINX_VERSION
    OUTPUT_STRIP_TRAILING_WHITESPACE)
  
  set(HAVE_SPHINX 0)
  if(SPHINX_PROCESS EQUAL 0)
    find_program(SPHINX_BUILD sphinx-build)
    if(SPHINX_BUILD)
      set(HAVE_SPHINX 1)
      message(STATUS "    Found Sphinx ${SPHINX_VERSION}: ${SPHINX_BUILD}")
    else()
      # Sphinx is required to generate documention, so it is an error
      # if we cannot find it.
      message(SEND_ERROR "must be able to find sphinx-build when BUILD_DOCS is enabled")
    endif()
  endif()
  
  # Find pdflatex, which is only required for doc-pdf.
  find_package(LATEX)
  if (PDFLATEX_COMPILER)
    message( STATUS "Found PDFLATEX_COMPILER=${PDFLATEX_COMPILER}" )
  else()
    message( STATUS "Found PDFLATEX_COMPILER NOT found" )
  endif()
endif()

# Compiler definitions for non-Windows builds.
add_definitions(-fPIC)

if (BUILD_TESTS)
  # enable ctest targets
  ENABLE_TESTING()
endif()

# Turn on Visual Studio solution folders.
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# Global settings for include paths.
include_directories(
  ${PROJECT_SOURCE_DIR}
  ${PYTHON_INCLUDE_DIRS}
  ${NUMPY_INCLUDE_DIRS}
  ${Boost_INCLUDE_DIRS} 
  )

# Install headers.
install(DIRECTORY boost
    DESTINATION "include"
    FILES_MATCHING
    PATTERN "*.hpp"
    ${INSTALL_PERMISSIONS_SRC}
  )

# add submodules
ADD_SUBDIRECTORY(libs/numpy/src)
if (BUILD_EXAMPLES)
  ADD_SUBDIRECTORY(libs/numpy/example)
endif()
if (BUILD_TESTS)
  ADD_SUBDIRECTORY(libs/numpy/test)
endif()

if(BUILD_DOCS)
  add_subdirectory(libs/numpy/doc)
endif()

# Create CMake package configuration.
set(INCLUDE_INSTALL_DIR include/ CACHE PATH
    "The subdirectory relative to the install prefix where header will be installed (default prefix/include)")
set(LIB_INSTALL_DIR lib/ CACHE PATH
    "The subdirectory relative to the install prefix where libraries will be installed (default is prefix/lib)")

include(CMakePackageConfigHelpers)
configure_package_config_file(cmake/BoostNumpyConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/${boost_numpy_package}Config.cmake
    INSTALL_DESTINATION ${LIB_INSTALL_DIR}/${boost_numpy_package}/cmake
    PATH_VARS INCLUDE_INSTALL_DIR LIB_INSTALL_DIR)
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/${boost_numpy_package}ConfigVersion.cmake
    VERSION 0.24.0
    COMPATIBILITY SameMajorVersion)

install(EXPORT "${boost_numpy_target}Targets"
        FILE "${boost_numpy_package}Targets.cmake"
        DESTINATION "${LIB_INSTALL_DIR}/${boost_numpy_package}/cmake")

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/${boost_numpy_package}Config.cmake
              ${CMAKE_CURRENT_BINARY_DIR}/${boost_numpy_package}ConfigVersion.cmake
        DESTINATION ${LIB_INSTALL_DIR}/${boost_numpy_package}/cmake)

# Building a package with CPack.
set(CPACK_PACKAGE_NAME boost-numpy)
set(CPACK_PACKAGE_VERSION 24)
set(CPACK_PACKAGE_CONTACT "ndarray-dev@googlegroups.com")
set(CPACK_DEBIAN_PACKAGE_DEPENDS "python-dev, python-numpy, libboost-python-dev")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Boost.NumPy is an extension for Boost.Python that adds NumPy support.")
set(CPACK_PACKAGE_DESCRIPTION "It is intended as a replacement for the old numeric support in Boost.Python proper, which is now out-of-date and does not support C/C++ pointer access to the data.  Because it builds against the NumPy headers and hence has a dependency that Boost.Python does not, it is intended to be built as a separate library.  This is also not intended to be a high-level C++ array library; it would be more accurate to consider it a C++ NumPy API, with the intent of making the NumPy C-API available in a form that is safer and more convenient for C++ users (and particularly those using Boost.Python, of course).")
include(CPack)
