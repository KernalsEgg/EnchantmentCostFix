cmake_minimum_required(VERSION 3.20)

# ---- Cache build vars ----

set(Boost_USE_STATIC_RUNTIME OFF CACHE BOOL "")
set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>" CACHE STRING "")
set(CMAKE_TOOLCHAIN_FILE "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake" CACHE STRING "")
set(VCPKG_TARGET_TRIPLET "x64-windows-static" CACHE STRING "")

# ---- Project ----

project(
	ScrambledEggchantments
	VERSION 1.0.0
	LANGUAGES CXX
)

configure_file(
	${CMAKE_CURRENT_SOURCE_DIR}/version.rc.in
	${CMAKE_CURRENT_BINARY_DIR}/version.rc
	@ONLY
)

# ---- Include guards ----

if(PROJECT_SOURCE_DIR STREQUAL PROJECT_BINARY_DIR)
	message(
		FATAL_ERROR
			"In-source builds not allowed. Please make a new directory (called a build directory) and run CMake from there."
	)
endif()

# ---- Dependencies ----

set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION_DEBUG OFF)

set(Boost_USE_MULTITHREADED ON)
set(Boost_USE_STATIC_LIBS ON)
set(Boost_USE_DEBUG_RUNTIME ON)

if (MSVC)
	add_compile_definitions(
		_UNICODE
	)

	add_compile_options(
		/MP						# Build with Multiple Processes
		/sdl					# Enable Additional Security Checks
		/utf-8					# Set Source and Executable character sets to UTF-8
		/Zi						# Debug Information Format

		/permissive-			# Standards conformance

		/Zc:alignedNew			# C++17 over-aligned allocation
		/Zc:auto				# Deduce Variable Type
		/Zc:char8_t
		/Zc:__cplusplus			# Enable updated __cplusplus macro
		/Zc:externC
		/Zc:externConstexpr		# Enable extern constexpr variables
		/Zc:forScope			# Force Conformance in for Loop Scope
		/Zc:hiddenFriend
		/Zc:implicitNoexcept	# Implicit Exception Specifiers
		/Zc:lambda
		/Zc:noexceptTypes		# C++17 noexcept rules
		/Zc:preprocessor		# Enable preprocessor conformance mode
		/Zc:referenceBinding	# Enforce reference binding rules
		/Zc:rvalueCast			# Enforce type conversion rules
		/Zc:sizedDealloc		# Enable Global Sized Deallocation Functions
		/Zc:strictStrings		# Disable string literal type conversion
		/Zc:ternary				# Enforce conditional operator rules
		/Zc:threadSafeInit		# Thread-safe Local Static Initialization
		/Zc:tlsGuards
		/Zc:trigraphs			# Trigraphs Substitution
		/Zc:wchar_t				# wchar_t Is Native Type

		/experimental:external
		/external:anglebrackets
		/external:W0

		/W4						# Warning level (all warnings)
		/WX						# Warning level (warnings are errors)

		"$<$<CONFIG:DEBUG>:>"
		"$<$<CONFIG:RELEASE>:/Zc:inline;/JMC->"
	)
endif()

add_subdirectory("$ENV{CommonLibSSEPath}" CommonLibSSE)

find_package(nlohmann_json REQUIRED)
find_package(spdlog REQUIRED CONFIG)

# ---- Add source files ----

include(cmake/headerlist.cmake)
include(cmake/sourcelist.cmake)

source_group(
	TREE
		${CMAKE_CURRENT_SOURCE_DIR}
	FILES
		${headers}
		${sources}
)

# ---- Create DLL ----

add_library(
	${PROJECT_NAME}
	SHARED
	${headers}
	${sources}
	${CMAKE_CURRENT_BINARY_DIR}/version.rc
	.clang-format
	ScrambledEggchantments.json
)

target_compile_features(
	${PROJECT_NAME}
	PRIVATE
		cxx_std_20
)

target_include_directories(
	${PROJECT_NAME}
	PRIVATE
		${CMAKE_CURRENT_SOURCE_DIR}/src
)

target_link_libraries(
	${PROJECT_NAME}
	PRIVATE
		CommonLibSSE::CommonLibSSE
		nlohmann_json::nlohmann_json
		spdlog::spdlog
)

if (MSVC)
	target_link_options(
		${PROJECT_NAME}
		PRIVATE
			"$<$<CONFIG:DEBUG>:/INCREMENTAL;/OPT:NOREF;/OPT:NOICF>"
			"$<$<CONFIG:RELEASE>:/INCREMENTAL:NO;/OPT:REF;/OPT:ICF;/DEBUG:FULL>"
	)
endif()

target_precompile_headers(
	${PROJECT_NAME}
	PRIVATE
		src/PCH.h
)

# ---- Post build ----

add_custom_command(
	TARGET ${PROJECT_NAME}
	POST_BUILD
	COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:${PROJECT_NAME}> $ENV{Skyrim64Path}/Data/SKSE/Plugins/
	COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_PDB_FILE:${PROJECT_NAME}> $ENV{Skyrim64Path}/Data/SKSE/Plugins/
	COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/ScrambledEggchantments.json $ENV{Skyrim64Path}/Data/SKSE/Plugins/
)
