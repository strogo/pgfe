# -*- cmake -*-
# Copyright (C) Dmitry Igrishin
# For conditions of distribution and use, see files LICENSE.txt or pgfe.hpp

if(NOT PGFE_BUILD_TESTS)
  return()
endif()

# ------------------------------------------------------------------------------

add_executable(hello_world hello_world.cpp)
target_link_libraries(hello_world dmitigr_pgfe)

add_executable(test-unit-data unit-data.cpp unit.hpp)
target_link_libraries(test-unit-data dmitigr_pgfe)

add_executable(test-unit-sql_string unit-sql_string.cpp unit.hpp)
target_link_libraries(test-unit-sql_string dmitigr_pgfe)

add_executable(test-unit-conversions unit-conversions.cpp unit.hpp)
target_link_libraries(test-unit-conversions dmitigr_pgfe)

add_executable(test-unit-connection_options unit-connection_options.cpp unit.hpp)
target_link_libraries(test-unit-connection_options dmitigr_pgfe)

add_executable(test-unit-connection unit-connection.cpp unit.hpp)
target_link_libraries(test-unit-connection dmitigr_pgfe)

add_executable(test-unit-connection-ssl unit-connection-ssl.cpp unit.hpp)
target_link_libraries(test-unit-connection-ssl dmitigr_pgfe)

add_executable(test-unit-connection-deferrable unit-connection-deferrable.cpp unit.hpp)
target_link_libraries(test-unit-connection-deferrable dmitigr_pgfe)

add_executable(test-unit-connection-err_in_mid unit-connection-err_in_mid.cpp unit.hpp)
target_link_libraries(test-unit-connection-err_in_mid dmitigr_pgfe)

add_executable(test-unit-ps unit-ps.cpp unit.hpp)
target_link_libraries(test-unit-ps dmitigr_pgfe)

add_executable(test-unit-conversions-online unit-conversions-online.cpp unit.hpp)
target_link_libraries(test-unit-conversions-online dmitigr_pgfe)

add_executable(test-benchmark-sql_string_replace benchmark-sql_string_replace.cpp unit.hpp)
target_link_libraries(test-benchmark-sql_string_replace dmitigr_pgfe)

add_executable(test-benchmark-array-client benchmark-array-client.cpp unit.hpp)
target_link_libraries(test-benchmark-array-client dmitigr_pgfe)

add_executable(test-benchmark-array-server benchmark-array-server.cpp unit.hpp)
target_link_libraries(test-benchmark-array-server dmitigr_pgfe)

# ------------------------------------------------------------------------------

# Note: Multi-configuration generators (VS, Xcode) append a per-configuration
# subdirectory to the RUNTIME_OUTPUT_DIRECTORY unless a generator expression is used.
set_target_properties(
  hello_world
  test-unit-data
  test-unit-sql_string
  test-unit-conversions
  test-unit-connection_options
  test-unit-connection
  test-unit-connection-ssl
  test-unit-connection-deferrable
  test-unit-connection-err_in_mid
  test-unit-ps
  test-unit-conversions-online
  test-benchmark-sql_string_replace
  test-benchmark-array-client
  test-benchmark-array-server
  PROPERTIES
  RUNTIME_OUTPUT_DIRECTORY "${PGFE_LIBRARY_OUTPUT_DIR}")

# ------------------------------------------------------------------------------

add_test(NAME hello_world COMMAND hello_world)
add_test(NAME unit-data COMMAND test-unit-data)
add_test(NAME unit-sql_string COMMAND test-unit-sql_string)
add_test(NAME unit-conversions COMMAND test-unit-conversions)
add_test(NAME unit-connection_options COMMAND test-unit-connection_options)
add_test(NAME unit-connection COMMAND test-unit-connection)
add_test(NAME unit-connection-ssl COMMAND test-unit-connection-ssl)
add_test(NAME unit-connection-deferrable COMMAND test-unit-connection-deferrable)
add_test(NAME unit-connection-err_in_mid COMMAND test-unit-connection-err_in_mid)
add_test(NAME unit-ps COMMAND test-unit-ps)
add_test(NAME unit-conversions-online COMMAND test-unit-conversions-online)
add_test(NAME benchmark-sql_string_replace COMMAND test-benchmark-sql_string_replace)
add_test(NAME benchmark-array-client COMMAND test-benchmark-array-client)
add_test(NAME benchmark-array-server COMMAND test-benchmark-array-server)
