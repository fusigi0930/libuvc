cmake_minimum_required(VERSION 3.23)
cmake_policy(SET CMP0042 NEW)
project(libuvc
  VERSION 0.1.0
  LANGUAGES C
  DESCRIPTION "A cross-platform library for USB video devices"
  HOMEPAGE_URL "https://github.com/pupil-labs/libuvc"
)
cmake_policy(SET CMP0054 NEW)

if(NOT CMAKE_BUILD_TYPE)
  message(STATUS "No build type selected, default to Release")
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "" FORCE)
endif()

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBUSB REQUIRED IMPORTED_TARGET libusb-1.0)

if(WIN32)
  # winmm is necessary for __imp_timeGetTime
  #add_definitions("-D_TIMESPEC_DEFINED")
  message("ignore validate on win32")
else()
  find_package(PkgConfig)
  pkg_check_modules(LIBUSB REQUIRED libusb-1.0)
  message(STATUS "    " ${LIBUSB_LIBRARY_DIRS})
  SET(CMAKE_C_FLAGS_DEBUG "-g  -DUVC_DEBUGGING")
endif()

include(GNUInstallDirs)

# "Installation directory for CMake files")
set(SOURCES
${CMAKE_CURRENT_LIST_DIR}/src/ctrl.c
${CMAKE_CURRENT_LIST_DIR}/src/ctrl-gen.c
${CMAKE_CURRENT_LIST_DIR}/src/device.c
${CMAKE_CURRENT_LIST_DIR}/src/diag.c
${CMAKE_CURRENT_LIST_DIR}/src/frame.c
${CMAKE_CURRENT_LIST_DIR}/src/init.c
${CMAKE_CURRENT_LIST_DIR}/src/stream.c
${CMAKE_CURRENT_LIST_DIR}/src/misc.c
)
add_library(uvc_static STATIC ${SOURCES})
#add_library(PLLibUVC::uvc STATIC ALIAS uvc)

set_target_properties(
  uvc_static PROPERTIES
  OUTPUT_NAME uvc
)

add_library(LibUVC::UVCStatic ALIAS uvc_static)

target_include_directories(
  uvc_static
  PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/include>
  $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>

  # Unix vars
  ${LIBUSB_INCLUDE_DIRS}
  ${PTHREAD_INCLUDE_DIRS}

  # Windows vars
  ${PTHREADS_WIN_INCLUDE_DIR}
  ${LIBUSB_WIN_INCLUDE_DIR}
)

target_sources(uvc_static
  PUBLIC
  FILE_SET HEADERS
  BASE_DIRS ${CMAKE_CURRENT_SOURCE_DIR}/include
  FILES
  ${CMAKE_CURRENT_SOURCE_DIR}/include/libuvc/libuvc.h
)

target_compile_features(uvc_static PUBLIC c_std_17)

target_link_directories(
  uvc_static
  PUBLIC
  ${LIBUSB_LIBRARY_DIRS}
  ${PTHREAD_LIBRARY_DIRS}
)

if(UNIX AND NOT APPLE)
  message(STATUS "Adding extra -pthread c flag")
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -pthread")
endif()

target_link_libraries(
  uvc_static
  PUBLIC

  # Unix vars
  ${LIBUSB_LIBRARIES}
  ${PTHREAD_LIBRARIES}

  # Windows vars
  ${PTHREADS_WIN_IMPORT_LIB_PATH}
  ${LIBUSB_WIN_IMPORT_LIB_PATH}
)

install(TARGETS uvc_static
  EXPORT libuvcTargets
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
  FILE_SET HEADERS
)

install(EXPORT libuvcTargets
  FILE libuvcTargets.cmake
  NAMESPACE LibUVC::
  DESTINATION ${CMAKE_INSTALL_CMAKEDIR}
)

install(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/libuvcConfig.cmake
  ${CMAKE_CURRENT_BINARY_DIR}/libuvcConfigVersion.cmake
  DESTINATION ${CMAKE_INSTALL_CMAKEDIR}
)


include(CMakePackageConfigHelpers)

set(LIBRARY_DIRS ${CMAKE_INSTALL_LIBDIR})

# generate the config file that is includes the exports
configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
  "${CMAKE_CURRENT_BINARY_DIR}/libuvcConfig.cmake"
  INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/libuvc
  PATH_VARS LIBRARY_DIRS
)

write_basic_package_version_file(
  libuvcConfigVersion.cmake
  VERSION ${libuvc_VERSION}
  COMPATIBILITY SameMajorVersion
)
