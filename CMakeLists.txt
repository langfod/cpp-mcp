cmake_minimum_required(VERSION 3.10)
project(MCP VERSION 2025.03.26 LANGUAGES CXX)

set(CMAKE_WARN_UNUSED_CLI YES)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if (NOT XCODE AND NOT MSVC AND NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

option(BUILD_SHARED_LIBS "build shared libraries" ${BUILD_SHARED_LIBS_DEFAULT})

if (WIN32)
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)
endif()

if (MSVC)
    add_compile_options("$<$<COMPILE_LANGUAGE:C>:/utf-8>")
    add_compile_options("$<$<COMPILE_LANGUAGE:CXX>:/utf-8>")
    add_compile_options("$<$<COMPILE_LANGUAGE:C>:/bigobj>")
    add_compile_options("$<$<COMPILE_LANGUAGE:CXX>:/bigobj>")
endif()

# Set C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find required packages
find_package(Threads REQUIRED)

# Third-party header-only dependencies: nlohmann_json and cpp-httplib.
# An already-defined target (parent project) or an installed package (e.g. vcpkg) is preferred;
# otherwise the copies vendored under common/ are used.
option(MCP_VENDORED_JSON "Always use the vendored nlohmann_json" OFF)
option(MCP_VENDORED_HTTPLIB "Always use the vendored cpp-httplib" OFF)

if(NOT MCP_VENDORED_JSON AND NOT TARGET nlohmann_json::nlohmann_json)
    find_package(nlohmann_json CONFIG QUIET)
endif()
if(TARGET nlohmann_json::nlohmann_json)
    message(STATUS "MCP: using external nlohmann_json")
else()
    message(STATUS "MCP: using vendored nlohmann_json")
    add_library(mcp_vendored_json INTERFACE)
    add_library(nlohmann_json::nlohmann_json ALIAS mcp_vendored_json)
    target_include_directories(mcp_vendored_json INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/common/json)
endif()

if(NOT MCP_VENDORED_HTTPLIB AND NOT TARGET httplib::httplib)
    find_package(httplib CONFIG QUIET)
endif()
if(TARGET httplib::httplib)
    message(STATUS "MCP: using external cpp-httplib")
else()
    message(STATUS "MCP: using vendored cpp-httplib")
    add_library(mcp_vendored_httplib INTERFACE)
    add_library(httplib::httplib ALIAS mcp_vendored_httplib)
    target_include_directories(mcp_vendored_httplib INTERFACE ${CMAKE_CURRENT_SOURCE_DIR}/common/httplib)
endif()

option(MCP_SSL "Enable SSL support" OFF)
set(MCP_MAX_SESSIONS 10 CACHE STRING "Maximum concurrent MCP sessions (0 = unlimited)")
set(MCP_SESSION_TIMEOUT 30 CACHE STRING "Inactive session timeout in seconds (0 = disabled)")

if(MCP_SSL)
    find_package(OpenSSL 3.0.0 COMPONENTS Crypto SSL REQUIRED)
    include_directories(${OPENSSL_INCLUDE_DIR}) # before including cpp-httplib to avoid OpenSSL with lower version included 
    message(STATUS "OpenSSL include directory: ${OPENSSL_INCLUDE_DIR}")
    add_compile_definitions(MCP_SSL CPPHTTPLIB_OPENSSL_SUPPORT)
endif()

include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)
# common/ itself only holds base64.hpp; the vendored libraries live in subdirectories so
# they can never shadow an external copy
# common/ itself only holds base64.hpp; the vendored libraries live in subdirectories so
# they can never shadow an external copy
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/common)

# Add MCP library
add_subdirectory(src)

# Add examples
add_subdirectory(examples)

# Add test directory
option(MCP_BUILD_TESTS "Build the tests" OFF)
if(MCP_BUILD_TESTS)
    enable_testing()
    add_subdirectory(test)
endif()
