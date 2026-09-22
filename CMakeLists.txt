#
# project: fips-openpnp-capture
#
# Upstream builds a shared library with its own version, MSVC runtime and turbojpeg machinery, this
# replaces that with a single static fips lib built from the same per platform sources.
#

if (NOT FIPS_IMPORT)
    cmake_minimum_required(VERSION 3.21)
    project(fips-openpnp-capture)
    # include the fips main cmake file
    get_filename_component(FIPS_ROOT_DIR "../fips" ABSOLUTE)
    include("${FIPS_ROOT_DIR}/cmake/fips.cmake")

    fips_setup(PROJECT fips-openpnp-capture)
endif()

set(OPENPNP_CAPTURE_LIB_VERSION "0.0.28")

# common/libmain.cpp includes version.h unconditionally and upstream generates it from a git tag into
# the source tree, generate it into the build tree instead so the checkout stays clean
set(GITVERSION "v${OPENPNP_CAPTURE_LIB_VERSION}")
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/version.h.in ${CMAKE_CURRENT_BINARY_DIR}/version.h)

include_directories(include)
include_directories(${CMAKE_CURRENT_BINARY_DIR})

if (CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(COMPILERBITS "64 bit")
else()
    set(COMPILERBITS "32 bit")
endif()

if (CMAKE_BUILD_TYPE MATCHES Release)
    set(BUILDTYPE_STRING "release")
else()
    set(BUILDTYPE_STRING "debug")
endif()

fips_begin_lib(openpnp-capture)

    fips_dir(common)
    fips_files(
        context.cpp
        context.h
        deviceinfo.h
        libmain.cpp
        logging.cpp
        logging.h
        stream.cpp
        stream.h
    )

    if (FIPS_WINDOWS)
        # DirectShow backend, the sample grabber filter is hand rolled in samplegrabber.h because the
        # one shipped with Windows was removed. Blackmagic cards show up here as "Blackmagic WDM Capture"
        fips_dir(win)
        fips_files(
            platformcontext.cpp
            platformcontext.h
            platformdeviceinfo.h
            platformstream.cpp
            platformstream.h
            samplegrabber.h
            scopedcomptr.h
        )
        fips_libs(strmiids ole32 oleaut32)
        set(PLATFORM_STRING "Win ${COMPILERBITS}")
    elseif (FIPS_OSX)
        # AVFoundation backend, uvcctrl drives the manual exposure/focus controls over raw USB
        fips_dir(mac)
        fips_files(
            platformcontext.h
            platformcontext.mm
            platformdeviceinfo.h
            platformstream.h
            platformstream.mm
            uvcctrl.h
            uvcctrl.mm
        )
        fips_frameworks_osx(AVFoundation Foundation CoreMedia CoreVideo Accelerate IOKit)
        set(PLATFORM_STRING "OSX ${COMPILERBITS}")
    else()
        # V4L2 backend, MJPEG frames are decoded with libjpeg-turbo
        fips_dir(linux)
        fips_files(
            mjpeghelper.cpp
            mjpeghelper.h
            platformcontext.cpp
            platformcontext.h
            platformdeviceinfo.h
            platformstream.cpp
            platformstream.h
            scopedptr.h
            yuvconverters.cpp
            yuvconverters.h
        )
        fips_libs(pthread)
        set(PLATFORM_STRING "Linux ${COMPILERBITS}")
        set(OPENPNP_LINUX_BACKEND TRUE)
    endif()

fips_end_lib()

# static lib, so the api must not be marked dllimport in the consumers either
# libmain.cpp defines BUILD_OPENPNP_LIBRARY itself, defining it here too only warns
target_compile_definitions(openpnp-capture PUBLIC OPENPNPCAPTURE_STATIC)

# reported back by Cap_getLibraryVersion
target_compile_definitions(openpnp-capture PRIVATE "__PLATFORM__=\"${PLATFORM_STRING}\"")
target_compile_definitions(openpnp-capture PRIVATE "__BUILDTYPE__=\"${BUILDTYPE_STRING}\"")

if (FIPS_WINDOWS)
    target_compile_definitions(openpnp-capture PRIVATE _CRT_SECURE_NO_WARNINGS)
endif()

if (OPENPNP_LINUX_BACKEND)
    # prefer a system libjpeg-turbo, fall back to building the vendored copy
    find_package(PkgConfig)
    if (PkgConfig_FOUND)
        pkg_search_module(TurboJPEG libturbojpeg)
    endif()

    if (TurboJPEG_FOUND)
        target_include_directories(openpnp-capture PRIVATE ${TurboJPEG_INCLUDE_DIRS})
        target_link_directories(openpnp-capture PRIVATE ${TurboJPEG_LIBDIR})
        target_link_libraries(openpnp-capture ${TurboJPEG_LIBRARIES})
    else()
        include(ExternalProject)
        set(LIBJPEG_TURBO_INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/libjpeg-turbo/install)

        ExternalProject_Add(libjpeg-turbo-external
            SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/linux/contrib/libjpeg-turbo-3.1.2
            CMAKE_ARGS
                -DCMAKE_INSTALL_PREFIX=${LIBJPEG_TURBO_INSTALL_DIR}
                -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
                -DENABLE_SHARED=OFF
                -DWITH_SIMD=ON
                -DWITH_TOOLS=OFF
                -DWITH_JAVA=OFF
                -DCMAKE_C_FLAGS=-fPIC
                -DCMAKE_CXX_FLAGS=-fPIC
            BUILD_BYPRODUCTS
                ${LIBJPEG_TURBO_INSTALL_DIR}/lib/libturbojpeg.a
        )

        add_library(turbojpeg-static STATIC IMPORTED)
        set_target_properties(turbojpeg-static PROPERTIES
            IMPORTED_LOCATION ${LIBJPEG_TURBO_INSTALL_DIR}/lib/libturbojpeg.a
        )

        target_include_directories(openpnp-capture PRIVATE ${LIBJPEG_TURBO_INSTALL_DIR}/include)
        add_dependencies(turbojpeg-static libjpeg-turbo-external)
        add_dependencies(openpnp-capture libjpeg-turbo-external)
        target_link_libraries(openpnp-capture turbojpeg-static)
    endif()

    # the static turbojpeg is built -fPIC, keep this lib matching it
    set_target_properties(openpnp-capture PROPERTIES POSITION_INDEPENDENT_CODE ON)
endif()
