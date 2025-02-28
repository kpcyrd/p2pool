cmake_minimum_required(VERSION 2.8.12)
project(p2pool_tests)

option(STATIC_LIBS "Use locally built libuv and libzmq static libs" OFF)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/cmake")

if (${CMAKE_VERSION} VERSION_GREATER_EQUAL "3.6.0") 
	set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT p2pool_tests)
endif()

add_subdirectory(googletest)
set(LIBS gtest)

add_subdirectory(../external/src/RandomX RandomX)
set(LIBS ${LIBS} randomx)
add_definitions(-DWITH_RANDOMX)

include(cmake/flags.cmake)

set(HEADERS
)

set(SOURCES
	src/crypto_tests.cpp
	src/difficulty_type_tests.cpp
	src/hash_tests.cpp
	src/keccak_tests.cpp
	src/main.cpp
	src/pool_block_tests.cpp
	src/util_tests.cpp
	src/wallet_tests.cpp
	../external/src/cryptonote/crypto-ops-data.c
	../external/src/cryptonote/crypto-ops.c
	../src/block_cache.cpp
	../src/block_template.cpp
	../src/console_commands.cpp
	../src/crypto.cpp
	../src/json_rpc_request.cpp
	../src/keccak.cpp
	../src/log.cpp
	../src/memory_leak_debug.cpp
	../src/mempool.cpp
	../src/miner.cpp
	../src/p2p_server.cpp
	../src/p2pool.cpp
	../src/p2pool_api.cpp
	../src/params.cpp
	../src/pool_block.cpp
	../src/pow_hash.cpp
	../src/side_chain.cpp
	../src/stratum_server.cpp
	../src/util.cpp
	../src/wallet.cpp
	../src/zmq_reader.cpp
)

include_directories(../src)
include_directories(../external/src)
include_directories(../external/src/cryptonote)
include_directories(../external/src/curl/include)
include_directories(../external/src/libuv/include)
include_directories(../external/src/cppzmq)
include_directories(../external/src/libzmq/include)
include_directories(../external/src/RandomX/src)
include_directories(../external/src/rapidjson/include)
include_directories(../external/src/robin-hood-hashing/src/include)
include_directories(src)
include_directories(googletest/googletest/include)

if (WIN32)
	set(LIBS ${LIBS} ws2_32 iphlpapi userenv psapi wldap32)
	add_definitions(-DCURL_STATICLIB)
elseif (NOT APPLE)
	set(LIBS ${LIBS} pthread gss dl)
endif()

if (STATIC_LIBS)
	set(ZMQ_LIBRARY "${CMAKE_SOURCE_DIR}/../external/src/libzmq/build/lib/libzmq.a")
	set(ZMQ_LIBRARY_DEBUG "${CMAKE_SOURCE_DIR}/../external/src/libzmq/build/lib/libzmq.a")
	set(UV_LIBRARY_DEBUG "${CMAKE_SOURCE_DIR}/../external/src/libuv/build/libuv_a.a")
	set(UV_LIBRARY "${CMAKE_SOURCE_DIR}/../external/src/libuv/build/libuv_a.a")
	set(CURL_LIBRARY_DEBUG "${CMAKE_SOURCE_DIR}/../external/src/curl/lib/.libs/libcurl.a")
	set(CURL_LIBRARY "${CMAKE_SOURCE_DIR}/../external/src/curl/lib/.libs/libcurl.a")
else()
	if (CMAKE_CXX_COMPILER_ID MATCHES MSVC)
		find_library(ZMQ_LIBRARY_DEBUG NAMES libzmq-v142-mt-sgd-4_3_5 PATHS "../external/lib/libzmq/Debug")
		find_library(ZMQ_LIBRARY NAMES libzmq-v142-mt-s-4_3_5 PATHS "../external/lib/libzmq/Release")
		find_library(UV_LIBRARY_DEBUG NAMES uv_a PATHS "../external/lib/libuv/Debug")
		find_library(UV_LIBRARY NAMES uv_a PATHS "../external/lib/libuv/Release")
		find_library(CURL_LIBRARY_DEBUG NAMES libcurld PATHS "../external/lib/libcurl/Debug")
		find_library(CURL_LIBRARY NAMES libcurl PATHS "../external/lib/libcurl/Release")
	elseif (CMAKE_CXX_COMPILER_ID MATCHES GNU OR CMAKE_CXX_COMPILER_ID MATCHES Clang)
		find_library(ZMQ_LIBRARY_DEBUG NAMES zmq libzmq.a)
		find_library(ZMQ_LIBRARY NAMES zmq libzmq.a)
		find_library(UV_LIBRARY_DEBUG NAMES uv libuv.a)
		find_library(UV_LIBRARY NAMES uv libuv.a)
		if (WIN32)
			find_library(CURL_LIBRARY_DEBUG NAMES libcurl.a PATHS "../external/src/curl/lib/.libs" NO_DEFAULT_PATH)
			find_library(CURL_LIBRARY NAMES libcurl.a PATHS "../external/src/curl/lib/.libs" NO_DEFAULT_PATH)
		else()
			find_library(CURL_LIBRARY_DEBUG NAMES curl)
			find_library(CURL_LIBRARY NAMES curl)
		endif()
		find_library(SODIUM_LIBRARY sodium)
	endif()

	if (SODIUM_LIBRARY)
		set(LIBS ${LIBS} ${SODIUM_LIBRARY})
	endif()
endif()

find_library(PGM_LIBRARY pgm)
find_library(NORM_LIBRARY norm)

if (PGM_LIBRARY)
	set(LIBS ${LIBS} ${PGM_LIBRARY})
endif()

if (NORM_LIBRARY)
	set(LIBS ${LIBS} ${NORM_LIBRARY})
endif()

if(APPLE)
	find_library(FOUNDATION_LIB Foundation)
	find_library(CORE_FOUNDATION_LIB CoreFoundation)
	find_library(SYSTEM_CONFIGURATION_LIB SystemConfiguration)
	set(LIBS ${LIBS} ${FOUNDATION_LIB} ${CORE_FOUNDATION_LIB} ${SYSTEM_CONFIGURATION_LIB})
endif()

add_definitions(/DZMQ_STATIC /DP2POOL_LOG_DISABLE)

include(CheckCXXSourceCompiles)

check_cxx_source_compiles("int main(){ return __builtin_clzll(1);}" HAVE_BUILTIN_CLZLL)
check_cxx_source_compiles("#include <intrin.h>\n#pragma intrinsic(_BitScanReverse64)\nint main(){unsigned long r;_BitScanReverse64(&r,1);return r;}" HAVE_BITSCANREVERSE64)

if (HAVE_BUILTIN_CLZLL)
	add_definitions(/DHAVE_BUILTIN_CLZLL)
endif()

if (HAVE_BITSCANREVERSE64)
	add_definitions(/DHAVE_BITSCANREVERSE64)
endif()

add_executable(${CMAKE_PROJECT_NAME} ${HEADERS} ${SOURCES})
target_link_libraries(${CMAKE_PROJECT_NAME} debug ${ZMQ_LIBRARY_DEBUG} debug ${UV_LIBRARY_DEBUG} debug ${CURL_LIBRARY_DEBUG} optimized ${ZMQ_LIBRARY} optimized ${UV_LIBRARY} optimized ${CURL_LIBRARY} ${LIBS})
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_if_different "${CMAKE_SOURCE_DIR}/src/crypto_tests.txt" $<TARGET_FILE_DIR:${CMAKE_PROJECT_NAME}>)
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_if_different "${CMAKE_SOURCE_DIR}/src/mainnet_test2_block.dat" $<TARGET_FILE_DIR:${CMAKE_PROJECT_NAME}>)
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_if_different "${CMAKE_SOURCE_DIR}/src/sidechain_dump.dat" $<TARGET_FILE_DIR:${CMAKE_PROJECT_NAME}>)
