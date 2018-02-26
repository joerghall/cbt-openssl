# MIT License
#
# Copyright (c) 2018 Joerg Hallmann
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.
#
# https://github.com/joerghall/cbt-openssl
#
cmake_minimum_required(VERSION 3.5)

if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/_submodules/cbt/cmake/setup.cmake)
    include(${CMAKE_CURRENT_SOURCE_DIR}/_submodules/cbt/cmake/setup.cmake)
else ()
    message(FATAL_ERROR "Unable to find buildtools, please check if the submodule is initialized")
endif ()

add_toolset(python2 2.7.14.49.re579c52b config=default )
add_toolset(ninja 1.8.2.11.rc3e3b700 config=default )
# add_toolset(pigz 1.8.2.7.r8fe061a4 config=default OVERRIDE_VERSION_WINDOWS=2.3.1)
#https://h1grid.com/artifactory/cbt/tools/ninja/1.8.2.11.rc3e3b700/

# Deal with build tools
#
if ("${CMAKE_GENERATOR}" STREQUAL "Ninja")
    if (BUILD_WINDOWS)
        set(CMAKE_MAKE_PROGRAM ${ninja_LOCATION}/ninja.exe CACHE FILEPATH "Make program" FORCE)
    else ()
        set(CMAKE_MAKE_PROGRAM ${ninja_LOCATION}/ninja CACHE FILEPATH "Make program" FORCE)
    endif ()
    if (NOT EXISTS ${CMAKE_MAKE_PROGRAM})
        message(FATAL_ERROR "Unable to find ninja in the expected location ${CMAKE_MAKE_PROGRAM}")
    else ()
        message(STATUS "Setting make program to ${CMAKE_MAKE_PROGRAM}")
    endif ()
endif ()

include(python2)

set(CMAKE_CXX_STANDARD 14)

project(cbt-openssl)

enable_testing()

file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")
set(CPACK_OUTPUT_FILE_PREFIX ${CMAKE_BINARY_DIR}/packages)

# Standard locations
#
set(SOURCE_UPLOAD_DIR ${CMAKE_BINARY_DIR}/upload)
set(SOURCE_DOWNLOAD_DIR ${CMAKE_BINARY_DIR}/download)
set(SOURCE_INSTALL_DIR ${CMAKE_BINARY_DIR}/install)
set(SOURCE_SRC_DIR ${CMAKE_BINARY_DIR}/src)
set(TARGET_INSTALL_DIR ${CMAKE_BINARY_DIR}/build)

# Generate version information for package
#
execute_process(COMMAND git rev-list HEAD
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    RESULT_VARIABLE VERSION_RESULT
    OUTPUT_VARIABLE VERSION_DATA
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

string(REPLACE "\n" ";" VERSION_DATA ${VERSION_DATA} )
list(LENGTH VERSION_DATA VERSION_REVISION)
list(GET VERSION_DATA 0 VERSION_HASH)
string(SUBSTRING ${VERSION_HASH} 0 8 VERSION_SHORTHASH)

# openssl specifics
#
set(PACKAGE_VERSION 1.0.2n)
set(PACKAGE_NAME openssl)
set(PACKAGE_VERSION_PACKAGE ${PACKAGE_VERSION}.${VERSION_REVISION}.r${VERSION_SHORTHASH})
set(PACKAGE_PUBLISH_URL https://h1grid.com:443/artifactory/cbt/components)

message(STATUS "Building package version ${PACKAGE_NAME}-${PACKAGE_VERSION_PACKAGE}-${OS}.tgz")
set(PACKAGE_SOURCE_SOURCE_URL https://github.com/openssl/openssl/archive/OpenSSL_1_0_2n.tar.gz)
set(PACKAGE_SOURCE_TARGET_FILE ${SOURCE_DOWNLOAD_DIR}/${PACKAGE_NAME}/OpenSSL_1_0_2n.tar.gz)

include(ProcessorCount)
ProcessorCount(N)
message(STATUS "Processor count ${N}")
if(N EQUAL 0)
    set(N 4)
endif()

add_custom_target(download_${PACKAGE_NAME}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${SOURCE_DOWNLOAD_DIR}/${PACKAGE_NAME}
    COMMAND ${CMAKE_COMMAND} -DSOURCE=${PACKAGE_SOURCE_SOURCE_URL} -DTARGET=${PACKAGE_SOURCE_TARGET_FILE} -P ${download_COMMAND}
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)

add_custom_target(unpack_${PACKAGE_NAME}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${SOURCE_SRC_DIR}
    COMMAND ${CMAKE_COMMAND} -DSOURCE=${PACKAGE_SOURCE_TARGET_FILE} -DTARGET=${SOURCE_SRC_DIR} -P ${untar_COMMAND}
    DEPENDS download_${PACKAGE_NAME}
)

if (BUILD_POSIX)
    if (BUILD_OSX)
        set(compilertriplet darwin64-x86_64-cc)
        set(TARGET_BUILD ${TARGET_INSTALL_DIR}/${PACKAGE_NAME}/${PACKAGE_VERSION_PACKAGE}/osx-x64-)
        set(TARGET_PKG ${PACKAGE_NAME}-${PACKAGE_VERSION_PACKAGE}-osx-x64-10.10)
    elseif (BUILD_LINUX)
        set(compilertriplet linux-x86_64)
        set(TARGET_BUILD ${TARGET_INSTALL_DIR}/${PACKAGE_NAME}/${PACKAGE_VERSION_PACKAGE}/linux-x64-)
        set(TARGET_PKG ${PACKAGE_NAME}-${PACKAGE_VERSION_PACKAGE}-linux-x64-)
    endif ()

    if (BUILD_TYPE STREQUAL "debug")
        set(compilertriplet debug-${compilertriplet})
        set(TARGET_BUILD ${TARGET_BUILD}dtyp0)
        set(TARGET_PKG ${TARGET_PKG}dtyp0)
    else ()
        set(TARGET_BUILD ${TARGET_BUILD}rtyp0)
        set(TARGET_PKG ${TARGET_PKG}dtyp0)
    endif()

    #Configuring for
    #    Usage: Configure [no-<cipher> ...] [enable-<cipher> ...] [experimental-<cipher> ...]
    #              [-Dxxx] [-lxxx] [-Lxxx] [-fxxx] [-Kxxx] [no-hw-xxx|no-hw] [[no-]threads] [[no-]shared] [[no-]zlib|zlib-dynamic]
    #              [no-asm] [no-dso] [no-krb5] [sctp] [386] [--prefix=DIR] [--openssldir=OPENSSLDIR] [--with-xxx[=vvv]] [--test-sanity] os/compiler[:flags]
    #    linux-x86_64
    #    debug-linux-x86_64
    # https://wiki.openssl.org/index.php/Compilation_and_Installation
    add_custom_target(build_${PACKAGE_NAME}
        COMMAND ${CMAKE_COMMAND} -E make_directory ${TARGET_INSTALL_DIR}
        COMMAND ./Configure enable-ssl2 shared no-asm enable-ssl2 --debug --prefix=${TARGET_BUILD} --openssldir=${TARGET_BUILD} ${compilertriplet}
        COMMAND make depend
        COMMAND make -j${N}
        COMMAND make install_sw
        WORKING_DIRECTORY ${SOURCE_SRC_DIR}/openssl-OpenSSL_1_0_2n
        DEPENDS unpack_${PACKAGE_NAME}
    )
elseif (BUILD_WINDOWS)

    set(compilertriplet VC-WIN64A)
    set(TARGET_BUILD ${TARGET_INSTALL_DIR}/${PACKAGE_NAME}/${PACKAGE_VERSION_PACKAGE}/windows-x64-)
    set(TARGET_PKG ${PACKAGE_NAME}-${PACKAGE_VERSION_PACKAGE}-windows-x64-)

    if (BUILD_TYPE STREQUAL "debug")
        set(compilertriplet debug-${compilertriplet})
        set(TARGET_BUILD ${TARGET_BUILD}dvc140typ0)
        set(TARGET_PKG ${TARGET_PKG}rvc140typ0)
    else ()
        set(TARGET_BUILD ${TARGET_BUILD}rvc140typ0)
        set(TARGET_PKG ${TARGET_PKG}rvc140typ0)
    endif()

    add_custom_target(build_${PACKAGE_NAME}
        COMMAND perl Configure ${compilertriplet} no-asm  enable-ssl2 -DOPENSSL_CAPIENG_DIALOG --prefix=${TARGET_BUILD} --openssldir=${TARGET_BUILD}
        COMMAND ms/do_win64a.bat
        COMMAND nmake -f ms/ntdll.mak
        COMMAND nmake -f ms/ntdll.mak test
        COMMAND nmake -f ms/ntdll.mak install
        WORKING_DIRECTORY ${SOURCE_SRC_DIR}/openssl-OpenSSL_1_0_2n
        DEPENDS unpack_${PACKAGE_NAME}
    )
endif()

add_custom_target(package_${PACKAGE_NAME} ALL
    COMMAND ${CMAKE_COMMAND} -E make_directory ${SOURCE_UPLOAD_DIR}
    COMMAND ${CMAKE_COMMAND} -DTARGET=${SOURCE_UPLOAD_DIR}/${TARGET_PKG}.tgz -DSOURCE=${TARGET_BUILD}/${PACKAGE_NAME} -P ${tar_COMMAND}
    DEPENDS build_${PACKAGE_NAME}
)

add_custom_target(publish_${PACKAGE_NAME}
    COMMAND ${python2_BIN} ${upload_py_COMMAND} --file ${SOURCE_UPLOAD_DIR}/${TARGET_PKG}.tgz --url ${PACKAGE_PUBLISH_URL}/${PACKAGE_NAME}/${PACKAGE_VERSION_PACKAGE}/${TARGET_PKG}.tgz
    COMMAND ${python2_BIN} ${upload_py_COMMAND} --file ${SOURCE_UPLOAD_DIR}/${TARGET_PKG}.tgz --url ${PACKAGE_PUBLISH_URL}/${PACKAGE_NAME}/${PACKAGE_VERSION_PACKAGE}/${TARGET_PKG}.tgz
    DEPENDS package_${PACKAGE_NAME}
)
