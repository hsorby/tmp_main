CMAKE_MINIMUM_REQUIRED(VERSION 2.4 FATAL_ERROR)
project(OpenCMISS)

# ==============================
# Initial setup instructions
# ==============================
option(OCM_DEVELOPER_MODE "Handle the packages as proper submodules. DEVELOPERS ONLY." ON)

option(BUILD_IRON "Build OpenCMISS-Iron" YES)
option(BUILD_ZINC "Build OpenCMISS-Iron" NO)

if (NOT CMAKE_BUILD_TYPE AND NOT WIN32)
    SET(CMAKE_BUILD_TYPE RELEASE)
    message(STATUS "No CMAKE_BUILD_TYPE has been defined. Using RELEASE.")
endif()

SET(OPENCMISS_UTILITIES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/utilities)
SET(OPENCMISS_DEPENDENCIES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/dependencies)
SET(OPENCMISS_INSTALL_PREFIX ${CMAKE_CURRENT_SOURCE_DIR}/install)

# Check utilities first (this ensures that CMake is new enough and other tools are available)
add_subdirectory(utilities)

LIST(APPEND CMAKE_MODULE_PATH ${OPENCMISS_UTILITIES_DIR}/CMakeModules ${OPENCMISS_UTILITIES_DIR}/CMakeScripts)
include(BuildMacros)
include(ExternalProject)

if (BUILD_IRON)
    SET(PROJECT_CMAKE_ARGS )
    # As the CMAKE_ARGS are a list themselves, we need to treat the ; in the (possible) list of module_paths
	# specially. Therefore CMAKE has a special command $<SEMICOLON>
	STRING(REPLACE ";" "$<SEMICOLON>" CMAKE_MODULE_PATH_ESC "${CMAKE_MODULE_PATH}")
	LIST(APPEND PROJECT_CMAKE_ARGS
	    -DCMAKE_INSTALL_PREFIX:PATH=${OPENCMISS_INSTALL_PREFIX}
	    -DCMAKE_BUILD_TYPE:PATH=${CMAKE_BUILD_TYPE}
	    #-DBUILD_PRECISION=${BUILD_PRECISION}
	    #-DBUILD_TESTS=${BUILD_TESTS}
	    -DCMAKE_PREFIX_PATH=${OPENCMISS_DEPENDENCIES_DIR}/install/release/lib
	    -DCMAKE_MODULE_PATH=${CMAKE_MODULE_PATH_ESC}
	)
	
    SET(PROJECT_FOLDER ${CMAKE_CURRENT_BINARY_DIR}/iron)
    GET_BUILD_COMMANDS(LOCAL_PLATFORM_BUILD_COMMAND LOCAL_PLATFORM_INSTALL_COMMAND ${PROJECT_FOLDER})
    ExternalProject_Add(IRON #DEPENDS ${${PROJECT_NAME}_DEPS}
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
	if (OCM_DEVELOPER_MODE)
	    #ADD_SUBMODULE_CHECKOUT_STEPS(IRON ${OpenCMISS_SOURCE_DIR} iron NO)
    endif()
endif()