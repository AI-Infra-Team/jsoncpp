# vim: et ts=4 sts=4 sw=4 tw=0

# ==== Define cmake build policies that affect compilation and linkage default behaviors
#
# Set the JSONCPP_NEWEST_VALIDATED_POLICIES_VERSION string to the newest cmake version
# policies that provide successful builds. By setting JSONCPP_NEWEST_VALIDATED_POLICIES_VERSION
# to a value greater than the oldest policies, all policies between
# JSONCPP_OLDEST_VALIDATED_POLICIES_VERSION and CMAKE_VERSION (used for this build)
# are set to their NEW behavior, thereby suppressing policy warnings related to policies
# between the JSONCPP_OLDEST_VALIDATED_POLICIES_VERSION and CMAKE_VERSION.
#
# CMake versions greater than the JSONCPP_NEWEST_VALIDATED_POLICIES_VERSION policies will
# continue to generate policy warnings "CMake Warning (dev)...Policy CMP0XXX is not set:"
#
set(JSONCPP_OLDEST_VALIDATED_POLICIES_VERSION "3.8.0")
set(JSONCPP_NEWEST_VALIDATED_POLICIES_VERSION "3.13.2")
cmake_minimum_required(VERSION ${JSONCPP_OLDEST_VALIDATED_POLICIES_VERSION})
if("${CMAKE_VERSION}" VERSION_LESS "${JSONCPP_NEWEST_VALIDATED_POLICIES_VERSION}")
    #Set and use the newest available cmake policies that are validated to work
    set(JSONCPP_CMAKE_POLICY_VERSION "${CMAKE_VERSION}")
else()
    set(JSONCPP_CMAKE_POLICY_VERSION "${JSONCPP_NEWEST_VALIDATED_POLICIES_VERSION}")
endif()
cmake_policy(VERSION ${JSONCPP_CMAKE_POLICY_VERSION})
if(POLICY CMP0091)
    cmake_policy(SET CMP0091 NEW)
endif()
#
# Now enumerate specific policies newer than JSONCPP_NEWEST_VALIDATED_POLICIES_VERSION
# that may need to be individually set to NEW/OLD
#
foreach(pnew "") # Currently Empty
    if(POLICY ${pnew})
        cmake_policy(SET ${pnew} NEW)
    endif()
endforeach()
foreach(pold "") # Currently Empty
    if(POLICY ${pold})
        cmake_policy(SET ${pold} OLD)
    endif()
endforeach()

# Build the library with C++11 standard support, independent from other including
# software which may use a different CXX_STANDARD or CMAKE_CXX_STANDARD.
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Ensure that CMAKE_BUILD_TYPE has a value specified for single configuration generators.
if(NOT DEFINED CMAKE_BUILD_TYPE AND NOT DEFINED CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING
        "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel Coverage.")
endif()

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_CURRENT_SOURCE_DIR}/cmake")

project(jsoncpp
        # Note: version must be updated in four places when doing a release. This
        # annoying process ensures that amalgamate, CMake, and meson all report the
        # correct version.
        # 1. ./meson.build
        # 2. ./include/json/version.h
        # 3. ./CMakeLists.txt
        # 4. ./MODULE.bazel
        # IMPORTANT: also update the PROJECT_SOVERSION!!
        VERSION 1.9.7 # <major>[.<minor>[.<patch>[.<tweak>]]]
        LANGUAGES CXX)

message(STATUS "JsonCpp Version: ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}")
set(PROJECT_SOVERSION 27)

include(${CMAKE_CURRENT_SOURCE_DIR}/include/PreventInSourceBuilds.cmake)
include(${CMAKE_CURRENT_SOURCE_DIR}/include/PreventInBuildInstalls.cmake)

option(JSONCPP_WITH_TESTS "Compile and (for jsoncpp_check) run JsonCpp test executables" ON)
option(JSONCPP_WITH_POST_BUILD_UNITTEST "Automatically run unit-tests as a post build step" ON)
option(JSONCPP_WITH_WARNING_AS_ERROR "Force compilation to fail if a warning occurs" OFF)
option(JSONCPP_WITH_STRICT_ISO "Issue all the warnings demanded by strict ISO C and ISO C++" ON)
option(JSONCPP_WITH_PKGCONFIG_SUPPORT "Generate and install .pc files" ON)
option(JSONCPP_WITH_CMAKE_PACKAGE "Generate and install cmake package files" ON)
option(JSONCPP_WITH_EXAMPLE "Compile JsonCpp example" OFF)
option(JSONCPP_STATIC_WINDOWS_RUNTIME "Use static (MT/MTd) Windows runtime" OFF)
option(BUILD_SHARED_LIBS "Build jsoncpp_lib as a shared library." ON)
option(BUILD_STATIC_LIBS "Build jsoncpp_lib as a static library." ON)
option(BUILD_OBJECT_LIBS "Build jsoncpp_lib as a object library." ON)

# Adhere to GNU filesystem layout conventions
include(GNUInstallDirs)

if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
    set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib" CACHE PATH "Archive output dir.")
    set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib" CACHE PATH "Library output dir.")
    set(CMAKE_PDB_OUTPUT_DIRECTORY     "${CMAKE_BINARY_DIR}/bin" CACHE PATH "PDB (MSVC debug symbol)output dir.")
    set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin" CACHE PATH "Executable/dll output dir.")
endif()

include(CheckFunctionExists)
check_function_exists(memset_s HAVE_MEMSET_S)
if(HAVE_MEMSET_S)
    add_definitions("-DHAVE_MEMSET_S=1")
endif()

if(JSONCPP_USE_SECURE_MEMORY)
    add_definitions("-DJSONCPP_USE_SECURE_MEMORY=1")
endif()

configure_file("${PROJECT_SOURCE_DIR}/version.in"
    "${PROJECT_BINARY_DIR}/version"
    NEWLINE_STYLE UNIX)

macro(use_compilation_warning_as_error)
    if(MSVC)
        # Only enabled in debug because some old versions of VS STL generate
        # warnings when compiled in release configuration.
        add_compile_options($<$<CONFIG:Debug>:/WX>)
    elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        add_compile_options(-Werror)
        if(JSONCPP_WITH_STRICT_ISO)
            add_compile_options(-pedantic-errors)
        endif()
    endif()
endmacro()

# Include our configuration header
include_directories(${jsoncpp_SOURCE_DIR}/include)

if(MSVC)
    # Only enabled in debug because some old versions of VS STL generate
    # unreachable code warning when compiled in release configuration.
    add_compile_options($<$<CONFIG:Debug>:/W4>)
    if (JSONCPP_STATIC_WINDOWS_RUNTIME)
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    endif()
endif()

if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    # using regular Clang or AppleClang
    add_compile_options(-Wall -Wconversion -Wshadow)

    if(JSONCPP_WITH_WARNING_AS_ERROR)
        add_compile_options(-Werror=conversion -Werror=sign-compare)
    endif()
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    # using GCC
    add_compile_options(-Wall -Wconversion -Wshadow -Wextra)
    # not yet ready for -Wsign-conversion

    if(JSONCPP_WITH_STRICT_ISO)
        add_compile_options(-Wpedantic)
    endif()
    if(JSONCPP_WITH_WARNING_AS_ERROR)
        add_compile_options(-Werror=conversion)
    endif()
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Intel")
    # using Intel compiler
    add_compile_options(-Wall -Wconversion -Wshadow -Wextra)

    if(JSONCPP_WITH_WARNING_AS_ERROR)
        add_compile_options(-Werror=conversion)
    elseif(JSONCPP_WITH_STRICT_ISO)
        add_compile_options(-Wpedantic)
    endif()
endif()

if(JSONCPP_WITH_WARNING_AS_ERROR)
    use_compilation_warning_as_error()
endif()

if(JSONCPP_WITH_PKGCONFIG_SUPPORT)
    include(JoinPaths)

    join_paths(libdir_for_pc_file "\${exec_prefix}" "${CMAKE_INSTALL_LIBDIR}")
    join_paths(includedir_for_pc_file "\${prefix}" "${CMAKE_INSTALL_INCLUDEDIR}")

    configure_file(
        "pkg-config/jsoncpp.pc.in"
        "pkg-config/jsoncpp.pc"
        @ONLY)
    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/pkg-config/jsoncpp.pc"
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
endif()

if(JSONCPP_WITH_CMAKE_PACKAGE)
    include(CMakePackageConfigHelpers)
    install(EXPORT jsoncpp
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/jsoncpp
        FILE        jsoncpp-targets.cmake)
    configure_package_config_file(jsoncppConfig.cmake.in ${CMAKE_CURRENT_BINARY_DIR}/jsoncppConfig.cmake
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/jsoncpp)

    write_basic_package_version_file("${CMAKE_CURRENT_BINARY_DIR}/jsoncppConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion)
    install(FILES
        ${CMAKE_CURRENT_BINARY_DIR}/jsoncppConfigVersion.cmake ${CMAKE_CURRENT_BINARY_DIR}/jsoncppConfig.cmake
        ${CMAKE_CURRENT_SOURCE_DIR}/jsoncpp-namespaced-targets.cmake
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/jsoncpp)
endif()

if(JSONCPP_WITH_TESTS)
    enable_testing()
    include(CTest)
endif()

# Build the different applications
add_subdirectory(src)

#install the includes
add_subdirectory(include)

#install the example
if(JSONCPP_WITH_EXAMPLE)
    add_subdirectory(example)
endif()
