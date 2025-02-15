# Copyright (c) Open Enclave SDK contributors.
# Licensed under the MIT License.

set(OE_INCLUDE_DIR ${PROJECT_SOURCE_DIR}/include)
set(EDL_DIR ${PROJECT_SOURCE_DIR}/include/openenclave/edl)

##==============================================================================
##
## These rules generate the edge routines for the internal TEE-agnostic
## ECALLs/OCALLs used by liboehost/liboecore.
##
##==============================================================================

set(CORE_EDL_FILE ${EDL_DIR}/core.edl)
add_custom_command(
  OUTPUT core_t.h core_args.h
  DEPENDS ${CORE_EDL_FILE} edger8r
  COMMAND edger8r --header-only --search-path ${OE_INCLUDE_DIR} --trusted
          ${CORE_EDL_FILE})

add_custom_target(core_trusted_edl DEPENDS core_t.h core_args.h)

##==============================================================================
##
## These rules generate the edge routines for the internal SGX-specific
## ECALLs/OCALLs used by liboehost/liboecore.
##
##==============================================================================

set(SGX_EDL_FILE ${EDL_DIR}/sgx/platform.edl)
if (OE_SGX)
  # Only generate headers
  add_custom_command(
    OUTPUT platform_t.h platform_args.h
    DEPENDS ${SGX_EDL_FILE} edger8r
    COMMAND edger8r --header-only --search-path ${OE_INCLUDE_DIR} --trusted
            ${SGX_EDL_FILE} -DOE_SGX_ENTROPY)

  add_custom_target(platform_trusted_edl DEPENDS platform_t.h platform_args.h)
endif ()

##==============================================================================
##
## These rules build the oecore target.
##
##==============================================================================

set(MUSL_SRC_DIR ${PROJECT_SOURCE_DIR}/3rdparty/musl/musl/src)

if (OE_SGX)
  list(
    APPEND
    PLATFORM_SRC
    ${MUSL_SRC_DIR}/string/x86_64/memcpy.s
    ${MUSL_SRC_DIR}/string/x86_64/memmove.s
    ${MUSL_SRC_DIR}/string/x86_64/memset.s
    ../../common/sgx/endorsements.c
    ../../common/sgx/rand.S
    ../../common/sgx/cpuid.c
    sgx/arena.c
    sgx/asmdefs.c
    sgx/backtrace.c
    sgx/calls.c
    sgx/cpuid.c
    sgx/enter.S
    sgx/entropy.c
    sgx/errno.c
    sgx/exception.c
    sgx/exit.S
    sgx/getkey.S
    sgx/globals.c
    sgx/hostcalls.c
    sgx/init.c
    sgx/keys.c
    sgx/longjmp.S
    sgx/memory.c
    sgx/properties.c
    sgx/random_internal.c
    sgx/reloc.c
    sgx/report.c
    sgx/sched_yield.c
    sgx/setjmp.S
    sgx/spinlock.c
    sgx/switchlesscalls.c
    sgx/td.c
    sgx/td_basic.c
    sgx/thread.c
    sgx/threadlocal.c
    sgx/tracee.c
    sgx/writebarrier.c
    sgx/xstate.c)

  # Functions in td_basic.c will change the status of td and may trigger
  # stack check fail, thus it is necessary to turn off stack check.
  set_source_files_properties(sgx/td_basic.c PROPERTIES COMPILE_FLAGS
                                                        -fno-stack-protector)

  # To avoid the `unused-command-line-argument` warning, which we treat as an
  # error, we explicitly turn off the warning when compiling these assembly
  # files.
  list(
    APPEND
    W_NO_UNUSED_COMMAND_LINE_ARGUMENT
    ${MUSL_SRC_DIR}/string/x86_64/memcpy.s
    ${MUSL_SRC_DIR}/string/x86_64/memmove.s
    ${MUSL_SRC_DIR}/string/x86_64/memset.s)

  set_property(
    SOURCE ${W_NO_UNUSED_COMMAND_LINE_ARGUMENT}
    APPEND_STRING
    PROPERTY COMPILE_FLAGS "-Wno-error=unused-command-line-argument")
elseif (OE_TRUSTZONE)
  list(
    APPEND
    PLATFORM_SRC
    ${MUSL_SRC_DIR}/string/memcpy.c
    ${MUSL_SRC_DIR}/string/memmove.c
    ${MUSL_SRC_DIR}/string/memset.c
    optee/backtrace.c
    optee/bounds.c
    optee/calls.c
    optee/entropy.c
    optee/errno.c
    optee/header.c
    optee/hostcalls.c
    optee/globals.c
    optee/gp.c
    optee/keys.c
    optee/printf.c
    optee/random_internal.c
    optee/sched_yield.c
    optee/spinlock.c
    optee/stubs.c
    optee/thread.c
    optee/tracee.c)

  list(APPEND NEEDS_STDC_NAMES ${MUSL_SRC_DIR}/string/memmove.c
       ${MUSL_SRC_DIR}/string/memset.c ${MUSL_SRC_DIR}/string/memcpy.c)

  list(APPEND W_NO_CONVERSION ${MUSL_SRC_DIR}/string/memmove.c
       ${MUSL_SRC_DIR}/string/memset.c)
endif ()

if (OE_TRUSTZONE OR (OE_SGX AND (UNIX OR USE_CLANGW)))
  list(APPEND PLATFORM_SRC init_fini.c)
endif ()

if (USE_SNMALLOC)
  set(DEFAULT_ALLOCATOR_LIB oesnmalloc_obj)
else ()
  set(DEFAULT_ALLOCATOR_LIB oedlmalloc_obj)
endif ()

add_enclave_library(
  oecore
  STATIC
  ../../common/safecrt.c
  ../../common/argv.c
  ${MUSL_SRC_DIR}/prng/rand.c
  ${MUSL_SRC_DIR}/string/memcmp.c
  __stack_chk_fail.c
  assert.c
  atexit.c
  backtrace.c
  ctype.c
  gmtime.c
  hexdump.c
  hostcalls.c
  intstr.c
  malloc.c
  once.c
  printf.c
  pthread.c
  random.c
  result.c
  seal.c
  stdio.c
  strerror.c
  string.c
  strtok_r.c
  strtoul.c
  time.c
  tracee.c
  wchar.c
  ${PLATFORM_SRC})

# Some files in oecore come directly from the musl source. For these files,
# corelibc functions with stdc names are required.
# Additionally, suppress type conversion warnings introduced by 3rdparty code
set(CORELIBC_INCLUDES ${PROJECT_SOURCE_DIR}/include/openenclave/corelibc)

list(APPEND NEEDS_STDC_NAMES ${MUSL_SRC_DIR}/prng/rand.c
     ${MUSL_SRC_DIR}/string/memcmp.c strtok_r.c)

list(APPEND W_NO_CONVERSION ${MUSL_SRC_DIR}/prng/rand.c)

set_property(SOURCE ${W_NO_CONVERSION} APPEND_STRING PROPERTY COMPILE_FLAGS
                                                              "-Wno-conversion")

set_property(SOURCE ${NEEDS_STDC_NAMES} APPEND_STRING
             PROPERTY COMPILE_FLAGS " -I${CORELIBC_INCLUDES}")
set_property(SOURCE ${NEEDS_STDC_NAMES} APPEND PROPERTY COMPILE_DEFINITIONS
                                                        OE_NEED_STDC_NAMES)

maybe_build_using_clangw(oecore)

add_enclave_dependencies(oecore core_trusted_edl)
if (OE_SGX)
  add_enclave_dependencies(oecore platform_trusted_edl)
endif ()

enclave_include_directories(oecore PRIVATE ${CMAKE_CURRENT_BINARY_DIR})

enclave_link_libraries(oecore PUBLIC oe_includes)
if (OE_TRUSTZONE)
  enclave_link_libraries(oecore PUBLIC oelibutee_includes)

  # OP-TEE header 3rdparty/optee/libutee/liboeutee/user_ta_header.h
  # includes stdint.h. As a result all OP-TEE needs the corelibc headers
  # in its include path.
  enclave_include_directories(
    oecore PRIVATE ${PROJECT_SOURCE_DIR}/include/openenclave/corelibc)
endif ()

enclave_link_libraries(oecore PUBLIC oe_includes ${DEFAULT_ALLOCATOR_LIB})

if (CMAKE_C_COMPILER_ID MATCHES GNU)
  enclave_compile_options(oecore PRIVATE -Wjump-misses-init)
endif ()

if (OE_SGX)
  set_source_files_properties(sgx/keys.c PROPERTIES COMPILE_FLAGS
                                                    -Wno-type-limits)

  # -m64 is an x86_64 specific flag
  enclave_compile_options(oecore PUBLIC -m64)
endif ()

if (CMAKE_C_COMPILER_ID MATCHES GNU)
  enclave_compile_options(oecore PRIVATE -Wjump-misses-init)
endif ()

# NOTE: All code inside an enclave must be compile with "position
# independent executable" semantics via -fPIE, and linked with -pie.
# Hence the PUBLIC (and INTERFACE) use of these flags instead of -fPIC
# or POSITION_INDEPENDENT_CODE, which would use the incorrect flag as
# enclaves actually are executables.
if (OE_SGX)
  enclave_compile_options(
    oecore
    PUBLIC
    -fPIE
    -nostdinc
    -fstack-protector-strong
    -fvisibility=hidden
    # Preserve frame-pointer in Release mode to enable oe_backtrace.
    -fno-omit-frame-pointer
    # Put each function or data in its own section.
    # This allows aggressively eliminating unused code.
    -ffunction-sections
    -fdata-sections
    # "The default without -fpic is 'initial-exec'; with -fpic the
    # default is 'global-dynamic'."
    # https://gcc.gnu.org/onlinedocs/gcc/Code-Gen-Options.html#Code-Gen-Options
    #
    # Enclaves are linked using -pie and therefore global-dynamic is
    # too conservative. Of the two efficient static thread-local
    # storage models, inital-exec and local-exec, we choose the most
    # optimal one.
    -ftls-model=local-exec
    # Disable builtin functions for enclaves, but only in our build.
    #
    # We do this to work-around compiler bugs (see #1429) due to our
    # redefinition of `memmove` to `oe_memmove` causing an undefined
    # symbol error when a built-in was inlined. However, we only do
    # this for our code as we don't want to force these flags on the
    # user. There are valid reasons for an end user to use built-ins.
    $<BUILD_INTERFACE:-fno-builtin-malloc
    -fno-builtin-calloc
    -fno-builtin>)
elseif (OE_TRUSTZONE)
  enclave_compile_options(oecore PUBLIC -fvisibility=hidden ${OE_TZ_TA_C_FLAGS})
endif ()

if (OE_SGX)
  list(
    APPEND
    DEFAULT_VISIBILITY
    __stack_chk_fail.c
    ${MUSL_SRC_DIR}/sting/memcmp.c
    ${MUSL_SRC_DIR}/string/x86_64/memcpy.s
    ${MUSL_SRC_DIR}/string/x86_64/memmove.s
    ${MUSL_SRC_DIR}/string/x86_64/memset.s)

  set_property(SOURCE ${DEFAULT_VISIBILITY} APPEND_STRING
               PROPERTY COMPILE_FLAGS " -fvisibility=default")
endif ()

enclave_compile_options(oecore INTERFACE $<$<COMPILE_LANGUAGE:CXX>:-nostdinc++>)

enclave_enable_code_coverage(oecore)
if (CODE_COVERAGE)
  # Disable the complier flag to the following sources to
  # avoid conflicts.
  set_source_files_properties(once.c sgx/calls.c sgx/init.c
                              PROPERTIES COMPILE_FLAGS -fno-profile-arcs)
endif ()

enclave_compile_definitions(
  oecore
  PUBLIC
  OE_BUILD_ENCLAVE
  # NOTE: This definition is public to the rest of our project's
  # targets, but should not yet be exposed to consumers of our
  # package.
  $<BUILD_INTERFACE:OE_API_VERSION=2>)

if (WITH_EEID)
  enclave_compile_definitions(oecore PRIVATE OE_WITH_EXPERIMENTAL_EEID)
endif ()

# Interface link flags for enclaves.
if (OE_SGX)
  enclave_link_libraries(
    oecore
    INTERFACE
    -nostdlib
    -nodefaultlibs
    -nostartfiles
    -Wl,--no-undefined,-Bstatic,-Bsymbolic,--export-dynamic,-pie,--build-id
    -Wl,-z,noexecstack
    -Wl,-z,now
    -Wl,-gc-sections)
elseif (OE_TRUSTZONE)
  enclave_link_libraries(
    oecore
    INTERFACE
    -nostdlib
    -nodefaultlibs
    -nostartfiles
    --no-undefined
    -pie
    --sort-section=alignment
    "-z max-page-size=4096"
    --as-needed)
endif ()

set_enclave_property(TARGET oecore PROPERTY ARCHIVE_OUTPUT_DIRECTORY
                     ${OE_LIBDIR}/openenclave/enclave)
install_enclaves(
  TARGETS
  oecore
  EXPORT
  openenclave-targets
  ARCHIVE
  DESTINATION
  ${CMAKE_INSTALL_LIBDIR}/openenclave/enclave)

add_enclave_library(oedebugmalloc STATIC debugmalloc.c)

enclave_compile_definitions(oedebugmalloc PUBLIC OE_USE_DEBUG_MALLOC)

enclave_link_libraries(oedebugmalloc oecore)

set_enclave_property(TARGET oedebugmalloc PROPERTY ARCHIVE_OUTPUT_DIRECTORY
                     ${OE_LIBDIR}/openenclave/enclave)

if (OE_TRUSTZONE)
  # dependent header 3rdparty/optee/libutee/liboeutee/user_ta_header.h
  # includes inttypes.h. As a result oedebugmalloc needs the corelibc headers
  # in its include path.
  enclave_include_directories(
    oedebugmalloc PRIVATE ${PROJECT_SOURCE_DIR}/include/openenclave/corelibc)
endif ()

install_enclaves(
  TARGETS
  oedebugmalloc
  EXPORT
  openenclave-targets
  ARCHIVE
  DESTINATION
  ${CMAKE_INSTALL_LIBDIR}/openenclave/enclave)

enclave_enable_fuzzing(oecore)
