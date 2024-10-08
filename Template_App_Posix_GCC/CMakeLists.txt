cmake_minimum_required(VERSION 3.15)

project( template_app )

add_compile_options( -D_WINDOWS_ )

add_compile_options( -Wall -Wextra -Wpedantic )

if( TRACE_ON_ENTER )
    add_compile_options( -DTRACE_ON_ENTER=1 )
else()
    add_compile_options( -DTRACE_ON_ENTER=0 )
endif()

if( COVERAGE_TEST )
    set( COVERAGE_TEST 1 )
    set( NO_TRACING 1 )
    add_compile_options( -DprojCOVERAGE_TEST=1 )
    add_compile_options( -DprojENABLE_TRACING=0 )
else()
    if( NO_TRACING )
        set( NO_TRACING 1 )
        add_compile_options( -DprojENABLE_TRACING=0 )
    else()
        set( NO_TRACING 0 )
        add_compile_options( -DprojENABLE_TRACING=1 )
    endif()
    set( COVERAGE_TEST 0 )
    add_compile_options( -DprojCOVERAGE_TEST=0 )
endif()

if( PROFILE )
    set( CMAKE_BUILD_TYPE "release" )
else()
    set( CMAKE_BUILD_TYPE "debug" )
endif()

set(CMAKE_CXX_FLAGS_RELEASE "-O3")
set(CMAKE_CXX_FLAGS_DEBUG "-O0 -g3")

if( SANITIZE_ADDRESS )
    add_compile_options( -fsanitize=address -fsanitize=alignment )
endif()

if( SANITIZE_LEAK )
    add_compile_options( -fsanitize=leak )
endif()

set( FREERTOS_KERNEL_PATH "../FreeRTOS/FreeRTOS/Source" )
set( FREERTOS_PLUS_TRACE_PATH "../FreeRTOS/FreeRTOS-Plus/Source/FreeRTOS-Plus-Trace" )
set( FREERTOS_PLUS_CLI_PATH "../FreeRTOS/FreeRTOS-Plus/Source/FreeRTOS-Plus-CLI" )

# Add the freertos_config for FreeRTOS-Kernel
add_library( freertos_config INTERFACE )

target_include_directories( freertos_config
    INTERFACE
        ./
        ./Trace_Recorder_Configuration
        ${FREERTOS_PLUS_CLI_PATH}/
        ${FREERTOS_PLUS_TRACE_PATH}/include
        ${FREERTOS_PLUS_TRACE_PATH}/kernelports/FreeRTOS/include
)

# Select the heap port
set( FREERTOS_HEAP "3" CACHE STRING "" FORCE)

# Select the native compile PORT
set( FREERTOS_PORT "GCC_POSIX" CACHE STRING "" FORCE)

# Adding the FreeRTOS-Kernel subdirectory
add_subdirectory( ${FREERTOS_KERNEL_PATH} ${CMAKE_CURRENT_BINARY_DIR}/FreeRTOS-Kernel )

target_compile_options( freertos_kernel
    PRIVATE
        # Trace macro cast pointer to int to store memory management event
        $<IF:${NO_TRACING},,-Wno-pointer-to-int-cast>
)

file( GLOB FREERTOS_PLUS_TRACE_SOURCES ${FREERTOS_PLUS_TRACE_PATH}/*.c ${FREERTOS_PLUS_TRACE_PATH}/kernelports/FreeRTOS/*.c )

add_executable( template_app
                code_coverage_additions.c
                console.c
                main.c
                main_cli_app.c
                run-time-stats-utils.c
                ${FREERTOS_PLUS_CLI_PATH}/FreeRTOS_CLI.c
                $<$<NOT:${NO_TRACING}>:${FREERTOS_PLUS_TRACE_SOURCES}>
              )

target_include_directories( template_app
    PRIVATE
        ${CMAKE_CURRENT_LIST_DIR}
        ${FREERTOS_PLUS_CLI_PATH}/
        ${FREERTOS_KERNEL_PATH}/../Demo/Common/include
        ${FREERTOS_PLUS_TRACE_PATH}/Include
        ${FREERTOS_PLUS_TRACE_PATH}/streamports/File/include
        ${FREERTOS_PLUS_TRACE_PATH}/streamports/File/config
)

target_compile_definitions( template_app
    PRIVATE
        $<IF:$<STREQUAL:${USER_DEMO},BLINKY_DEMO>,USER_DEMO=0,>
        $<IF:$<STREQUAL:${USER_DEMO},FULL_DEMO>,USER_DEMO=1,>
)

target_link_libraries( template_app freertos_kernel freertos_config )
