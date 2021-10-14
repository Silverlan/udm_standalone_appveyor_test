cmake_minimum_required(VERSION 3.12)

set(INCLUDE_DIRS)
function(add_include_dir IDENTIFIER DEFAULT_INCLUDE_DIR)
	set(PRETTYNAME ${IDENTIFIER})
	set(ARGV ${ARGN})
	list(LENGTH ARGV ARGC)
	if(${ARGC} GREATER 0)
		list(GET ARGV 0 PRETTYNAME)
	endif()
	string(TOUPPER ${IDENTIFIER} UIDENTIFIER)

	set(DEPENDENCY_${UIDENTIFIER}_INCLUDE "${DEFAULT_INCLUDE_DIR}" CACHE PATH "Path to ${PRETTYNAME} include directory.")
	set(INCLUDE_DIRS ${INCLUDE_DIRS} DEPENDENCY_${UIDENTIFIER}_INCLUDE PARENT_SCOPE)
endfunction(add_include_dir)

option(WGUI_INCLUDE_DEPENDENCIES "Include dependencies required to build the wgui library?" ON)
function(setup_library_dependency_var LIB_PATH LIB_NAME STATIC_LIB VAR_NAME)
	if(WGUI_INCLUDE_DEPENDENCIES)
		if(WIN32)
			set(LIB_FILE_NAME "${LIB_NAME}.lib")
		else()
			if(STATIC_LIB)
				set(FILE_EXTENSION ".a")
			else()
				set(FILE_EXTENSION ".so")
			endif()
			
			set(LIB_FILE_NAME "${LIB_NAME}${FILE_EXTENSION}")
		endif()
		set(${VAR_NAME} "${LIB_PATH}/${LIB_FILE_NAME}" CACHE PATH "Path to library." FORCE)
	endif()
endfunction(setup_library_dependency_var)

set(LIBRARIES)
function(add_external_library IDENTIFIER DEFAULT_LIBRARY_DIR LIB_NAME STATIC_LIB)
	set(PRETTYNAME ${IDENTIFIER})
	set(ARGV ${ARGN})
	list(LENGTH ARGV ARGC)
	if(${ARGC} GREATER 0)
		list(GET ARGV 0 PRETTYNAME)
	endif()
	string(TOUPPER ${IDENTIFIER} UIDENTIFIER)

	setup_library_dependency_var("${DEFAULT_LIBRARY_DIR}" "${LIB_NAME}" ${STATIC_LIB} DEPENDENCY_${UIDENTIFIER}_LIBRARY)
	# set(DEPENDENCY_${UIDENTIFIER}_LIBRARY "" CACHE FILEPATH "Path to ${PRETTYNAME} library.")
	set(LIBRARIES ${LIBRARIES} DEPENDENCY_${UIDENTIFIER}_LIBRARY PARENT_SCOPE)
endfunction(add_external_library)

function(link_external_library IDENTIFIER DEFAULT_INCLUDE_DIR DEFAULT_LIBRARY_DIR LIB_NAME STATIC_LIB)
	set(PRETTYNAME ${IDENTIFIER})
	set(ARGV ${ARGN})
	list(LENGTH ARGV ARGC)
	if(${ARGC} GREATER 0)
		list(GET ARGV 0 PRETTYNAME)
	endif()
	string(TOUPPER ${IDENTIFIER} UIDENTIFIER)

	set(DEPENDENCY_${UIDENTIFIER}_INCLUDE "${DEFAULT_INCLUDE_DIR}" CACHE PATH "Path to ${PRETTYNAME} include directory.")
	set(INCLUDE_DIRS ${INCLUDE_DIRS} DEPENDENCY_${UIDENTIFIER}_INCLUDE PARENT_SCOPE)

	setup_library_dependency_var("${DEFAULT_LIBRARY_DIR}" "${LIB_NAME}" ${STATIC_LIB} DEPENDENCY_${UIDENTIFIER}_LIBRARY)
	# set(DEPENDENCY_${UIDENTIFIER}_LIBRARY "${DEFAULT_LIBRARY_DIR}" CACHE FILEPATH "Path to ${PRETTYNAME} library.")
	set(LIBRARIES ${LIBRARIES} DEPENDENCY_${UIDENTIFIER}_LIBRARY PARENT_SCOPE)
endfunction(link_external_library)

function(register_third_party_library LIB_NAME)
	message("Processing third-party library '${LIB_NAME}'...")
	set(extra_macro_args ${ARGN})

	list(LENGTH extra_macro_args num_extra_args)
	if(${num_extra_args} GREATER 0)
		list(GET extra_macro_args 0 optional_arg)
		add_subdirectory(third_party_libs/${LIB_NAME} third_party_libs/${optional_arg})
		return()
	endif()
	add_subdirectory(third_party_libs/${LIB_NAME})
endfunction(register_third_party_library)

##### CONFIGURATION #####

set(PROJ_NAME udm_standalone)

project(${PROJ_NAME} CXX)

set(CMAKE_CXX_STANDARD 20)
set(ENABLE_STATIC_LIBRARY_FLAG 0)

add_definitions(-DSHUTIL_STATIC)
add_definitions(-DMUTIL_STATIC)
add_definitions(-DVFILESYSTEM_STATIC)
add_definitions(-DGLM_FORCE_DEPTH_ZERO_TO_ONE)
set(MATHUTIL_ENABLE_MESH_FUNCTIONS OFF CACHE BOOL OFF FORCE)
set(MATHUTIL_STATIC ON CACHE BOOL ON FORCE)
set(SHAREDUTILS_STATIC ON CACHE BOOL ON FORCE)
set(VFILESYSTEM_STATIC ON CACHE BOOL ON FORCE)
set(UTIL_UDM_STATIC OFF CACHE BOOL OFF FORCE)

# Build Type
if(WIN32)
	set(CONFIG_BUILD_TYPE "RelWithDebInfo" CACHE STRING "Build Type")
else()
	set(CONFIG_BUILD_TYPE "Release" CACHE STRING "Build Type")
endif()
set_property(CACHE CONFIG_BUILD_TYPE PROPERTY STRINGS RelWithDebInfo Release Debug)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)
if(UNIX)
	set(CMAKE_POSITION_INDEPENDENT_CODE ON)
endif()
set(CMAKE_BUILD_TYPE ${CONFIG_BUILD_TYPE} CACHE STRING "" FORCE)
#

link_external_library(sharedutils_static "${CMAKE_CURRENT_LIST_DIR}/external_libs/sharedutils/include" "${CMAKE_CURRENT_BINARY_DIR}/external_libs/sharedutils-static/${CMAKE_BUILD_TYPE}" "sharedutils-static" 1)
link_external_library(sharedutils "${CMAKE_CURRENT_LIST_DIR}/external_libs/sharedutils/include" "${CMAKE_CURRENT_BINARY_DIR}/external_libs/sharedutils-static/${CMAKE_BUILD_TYPE}" "sharedutils-static" 1)
set(DEPENDENCY_SHAREDUTILS_LIBRARY_STATIC ${DEPENDENCY_SHAREDUTILS_STATIC_LIBRARY} CACHE STRING "" FORCE)
link_external_library(mathutil "${CMAKE_CURRENT_LIST_DIR}/external_libs/mathutil/include" "${CMAKE_CURRENT_BINARY_DIR}/external_libs/mathutil/${CMAKE_BUILD_TYPE}" "mathutil" 1)
link_external_library(vfilesystem "${CMAKE_CURRENT_LIST_DIR}/external_libs/vfilesystem/include" "${CMAKE_CURRENT_BINARY_DIR}/external_libs/vfilesystem/${CMAKE_BUILD_TYPE}" "vfilesystem" 1)
link_external_library(util_udm "${CMAKE_CURRENT_LIST_DIR}/external_libs/util_udm/include" "${CMAKE_CURRENT_BINARY_DIR}/external_libs/util_udm/${CMAKE_BUILD_TYPE}" "util_udm" 0)
link_external_library(lz4 "${CMAKE_CURRENT_LIST_DIR}/third_party_libs/lz4/lib" "${CMAKE_CURRENT_BINARY_DIR}/third_party_libs/lz4-cmake/${CMAKE_BUILD_TYPE}" "lz4" 1)
add_include_dir(glm "${CMAKE_CURRENT_LIST_DIR}/third_party_libs/glm")
add_include_dir(bzip2 "${CMAKE_CURRENT_LIST_DIR}/third_party_libs/bzip2")

add_subdirectory("external_libs")
add_subdirectory("third_party_libs")

set(DEFINITIONS)

##### CONFIGURATION #####

set(LIB_TYPE SHARED)
if(ENABLE_STATIC_LIBRARY_FLAG)
	option(CONFIG_STATIC_LIBRARY "Build as static library?" OFF)
endif()
if(DEFINED ${CONFIG_STATIC_LIBRARY} AND ${CONFIG_STATIC_LIBRARY})
	set(LIB_TYPE STATIC)
endif()

foreach(def IN LISTS DEFINITIONS)
	add_definitions(-D${def})
endforeach(def)

function(def_vs_filters FILE_LIST)
	foreach(source IN LISTS FILE_LIST)
	    get_filename_component(source_path "${source}" PATH)
	    string(REPLACE "${CMAKE_CURRENT_LIST_DIR}" "" source_path_relative "${source_path}")
	    string(REPLACE "/" "\\" source_path_msvc "${source_path_relative}")
	    source_group("${source_path_msvc}" FILES "${source}")
	endforeach()
endfunction(def_vs_filters)

file(GLOB_RECURSE SRC_FILES
    "${CMAKE_CURRENT_LIST_DIR}/src/*.h"
    "${CMAKE_CURRENT_LIST_DIR}/src/*.hpp"
    "${CMAKE_CURRENT_LIST_DIR}/src/*.cpp"
    "${CMAKE_CURRENT_LIST_DIR}/include/*.h"
    "${CMAKE_CURRENT_LIST_DIR}/include/*.hpp"
)
# add_library(${PROJ_NAME} ${LIB_TYPE} ${SRC_FILES})
add_executable(${PROJ_NAME} ${SRC_FILES})
if(WIN32)
	target_compile_options(${PROJ_NAME} PRIVATE /wd4251)
	target_compile_options(${PROJ_NAME} PRIVATE /wd4996)
endif()
def_vs_filters("${SRC_FILES}")

foreach(LIB IN LISTS LIBRARIES)
	target_link_libraries(${PROJ_NAME} ${${LIB}})
endforeach(LIB)

target_include_directories(${PROJ_NAME} PRIVATE ${CMAKE_CURRENT_LIST_DIR}/include)
target_include_directories(${PROJ_NAME} PRIVATE ${CMAKE_CURRENT_LIST_DIR}/src)

foreach(INCLUDE_PATH IN LISTS INCLUDE_DIRS)
	target_include_directories(${PROJ_NAME} PRIVATE ${${INCLUDE_PATH}})
endforeach(INCLUDE_PATH)

set(TARGET_PROPERTIES LINKER_LANGUAGE CXX)
set_target_properties(${PROJ_NAME} PROPERTIES ${TARGET_PROPERTIES})

add_dependencies(util_udm mathutil vfilesystem sharedutils-static lz4)
add_dependencies(${PROJ_NAME} util_udm)
