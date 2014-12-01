CMAKE_MINIMUM_REQUIRED(VERSION 2.4 FATAL_ERROR)

#include(utilities/FunctionDefinitions.cmake)
#PRINT_VARS()
#return()

# These are default locations for utilities and dependencies
SET(OPENCMISS_UTILITIES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/utilities)
SET(OPENCMISS_DEPENDENCIES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/dependencies)

# ==========================
# First-time only instructions
# Make sure the localconfig file exists
if (NOT EXISTS ${CMAKE_CURRENT_LIST_DIR}/OpenCMISSLocalConfig.cmake)
    configure_file(${OPENCMISS_UTILITIES_DIR}/OpenCMISSLocalConfig.template.cmake
        ${CMAKE_CURRENT_LIST_DIR}/OpenCMISSLocalConfig.cmake COPYONLY)
endif()

# Set up include path
LIST(APPEND CMAKE_MODULE_PATH 
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${OPENCMISS_UTILITIES_DIR}/CMakeFindModuleWrappers
    ${OPENCMISS_UTILITIES_DIR}/CMakeModules
    ${OPENCMISS_UTILITIES_DIR}/CMakeScripts)

# Check utilities first (this ensures that CMake is new enough and other tools are available)
include(CMakeCheck)

# Start actual project
CMAKE_MINIMUM_REQUIRED(VERSION 3.0 FATAL_ERROR)
project(OpenCMISS VERSION 1.0 LANGUAGES C CXX Fortran)
if (NOT WIN32 AND NOT CMAKE_BUILD_TYPE)
    SET(CMAKE_BUILD_TYPE RELEASE)
    message(STATUS "No CMAKE_BUILD_TYPE has been defined. Using RELEASE.")
endif()

# Load macros
include(OCMUtilsBuildMacros)
include(OCMUtilsArchitecture)
include(ExternalProject)

# Include utilities dir (thus far, will probably be own external project)

########################################################################
# Read configuration (defaults & local)
include(OpenCMISSConfig)

# ABI detection
#if (NOT ABI)
    # Rough detection if not specified..
#    if (${CMAKE_SYSTEM_PROCESSOR} MATCHES 64)
#        SET(ABI 64)
#    else()
#        SET(ABI 32)
#    endif()
#endif()
#foreach(lang C CXX Fortran)
#    SET(CMAKE_${lang}_FLAGS "-m${ABI} ${CMAKE_${lang}_FLAGS}")
#endforeach()

if (OCM_USE_MPI)
    find_package(MPI MODULE QUIET)
endif()
#message(STATUS "MPI_C_COMPILER: ${MPI_C_COMPILER}, MPI_CXX_COMPILER: ${MPI_CXX_COMPILER}")

if(OCM_USE_MT)
    find_package(OpenMP REQUIRED)
endif()

####################################
# Collect common arguments
SET(PROJECT_CMAKE_ARGS )
# As the CMAKE_ARGS are a list themselves, we need to treat the ; in the (possible) list of module_paths
# specially. Therefore CMAKE has a special command $<SEMICOLON>
STRING(REPLACE ";" "$<SEMICOLON>" CMAKE_MODULE_PATH_ESC "${CMAKE_MODULE_PATH}")
LIST(APPEND PROJECT_CMAKE_ARGS
    -DOPENCMISS_BUILD=YES # Flag for simple check
    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
    -DCMAKE_MODULE_PATH=${CMAKE_MODULE_PATH_ESC}
    -DBUILD_SHARED_LIBS=${BUILD_SHARED_LIBS}
)
# Add compilers and flags
foreach(lang C CXX Fortran)
    if (CMAKE_${lang}_FLAGS)
        LIST(APPEND PROJECT_CMAKE_ARGS
            -DCMAKE_${lang}_FLAGS=${CMAKE_${lang}_FLAGS}
        )
    endif()
    if(CMAKE_${lang}_COMPILER)
        LIST(APPEND PROJECT_CMAKE_ARGS
            -DCMAKE_${lang}_COMPILER=${CMAKE_${lang}_COMPILER}
        )
    endif()
endforeach()
#message(STATUS "OpenCMISS common external project args:\n${PROJECT_CMAKE_ARGS}")

####################################
# Set up installation paths (iron, zinc)
get_architecture_path(ARCHITECTURE_PATH)
# Extra path segment for single configuration - will give release/debug/...
get_build_type_extra(BUILDTYPEEXTRA)
# everything from the OpenCMISS main project goes into ./install/...
# (except utilities and dependencies, they have their own install subtree)
SET(CMAKE_INSTALL_PREFIX ${CMAKE_CURRENT_SOURCE_DIR}/install/${ARCHITECTURE_PATH}/${BUILDTYPEEXTRA})

########################################################################
# Utilities
########################################################################
SET(OPENCMISS_UTILITIES_BINARY_DIR ${OPENCMISS_UTILITIES_DIR}/build) #/${ARCHITECTURE_PATH}
GET_BUILD_COMMANDS(BUILD_COMMAND DUMMY_INSTALL_COMMAND ${OPENCMISS_UTILITIES_BINARY_DIR} FALSE)
ExternalProject_Add(UTILITIES
    PREFIX ${OPENCMISS_UTILITIES_DIR}
	TMP_DIR ${OPENCMISS_UTILITIES_BINARY_DIR}/ep_tmp
	STAMP_DIR ${OPENCMISS_UTILITIES_BINARY_DIR}/ep_stamp
	#--Configure step-------------
	CMAKE_COMMAND ${CMAKE_COMMAND} --no-warn-unused-cli
	SOURCE_DIR ${OPENCMISS_UTILITIES_DIR}
	BINARY_DIR ${OPENCMISS_UTILITIES_BINARY_DIR}
	CMAKE_ARGS ${PROJECT_CMAKE_ARGS}
	#--Build step-----------------
	BUILD_COMMAND ${BUILD_COMMAND}
	INSTALL_COMMAND "" # No install for utilities main project
)

########################################################################
# Dependencies
########################################################################
SET(OPENCMISS_DEPENDENCIES_BINARY_DIR ${OPENCMISS_DEPENDENCIES_DIR}/build/${ARCHITECTURE_PATH})
# Dependencies go into dependencies subfolder
SET(OPENCMISS_DEPENDENCIES_INSTALL_PREFIX )
GET_BUILD_COMMANDS(BUILD_COMMAND DUMMY_INSTALL_COMMAND ${OPENCMISS_DEPENDENCIES_BINARY_DIR} FALSE)
# Add dependency-specific arguments
SET(DEPENDENCIES_CMAKE_ARGS ${PROJECT_CMAKE_ARGS})
LIST(APPEND DEPENDENCIES_CMAKE_ARGS
    -DCMAKE_INSTALL_PREFIX=${OPENCMISS_DEPENDENCIES_DIR}/install/${ARCHITECTURE_PATH}/${BUILDTYPEEXTRA}
)
ExternalProject_Add(DEPENDENCIES
    DEPENDS UTILITIES
    PREFIX ${OPENCMISS_DEPENDENCIES_DIR}
	TMP_DIR ${OPENCMISS_DEPENDENCIES_BINARY_DIR}/ep_tmp
	STAMP_DIR ${OPENCMISS_DEPENDENCIES_BINARY_DIR}/ep_stamp
	#--Configure step-------------
	CMAKE_COMMAND ${CMAKE_COMMAND} --no-warn-unused-cli
	SOURCE_DIR ${OPENCMISS_DEPENDENCIES_DIR}
	BINARY_DIR ${OPENCMISS_DEPENDENCIES_BINARY_DIR}
	CMAKE_ARGS ${DEPENDENCIES_CMAKE_ARGS}
	#--Build step-----------------
	BUILD_COMMAND ${BUILD_COMMAND}
	INSTALL_COMMAND "" # No install for dependencies main project
)

########################################################################
# Remove the CMake Cache of utils/dependencies when calling make clean!
# Otherwise, changes in the localconfig wont be propagated
set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
    PROPERTY ADDITIONAL_MAKE_CLEAN_FILES 
    ${OPENCMISS_UTILITIES_BINARY_DIR}/CMakeCache.txt
    ${OPENCMISS_DEPENDENCIES_BINARY_DIR}/CMakeCache.txt
)

########################################################################
# Iron
########################################################################
if (BUILD_IRON)
    SET(IRON_CMAKE_ARGS ${PROJECT_CMAKE_ARGS})
	LIST(APPEND IRON_CMAKE_ARGS
	    -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
	)
	# iron will include config directly?! (later)
	if (MPI_HOME)
        LIST(APPEND IRON_CMAKE_ARGS
            -DMPI_HOME=${MPI_HOME}
        )
    endif()
	
    SET(PROJECT_FOLDER ${CMAKE_CURRENT_BINARY_DIR}/iron/${ARCHITECTURE_PATH})
    GET_BUILD_COMMANDS(LOCAL_PLATFORM_BUILD_COMMAND LOCAL_PLATFORM_INSTALL_COMMAND ${PROJECT_FOLDER} TRUE)
    ExternalProject_Add(IRON 
        DEPENDS DEPENDENCIES
        PREFIX ${PROJECT_FOLDER}
		TMP_DIR ${PROJECT_FOLDER}/ep_tmp
		STAMP_DIR ${PROJECT_FOLDER}/ep_stamp
		#--Download step--------------
		DOWNLOAD_DIR ${PROJECT_FOLDER}/ep_dl
        ${USERMODE_DOWNLOAD_CMDS}
		#--Configure step-------------
		CMAKE_COMMAND ${CMAKE_COMMAND} --no-warn-unused-cli
		SOURCE_DIR ${OpenCMISS_SOURCE_DIR}/iron
		BINARY_DIR ${PROJECT_FOLDER}
		CMAKE_ARGS ${IRON_CMAKE_ARGS}
		#--Build step-----------------
		BUILD_COMMAND ${LOCAL_PLATFORM_BUILD_COMMAND}
		#--Install step---------------
		INSTALL_COMMAND ${LOCAL_PLATFORM_INSTALL_COMMAND}
	)
	#if (OCM_DEVELOPER_MODE)
	#    ADD_SUBMODULE_CHECKOUT_STEPS(IRON ${OpenCMISS_SOURCE_DIR} iron NO)
    #endif()
endif()