CMAKE_MINIMUM_REQUIRED(VERSION 3.13)

PROJECT(pg_query CXX C)

FILE(GLOB pg_query_srcs ${CMAKE_CURRENT_SOURCE_DIR}/src/*.c ${CMAKE_CURRENT_SOURCE_DIR}/src/postgres/*.c)
LIST(APPEND pg_query_srcs vendor/protobuf-c/protobuf-c.c vendor/xxhash/xxhash.c protobuf/pg_query.pb-c.c)

INCLUDE_DIRECTORIES(.)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/src/include)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/src/postgres/include)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR}/vendor)

ADD_LIBRARY(pg_query STATIC ${pg_query_srcs})

SET_TARGET_PROPERTIES(pg_query PROPERTIES LINKER_LANGUAGE C)
SET_TARGET_PROPERTIES(pg_query PROPERTIES POSITION_INDEPENDENT_CODE ON)

# Build examples
FUNCTION(pg_query_add_example name)
    ADD_EXECUTABLE(${name} ${CMAKE_CURRENT_SOURCE_DIR}/examples/${name}.c)
    TARGET_LINK_LIBRARIES(${name} pg_query pthread)
ENDFUNCTION()

PG_QUERY_ADD_EXAMPLE(normalize)
PG_QUERY_ADD_EXAMPLE(normalize_error)
PG_QUERY_ADD_EXAMPLE(scan)
PG_QUERY_ADD_EXAMPLE(simple)
PG_QUERY_ADD_EXAMPLE(simple_error)
PG_QUERY_ADD_EXAMPLE(simple_plpgsql)

# Build tests
FUNCTION(pg_query_add_test name)
    LIST(TRANSFORM ARGN PREPEND "${CMAKE_CURRENT_SOURCE_DIR}/test/")
    LIST(JOIN ARGN " " extra_srcs)
    SET(test_srcs ${CMAKE_CURRENT_SOURCE_DIR}/test/${name}.c ${extra_srcs})
    SET(test_name ${name}_test)
    ADD_EXECUTABLE(${test_name} ${test_srcs})
    TARGET_LINK_LIBRARIES(${test_name} pg_query pthread)
    TARGET_INCLUDE_DIRECTORIES(${test_name} PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/src)
ENDFUNCTION()

SET(pg_query_test_targets
    complex
    concurrency
    deparse
    fingerprint
    fingerprint_opts
    normalize
    parse
    parse_opts
    parse_plpgsql
    parse_protobuf
    parse_protobuf_opts
    scan
    split
)

FOREACH(item ${pg_query_test_targets})
    PG_QUERY_ADD_TEST(${item})
ENDFOREACH()

LIST(TRANSFORM pg_query_test_targets APPEND "_test" OUTPUT_VARIABLE pg_query_test_target_names)

ADD_CUSTOM_TARGET(pg_run_all_tests echo "running tests..."
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/complex_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/concurrency_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/deparse_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/fingerprint_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/fingerprint_opts_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/normalize_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/parse_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/parse_opts_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/parse_plpgsql_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/parse_protobuf_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/parse_protobuf_opts_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/scan_test
    COMMAND ${EXECUTABLE_OUTPUT_PATH}/split_test
    COMMAND diff -Naur test/plpgsql_samples.expected.json test/plpgsql_samples.actual.json
    DEPENDS
        ${pg_query_test_target_names}
    WORKING_DIRECTORY
        ${CMAKE_CURRENT_SOURCE_DIR}
)