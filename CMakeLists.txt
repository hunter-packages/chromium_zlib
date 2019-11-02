cmake_minimum_required(VERSION 3.2)
project(ZLIB)

set(cpu_is_x64 FALSE)
set(cpu_is_arm64 FALSE)

set(is_clang FALSE)
if("${CMAKE_C_COMPILER_ID}" STREQUAL "Clang")
  set(is_clang TRUE)
endif()

# Neon {
if(CMAKE_SYSTEM_PROCESSOR MATCHES "^arm.*")
  set(default ON)
  if("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "arm64")
    set(cpu_is_arm64 TRUE)
  endif()
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "aarch64")
  set(default ON)
  set(cpu_is_arm64 TRUE)
else()
  set(default OFF)
endif()

option(ZLIB_ARM_NEON_OPTIMIZATIONS "ARM Neon optimizations" ${default})
# }

# x86/x64 {
if(IOS)
  set(default OFF)
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "x86")
  set(default ON)
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "x86_64")
  set(default ON)
  set(cpu_is_x64 TRUE)
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "AMD64")
  set(default ON)
  set(cpu_is_x64 TRUE)
else()
  set(default OFF)
endif()

option(ZLIB_X86_X64_OPTIMIZATIONS "x86/x64 optimizations" ${default})
# }

set(sources "")

if(ZLIB_X86_X64_OPTIMIZATIONS)
  list(APPEND sources adler32_simd.c adler32_simd.h)
  if(NOT WIN32 OR is_clang)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -mssse3")
  endif()
endif()

if(ZLIB_ARM_NEON_OPTIMIZATIONS)
  list(APPEND sources adler32_simd.c adler32_simd.h)

  if(NOT IOS)
    if(ANDROID)
      # third_party/android_ndk:cpu_features
      include_directories("${CMAKE_ANDROID_NDK}/sources/android/cpufeatures")
      list(
          APPEND
          sources
          "${CMAKE_ANDROID_NDK}/sources/android/cpufeatures/cpu-features.c"
      )
    elseif(NOT WIN32 AND NOT is_clang)
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=armv8-a+crc")
    endif()

    list(APPEND sources arm_features.c arm_features.h crc32_simd.c crc32_simd.h)
  endif()
endif()

if(ZLIB_X86_X64_OPTIMIZATIONS OR ZLIB_ARM_NEON_OPTIMIZATIONS)
  list(
      APPEND
      sources
      contrib/optimizations/chunkcopy.h
      contrib/optimizations/inffast_chunk.c
      contrib/optimizations/inffast_chunk.h
      contrib/optimizations/inflate.c
  )
endif()

if(ZLIB_X86_X64_OPTIMIZATIONS)
  list(APPEND sources crc32_simd.c crc32_simd.h crc_folding.c fill_window_sse.c)
  if(NOT WIN32 OR is_clang)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -msse4.2 -mpclmul")
  endif()
else()
  list(APPEND sources simd_stub.c)
endif()

list(
    APPEND
    sources
    adler32.c
    chromeconf.h
    compress.c
    crc32.c
    crc32.h
    deflate.c
    deflate.h
    gzclose.c
    gzguts.h
    gzlib.c
    gzread.c
    gzwrite.c
    infback.c
    inffast.c
    inffast.h
    inffixed.h
    inflate.h
    inftrees.c
    inftrees.h
    trees.c
    trees.h
    uncompr.c
    x86.h
    zconf.h
    zlib.h
    zutil.c
    zutil.h
)

if(ZLIB_X86_X64_OPTIMIZATIONS)
  list(APPEND sources x86.c)
elseif(ZLIB_ARM_NEON_OPTIMIZATIONS)
  list(APPEND sources contrib/optimizations/slide_hash_neon.h)
else()
  list(APPEND sources inflate.c)
endif()

add_library(ZLIB ${sources})
target_include_directories(
    ZLIB PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}>
)

find_package(Threads REQUIRED)
target_link_libraries(ZLIB PUBLIC Threads::Threads)

target_compile_definitions(ZLIB PRIVATE ZLIB_IMPLEMENTATION)

if(ZLIB_ARM_NEON_OPTIMIZATIONS)
  target_compile_definitions(ZLIB PRIVATE ADLER32_SIMD_NEON)
endif()

if(ZLIB_X86_X64_OPTIMIZATIONS)
  target_compile_definitions(ZLIB PRIVATE ADLER32_SIMD_SSSE3)
endif()

if(ZLIB_ARM_NEON_OPTIMIZATIONS)
  if(NOT IOS)
    target_compile_definitions(ZLIB PRIVATE CRC32_ARMV8_CRC32)
    if(ANDROID)
      target_compile_definitions(ZLIB PRIVATE ARMV8_OS_ANDROID)
    elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "Linux")
      target_compile_definitions(ZLIB PRIVATE ARMV8_OS_LINUX)
    elseif(WIN32)
      target_compile_definitions(ZLIB PRIVATE ARMV8_OS_WINDOWS)
    else()
      message(FATAL_ERROR "Unsupported ARM OS")
    endif()
  endif()
endif()

if(ZLIB_X86_X64_OPTIMIZATIONS)
  target_compile_definitions(ZLIB PRIVATE INFLATE_CHUNK_SIMD_SSE2)
  if(cpu_is_x64)
    target_compile_definitions(ZLIB PRIVATE INFLATE_CHUNK_READ_64LE)
  endif()
endif()

if(ZLIB_ARM_NEON_OPTIMIZATIONS)
  target_compile_definitions(ZLIB PRIVATE INFLATE_CHUNK_SIMD_NEON)
  if(cpu_is_arm64)
    target_compile_definitions(ZLIB PRIVATE INFLATE_CHUNK_READ_64LE)
  endif()
endif()

if(ZLIB_X86_X64_OPTIMIZATIONS)
  target_compile_definitions(ZLIB PRIVATE CRC32_SIMD_SSE42_PCLMUL)
endif()

### Installation (https://github.com/forexample/package-example) {

# Introduce variables:
# * CMAKE_INSTALL_LIBDIR
# * CMAKE_INSTALL_BINDIR
# * CMAKE_INSTALL_INCLUDEDIR
include(GNUInstallDirs)

set(config_install_dir "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}")
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(TARGETS_EXPORT_NAME "${PROJECT_NAME}Targets")
set(NAMESPACE "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)

# Use:
# * TARGETS_EXPORT_NAME
# * PROJECT_NAME
configure_package_config_file(
    "cmake/template/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

install(
    FILES zlib.h zconf.h chromeconf.h
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

install(
    FILES "${project_config}"
    DESTINATION "${config_install_dir}"
)

install(
    EXPORT "${TARGETS_EXPORT_NAME}"
    NAMESPACE "${NAMESPACE}"
    DESTINATION "${config_install_dir}"
)

install(
    TARGETS ZLIB
    EXPORT "${TARGETS_EXPORT_NAME}"
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    INCLUDES DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}"
)

### }
