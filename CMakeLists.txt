#################
# Sparse2D 2017 #
#################

cmake_minimum_required(VERSION 2.6)
include(ExternalProject)
include(FindPkgConfig)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/modules/")

project(sparse2d)

  # Set build type
  if(CMAKE_BUILD_TYPE STREQUAL "")
    set(CMAKE_BUILD_TYPE "Release")
  endif()
  message(STATUS "CMake Build Type: ${CMAKE_BUILD_TYPE}")

  # Set the default installation path
  if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    set(CMAKE_INSTALL_PREFIX "${CMAKE_CURRENT_BINARY_DIR}/sparse2d")
  endif()
  message(STATUS "Installing to: ${CMAKE_INSTALL_PREFIX}")

  # Locate CFITSIO using pkg-config or use command line arguments to configure
  # CFITSIO
  if(CFITSIO_LIBRARIES STREQUAL "" OR  CFITSIO_LIBRARY_DIRS STREQUAL "" OR
      CFITSIO_INCLUDE_DIRS STREQUAL "")
    pkg_check_modules(CFITSIO REQUIRED cfitsio)
  else()
    message(STATUS "Use manually configured cfitsio")
    message(STATUS "  includes: ${CFITSIO_INCLUDE_DIRS}")
    message(STATUS "  libs: ${CFITSIO_LIBRARY_DIRS}")
    message(STATUS "  flags: ${CFITSIO_LIBRARIES}")
  endif()
  include_directories(${CFITSIO_INCLUDE_DIRS})
  link_directories(${CFITSIO_LIBRARY_DIRS})

  # Build location for external modules
  set(MODULE_BUILD_DIR "${CMAKE_CURRENT_BINARY_DIR}/module_build/")

  # Optional build of FFTW3
  option(USE_FFTW "Use FFTW for Fourier transformation" OFF)
  include(BuildFFTW3)

  # Optional build of NFFT
  option(COMPILE_NFFT "Get and compile NFFT" OFF)
  include(BuildNFFT)

  # Set compilation flags Mac OSX or any other system
  if(APPLE)

    set(CMAKE_CXX_FLAGS_RELEASE "-DMACOS -DNO_DISP_IO -w -g0 -O2 -fPIC \
        -fomit-frame-pointer -fopenmp -Wno-write-strings -DNDEBUG \
        ${FFTW_FLAGS}"
    )

    set(CMAKE_CXX_FLAGS_DEBUG "-DMACOS -DNO_DISP_IO -w -g0 -fPIC -fopenmp \
        -Wno-write-strings ${FFTW_CXX_FLAGS}"
    )

    message(STATUS "Using CXX Flags for Mac OSX: ${CMAKE_SYSTEM}")

  else(APPLE)

    set(CMAKE_CXX_FLAGS_RELEASE "-DNO_DISP_IO -g0 -O2 -fPIC \
        -fomit-frame-pointer -fopenmp -Wno-write-strings -DNDEBUG \
        ${FFTW_FLAGS}"
    )

    set(CMAKE_CXX_FLAGS_DEBUG "-DNO_DISP_IO -g -fPIC -fopenmp \
        -Wno-write-strings ${FFTW_CXX_FLAGS}"
    )

    message(STATUS "Using CXX Flags for ${CMAKE_SYSTEM}")

  endif(APPLE)

  # Build tools library
  FILE(GLOB src_lib1  "${PROJECT_SOURCE_DIR}/src/libtools/*.cc")
  include_directories("${PROJECT_SOURCE_DIR}/src/libtools")
  add_library(tools STATIC ${src_lib1})
  target_link_libraries(tools ${CFITSIO_LIBRARIES} ${FFTW_LD_FLAGS})
  if(USE_FFTW)
    add_dependencies(tools fftw3)
  endif(USE_FFTW)

  # Build sparse1d library
  FILE(GLOB src_lib2  "${PROJECT_SOURCE_DIR}/src/libsparse1d/*.cc")
  include_directories("${PROJECT_SOURCE_DIR}/src/libsparse1d")
  add_library(sparse1d STATIC ${src_lib2})
  target_link_libraries(sparse1d ${CFITSIO_LIBRARIES} ${FFTW_LD_FLAGS})
  if(USE_FFTW)
    add_dependencies(sparse1d fftw3)
  endif(USE_FFTW)

  # Build sparse2d library
  FILE(GLOB src_lib2  "${PROJECT_SOURCE_DIR}/src/libsparse2d/*.cc")
  include_directories("${PROJECT_SOURCE_DIR}/src/libsparse2d")
  add_library(sparse2d STATIC ${src_lib2})
  target_link_libraries(sparse2d ${CFITSIO_LIBRARIES} ${FFTW_LD_FLAGS})
  if(USE_FFTW)
    add_dependencies(sparse2d fftw3)
  endif(USE_FFTW)

  # Build mga2d library
  FILE(GLOB src_mgalib2  "${PROJECT_SOURCE_DIR}/src/libmga2d/*.cc")
  include_directories("${PROJECT_SOURCE_DIR}/src/libmga2d")
  add_library(mga2d STATIC ${src_mgalib2})
  target_link_libraries(mga2d ${CFITSIO_LIBRARIES} ${FFTW_LD_FLAGS})
  if(USE_FFTW)
    add_dependencies(mga2d fftw3)
  endif(USE_FFTW)

  # Compile and link executables
  set(BINMR2D mr_transform mr_recons mr_filter mr_deconv mr_info cur_contrast
      cur_deconv cur_filter cur_stat cur_trans)

  foreach(program ${BINMR2D})
    add_executable(${program} ${PROJECT_SOURCE_DIR}/src/${program}.cc)
    target_link_libraries(${program} mga2d sparse2d sparse1d tools)
  endforeach(program)

  # Install headers
  FILE(GLOB inc_lib  "${PROJECT_SOURCE_DIR}/src/libsparse1d/*.h")
  INSTALL(FILES ${inc_lib} DESTINATION include)
  FILE(GLOB inc_lib  "${PROJECT_SOURCE_DIR}/src/libsparse2d/*.h")
  INSTALL(FILES ${inc_lib} DESTINATION include)
  FILE(GLOB inc_lib  "${PROJECT_SOURCE_DIR}/src/libmga2d/*.h")
  INSTALL(FILES ${inc_lib} DESTINATION include)
  FILE(GLOB inc_lib  "${PROJECT_SOURCE_DIR}/src/libtools/*.h")
  INSTALL(FILES ${inc_lib} DESTINATION include)

  # Install library
  INSTALL(TARGETS sparse1d sparse2d mga2d tools DESTINATION lib)

  # install  executables
  INSTALL(TARGETS ${BINMR2D} DESTINATION bin)

  # ---------- #
  # UNIT TESTS #
  # -----------#

  # Add unit tests
  enable_testing()
  set(UNIT_TESTS mr_transform_test example)

  foreach(program ${UNIT_TESTS})
    add_executable(${program} ${PROJECT_SOURCE_DIR}/tests/${program}.cpp)
    target_link_libraries(${program} mga2d sparse2d sparse1d tools)
    add_test(${program} ${program})
  endforeach(program)
