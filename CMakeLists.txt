cmake_minimum_required(VERSION 3.15)
include(CMakeDependentOption)
project(cppcoro
    VERSION 0.0.1
    LANGUAGES CXX
    DESCRIPTION "C++20 coroutine library"
)

include(GNUInstallDirs)
include(GenerateExportHeader)

if (NOT "$ENV{version}" STREQUAL "")
    set(PROJECT_VERSION "$ENV{version}" CACHE INTERNAL "Copied from environment variable")
endif()

option(CPPCORO_EXTERNAL_DEPENDENCIES "Use Cmake find_package to resolve dependencies instead of embedded libraries, Default=OFF." OFF)
option(CPPCORO_BUILD_TESTS           "Build the tests, Default=OFF." OFF)
option(CPPCORO_CODE_COVERAGE         "Enable code coverage, tests must also be enabled, Default=OFF" OFF)
option(CPPCORO_BUILD_EXAMPLES        "Build the examples, Default=ON." ON)
option(CPPCORO_BUILD_SHARED_LIBS     "Build shared libraries, Default=OFF." OFF)

option(CPPCORO_FEATURE_NETWORKING "Include networking features, Default=ON." ON)
option(CPPCORO_FEATURE_TLS "Include TLS encryption features, Default=ON." ON)

message("${PROJECT_NAME} CPPCORO_EXTERNAL_DEPENDENCIES = ${CPPCORO_EXTERNAL_DEPENDENCIES}")
message("${PROJECT_NAME} CPPCORO_BUILD_TESTS           = ${CPPCORO_BUILD_TESTS}")
message("${PROJECT_NAME} CPPCORO_CODE_COVERAGE         = ${CPPCORO_CODE_COVERAGE}")
message("${PROJECT_NAME} CPPCORO_BUILD_EXAMPLES        = ${CPPCORO_BUILD_EXAMPLES}")
message("${PROJECT_NAME} CPPCORO_FEATURE_NETWORKING    = ${CPPCORO_FEATURE_NETWORKING}")
message("${PROJECT_NAME} CPPCORO_FEATURE_TLS           = ${CPPCORO_FEATURE_TLS}")
message("${PROJECT_NAME} CPPCORO_BUILD_SHARED_LIBS     = ${CPPCORO_BUILD_SHARED_LIBS}")

if(CPPCORO_EXTERNAL_DEPENDENCIES)
    if(CPPCORO_FEATURE_NETWORKING)
        find_package(c-ares CONFIG REQUIRED)
    endif()
else()
    if(NOT CPPCORO_BUILD_TESTS)
        # Disable testing in expected
        set(BUILD_TESTING OFF CACHE INTERNAL "")
    endif()
    if(CPPCORO_FEATURE_NETWORKING)
        set(CARES_STATIC    $<NOT:${CPPCORO_BUILD_SHARED_LIBS}> CACHE INTERNAL "")
        set(CARES_SHARED    ${CPPCORO_BUILD_SHARED_LIBS} CACHE INTERNAL "")
        set(CARES_INSTALL   OFF CACHE INTERNAL "")
        add_subdirectory(thirdparty/c-ares)
    endif()
endif()

set(BUILD_SHARED_LIBS ${CPPCORO_BUILD_SHARED_LIBS} CACHE INTERNAL "")

set(CPPCORO_SOURCE_FILES
    include/coro/concepts/awaitable.hpp

    include/coro/coro.hpp
    include/coro/event.hpp src/event.cpp
    include/coro/task.hpp
)

if(CPPCORO_FEATURE_NETWORKING)
    list(APPEND CPPCORO_SOURCE_FILES
        include/coro/fd.hpp
        include/coro/epoll.hpp src/epoll.cpp
    )

    if(CPPCORO_FEATURE_TLS)
        find_package(OpenSSL REQUIRED)
        list(APPEND CPPCORO_SOURCE_FILES
        )
    endif()
endif()

add_library(${PROJECT_NAME} ${CPPCORO_SOURCE_FILES})
set_target_properties(${PROJECT_NAME} PROPERTIES LINKER_LANGUAGE CXX PREFIX "" VERSION ${PROJECT_VERSION} SOVERSION ${PROJECT_VERSION_MAJOR})
target_compile_features(${PROJECT_NAME} PUBLIC cxx_std_20)
target_include_directories(${PROJECT_NAME} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/include PUBLIC ${CMAKE_CURRENT_BINARY_DIR}/include)
generate_export_header(${PROJECT_NAME} BASE_NAME CORO EXPORT_FILE_NAME include/coro/export.hpp)

if(UNIX)
    target_link_libraries(${PROJECT_NAME} PUBLIC pthread)
endif()

if(CPPCORO_FEATURE_NETWORKING)
    target_link_libraries(${PROJECT_NAME} PUBLIC c-ares::cares)
    target_compile_definitions(${PROJECT_NAME} PUBLIC CPPCORO_FEATURE_NETWORKING)
    if(CPPCORO_FEATURE_TLS)
        target_link_libraries(${PROJECT_NAME} PUBLIC OpenSSL::SSL OpenSSL::Crypto)
        target_compile_definitions(${PROJECT_NAME} PUBLIC CPPCORO_FEATURE_TLS)
    endif()
endif()

if(${CMAKE_CXX_COMPILER_ID} MATCHES "GNU")
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "10.2.0")
        message(FATAL_ERROR "g++ version ${CMAKE_CXX_COMPILER_VERSION} is unsupported, please upgrade to at least 10.2.0")
    endif()

    target_compile_options(${PROJECT_NAME} PUBLIC
        $<$<COMPILE_LANGUAGE:CXX>:-fcoroutines>
        $<$<COMPILE_LANGUAGE:CXX>:-fconcepts>
        $<$<COMPILE_LANGUAGE:CXX>:-fexceptions>
        $<$<COMPILE_LANGUAGE:CXX>:-Wall>
        $<$<COMPILE_LANGUAGE:CXX>:-Wextra>
        $<$<COMPILE_LANGUAGE:CXX>:-pipe>
    )
elseif(${CMAKE_CXX_COMPILER_ID} MATCHES "Clang")
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS "16.0.0")
        message(FATAL_ERROR "Clang version ${CMAKE_CXX_COMPILER_VERSION} is unsupported, please upgrade to at least 16.0.0")
    endif()

    target_compile_options(${PROJECT_NAME} PUBLIC
        $<$<COMPILE_LANGUAGE:CXX>:-fexceptions>
        $<$<COMPILE_LANGUAGE:CXX>:-Wall>
        $<$<COMPILE_LANGUAGE:CXX>:-Wextra>
        $<$<COMPILE_LANGUAGE:CXX>:-pipe>
    )
endif()

if(CPPCORO_BUILD_TESTS)
    if(CPPCORO_CODE_COVERAGE)
        target_compile_options(${PROJECT_NAME} PRIVATE --coverage)
        target_link_libraries(${PROJECT_NAME} PRIVATE gcov)
    endif()

    include(CTest)
    add_subdirectory(test)
endif()

if(CPPCORO_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

install(TARGETS cppcoro)
install(DIRECTORY include/coro TYPE INCLUDE)
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/include/coro TYPE INCLUDE)