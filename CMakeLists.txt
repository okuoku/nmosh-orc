cmake_minimum_required(VERSION 2.8)
project(orc)

macro(prefix pref var)
    set(_r "")
    foreach(e ${${var}})
        list(APPEND _r "${pref}/${e}")
    endforeach()
    set(${var} ${_r})
endmacro()

if(NOT CMAKE_SIZEOF_VOID_P)
    message(WARNING "CMAKE_SIZEOF_VOID_P was not defined!!")
elseif(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(BUILD64 1)
elseif(CMAKE_SIZEOF_VOID_P EQUAL 4)
    set(BUILD32 1)
else()
    message(WARNING "Unsupported pointer size!(${CMAKE_SIZEOF_VOID_P}")
endif()

if(BUILD32)
    add_definitions(-DHAVE_I386)
elseif(BUILD64)
    add_definitions(-DHAVE_AMD64)
endif()

if(MSVC)
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)
    if(BUILD64)
        add_definitions(-D__amd64__)
    endif()
endif()


# FIXME: Replace with dynamic detection
if(WIN32)
    add_definitions(
        -DHAVE_THREAD_WIN32
        -DHAVE_CODEMEM_VIRTUALALLOC
        -DHAVE_OS_WIN32
        )
else()
    # Assumes POSIX here.
    # FIXME: CLOCK_GETTIME in rt
    add_definitions(
        -DHAVE_MMAP
        -DHAVE_POSIX_MEMALIGN
        -DSYS_TIME_H
        -DGETTIMEOFDAY
        -DHAVE_COREMEM_MMAP
        -DHAVE_THREAD_PTHREAD)
endif()

# Backends
add_definitions(
    -DENABLE_BACKEND_C64X
    -DENABLE_BACKEND_MMX
    -DENABLE_BACKEND_SSE
    -DENABLE_BACKEND_ALTIVEC
    -DENABLE_BACKEND_ARM
    -DENABLE_BACKEND_NEON
    -DENABLE_BACKEND_MIPS
    )

# Common
add_definitions(
    -DVERSION=\"0.5.0\"
    -DPACKAGE_VERSION=\"0.5.0\"
    -DORC_EXPORTS
    -DORC_ENABLE_UNSTABLE_API
    )

# orc
set(orc_cpu_srcs
    orccpu-x86.c
    )
set(orc_srcs
    ${orc_cpu_srcs}
    orc.c
    orcbytecode.c
    orcemulateopcodes.c
    orcexecutor.c
    orcfunctions.c
    orcutils.c
    orcrule.c
    orccodemem.c
    orcprogram.c
    orccompiler.c
    orcprogram-c.c
    orcprogram.h
    orcopcodes.c
    orcparse.c
    orconce.c
    orcdebug.c
    orccode.c

    orcx86.c
    # SSE
    orcsse.c
    orcrules-sse.c
    orcprogram-sse.c
    orcx86insn.c
    # MMX
    orcmmx.c
    orcrules-mmx.c
    orcprogram-mmx.c

    orcrules-altivec.c
    orcprogram-altivec.c
    orcpowerpc.c
    orcprogram-neon.c
    orcrules-neon.c
    orcarm.c
    orcprogram-arm.c
    orcrules-arm.c

    orcprogram-c64x-c.c

    orcmips.c
    orcprogram-mips.c
    orcrules-mips.c
    )

prefix(orc orc_srcs)

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/config.h "/* DUMMY */")
include_directories(${CMAKE_CURRENT_BINARY_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}
    )

add_library(orc STATIC ${orc_srcs})

# orcc
add_executable(orcc tools/orcc.c)
target_link_libraries(orcc orc)

# tests
add_library(orc_test STATIC
    orc-test/orcarray.c
    orc-test/orcprofile.c
    orc-test/orcrandom.c
    orc-test/orctest.c)
macro(add_orc_test nam)
    add_executable(${nam} ${ARGN})
    target_link_libraries(${nam} orc orc_test)
endmacro()
macro(testsuite)
    foreach(e ${ARGN})
        add_orc_test(${e} testsuite/${e}.c)
    endforeach()
endmacro()

add_orc_test(benchmorc testsuite/benchmorc/benchmorc.c)
configure_file(testsuite/benchmorc/bench10.orc bench10.orc COPYONLY)
testsuite(
    test_accsadubl test-schro 
    exec_opcodes_sys 
    exec_parse 
    perf_opcodes_sys perf_parse 
    memcpy_speed 
    abi)

