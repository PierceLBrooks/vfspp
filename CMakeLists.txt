cmake_minimum_required(VERSION 2.8)
include(ExternalProject)
set_property (GLOBAL PROPERTY USE_FOLDERS ON)

project(vfspp)
set(VERSION "1.0.0")

if (NOT DEFINED PLATFORM)
    message (FATAL_ERROR "Please specify the target platform: Windows, Linux, Android, macOS, iOS, tvOS, watchOS. -DPLATFORM=macOS")
endif()

set(INSTALL_BIN_DIR "${CMAKE_INSTALL_PREFIX}/bin" CACHE PATH "Installation directory for executables")
set(INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib" CACHE PATH "Installation directory for libraries")
set(INSTALL_INC_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE PATH "Installation directory for headers")
set(INSTALL_MAN_DIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE PATH "Installation directory for manual pages")
set(INSTALL_PKGCONFIG_DIR "${CMAKE_INSTALL_PREFIX}/share/pkgconfig" CACHE PATH "Installation directory for pkgconfig (.pc) files")

if(${PLATFORM} STREQUAL "Windows")
    set(LIB_PREFIX "")
    set(LIB_SUFFIX ".lib")
else()
    set(LIB_PREFIX "lib")
    set(LIB_SUFFIX ".a")
endif()

macro (link_library TARGET_NAME LIB_NAME)
    target_link_libraries(${TARGET_NAME} ${CMAKE_INSTALL_PREFIX}/${LIB_NAME}/lib/${LIB_PREFIX}${LIB_NAME}${LIB_SUFFIX})
endmacro()

#============================================================================
# zlib
#============================================================================

if(UNIX)
    set(ZLIB_NAME z)
else()
    set(ZLIB_NAME zlibstatic)
endif()
set(ZLIB_DIR ${CMAKE_SOURCE_DIR}/external/zlib)

if (${PLATFORM} STREQUAL "macOS" OR
    ${PLATFORM} STREQUAL "iOS" OR
    ${PLATFORM} STREQUAL "tvOS" OR
    ${PLATFORM} STREQUAL "watchOS")
    set(ZLIB_C_FLAGS " -Wno-error-implicit-function-declaration")
endif()

add_subdirectory(external)

#============================================================================
# minizip
#============================================================================

set(MINIZIP_DIR ${ZLIB_DIR}/contrib/minizip)

set(MINIZIP_HDRS
    ${MINIZIP_DIR}/crypt.h
    ${MINIZIP_DIR}/ioapi.h
    ${MINIZIP_DIR}/mztools.h
    ${MINIZIP_DIR}/unzip.h
    ${MINIZIP_DIR}/zip.h
)

set(MINIZIP_SRCS
    ${MINIZIP_DIR}/ioapi.c
    ${MINIZIP_DIR}/mztools.c
    ${MINIZIP_DIR}/unzip.c
    ${MINIZIP_DIR}/zip.c
)

if(WIN32)
    set(MINIZIP_HDRS ${MINIZIP_HDRS} ${MINIZIP_DIR}/iowin32.h)
    set(MINIZIP_SRCS ${MINIZIP_SRCS} ${MINIZIP_DIR}/iowin32.c)
endif()

add_library(minizip ${MINIZIP_SRCS} ${MINIZIP_HDRS})
add_dependencies(minizip zlibstatic)
link_library(minizip ${ZLIB_NAME})

target_include_directories(minizip PUBLIC ${CMAKE_INSTALL_PREFIX}/${ZLIB_NAME}/include)

#============================================================================
# vfspp
#============================================================================

set(LIBVFSPP_PUBLIC_HDRS_DIR include)
set(LIBVFSPP_PRIVATE_HDRS_DIR src)
set(LIBVFSPP_SRCS_DIR src)

set(LIBVFSPP_PUBLIC_HDRS
    ${LIBVFSPP_PUBLIC_HDRS_DIR}/CFileInfo.h
    ${LIBVFSPP_PUBLIC_HDRS_DIR}/CMemoryFile.h
    ${LIBVFSPP_PUBLIC_HDRS_DIR}/CMemoryFileSystem.h
    ${LIBVFSPP_PUBLIC_HDRS_DIR}/CNativeFile.h
    ${LIBVFSPP_PUBLIC_HDRS_DIR}/CNativeFileSystem.h
    ${LIBVFSPP_PUBLIC_HDRS_DIR}/CVirtualFileSystem.h
    ${LIBVFSPP_PUBLIC_HDRS_DIR}/CZipFile.h
    ${LIBVFSPP_PUBLIC_HDRS_DIR}/CZipFileSystem.h
    ${LIBVFSPP_PUBLIC_HDRS_DIR}/VFS.h
    ${LIBVFSPP_PUBLIC_HDRS_DIR}/IFile.h
    ${LIBVFSPP_PUBLIC_HDRS_DIR}/IFileSystem.h
)

set(LIBVFSPP_PRIVATE_HDRS
    ${LIBVFSPP_PRIVATE_HDRS_DIR}/CStringUtilsVFS.h
    ${LIBVFSPP_PRIVATE_HDRS_DIR}/miniz.h
)

set(LIBVFSPP_SRCS
    ${LIBVFSPP_SRCS_DIR}/CFileInfo.cpp
    ${LIBVFSPP_SRCS_DIR}/CMemoryFile.cpp
    ${LIBVFSPP_SRCS_DIR}/CMemoryFileSystem.cpp
    ${LIBVFSPP_SRCS_DIR}/CNativeFile.cpp
    ${LIBVFSPP_SRCS_DIR}/CNativeFileSystem.cpp
    ${LIBVFSPP_SRCS_DIR}/CStringUtilsVFS.cpp
    ${LIBVFSPP_SRCS_DIR}/CVirtualFileSystem.cpp
    ${LIBVFSPP_SRCS_DIR}/CZipFile.cpp
    ${LIBVFSPP_SRCS_DIR}/CZipFileSystem.cpp
    ${LIBVFSPP_SRCS_DIR}/miniz.c
)

if (${PLATFORM} STREQUAL "macOS" OR
    ${PLATFORM} STREQUAL "iOS" OR
    ${PLATFORM} STREQUAL "tvOS" OR
    ${PLATFORM} STREQUAL "watchOS")
    set(CMAKE_MACOSX_RPATH 1)
endif()

set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++11 -stdlib=libc++ -fno-rtti -fno-exceptions -g -Wall")

add_library(${PROJECT_NAME} STATIC ${LIBVFSPP_SRCS} ${LIBVFSPP_HDRS})

target_include_directories(${PROJECT_NAME} PUBLIC ${LIBVFSPP_PUBLIC_HDRS_DIR})
target_include_directories(${PROJECT_NAME} PUBLIC ${CMAKE_INSTALL_PREFIX}/${ZLIB_NAME}/include)
target_include_directories(${PROJECT_NAME} PUBLIC ${MINIZIP_DIR})
add_dependencies(${PROJECT_NAME} minizip)
link_library(${PROJECT_NAME} minizip)

# Installation
if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL)
    install(FILES ${CMAKE_INSTALL_PREFIX}/${PROJECT_NAME}/lib/${LIB_PREFIX}${PROJECT_NAME}${LIB_SUFFIX}
            DESTINATION "${INSTALL_LIB_DIR}")
endif()
if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL)
    install(FILES ${LIBVFSPP_PUBLIC_HDRS} DESTINATION "${INSTALL_INC_DIR}")
endif()

#============================================================================
# Example binaries
#============================================================================

if(WITH_EXAMPLES)
    set(EXAMPLES_DIR examples)

    add_executable(example ${EXAMPLES_DIR}/example.cpp)
    add_dependencies(example ${PROJECT_NAME})
    link_library(example ${PROJECT_NAME})
    target_include_directories(example PUBLIC ${LIBVFSPP_PUBLIC_HDRS_DIR})
endif()
