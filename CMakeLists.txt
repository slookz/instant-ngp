# Copyright (c) 2021-2022, NVIDIA CORPORATION.  All rights reserved.
#
# NVIDIA CORPORATION and its licensors retain all intellectual property
# and proprietary rights in and to this software, related documentation
# and any modifications thereto.  Any use, reproduction, disclosure or
# distribution of this software and related documentation without an express
# license agreement from NVIDIA CORPORATION is strictly prohibited.

cmake_minimum_required(VERSION 3.18)

project(instant-ngp
	VERSION 2.0
	DESCRIPTION "Instant Neural Graphics Primitives"
	LANGUAGES C CXX CUDA
)
set(NGP_VERSION "${CMAKE_PROJECT_VERSION}")

if (NOT NGP_DEPLOY)
	set(NGP_VERSION "${NGP_VERSION}dev")
endif()

option(NGP_BUILD_EXECUTABLE "Build instant-ngp.exe?" ON)
option(NGP_BUILD_WITH_GUI "Build with GUI support (requires GLFW and GLEW)?" ON)
option(NGP_BUILD_WITH_OPTIX "Build with OptiX to enable hardware ray tracing?" ON)
option(NGP_BUILD_WITH_PYTHON_BINDINGS "Build bindings that allow instrumenting instant-ngp with Python?" ON)
option(NGP_BUILD_WITH_RTC "Build support for runtime compilation of fully fused kernels?" ON)
option(NGP_BUILD_WITH_VULKAN "Build with Vulkan to enable DLSS support?" ON)

set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

###############################################################################
# Build type and C++ compiler setup
###############################################################################

# Set a default configuration if none was specified
if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
	message(STATUS "No release type specified. Setting to 'Release'.")
	set(CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build." FORCE)
	set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "RelWithDebInfo")
endif()

if (NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/glfw/CMakeLists.txt")
	message(FATAL_ERROR
		"Some instant-ngp dependencies are missing. "
		"If you forgot the \"--recursive\" flag when cloning this project, "
		"this can be fixed by calling \"git submodule update --init --recursive\"."
	)
endif()

if (APPLE)
	set(CMAKE_MACOSX_RPATH ON)
endif()

if (CMAKE_EXPORT_COMPILE_COMMANDS)
    set(CMAKE_CXX_STANDARD_INCLUDE_DIRECTORIES ${CMAKE_CXX_IMPLICIT_INCLUDE_DIRECTORIES})
    set(CMAKE_CUDA_STANDARD_INCLUDE_DIRECTORIES ${CMAKE_CUDA_IMPLICIT_INCLUDE_DIRECTORIES})
endif()

if (MSVC)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /D_CRT_SECURE_NO_WARNINGS")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP24")
else()
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
endif()

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_EXTENSIONS OFF)

###############################################################################
# CUDA compiler setup
###############################################################################

set(CMAKE_CUDA_STANDARD 14)
set(CMAKE_CUDA_STANDARD_REQUIRED ON)
set(CMAKE_CUDA_EXTENSIONS OFF)
set(CUDA_LINK_LIBRARIES_KEYWORD PUBLIC)
set(CMAKE_CUDA_RUNTIME_LIBRARY Shared)

if (MSVC)
	list(APPEND CUDA_NVCC_FLAGS "-Xcompiler=/bigobj")
else()
	list(APPEND CUDA_NVCC_FLAGS "-Xcompiler=-Wno-float-conversion")
	list(APPEND CUDA_NVCC_FLAGS "-Xcompiler=-fno-strict-aliasing")
	list(APPEND CUDA_NVCC_FLAGS "-Xcompiler=-fPIC")
endif()
list(APPEND CUDA_NVCC_FLAGS "--extended-lambda")
list(APPEND CUDA_NVCC_FLAGS "--expt-relaxed-constexpr")
list(APPEND CUDA_NVCC_FLAGS "--use_fast_math")

###############################################################################
# Dependencies
###############################################################################

set(TCNN_BUILD_BENCHMARK OFF)
set(TCNN_BUILD_EXAMPLES OFF)
set(TCNN_BUILD_WITH_RTC ${NGP_BUILD_WITH_RTC})
set(TCNN_ALLOW_CUBLAS_CUSOLVER OFF)
add_subdirectory(dependencies/tiny-cuda-nn)

set(CMAKE_CUDA_ARCHITECTURES ${TCNN_CUDA_ARCHITECTURES})

if (NGP_BUILD_WITH_GUI)
	find_package(Vulkan)
	if (Vulkan_FOUND AND NGP_BUILD_WITH_VULKAN)
		set(NGP_VULKAN ON)
		list(APPEND NGP_DEFINITIONS -DNGP_VULKAN -DGLFW_INCLUDE_VULKAN)
		list(APPEND NGP_INCLUDE_DIRECTORIES "${Vulkan_INCLUDE_DIRS}")
		list(APPEND NGP_LIBRARIES ${Vulkan_LIBRARIES})

		list(APPEND GUI_SOURCES src/dlss.cu)

		# DLSS depends on vulkan, so appears here
		list(APPEND NGP_INCLUDE_DIRECTORIES "dependencies/dlss/include")
		if (MSVC)
			list(APPEND NGP_LINK_DIRECTORIES "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/dlss/lib/Windows_x86_64/x86_64")
			list(APPEND NGP_LIBRARIES "$<IF:$<CONFIG:Debug>,nvsdk_ngx_d_dbg,nvsdk_ngx_d>")
		else()
			list(APPEND NGP_LINK_DIRECTORIES "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/dlss/lib/Linux_x86_64")
			list(APPEND NGP_LIBRARIES nvsdk_ngx)
		endif()
	else()
		set(NGP_VULKAN OFF)
		if (NGP_BUILD_WITH_VULKAN)
			message(WARNING
				"Vulkan was not found. Instant neural graphics primitives will still "
				"compile and run correctly, but DLSS will not be supported."
			)
		endif()
	endif()

	# OpenXR
	if (WIN32)
		list(APPEND NGP_DEFINITIONS -DXR_USE_PLATFORM_WIN32 -DGLFW_EXPOSE_NATIVE_WGL)
	elseif (UNIX AND NOT APPLE)
		list(APPEND NGP_DEFINITIONS -DGLFW_EXPOSE_NATIVE_GLX)
		if (JK_USE_WAYLAND)
			set(PRESENTATION_BACKEND wayland CACHE STRING " " FORCE)
			set(BUILD_WITH_XLIB_HEADERS OFF CACHE BOOL " " FORCE)
			set(BUILD_WITH_XCB_HEADERS OFF CACHE BOOL " " FORCE)
			set(BUILD_WITH_WAYLAND_HEADERS ON CACHE BOOL " " FORCE)
			list(APPEND NGP_DEFINITIONS -DGLFW_EXPOSE_NATIVE_WAYLAND -DXR_USE_PLATFORM_WAYLAND)
		else()
			set(PRESENTATION_BACKEND xlib CACHE STRING " " FORCE)
			set(BUILD_WITH_XLIB_HEADERS ON CACHE BOOL " " FORCE)
			set(BUILD_WITH_XCB_HEADERS OFF CACHE BOOL " " FORCE)
			set(BUILD_WITH_WAYLAND_HEADERS OFF CACHE BOOL " " FORCE)
			list(APPEND NGP_DEFINITIONS -DGLFW_EXPOSE_NATIVE_X11 -DXR_USE_PLATFORM_XLIB)
		endif()
	else()
		message(FATAL_ERROR "No OpenXR platform set for this OS")
	endif()

	add_subdirectory(dependencies/OpenXR-SDK)

	list(APPEND NGP_INCLUDE_DIRECTORIES "dependencies/OpenXR-SDK/include" "dependencies/OpenXR-SDK/src/common")
	list(APPEND NGP_LIBRARIES openxr_loader)
	list(APPEND GUI_SOURCES src/openxr_hmd.cu)

	# OpenGL
	find_package(OpenGL REQUIRED)

	# GLFW
	set(GLFW_BUILD_EXAMPLES OFF CACHE BOOL " " FORCE)
	set(GLFW_BUILD_TESTS OFF CACHE BOOL " " FORCE)
	set(GLFW_BUILD_DOCS OFF CACHE BOOL " " FORCE)
	set(GLFW_BUILD_INSTALL OFF CACHE BOOL " " FORCE)
	set(GLFW_INSTALL OFF CACHE BOOL " " FORCE)
	set(GLFW_USE_CHDIR OFF CACHE BOOL " " FORCE)
	set(GLFW_VULKAN_STATIC OFF CACHE BOOL " " FORCE)
	set(BUILD_SHARED_LIBS ON CACHE BOOL " " FORCE)

	add_subdirectory(dependencies/glfw)

	set_target_properties(glfw PROPERTIES EXCLUDE_FROM_ALL 1 EXCLUDE_FROM_DEFAULT_BUILD 1)

	mark_as_advanced(
		GLFW_BUILD_DOCS GLFW_BUILD_EXAMPLES GLFW_BUILD_INSTALL GLFW_BUILD_TESTS
		GLFW_DOCUMENT_INTERNALS GLFW_INSTALL GLFW_USE_CHDIR GLFW_USE_MENUBAR
		GLFW_USE_OSMESA GLFW_VULKAN_STATIC GLFW_USE_RETINA GLFW_USE_MIR
		BUILD_SHARED_LIBS USE_MSVC_RUNTIME_LIBRARY_DLL
	)

	list(APPEND NGP_INCLUDE_DIRECTORIES "dependencies/glfw/include" "dependencies/imgui")

	if (MSVC)
		list(APPEND NGP_INCLUDE_DIRECTORIES "dependencies/gl3w")
		list(APPEND GUI_SOURCES "dependencies/gl3w/GL/gl3w.c")
		list(APPEND NGP_LIBRARIES opengl32 $<TARGET_OBJECTS:glfw_objects>)
	else()
		find_package(GLEW REQUIRED)
		list(APPEND NGP_INCLUDE_DIRECTORIES ${GLEW_INCLUDE_DIRS})
		list(APPEND NGP_LIBRARIES GL ${GLEW_LIBRARIES} $<TARGET_OBJECTS:glfw_objects>)
	endif()

	list(APPEND GUI_SOURCES
		dependencies/imguizmo/ImGuizmo.cpp
		dependencies/imgui/imgui.cpp
		dependencies/imgui/backends/imgui_impl_glfw.cpp
		dependencies/imgui/backends/imgui_impl_opengl3.cpp
		dependencies/imgui/imgui_draw.cpp
		dependencies/imgui/imgui_tables.cpp
		dependencies/imgui/imgui_widgets.cpp
		dependencies/imgui/misc/cpp/imgui_stdlib.cpp
	)

	list(APPEND NGP_DEFINITIONS -DNGP_GUI)
endif(NGP_BUILD_WITH_GUI)

list(APPEND NGP_INCLUDE_DIRECTORIES
	"dependencies"
	"dependencies/filesystem"
	"dependencies/nanovdb"
	"dependencies/NaturalSort"
	"dependencies/tinylogger"
)

find_package(OpenMP)
if (OPENMP_FOUND)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
endif()

if (NGP_BUILD_WITH_OPTIX)
	set(NGP_OPTIX ON)
	list(APPEND NGP_INCLUDE_DIRECTORIES "dependencies/optix")
	list(APPEND NGP_DEFINITIONS -DNGP_OPTIX)
endif()

if (NGP_BUILD_WITH_PYTHON_BINDINGS)
	find_package(Python 3.7 COMPONENTS Interpreter Development)
	if (Python_FOUND)
		add_subdirectory("dependencies/pybind11")
	endif()
endif()

# Compile zlib (only on Windows)
if (WIN32)
	set(ZLIB_USE_STATIC_LIBS ON CACHE BOOL " " FORCE)
	set(ZLIB_BUILD_STATIC_LIBS ON CACHE BOOL " " FORCE)
	set(ZLIB_BUILD_SHARED_LIBS OFF CACHE BOOL " " FORCE)
	set(SKIP_INSTALL_ALL ON CACHE BOOL " " FORCE)
	add_subdirectory("dependencies/zlib")
	set_property(TARGET zlibstatic PROPERTY FOLDER "dependencies")

	set(ZLIB_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/zlib" CACHE PATH " " FORCE)
	set(ZLIB_LIBRARY zlibstatic)

	include_directories(${ZLIB_INCLUDE_DIR} "${CMAKE_CURRENT_BINARY_DIR}/dependencies/zlib")

	list(APPEND NGP_LIBRARIES zlibstatic)
endif()

add_subdirectory("dependencies/zstr")
list(APPEND NGP_LIBRARIES zstr::zstr)


###############################################################################
# Program
###############################################################################

list(APPEND NGP_DEFINITIONS -DNGP_VERSION="${NGP_VERSION}")
list(APPEND NGP_INCLUDE_DIRECTORIES "include")
if (NOT MSVC)
	list(APPEND NGP_LIBRARIES ${CMAKE_DL_LIBS})
endif()
list(APPEND NGP_SOURCES
	${GUI_SOURCES}
	src/camera_path.cu
	src/common_host.cu
	src/marching_cubes.cu
	src/nerf_loader.cu
	src/render_buffer.cu
	src/testbed.cu
	src/testbed_image.cu
	src/testbed_nerf.cu
	src/testbed_sdf.cu
	src/testbed_volume.cu
	src/thread_pool.cpp
	src/tinyexr_wrapper.cu
	src/tinyobj_loader_wrapper.cu
	src/triangle_bvh.cu
)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELEASE ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_RELWITHDEBINFO ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_MINSIZEREL ${CMAKE_BINARY_DIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY_DEBUG ${CMAKE_BINARY_DIR})

get_filename_component(CUDA_COMPILER_BIN "${CMAKE_CUDA_COMPILER}" DIRECTORY)
get_filename_component(CUDA_DIR "${CUDA_COMPILER_BIN}" DIRECTORY)
set(CUDA_INCLUDE "${CUDA_DIR}/include")

if (NGP_OPTIX)
	add_library(optix_program OBJECT
		src/optix/pathescape.cu
		src/optix/raystab.cu
		src/optix/raytrace.cu
	)

	set_target_properties(optix_program PROPERTIES CUDA_PTX_COMPILATION ON CUDA_ARCHITECTURES OFF)
	target_compile_definitions(optix_program PUBLIC ${NGP_DEFINITIONS} -DTCNN_MIN_GPU_ARCH=0)
	target_compile_options(optix_program PUBLIC "--expt-relaxed-constexpr")

	get_target_property(TCNN_INCLUDE_DIRECTORIES tiny-cuda-nn INCLUDE_DIRECTORIES)
	target_include_directories(optix_program PUBLIC ${NGP_INCLUDE_DIRECTORIES} ${TCNN_INCLUDE_DIRECTORIES})

	# OptiX programs will be compiled as PTX and packaged
	# as headers to be included from the binary dir.
	list(APPEND NGP_INCLUDE_DIRECTORIES "${CMAKE_CURRENT_BINARY_DIR}")
	set(OPTIX_PTX_HEADER ${CMAKE_CURRENT_BINARY_DIR}/optix_ptx.h)

	find_program(bin_to_c NAMES bin2c PATHS ${CUDA_COMPILER_BIN})
	if (NOT bin_to_c)
		message(FATAL_ERROR
			"bin2c not found:\n"
			"  CMAKE_CUDA_COMPILER='${CMAKE_CUDA_COMPILER}'\n"
			"  CUDA_COMPILER_BIN='${CUDA_COMPILER_BIN}'\n"
		)
	endif()

	add_custom_command(
		OUTPUT "${OPTIX_PTX_HEADER}"
		COMMAND ${CMAKE_COMMAND}
		"-DBIN_TO_C_COMMAND=${bin_to_c}"
		"-DOBJECTS=$<TARGET_OBJECTS:optix_program>"
		"-DOUTPUT=${OPTIX_PTX_HEADER}"
		-P ${PROJECT_SOURCE_DIR}/cmake/bin2c_wrapper.cmake
		VERBATIM
		DEPENDS $<TARGET_OBJECTS:optix_program>
		COMMENT "Converting PTX files to a C header"
	)

	list(APPEND NGP_SOURCES ${OPTIX_PTX_HEADER})
endif()

cmrc_add_resource_library(ngp-resources NAMESPACE ngp)
list(APPEND NGP_LIBRARIES ngp-resources)
if (NGP_BUILD_WITH_RTC)
	# Create local directory that can serve as cache for the runtime compiled artifacts
	file(MAKE_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/rtc/cache")

	file(GLOB_RECURSE NGP_HEADERS "${CMAKE_CURRENT_SOURCE_DIR}/include/neural-graphics-primitives/*")
	cmrc_add_resources(ngp-resources WHENCE "${CMAKE_CURRENT_SOURCE_DIR}/include" ${NGP_HEADERS})
endif()

add_library(ngp STATIC ${NGP_SOURCES})
set_target_properties(ngp PROPERTIES CUDA_RESOLVE_DEVICE_SYMBOLS ON CUDA_SEPARABLE_COMPILATION ON)
target_compile_definitions(ngp PUBLIC ${NGP_DEFINITIONS})
target_compile_options(ngp PUBLIC $<$<COMPILE_LANGUAGE:CUDA>:${CUDA_NVCC_FLAGS}>)
target_include_directories(ngp PUBLIC ${NGP_INCLUDE_DIRECTORIES})
target_link_directories(ngp PUBLIC ${NGP_LINK_DIRECTORIES})
target_link_libraries(ngp PUBLIC ${NGP_LIBRARIES} tiny-cuda-nn)

# Copy shared libraries to the binary directory as needed
if (NGP_VULKAN)
	set(NGX_BUILD_DIR "$<IF:$<CONFIG:Debug>,dev,rel>")
	if (MSVC)
		set(NGX_SHARED_LIB "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/dlss/lib/Windows_x86_64/${NGX_BUILD_DIR}/nvngx_dlss.dll")
	else()
		set(NGX_SHARED_LIB "${CMAKE_CURRENT_SOURCE_DIR}/dependencies/dlss/lib/Linux_x86_64/${NGX_BUILD_DIR}/libnvidia-ngx-dlss.so.*")
	endif()

	add_custom_command(TARGET ngp POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy "${NGX_SHARED_LIB}" "${CMAKE_CURRENT_BINARY_DIR}" COMMAND_EXPAND_LISTS)
endif()

if (MSVC)
	file(GLOB CUDA_DLLS "${CUDA_COMPILER_BIN}/cudart64*.dll" "${CUDA_COMPILER_BIN}/nvrtc*.dll")
	list(FILTER CUDA_DLLS EXCLUDE REGEX "\\.alt\\.dll$")
	if (CUDA_DLLS)
		add_custom_command(TARGET ngp POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${CUDA_DLLS} "${CMAKE_CURRENT_BINARY_DIR}" COMMAND_EXPAND_LISTS)
	endif()
endif()

if (NGP_BUILD_EXECUTABLE)
	add_executable(instant-ngp src/main.cu)
	target_link_libraries(instant-ngp PRIVATE ngp)

	# Link the executable to the project directory and copy over DLLs such that instant-ngp can be invoked without going into the build folder.
	set(NGP_BINARY_FILE "\"${CMAKE_CURRENT_SOURCE_DIR}/$<TARGET_FILE_NAME:instant-ngp>\"")
	if (MSVC)
		add_custom_command(TARGET instant-ngp POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy $<TARGET_FILE:instant-ngp> ${CMAKE_CURRENT_SOURCE_DIR})
		file(GLOB NGP_REQUIRED_DLLS "${CUDA_COMPILER_BIN}/cudart64*.dll" "${CUDA_COMPILER_BIN}/nvrtc*.dll")
		list(FILTER NGP_REQUIRED_DLLS EXCLUDE REGEX "\\.alt\\.dll$")
		if (NGP_VULKAN)
			list(APPEND NGP_REQUIRED_DLLS "${NGX_SHARED_LIB}")
		endif()
		message("NGP_REQUIRED_DLLS: ${NGP_REQUIRED_DLLS}")
		if (NGP_REQUIRED_DLLS)
			add_custom_command(TARGET instant-ngp POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy ${NGP_REQUIRED_DLLS} "${CMAKE_CURRENT_SOURCE_DIR}" COMMAND_EXPAND_LISTS)
		endif()
	else()
		add_custom_command(TARGET instant-ngp POST_BUILD COMMAND ${CMAKE_COMMAND} -E create_symlink $<TARGET_FILE:instant-ngp> "${NGP_BINARY_FILE}")

		if (NGP_VULKAN)
			add_custom_command(TARGET instant-ngp POST_BUILD COMMAND "ln" -s -f "${NGX_SHARED_LIB}" "${CMAKE_CURRENT_SOURCE_DIR}/")
		endif()
	endif()
endif(NGP_BUILD_EXECUTABLE)

if (Python_FOUND)
	add_library(pyngp SHARED src/python_api.cu)
	set_target_properties(pyngp PROPERTIES CXX_VISIBILITY_PRESET "hidden" CUDA_VISIBILITY_PRESET "hidden")
	target_link_libraries(pyngp PRIVATE ngp PUBLIC ${PYTHON_LIBRARIES} pybind11::module)
	target_compile_definitions(pyngp PUBLIC -DNGP_PYTHON)
	pybind11_extension(pyngp)
endif()
