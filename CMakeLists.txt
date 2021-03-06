# CMake project for FOVE C++ Examples
cmake_minimum_required(VERSION 2.6)
project(FoveCppExamples)

# Find the Fove SDK
if("${CppExampleBuildInTree}")

	# This allows building within the the FOVE SDK source checkout (for FOVE developers)
	message(STATUS "Building FOVE CPP Examples in tree")

else()

	# Setup a FOVE_SDK_PATH variable to allow using a FOVE SDK from a custom location
	# This older variable is no longer used since probably it got cached to an old SDK name
	set(FOVE_SDK_PATH "" CACHE PATH "Directory of extracted FOVE SDK")
	unset(FOVE_SDK_FOLDER CACHE)

	# Find the sdk path (falling back to the built-in one if FOVE_SDK_PATH is not set)
	set(sdk_folder "${CMAKE_CURRENT_LIST_DIR}/FOVE SDK 1.0.0") # Path to builtin sdk folder
	if(NOT "${FOVE_SDK_PATH}" STREQUAL "")
		# Custom error message that suggests using the built-in release if you are pointing to a bad folder
		if(NOT EXISTS "${FOVE_SDK_PATH}" AND EXISTS "${sdk_folder}")
			message(FATAL_ERROR "Unable to find FOVE SDK at ${sdk_folder}\nPlease set FOVE_SDK_PATH to the location of the FOVE SDK.\nSuggested path: ${sdk_folder}")
		endif()
		set(sdk_folder "${FOVE_SDK_PATH}")
	endif()

	# Ensure the sdk folder is valid
	if(NOT EXISTS "${sdk_folder}")
		message(FATAL_ERROR "Unable to find FOVE SDK. Please set FOVE_SDK_PATH to the location of the FOVE SDK.\nNo folder exists at: ${sdk_folder}")
	endif()
	if(NOT EXISTS "${sdk_folder}/FoveAPI.h")
		message(FATAL_ERROR "Unable to find FOVE SDK headers. Please set FOVE_SDK_PATH to the location of the FOVE SDK.\nNo file exists at: ${sdk_folder}/FoveAPI.h")
	endif()
	message(STATUS "Using FOVE SDK at ${sdk_folder}")

	# Locate the client dynamic library
	set(FoveClient "${sdk_folder}/FoveClient.dll")
	if(NOT EXISTS "${FoveClient}")
		message(FATAL_ERROR "Missing FOVE library at ${FoveClient}")
	endif()

	# Setup build parameters
	set(CMAKE_CXX_STANDARD 11)
	add_definitions(-DUNICODE)
	include_directories("${sdk_folder}")
	link_libraries("${sdk_folder}/FoveClient.lib")
endif()

# Determine the utility headers we have for the current platform
if(WIN32)
	set(NativeUtilFiles WIN32 NativeUtil.h WindowsUtil.h WindowsUtil.cpp)
else()
	set(NativeUtilFiles NativeUtil.h)
endif()

# Create DirectX 11 example, and the option to enable/disable it
if(WIN32 AND MSVC)
	option(FOVE_BUILD_DIRECTX11_EXAMPLE "Enable building of the DirectX11 Example" ON)
else()
	option(FOVE_BUILD_DIRECTX11_EXAMPLE "Enable building of the DirectX11 Example" OFF)
endif()
if(FOVE_BUILD_DIRECTX11_EXAMPLE)
	# Declare the DirectX11 example target to cmake
	add_executable(FoveDirectX11Example ${NativeUtilFiles} Shader.vert.hlsl Shader.frag.hlsl DirectX11Example.cpp Util.h Util.cpp DXUtil.h DXUtil.cpp)

	# Tell the build system to compile the shaders to a header file, and give the shader compiler build info like the entry point
	set_source_files_properties(Shader.vert.hlsl PROPERTIES VS_SHADER_FLAGS "/Fh \"$(OutDir)%(Filename)_compiled.h\"" VS_SHADER_TYPE Vertex VS_SHADER_MODEL 4.0 VS_SHADER_ENTRYPOINT vert)
	set_source_files_properties(Shader.frag.hlsl PROPERTIES VS_SHADER_FLAGS "/Fh \"$(OutDir)%(Filename)_compiled.h\"" VS_SHADER_TYPE Pixel  VS_SHADER_MODEL 4.0 VS_SHADER_ENTRYPOINT frag)

	# Include the directory where the compiled shader header files will be put
	# Note: include_directories/target_include_directories directories don't work here
	# Cmake treats $(OutDir) as a relative path and prepends stuff to it to make it absolute
	# Instead, use the compiler flags to add the include directory to avoid changes from cmake
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} \"/I$(OutDir)\"")

	# Add the DirectX11 example to our list of targets which is used below
	list(APPEND alltargets FoveDirectX11Example)
endif()

# Create the OpenGL example, and the option to enable/disable it
if(WIN32)
	option(FOVE_BUILD_OPENGL_EXAMPLE "Enable building of the OpenGL Example" ON)
else()
	option(FOVE_BUILD_OPENGL_EXAMPLE "Enable building of the OpenGL Example" OFF)
endif()
if(FOVE_BUILD_OPENGL_EXAMPLE)
	# Declare platform specific files
	if (WIN32)
		set(FoveOpenGLExamplePlatformSourceFiles DXUtil.h DXUtil.cpp)
	endif()

	# Declare the OpenGL example target to cmake
	add_executable(FoveOpenGLExample ${NativeUtilFiles} OpenGLExample.cpp Util.h Util.cpp OpenGLUtil.h OpenGLUtil.cpp ${FoveOpenGLExamplePlatformSourceFiles})

	# Link to OpenGL libraries
	if (WIN32)
		target_link_libraries(FoveOpenGLExample "opengl32.lib")
	endif()

	# Add the OpenGL example to our list of targets which is used below
	list(APPEND alltargets FoveOpenGLExample)
endif()

# Create the data example, and the option to enable/disable it
option(FOVE_BUILD_DATA_EXAMPLE "Enable building of the Data Example" ON)
if(FOVE_BUILD_DATA_EXAMPLE)
	# Declare the Data example target to cmake
	add_executable(FoveDataExample DataExample.cpp Util.h Util.cpp)

	# Add the Data example to our list of targets which is used below
	list(APPEND alltargets FoveDataExample)
endif()

# Add a post-build command to each target to copy the FoveClient dynamic library to the executable location
# Otherwise the executable will not be able to find the dll, and will fail to launch
if(FoveClient)
	foreach(target ${alltargets})
		add_custom_command(TARGET ${target} POST_BUILD COMMAND ${CMAKE_COMMAND} -E copy_if_different "${FoveClient}" $<TARGET_FILE_DIR:${target}>)
	endforeach()
endif()

# Set default startup project to something other than cmake's ALL_BUILD target
# This way anyone hitting build-and-run will run one of the targets instead of ALL_BUILD which isn't runable
if(MSVC AND NOT "${CppExampleBuildInTree}" AND alltargets)
	list(GET alltargets 0 FirstTarget)
	set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT ${FirstTarget})
endif()
