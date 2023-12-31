#* Author: Wady <iswady@163.com>
#* Date:   Thu Aug 25 12:00:00 2022

cmake_minimum_required(VERSION 3.16.3)
project(Valgrind VERSION 3.21.0 LANGUAGES C CXX)

if(EXTRA_CMAKE)
    include("${EXTRA_CMAKE}")
    message(STATUS "Including extra cmake from ${EXTRA_CMAKE}")
endif()

# Used to configure generated files.
string(TOUPPER ${PROJECT_NAME} PROJECT_NAME_UPPER)
string(TOLOWER ${PROJECT_NAME} PROJECT_NAME_LOWER)

# Basic Information.
MESSAGE(STATUS "CMake version: " ${CMAKE_VERSION})
MESSAGE(STATUS "CMake system name: " ${CMAKE_SYSTEM_NAME})
MESSAGE(STATUS "CMake system processor: " ${CMAKE_SYSTEM_PROCESSOR})

# Specifying a search path for CMake modules to be loaded by the the include() or find_package() commands before checking the default modules that come with CMake.
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

# Provides install directory variables as defined by the GNU Coding Standards.
# GNU Coding Standards Reference 'https://www.gnu.org/prep/standards/html_node/Directory-Variables.html'.
include(GNUInstallDirs)

include(cmake/CMakeUtils.cmake)

set(INTERMEDIATE_DIR "${CMAKE_BINARY_DIR}/Intermediate")

set(CONFIGURE "${CMAKE_CURRENT_LIST_DIR}/configure")

set(CONFIGURE_ARCH_TXT "${INTERMEDIATE_DIR}/arch.txt")
set(CONFIGURE_LOG_FILE "${INTERMEDIATE_DIR}/config.log")
set(CONFIGURE_MAKEFILE "${INTERMEDIATE_DIR}/Makefile")

set(CONFIGURE_SUCCESS_INFO "configure: exit 0")

set(COMPILING_ARCH "${CMAKE_SYSTEM_NAME}-${CMAKE_SYSTEM_PROCESSOR}")

if(EXISTS "${CONFIGURE_ARCH_TXT}")
    file(READ "${CONFIGURE_ARCH_TXT}" ACTIVATED_ARCH)
endif()

if("${COMPILING_ARCH}" STREQUAL "${ACTIVATED_ARCH}")
    set(ENABLE_CLEAN OFF)

    message(STATUS "The same processor architecture (${COMPILING_ARCH}), checking configure result ...")

    if(EXISTS "${CONFIGURE_LOG_FILE}")
        file(STRINGS "${CONFIGURE_LOG_FILE}" ENABLE_CONFIGURE REGEX "${CONFIGURE_SUCCESS_INFO}")
        if("${ENABLE_CONFIGURE}" STREQUAL "${CONFIGURE_SUCCESS_INFO}")
            set(ENABLE_CONFIGURE OFF)
        else()
            set(ENABLE_CONFIGURE ON)
        endif()
    else()
        set(ENABLE_CONFIGURE ON)
    endif()
else()
    set(ENABLE_CLEAN ON)
    set(ENABLE_CONFIGURE ON)
    file(WRITE "${CONFIGURE_ARCH_TXT}" "${COMPILING_ARCH}")
endif()

file(GLOB SUPPORT_CLEAN "${CONFIGURE_MAKEFILE}")

utils_write_script("#!/bin/bash" TRUNCATE WRITE_ALL)

if(CMAKE_CROSSCOMPILING)
    string(REGEX REPLACE "^(.+)/.+$" "\\1" COMPILER_PATH ${CMAKE_C_COMPILER})
    utils_write_script("export PATH=\"${COMPILER_PATH}:$PATH\"" WRITE_ALL)

    string(REGEX REPLACE "^.+/(.+)-gcc$" "\\1" COMPILER_HOST ${CMAKE_C_COMPILER})

    utils_write_script("export CFLAGS=" WRITE_ALL)
    utils_write_script("export CPPFLAGS=" WRITE_ALL)
    utils_write_script("export CXXFLAGS=" WRITE_ALL)
    utils_write_script("export LDFLAGS=" WRITE_ALL)

    utils_write_script("export CC=${COMPILER_HOST}-gcc" WRITE_ALL)
    utils_write_script("export CXX=${COMPILER_HOST}-g++" WRITE_ALL)
    utils_write_script("export AR=${COMPILER_HOST}-ar" WRITE_ALL)
endif()

set(MAKE_CLEAN "${CMAKE_MAKE_PROGRAM} -C ${INTERMEDIATE_DIR} distclean")
set(SKIP_CLEAN "# There is nothing to be cleaned.")

if(SUPPORT_CLEAN)
    utils_write_script("${MAKE_CLEAN}" WRITE_CLEAN)
else()
    utils_write_script("${SKIP_CLEAN}" WRITE_CLEAN)
endif()

if(SUPPORT_CLEAN AND ENABLE_CLEAN)
    utils_write_script("${MAKE_CLEAN}" WRITE_COMPILE)
else()
    utils_write_script("${SKIP_CLEAN}" WRITE_COMPILE)
endif()

if(ENABLE_CONFIGURE)
    utils_write_script("cd ${INTERMEDIATE_DIR}" WRITE_COMPILE)

    set(CONFIGURE_COMMAND "${CONFIGURE}")

    list(APPEND CONFIGURE_COMMAND "--prefix=${CMAKE_INSTALL_PREFIX}")

    if(CMAKE_CROSSCOMPILING)
        if("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "armv7-a")
            list(APPEND CONFIGURE_COMMAND "--host=\"armv7-unknown-linux\"")
        else()
            message(FATAL_ERROR "Unknown system processor '${CMAKE_SYSTEM_PROCESSOR}'")
        endif()
    endif()

    string(REPLACE ";" " " CONFIGURE_COMMAND "${CONFIGURE_COMMAND}")

    utils_write_script("${CONFIGURE_COMMAND}" WRITE_COMPILE)
else()
    utils_write_script("# There is nothing to be configured." WRITE_COMPILE)
endif()

utils_write_script("${CMAKE_MAKE_PROGRAM} -C ${INTERMEDIATE_DIR}" WRITE_COMPILE)

utils_write_script("${CMAKE_MAKE_PROGRAM} -C ${INTERMEDIATE_DIR} install" WRITE_INSTALL)

add_custom_target(Customize_compile
    COMMAND "/bin/bash" "${COMPILE_SCRIPT}"
    COMMENT "Running `compile` ..."
    VERBATIM
)

add_custom_target(Customize_clean
    COMMAND "/bin/bash" "${CLEAN_SCRIPT}"
    COMMENT "Running `clean` ..."
    VERBATIM
)

add_custom_target(Customize_install
    COMMAND "/bin/bash" "${INSTALL_SCRIPT}"
    COMMENT "Running `install` ..."
    VERBATIM
)
