# Important: The Basis Universal encoder and transcoder libraries must be compiled with -fno-strict-aliasing (MSVC's default, and also the Linux kernel).
# It should also work without this option, but we do not test with it.
cmake_minimum_required(VERSION 3.10)

project(basisu C CXX)
set(CMAKE_CXX_STANDARD 17)

option(BASISU_SANITIZE "Enable sanitize" FALSE)
option(BASISU_BUILD_EXECUTABLES "Build executables" TRUE)
option(BASISU_BUILD_EXAMPLES "Build examples" TRUE)
option(BASISU_STATIC "Static link executables" FALSE)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/bin)

# For MSVC builds default to SSE enabled, and determine if it's a 64-bit (-A x64) vs. 32-bit (-A Win32) build.
if (MSVC)
    option(SSE "SSE 4.1 support" TRUE)
    if ( CMAKE_GENERATOR_PLATFORM STREQUAL Win32 )
        set(BUILD_X64 0)
    else()
        set(BUILD_X64 1)
    endif()
	add_compile_options(/W4)
else()
    option(SSE "SSE 4.1 support" FALSE)
    option(BUILD_X64 "build 64-bit" TRUE)
endif()

option(ZSTD "ZSTD support for KTX2 transcoding/encoding" TRUE)
option(OPENCL "OpenCL support in encoder" FALSE)

message("Initial BUILD_X64=${BUILD_X64}")
message("Initial CMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}")
message("Initial SSE=${SSE}")
message("Initial ZSTD=${ZSTD}")
message("Initial OPENCL=${OPENCL}")
message("Initial SANITIZE=${BASISU_SANITIZE}")

if ((NOT MSVC) AND OPENCL)
    # With MSVC builds we use the Khronos lib/include files in the project's "OpenCL" directory, to completely avoid requiring fiddly to install vendor SDK's.
    # Otherwise we use the system's (if any).
    find_package(OpenCL)
    message(STATUS "OpenCL found: ${OPENCL_FOUND}")
    message(STATUS "OpenCL includes: ${OpenCL_INCLUDE_DIRS}")
    message(STATUS "OpenCL libraries: ${OpenCL_LIBRARIES}")
endif()

if( NOT CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release )
endif()

message(${PROJECT_NAME} " build type: " ${CMAKE_BUILD_TYPE})

if (BUILD_X64)
    message("Building 64-bit")
else()
    message("Building 32-bit")
endif()

if (SSE)
    message("SSE enabled")
else()
    message("SSE disabled")
endif()

if (ZSTD)
    message("Zstandard enabled")
else()
    message("Zstandard disabled")
endif()

function(configure_basisu_target target_name is_executable)

	# configure SSE support
	if (SSE AND NOT EMSCRIPTEN)
		target_compile_definitions(${target_name} PRIVATE -DBASISU_SUPPORT_SSE=1)
		if (NOT MSVC)
			target_compile_options(${target_name} PRIVATE -msse4.1)
		endif()
	else()
		target_compile_definitions(${target_name} PRIVATE -DBASISU_SUPPORT_SSE=0)
	endif()

	# address sanitizer
	if (NOT MSVC AND BASISU_SANITIZE)
		message(STATUS "Enabling sanitizers for ${target_name}")
		target_compile_options(${target_name} PRIVATE
			-fsanitize=address
			-fno-omit-frame-pointer
			-fsanitize=undefined
			-fno-sanitize=alignment
		)
		target_link_options(${target_name} PRIVATE
			-fsanitize=address
			-fsanitize=undefined
		)
	endif()

	if (NOT MSVC)
		# common compiler flags
		target_compile_options(${target_name} PRIVATE
			-fvisibility=hidden
			-fPIC
			-fno-strict-aliasing
			-Wall
			-Wextra
			-Wno-unused-local-typedefs
			-Wno-unused-value
			-Wno-unused-parameter
			-Wno-unused-variable
			-Wno-misleading-indentation
			-Wno-unused-function
			-Wno-maybe-uninitialized
			-Wno-stringop-overflow
		)

		# C++ specific warnings
		target_compile_options(${target_name} PRIVATE
			$<$<COMPILE_LANGUAGE:CXX>:-Wno-reorder>
			$<$<COMPILE_LANGUAGE:CXX>:-Wno-deprecated-copy>
			$<$<COMPILE_LANGUAGE:CXX>:-Wno-class-memaccess>
		)

		# Common preprocessor definitions
	 	 target_compile_definitions(${target_name} PRIVATE
	        _LARGEFILE64_SOURCE=1
	        _FILE_OFFSET_BITS=64
	        $<$<CONFIG:Debug>:_DEBUG>
	    )

		# 32-bit build support
	    if(NOT BUILD_X64)
	        target_compile_options(${target_name} PRIVATE -m32)
	        target_link_options(${target_name} PRIVATE -m32)
	    endif()

		# Platform-specific configurations
		if(EMSCRIPTEN)
			# target_compile_options(${target_name} PRIVATE -s ALLOW_MEMORY_GROWTH=1)
	        target_link_options(${target_name} PRIVATE -s ALLOW_MEMORY_GROWTH=1)
	    elseif(BASISU_STATIC)
	        target_link_options(${target_name} PRIVATE
	            -static-libgcc
	            -static-libstdc++
	            -static
	        )
	    else()
	        target_link_options(${target_name} PRIVATE -Wl,-rpath,.)
	    endif()

		# OpenCL support
		if (OPENCL AND OPENCL_FOUND)
			# For Non-Windows builds, let cmake try and find the system OpenCL headers/libs for us.
			target_compile_definitions(${target_name} PRIVATE -DBASISU_SUPPORT_OPENCL=1)
			target_include_directories(${target_name} PRIVATE ${OpenCL_INCLUDE_DIRS})
			target_link_libraries(${target_name} PRIVATE ${OpenCL_LIBRARIES})
		endif()

		# linker extra libraries
		if (is_executable)
			target_link_libraries(${target_name} PRIVATE m)
			if (NOT ANDROID)
				target_link_libraries(${target_name} PRIVATE m pthread)
			endif()
		endif()
	else()
		if (OPENCL)
			# For Windows builds, we use our local copies of the OpenCL import lib and Khronos headers.
			target_compile_definitions(${target_name} PRIVATE -DBASISU_SUPPORT_OPENCL=1)
			target_include_directories(${target_name} PRIVATE "OpenCL")

			if (BUILD_X64)
				target_link_libraries(${target_name} PRIVATE "${CMAKE_SOURCE_DIR}/OpenCL/lib/OpenCL64.lib")
			else()
				target_link_libraries(${target_name} PRIVATE "${CMAKE_SOURCE_DIR}/OpenCL/lib/OpenCL.lib")
			endif()
		endif()
	endif()

	if (is_executable)
		target_link_libraries(${target_name} PRIVATE basisu_encoder)
	endif()

	if (ZSTD)
		target_compile_definitions(${target_name} PRIVATE BASISD_SUPPORT_KTX2_ZSTD=1)
	else()
		target_compile_definitions(${target_name} PRIVATE BASISD_SUPPORT_KTX2_ZSTD=0)
	endif()

endfunction()


# Define the source files for the static library
set(ENCODER_LIB_SRC_LIST
    encoder/basisu_backend.cpp
    encoder/basisu_basis_file.cpp
    encoder/basisu_comp.cpp
    encoder/basisu_enc.cpp
    encoder/basisu_etc.cpp
    encoder/basisu_frontend.cpp
    encoder/basisu_gpu_texture.cpp
    encoder/basisu_pvrtc1_4.cpp
    encoder/basisu_resampler.cpp
    encoder/basisu_resample_filters.cpp
    encoder/basisu_ssim.cpp
    encoder/basisu_uastc_enc.cpp
    encoder/basisu_bc7enc.cpp
    encoder/jpgd.cpp
    encoder/basisu_kernels_sse.cpp
    encoder/basisu_opencl.cpp
    encoder/pvpngreader.cpp
    encoder/basisu_uastc_hdr_4x4_enc.cpp
    encoder/basisu_astc_hdr_6x6_enc.cpp
    encoder/basisu_astc_hdr_common.cpp
    encoder/3rdparty/android_astc_decomp.cpp
    encoder/3rdparty/tinyexr.cpp
    transcoder/basisu_transcoder.cpp
)

if (ZSTD)
    set(ENCODER_LIB_SRC_LIST ${ENCODER_LIB_SRC_LIST} zstd/zstd.c)
endif()

# Create the static library
add_library(basisu_encoder STATIC ${ENCODER_LIB_SRC_LIST})
configure_basisu_target(basisu_encoder FALSE)

# Create the basisu executable and link against the static library
if (BASISU_BUILD_EXECUTABLES)
	add_executable(basisu basisu_tool.cpp)
	configure_basisu_target(basisu TRUE)
	if (MSVC)
		set_target_properties(basisu PROPERTIES
			RUNTIME_OUTPUT_NAME "basisu"
			RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
			RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
			RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
			RUNTIME_OUTPUT_DIRECTORY_MINSIZEREL ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
		)
	endif()
endif()

# Create the new example executable and link against the static library
if (BASISU_BUILD_EXAMPLES)
	add_executable(examples example/example.cpp)
	configure_basisu_target(examples TRUE)

	if (MSVC)
		set_target_properties(examples PROPERTIES
			RUNTIME_OUTPUT_NAME "examples"
			RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
			RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
			RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
			RUNTIME_OUTPUT_DIRECTORY_MINSIZEREL ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
		)
	endif()
endif()

if (NOT EMSCRIPTEN AND BASISU_BUILD_EXECUTABLES)
    if (UNIX)
        if (CMAKE_BUILD_TYPE STREQUAL Release)
            if (APPLE)
                add_custom_command(TARGET basisu POST_BUILD COMMAND strip -X -x ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/basisu)
				#message("strip command: strip -X -x ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/basisu")
            else()
                add_custom_command(TARGET basisu POST_BUILD COMMAND strip -g -X -x ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/basisu)
				#message("strip command: strip -g -X -x ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/basisu")
            endif()
        endif()
    endif()
endif()


