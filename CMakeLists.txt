CMAKE_MINIMUM_REQUIRED(VERSION 2.4 FATAL_ERROR)
project(OpenCMISS)

#include(utilities/FunctionDefinitions.cmake)
#PRINT_VARS()
#return()

# ==========================
# Initial setup instructions
# ==========================
if (NOT WIN32 AND NOT CMAKE_BUILD_TYPE)
    SET(CMAKE_BUILD_TYPE RELEASE)
    message(STATUS "No CMAKE_BUILD_TYPE has been defined. Using RELEASE.")
endif()

# These are default locations for utilities and dependencies
SET(OPENCMISS_UTILITIES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/utilities)
SET(OPENCMISS_DEPENDENCIES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/dependencies)

# Check utilities first (this ensures that CMake is new enough and other tools are available)
add_subdirectory(utilities)

# Make sure the localconfig file exists
if (NOT EXISTS ${CMAKE_CURRENT_LIST_DIR}/OpenCMISSLocalConfig.cmake)
    configure_file(${CMAKE_CURRENT_LIST_DIR}/OpenCMISSLocalConfig.default.cmake ${CMAKE_CURRENT_LIST_DIR}/OpenCMISSLocalConfig.cmake COPYONLY)
endif()

# Set up include path
LIST(APPEND CMAKE_MODULE_PATH 
    ${OPENCMISS_UTILITIES_DIR}/CMakeFindModuleWrappers
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${OPENCMISS_UTILITIES_DIR}/CMakeModules
    ${OPENCMISS_UTILITIES_DIR}/CMakeScripts)
include(OCMUtilsBuildMacros)
include(OCMUtilsArchitecture)
include(ExternalProject)
# Read local config
include(OpenCMISSLocalConfig)
# If an MPI home folder was specified make a check here if it works!
if (MPI_HOME)
    find_package(MPI MODULE REQUIRED)
endif()
#message(STATUS "MPI_C_COMPILER: ${MPI_C_COMPILER}, MPI_CXX_COMPILER: ${MPI_CXX_COMPILER}")

GET_ARCHITECTURE_PATH(ARCHITECTURE_PATH)
SET(CMAKE_INSTALL_PREFIX ${CMAKE_CURRENT_SOURCE_DIR}/install/${ARCHITECTURE_PATH})
SET(CMAKE_DEPENDENCIES_INSTALL_PREFIX ${OPENCMISS_DEPENDENCIES_DIR}/install/${ARCHITECTURE_PATH})

# Manual for now
SET(NEED_DEPENDENCIES YES)

if (NEED_DEPENDENCIES)
    # Use one build dir as all dependencies therein will use the architecture path 
    SET(OPENCMISS_DEPENDENCIES_BINARY_DIR ${OPENCMISS_DEPENDENCIES_DIR}/build)
    SET(PROJECT_CMAKE_ARGS )
    # As the CMAKE_ARGS are a list themselves, we need to treat the ; in the (possible) list of module_paths
	# specially. Therefore CMAKE has a special command $<SEMICOLON>
	STRING(REPLACE ";" "$<SEMICOLON>" CMAKE_MODULE_PATH_ESC "${CMAKE_MODULE_PATH}")
	LIST(APPEND PROJECT_CMAKE_ARGS
	    -DOPENCMISS_BUILD=YES # Flag for simple check
	    -DCMAKE_INSTALL_PREFIX=${CMAKE_DEPENDENCIES_INSTALL_PREFIX}
	    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
	    -DCMAKE_MODULE_PATH=${CMAKE_MODULE_PATH_ESC}
	    -DARCHITECTURE_PATH=${ARCHITECTURE_PATH}
	)
	
    GET_BUILD_COMMANDS(LOCAL_PLATFORM_BUILD_COMMAND LOCAL_PLATFORM_INSTALL_COMMAND ${OPENCMISS_DEPENDENCIES_BINARY_DIR})
    #message(STATUS "OpenCMISS dependencies extra args:\n${PROJECT_CMAKE_ARGS}")
    ExternalProject_Add(DEPENDENCIES
        PREFIX ${OPENCMISS_DEPENDENCIES_DIR}
		TMP_DIR ${OPENCMISS_DEPENDENCIES_BINARY_DIR}/ep_tmp
		STAMP_DIR ${OPENCMISS_DEPENDENCIES_BINARY_DIR}/ep_stamp
		#--Configure step-------------
		SOURCE_DIR ${OPENCMISS_DEPENDENCIES_DIR}
		BINARY_DIR ${OPENCMISS_DEPENDENCIES_BINARY_DIR}
		CMAKE_ARGS ${PROJECT_CMAKE_ARGS}
		#--Build step-----------------
		BUILD_COMMAND ${LOCAL_PLATFORM_BUILD_COMMAND}
		INSTALL_COMMAND ""
	)
	SET(IRON_DEPS DEPENDS DEPENDENCIES)
endif()

if (BUILD_IRON)
    SET(PROJECT_CMAKE_ARGS )
    # As the CMAKE_ARGS are a list themselves, we need to treat the ; in the (possible) list of module_paths
	# specially. Therefore CMAKE has a special command $<SEMICOLON>
	STRING(REPLACE ";" "$<SEMICOLON>" CMAKE_MODULE_PATH_ESC "${CMAKE_MODULE_PATH}")
	LIST(APPEND PROJECT_CMAKE_ARGS
	    -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
	    -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
	    #-DBUILD_PRECISION=${BUILD_PRECISION}
	    #-DBUILD_TESTS=${BUILD_TESTS}
	    -DCMAKE_PREFIX_PATH=${CMAKE_DEPENDENCIES_INSTALL_PREFIX}/lib/cmake
	    -DCMAKE_MODULE_PATH=${CMAKE_MODULE_PATH_ESC}
	)
	if (MPI_HOME)
        LIST(APPEND PROJECT_CMAKE_ARGS
            -DMPI_HOME=${MPI_HOME}
        )
    endif()
	
    SET(PROJECT_FOLDER ${CMAKE_CURRENT_BINARY_DIR}/iron/${ARCHITECTURE_PATH})
    GET_BUILD_COMMANDS(LOCAL_PLATFORM_BUILD_COMMAND LOCAL_PLATFORM_INSTALL_COMMAND ${PROJECT_FOLDER})
    ExternalProject_Add(IRON 
        ${IRON_DEPS}
        PREFIX ${PROJECT_FOLDER}
		TMP_DIR ${PROJECT_FOLDER}/ep_tmp
		STAMP_DIR ${PROJECT_FOLDER}/ep_stamp
		#--Download step--------------
		DOWNLOAD_DIR ${PROJECT_FOLDER}/ep_dl
        ${USERMODE_DOWNLOAD_CMDS}
		#--Configure step-------------
		SOURCE_DIR ${OpenCMISS_SOURCE_DIR}/iron
		BINARY_DIR ${PROJECT_FOLDER}
		CMAKE_ARGS ${PROJECT_CMAKE_ARGS}
		#--Build step-----------------
		BUILD_COMMAND ${LOCAL_PLATFORM_BUILD_COMMAND}
		#--Install step---------------
		INSTALL_COMMAND ${LOCAL_PLATFORM_INSTALL_COMMAND}
	)
	#if (OCM_DEVELOPER_MODE)
	#    ADD_SUBMODULE_CHECKOUT_STEPS(IRON ${OpenCMISS_SOURCE_DIR} iron NO)
    #endif()
endif()