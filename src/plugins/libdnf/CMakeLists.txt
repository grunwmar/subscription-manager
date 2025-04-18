CMAKE_MINIMUM_REQUIRED (VERSION 3.20)

project(product-id C)

include(GNUInstallDirs)

set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD 11)

#ADD_COMPILE_OPTIONS(-Wall -fPIC -g -Wl,as-needed)

# Build type
if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    message(STATUS "Setting build type to 'Debug' as none was specified.")
    set(CMAKE_BUILD_TYPE Debug CACHE STRING "Choose the type of build." FORCE)
    # Set the possible values of build type for cmake-gui
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release")
endif ()

if (CMAKE_COMPILER_IS_GNUCC)
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -fPIC -Wextra -pedantic -Wno-long-long")
    if (CMAKE_BUILD_TYPE STREQUAL "Debug")
        set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ggdb -O0 --coverage")
    elseif( CMAKE_BUILD_TYPE STREQUAL "Release" )
        set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DNDEBUG -g -O3 -fno-strict-aliasing")
    endif ()
endif (CMAKE_COMPILER_IS_GNUCC)

find_package(PkgConfig REQUIRED)
pkg_check_modules(GLIB REQUIRED IMPORTED_TARGET glib-2.0>=2.44.0)
pkg_check_modules(GIO REQUIRED IMPORTED_TARGET gio-2.0>=2.54.3)
pkg_check_modules(LIBDNF REQUIRED IMPORTED_TARGET libdnf>=0.22.0)
pkg_check_modules(ZLIB REQUIRED IMPORTED_TARGET zlib>=1.2.0)
pkg_check_modules(OPENSSL REQUIRED IMPORTED_TARGET libcrypto libssl)
pkg_check_modules(JSONC REQUIRED IMPORTED_TARGET json-c)

# Copy ./test_data directory to ${PROJECT_BINARY_DIR}
file (COPY "${PROJECT_SOURCE_DIR}/test_data/corrupted_compressed_productid.pem.gz"
        DESTINATION "${PROJECT_BINARY_DIR}/test_data/")
file (COPY "${PROJECT_SOURCE_DIR}/test_data/59803427316a729fb1d67fd08e7d0c8ccd2a4a5377729b747b76345851bdba6c-productid.gz"
        DESTINATION "${PROJECT_BINARY_DIR}/test_data/")
file (COPY "${PROJECT_SOURCE_DIR}/test_data/cert_dir/71.pem"
        DESTINATION "${PROJECT_BINARY_DIR}/test_data/cert_dir/")

add_library(product-id SHARED product-id.c util.c productdb.c)

# Don't put "lib" on the front
set_target_properties(product-id PROPERTIES PREFIX "")

target_link_libraries(product-id
    PkgConfig::GLIB
    PkgConfig::GIO
    PkgConfig::LIBDNF
    PkgConfig::ZLIB
    PkgConfig::OPENSSL
    PkgConfig::JSONC
)

# Note: libdnf use "hardcoded" directory /usr/lib64/libdnf/plugins for searching
# libdnf plugins. This directory cannot be changed by any configuration file ATM.
# It can be changed only during building of libdnf using CMAKE_INSTALL_FULL_LIBDIR
install(TARGETS product-id LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}/libdnf/plugins)

enable_testing()

# Testing of productdb
add_executable(test-productdb test-productdb.c)
target_link_libraries(test-productdb product-id)
add_test(productdb test-productdb)

# Testing of product-id
add_executable(test-product-id test-product-id.c)
target_link_libraries(test-product-id product-id)
add_test(product-id test-product-id)
