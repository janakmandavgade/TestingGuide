cmake_minimum_required(VERSION 3.10)
project(UnityOddTest C)

# --- 1. Coverage Option ---
option(ENABLE_COVERAGE "Enable code coverage reporting" OFF)
if(ENABLE_COVERAGE)
    if(NOT CMAKE_COMPILER_IS_GNUCC AND NOT CMAKE_C_COMPILER_ID MATCHES "Clang")
        message(FATAL_ERROR "Coverage requires GCC or Clang")
    endif()
    add_compile_options(--coverage -O0 -g)
    add_link_options(--coverage)
endif()

# --- 2. Include paths & Sources ---
include_directories(odd unity_lib)
set(SOURCES
    main.c
    odd/odd.c
    odd/test_odd.c
    unity_lib/unity.c
)

# --- 3. Create Executable ---
add_executable(run_tests ${SOURCES})
target_compile_options(run_tests PRIVATE -Wall -Wextra)

# --- 4. CPPCHECK LOGIC ---
find_program(CPPCHECK_PATH cppcheck)

if(CPPCHECK_PATH)
    message(STATUS "Cppcheck found: ${CPPCHECK_PATH}")
    
    add_custom_command(TARGET run_tests POST_BUILD
        COMMAND ${CPPCHECK_PATH} 
            --enable=all 
            --platform=unix64 
            --std=c11 
            --suppress=missingIncludeSystem 
            -i ${CMAKE_SOURCE_DIR}/unity_lib
            ${CMAKE_SOURCE_DIR}
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMENT "--- Running Static Analysis (ARM64/Unix64) ---"
    )
else()
    # If not found, provide the instruction instead of failing the build
    message(WARNING "
    ---------------------------------------------------------
    WARNING: cppcheck was not found!
    Static analysis will be skipped.
    
    To install it (requires internet):
        sudo apt update && sudo apt install cppcheck
    
    If you are in an air-gapped environment, please manually 
    transfer the cppcheck binary to your system.
    ---------------------------------------------------------")
endif()

# --- 5. Coverage Report Target ---
if(ENABLE_COVERAGE)
    find_program(LCOV_PATH lcov)
    find_program(GENHTML_PATH genhtml)

    if(LCOV_PATH AND GENHTML_PATH)
        add_custom_target(coverage_report
            COMMAND ${LCOV_PATH} --directory . --capture --output-file coverage.info
            COMMAND ${LCOV_PATH} --remove coverage.info '/usr/*' --output-file coverage.info
            COMMAND ${GENHTML_PATH} coverage.info --output-directory out_coverage
            COMMAND ${CMAKE_COMMAND} -E echo "Report generated at: ${CMAKE_BINARY_DIR}/out_coverage/index.html"
            WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
        )
    endif()
endif()