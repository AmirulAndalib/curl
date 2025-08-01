#***************************************************************************
#                                  _   _ ____  _
#  Project                     ___| | | |  _ \| |
#                             / __| | | | |_) | |
#                            | (__| |_| |  _ <| |___
#                             \___|\___/|_| \_\_____|
#
# Copyright (C) Daniel Stenberg, <daniel@haxx.se>, et al.
#
# This software is licensed as described in the file COPYING, which
# you should have received as part of this distribution. The terms
# are also available at https://curl.se/docs/copyright.html.
#
# You may opt to use, copy, modify, merge, publish, distribute and/or sell
# copies of the Software, and permit persons to whom the Software is
# furnished to do so, under the terms of the COPYING file.
#
# This software is distributed on an "AS IS" basis, WITHOUT WARRANTY OF ANY
# KIND, either express or implied.
#
# SPDX-License-Identifier: curl
#
###########################################################################
find_program(TEST_NGHTTPX "nghttpx")
if(NOT TEST_NGHTTPX)
  set(TEST_NGHTTPX "")
endif()
mark_as_advanced(TEST_NGHTTPX)
# Consumed variables: TEST_NGHTTPX
configure_file("config.in" "${CMAKE_CURRENT_BINARY_DIR}/config" @ONLY)

add_custom_target(testdeps)
if(BUILD_CURL_EXE)
  add_dependencies(testdeps curlinfo)
endif()

if(CURL_CLANG_TIDY)
  add_custom_target(tests-clang-tidy)
  add_dependencies(testdeps tests-clang-tidy)
endif()

add_subdirectory(http)
add_subdirectory(server)
add_subdirectory(libtest)
add_subdirectory(tunit)
add_subdirectory(unit)
add_subdirectory(certs)

# Add a runtests target with customized flags
function(curl_add_runtests _targetname _test_flags)
  if(NOT BUILD_LIBCURL_DOCS)
    string(APPEND _test_flags " !documentation")
  endif()
  set(_depends "")
  # Skip walking through dependent targets before running tests in CI.
  # This avoids: GNU Make doing a slow re-evaluation of all targets and
  # skipping them, MSBuild doing a re-evaluation, and actually rebuilding them.
  if(NOT _targetname STREQUAL "test-ci")
    if(BUILD_CURL_EXE)
      list(APPEND _depends ${EXE_NAME})
    endif()
    list(APPEND _depends "testdeps")
  endif()
  # Use a special '$TFLAGS' placeholder as last argument which will be
  # replaced by the contents of the environment variable in runtests.pl.
  # This is a workaround for CMake's limitation where commands executed by
  # 'make' or 'ninja' cannot portably reference environment variables.
  string(REPLACE " " ";" _test_flags_list "${_test_flags}")
  add_custom_target(${_targetname}
    COMMAND
      "${PERL_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/runtests.pl"
      ${_test_flags_list}
      "\$TFLAGS"
    DEPENDS "${_depends}"
    VERBATIM USES_TERMINAL
  )
endfunction()

# Add a pytests target with customized flags
function(curl_add_pytests _targetname _test_flags)
  set(_depends "")
  if(NOT _targetname STREQUAL "pytest-ci")
    if(BUILD_CURL_EXE)
      list(APPEND _depends ${EXE_NAME})
    endif()
    list(APPEND _depends "libtests")
  endif()
  string(REPLACE " " ";" _test_flags_list "${_test_flags}")
  add_custom_target(${_targetname}
    COMMAND pytest ${_test_flags_list} "${CMAKE_CURRENT_SOURCE_DIR}/http"
    DEPENDS "${_depends}"
    VERBATIM USES_TERMINAL
  )
endfunction()

# Create configurehelp.pm, used by tests needing to run the C preprocessor.
if(MSVC OR CMAKE_C_COMPILER_ID STREQUAL "GNU" OR CMAKE_C_COMPILER_ID MATCHES "Clang")
  set(CURL_CPP "\"${CMAKE_C_COMPILER}\" -E")
  if(APPLE AND CMAKE_OSX_SYSROOT)
    string(APPEND CURL_CPP " -isysroot ${CMAKE_OSX_SYSROOT}")
  endif()
  # Add header directories, like autotools builds do.
  get_property(_include_dirs TARGET ${LIB_SELECTED} PROPERTY INCLUDE_DIRECTORIES)
  foreach(_include_dir IN LISTS _include_dirs)
    string(APPEND CURL_CPP " -I${_include_dir}")
  endforeach()
else()
  set(CURL_CPP "cpp")
endif()
# Generate version script for the linker, for versioned symbols.
# Consumed variable:
#   CURL_CPP
configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/configurehelp.pm.in"
  "${CMAKE_CURRENT_BINARY_DIR}/configurehelp.pm" @ONLY)

curl_add_runtests(tests          "")  # Avoid 'test', which is a reserved target name in CMake
curl_add_runtests(test-quiet     "-a -s")
curl_add_runtests(test-am        "-a -am")
curl_add_runtests(test-full      "-a -p -r")
# ~flaky means that it ignores results of tests using the flaky keyword
curl_add_runtests(test-nonflaky  "-a -p ~flaky ~timing-dependent")
curl_add_runtests(test-ci        "-a -p ~flaky ~timing-dependent -r --retry=5 -j20")
curl_add_runtests(test-torture   "-a -t -j20")
curl_add_runtests(test-event     "-a -e")

curl_add_pytests(curl-pytest      "-n auto")
curl_add_pytests(curl-pytest-ci   "-n auto -v")
