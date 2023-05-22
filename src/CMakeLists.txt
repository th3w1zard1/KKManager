cmake_minimum_required(VERSION 3.5)

# Set variables
get_filename_component(VENDOR_SRC_DIR "${VENDOR_SRC_DIR}" REALPATH)
get_filename_component(UNZIPPED_DIR "${UNZIPPED_DIR}" REALPATH)
get_filename_component(ROOT_DIR "${ROOT_DIR}" REALPATH)
get_filename_component(CMAKE_INSTALL_PREFIX "${CMAKE_INSTALL_PREFIX}" REALPATH)
get_filename_component(CMAKE_SOURCE_DIR "${CMAKE_SOURCE_DIR}" REALPATH)

# Print variables
message("CMake: VENDOR_SRC_DIR: ${VENDOR_SRC_DIR}")
message("CMake: UNZIPPED_DIR: ${UNZIPPED_DIR}")
message("CMake: ROOT_DIR: ${ROOT_DIR}")
message("CMake: CMAKE_INSTALL_PREFIX: ${CMAKE_INSTALL_PREFIX}")
message("CMake: CMAKE_SOURCE_DIR: ${CMAKE_SOURCE_DIR}")
project(CWrapperGen)
enable_language(CXX)

# Export all symbols
set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)

# Use the default vcpkg path if it's installed with VS 2019
if (DEFINED ENV{VCPKG_ROOT} AND NOT DEFINED VCPKG_DIR)
    set(VCPKG_DIR "$ENV{VCPKG_ROOT}")
endif()

# Set the vcpkg path if it's not already defined
if (NOT DEFINED VCPKG_DIR)
    set(VCPKG_DIR ${CMAKE_INSTALL_PREFIX}/vcpkg)
endif()

# Set the toolchain file to use vcpkg's cmake toolchain
set(CMAKE_TOOLCHAIN_FILE ${VCPKG_DIR}/scripts/buildsystems/vcpkg.cmake CACHE STRING "")

# Stop CMake if an error occurs
cmake_policy(SET CMP0076 NEW)
set(CMAKE_ERROR_STOP ON)

# Unzip source directories
# message("Unzipping zipped source directories in '${VENDOR_SRC_DIR}/*.zip' into '${UNZIPPED_DIR}/'")
# file(GLOB ZIP_FILES ${VENDOR_SRC_DIR}/*.zip)
# foreach(ZIP_FILE ${ZIP_FILES})
#     get_filename_component(PROJECT_NAME ${ZIP_FILE} NAME_WE)
#     execute_process(COMMAND ${CMAKE_COMMAND} -E make_directory ${UNZIPPED_DIR})
#     execute_process(COMMAND ${CMAKE_COMMAND} -E tar xf ${ZIP_FILE} WORKING_DIRECTORY ${UNZIPPED_DIR} RESULT_VARIABLE RESULT)
#     if(RESULT EQUAL "0")
#         message("Zip file successfully extracted to ${UNZIPPED_DIR}/${PROJECT_NAME}.")
#     else()
#         message(FATAL_ERROR "Error extracting zip file ${PROJECT_NAME}: ${RESULT}")
#     endif()
#     file(GLOB_RECURSE EXISTING_FILES "${UNZIPPED}/*")
# endforeach()

function(get_version_number input_string output_variable)
  # Use regex to find the largest possible version number in the input string
  string(REGEX MATCHALL "[0-9]+([._][0-9]+)*" version_numbers ${input_string})

  # Check if version_numbers is empty
  if (NOT version_numbers)
    set(${output_variable} "" PARENT_SCOPE)
  else()
    # Set the output variable to the last version number found
    list(GET version_numbers -1 last_version_number)
    set(${output_variable} ${last_version_number} PARENT_SCOPE)
  endif()
endfunction()

function(get_max_version max_version VERSION_NUMBER)
    get_version_number(max_version, VERSION_NUMBER)
    if(VERSION_NUMBER GREATER max_version)
        set(${FOUND_MAX_VERSION} ${max_version} PARENT_SCOPE)
    endif()
endfunction()

set(FOUND_MAX_VERSION "")
set(FOUND_PROJECT_PATH "")
function(find_project_recursive PROJECT_NAME MIN_VERSION)
    set(SEARCH_DEPTH 1)
    set(SEARCH_PATH ${CMAKE_CURRENT_SOURCE_DIR})
    while(SEARCH_DEPTH GREATER 0)
        file(GLOB SUBDIRS RELATIVE ${SEARCH_PATH} ${SEARCH_PATH}/*)
        foreach(SUBDIR ${SUBDIRS})
            # message("Searching ${SUBDIR}")
            if(IS_DIRECTORY ${SEARCH_PATH}/${SUBDIR})
                string(TOUPPER ${SUBDIR} UPPER_SUBDIR)
                # message("Checking ${UPPER_SUBDIR}")
                if(UPPER_SUBDIR MATCHES ".*${PROJECT_NAME}.*")
                    message("Found ${PROJECT_NAME} in ${SUBDIR}")
                    set(VERSION_NUMBER "")
                    get_version_number(${SUBDIR} VERSION_NUMBER)
                    message("Version number found: ${VERSION_NUMBER}")
                    if(VERSION_NUMBER VERSION_GREATER_EQUAL ${MIN_VERSION})
                        if(VERSION_NUMBER VERSION_GREATER FOUND_MAX_VERSION)
                            message("Acceptable version ${VERSION_NUMBER} found for ${PROJECT_NAME}")
                            set(FOUND_MAX_VERSION ${VERSION_NUMBER} PARENT_SCOPE)
                            set(FOUND_PROJECT_PATH ${CMAKE_CURRENT_SOURCE_DIR}/${SUBDIR} PARENT_SCOPE)
                            return()
                        endif()
                    endif()
                    file(GLOB SUBDIR_FILES ${SEARCH_PATH}/${SUBDIR}/*)
                    list(LENGTH SUBDIR_FILES NUM_FILES)
                    if(NOT NUM_FILES EQUAL 0)
                        if(SEARCH_DEPTH GREATER 0)
                            math(EXPR SEARCH_DEPTH "${SEARCH_DEPTH}-1")
                        endif()
                    endif()
                endif()
            endif()
        endforeach()
        if(SEARCH_DEPTH GREATER 0)
            math(EXPR SEARCH_DEPTH "${SEARCH_DEPTH}-1")
            set(SEARCH_PATH ${SEARCH_PATH}/..)
        endif()
    endwhile()
endfunction()



##
## not all dependencies are defined, you should use vcpkg.
##
set(
    DEPENDENCY_PACKAGES
        PCRE2
        ZLib
        BZip2
        7zip
)


##
## Find the required projects or build it if not found
##
set(
    BUILD_ORDER
        VCPKG
        PkgConfig
        CLANG-TIDY
        CPPCHECK
        SWIG
        PNGCheck
)
set(
    MIN_REQUIRED_VERSIONS
        0.0.0
        0.0.0
        0.0.0
        0.0.0
        4.0.2
        0.0.0
)
set(
    BUILD_INSTRUCTIONS # from source
        vcpkg
        none
        make
        cmake
        make
)
set(
    VCPKG_INSTALL_NAMES
        vcpkg
        pkgconf
        clang-tidy
        cppcheck
        swig
        sourceonly
)
set(${I} -1)
set(VCPKG_EXE_DIR "")
foreach(PROJECT_NAME ${BUILD_ORDER})
    math(EXPR I "${I}+1")
    find_package(${PROJECT_NAME} QUIET)
    list(GET VCPKG_INSTALL_NAMES ${I} pkg_name)
    message("pkg_name: ${pkg_name}")
    if(NOT ${PROJECT_NAME}_FOUND AND NOT ${PROJECT_NAME} STREQUAL "VCPKG" AND NOT ${pkg_name} STREQUAL "sourceonly")
        message(STATUS "${PROJECT_NAME} not found. Installing via vcpkg...")
        message(STATUS "${VCPKG_EXE_DIR} install --recurse ${pkg_name} --triplet x86-windows")
        execute_process(
            COMMAND "${VCPKG_EXE_DIR}" install --recurse ${pkg_name} --triplet x86-windows
            OUTPUT_VARIABLE my_output
            ERROR_VARIABLE my_error
            RESULT_VARIABLE my_result
        )
        if(my_result)
            message(STATUS "VCPKG install failed with error code '${my_result}'")
        endif()

        if(my_output)
            message(STATUS "Command output: '${my_output}'")
        endif()

        if(my_error)
            message(STATUS "Command error output: '${my_error}'")
        endif()
    endif()
    if(NOT ${PROJECT_NAME}_FOUND)
        pkg_check_modules(AUTOGEN_DEPENDENCIES REQUIRED IMPORTED_TARGET ${PROJECT_NAME})
        # Check if dependencies are installed and install them using vcpkg if not
        foreach(DEP ${AUTOGEN_DEPENDENCIES})
            if(NOT TARGET ${DEP}::${DEP})
                message(STATUS "Dependency ${DEP} not found. Installing with vcpkg...")
                execute_process(
                    COMMAND vcpkg install ${DEP}
                    RESULT_VARIABLE VCPKG_INSTALL_RESULT
                )
                if(VCPKG_INSTALL_RESULT)
                    message(FATAL_ERROR "Failed to install ${DEP} with vcpkg")
                endif()
            endif()
        endforeach()
    endif()
    if(NOT ${PROJECT_NAME}_FOUND)
        ##################
        # foreach(DEPENDENCY_NAME ${DEPENDENCY_PACKAGES})
        # install the hundreds of tiny dependencies one by one the same way as below, you should get vcpkg instead.
        # endforeach()
        ##################
        if(DEFINED PARENT_SCOPE)
            set(FOUND_MAX_VERSION "" PARENT_SCOPE)
            set(FOUND_PROJECT_PATH "" PARENT_SCOPE)
        else()
            set(FOUND_MAX_VERSION "")
            set(FOUND_PROJECT_PATH "")
        endif()

        list(GET MIN_REQUIRED_VERSIONS ${I} PROJECT_MIN_VERSION)
        message(STATUS "${PROJECT_NAME} ${PROJECT_MIN_VERSION} not found. Searching for ${PROJECT_NAME} source files...")
        set(CMAKE_CURRENT_SOURCE_DIR ${UNZIPPED_DIR})
        string(TOUPPER ${PROJECT_NAME} PROJECT_NAME)
        find_project_recursive(${PROJECT_NAME} ${PROJECT_MIN_VERSION})

        if (${PROJECT_NAME} STREQUAL "VCPKG")
            message("Installing vcpkg to resolve required dependencies for our packages")
            execute_process(
                COMMAND ${CMAKE_COMMAND} -E chdir ${FOUND_PROJECT_PATH} "${FOUND_PROJECT_PATH}/bootstrap-vcpkg.bat"
                RESULT_VARIABLE VCPKG_BOOTSTRAP_RESULT
            )
            if(VCPKG_BOOTSTRAP_RESULT)
                message(FATAL_ERROR "Failed to bootstrap vcpkg")
            endif()
            set(VCPKG_EXE_DIR ${FOUND_PROJECT_PATH}/vcpkg.exe)
            continue()
        endif()
        message("FOUND_MAX_VERSION: ${FOUND_MAX_VERSION} FOUND_PROJECT_PATH: ${FOUND_PROJECT_PATH}")
        if(FOUND_PROJECT_PATH STREQUAL "")
            message(FATAL_ERROR "Cound not find ${PROJECT_NAME} source in ${FOUND_PROJECT_PATH}")
        elseif(FOUND_MAX_VERSION VERSION_GREATER ${PROJECT_NAME}_MIN_VERSION)
            message(STATUS "Attempting to build ${PROJECT_NAME} from source in ${FOUND_PROJECT_PATH}...")
            if(EXISTS ${FOUND_PROJECT_PATH}/CMakeLists.txt)
                message("Using CMakeLists.txt for project ${PROJECT_NAME}")
                add_subdirectory(${FOUND_PROJECT_PATH})
                include(${FOUND_PROJECT_PATH}/CMakeLists.txt)
            endif()
            file(MAKE_DIRECTORY "${FOUND_PROJECT_PATH}/build")
            list(GET BUILD_INSTRUCTIONS ${I} PROJECT_BUILD_INSTRUCTIONS)
            if(PROJECT_BUILD_INSTRUCTIONS STREQUAL "make")
                execute_process(
                    COMMAND ${CMAKE_COMMAND} -E make_directory "${FOUND_PROJECT_PATH}/build"
                    COMMAND ${CMAKE_COMMAND} -E chdir "${FOUND_PROJECT_PATH}/build" "${FOUND_PROJECT_PATH}/configure"
                    COMMAND ${CMAKE_COMMAND} -E chdir "${FOUND_PROJECT_PATH}/build" make
                    COMMAND ${CMAKE_COMMAND} -E chdir "${FOUND_PROJECT_PATH}/build" make install
                )
            endif()
            set(${PROJECT_NAME}_EXECUTABLE "${FOUND_PROJECT_PATH}/build/${PROJECT_NAME}" PARENT_SCOPE)
            if(EXISTS "${${PROJECT_NAME}_EXECUTABLE}")
                message(STATUS "Built ${PROJECT_NAME} ${FOUND_MAX_VERSION} executable: ${${PROJECT_NAME}_EXECUTABLE}")
                set(${PROJECT_NAME}_FOUND TRUE)
            endif()
        else()
            message(STATUS "PROJECT ${PROJECT_NAME} has a minimum version of ${PROJECT_MIN_VERSION} but max version found is ${FOUND_MAX_VERSION}")
        endif()
    endif()
    if (PROJECT_NAME STREQUAL "PNGCHECK")
        set(CMAKE_C_STANDARD 99)
        message("Adding executable ${PROJECT_NAME} with source file ${FOUND_PROJECT_PATH}/pngcheck.c")
        add_executable(${PROJECT_NAME} ${ABSOLUTE_PROJECT_PATH}/pngcheck.c)
        target_link_libraries(${PROJECT_NAME} png z)
    endif()
endforeach()

# Use SWIG to wrap C/C++ projects for C#
find_package(SWIG REQUIRED)
include(${SWIG_USE_FILE})

if(SWIG_FOUND)
    message("SWIG found: ${SWIG_EXECUTABLE}")
    set(CMAKE_SWIG_FLAGS "")
    set_source_files_properties(pngcheck.i PROPERTIES CPLUSPLUS ON)
    set_source_files_properties(pngcheck.i PROPERTIES SWIG_FLAGS "-includeall")
    swig_add_library(pngcheck TYPE SHARED LANGUAGE python SOURCES pngcheck.i ${SOURCES})
    swig_link_libraries(pngcheck png z)
endif()

# Set project properties
function(set_project_properties PROJECT_NAME)
    set_target_properties(${PROJECT_NAME} PROPERTIES
        POSITION_INDEPENDENT_CODE ON
        C_VISIBILITY_PRESET hidden
        CXX_VISIBILITY_PRESET hidden
        VISIBILITY_INLINES_HIDDEN ON
    )
    message("Set target properties for ${PROJECT_NAME}")
endfunction()

# Use SWIG to wrap C/C++ projects
foreach(PROJECT_DIR ${PROJECT_DIRS})
    if(IS_DIRECTORY ${PROJECT_DIR})
        if (${PROJECT_DIR}/CMakeLists.txt)
            include(${PROJECT_DIR}/CMakeLists.txt)
        endif()
        get_filename_component(PROJECT_NAME ${PROJECT_DIR} NAME)
        file(GLOB_RECURSE PROJECT_SRCS ${PROJECT_DIR}/*.c*)
        if(PROJECT_SRCS)
            set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -D__cplusplus -x c++ -externc -D__stdcall=__attribute__((stdcall)) -D__cdecl=__attribute__((cdecl))")
            swig_add_library(${PROJECT_NAME}_wrap
                TYPE SHARED
                LANGUAGE csharp
                SOURCES ${PROJECT_SRCS}
                INCLUDE_DIRECTORIES ${PROJECT_DIR}
                OPTIONS -csharp -outdir ${CWRAPPERDIR} -namespace ${PROJECT_NAME} -dllimport ${PROJECT_NAME}_dll
            )
            target_link_libraries(${PROJECT_NAME}_wrap PRIVATE ${PROJECT_NAME})
            swig_link_libraries(${PROJECT_NAME}_wrap ${PROJECT_NAME})
            set_target_properties(${PROJECT_NAME}_wrap PROPERTIES
                ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_INSTALL_PREFIX}
                LIBRARY_OUTPUT_DIRECTORY ${CMAKE_INSTALL_PREFIX}
                RUNTIME_OUTPUT_DIRECTORY ${CMAKE_INSTALL_PREFIX}
                OUTPUT_NAME ${PROJECT_NAME}
            )
            set(CMAKE_SWIG_FLAGS
                "-extern-c-header"               # Add extern "C" to function declarations
                "-D SWIG_CSharp_NO_CHAR16_T"     # Use char* instead of std::string for function arguments/returns
                "-D SWIG_CSHARP_CALL=__stdcall"  # Set calling convention to __stdcall
                "-D SWIG_CSHARP_INCLUDE_CONST"   # Handle const variables
                "-includetypesafe"               # Handle C/C++ structures
                "-namespace"
                "-includeexcept"                 # Handle C/C++ exception handling in C#
                "-includeall"                    # Use C/C++ templates
                "-includetypedef"
                "-includearray"
                "-includeenums"
            )

            add_custom_command(TARGET ${PROJECT_NAME}_wrap POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.cs ${CWRAPPERDIR}
            )
            
            set_source_files_properties(${PROJECT_SRCS} PROPERTIES SWIG_FLAGS "-includeall")
            foreach(src_file ${PROJECT_SRCS})
                get_filename_component(src_file_we ${src_file} NAME_WE)
                set_source_files_properties(${src_file} PROPERTIES SWIG_FLAGS "-include${src_file_we}_csharp.i")
                set_property(SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/${src_file_we}_csharp.i PROPERTY USE_TARGET_INCLUDE_DIRECTORIES ON)
            endforeach()
            # Handle overloaded functions
            foreach(src_file ${PROJECT_SRCS})
                get_filename_component(src_file_we ${src_file} NAME_WE)
                file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/${src_file_we}_csharp.i "")
                execute_process(COMMAND ${SWIG_EXECUTABLE} -csharp -c++ -external-runtime ${src_file}
                    COMMAND sed -E "s/(SWIGINTERN|SWIGEXPORT)\\s+.*\\)/SWIGINTERN void)/g" >> ${CMAKE_CURRENT_BINARY_DIR}/${src_file_we}_csharp.i
                    COMMAND ${SWIG_EXECUTABLE} -csharp -c++ -external-runtime -namespace ${PROJECT_NAME} -outdir ${CWRAPPERDIR} -o ${CMAKE_CURRENT_BINARY_DIR}/${src_file_we}_csharp_wrap.cxx -I${CMAKE_CURRENT_SOURCE_DIR} -I${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_CURRENT_BINARY_DIR}/${src_file_we}_csharp.i
                    COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_BINARY_DIR}/${src_file_we}_csharp.i ${CWRAPPERDIR}
                )
                list(APPEND PROJECT_SRCS ${CMAKE_CURRENT_BINARY_DIR}/${src_file_we}_csharp_wrap.cxx)
            endforeach()
        endif()
        add_custom_command(TARGET ${PROJECT_NAME}_wrap POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy ${SWIG_CSHARP_FILE} ${CWRAPPERDIR}
        )
    endif()
endforeach()

# Define a custom command to run Cppcheck
add_custom_command(
    OUTPUT cppcheck_result.txt
    COMMAND cppcheck --enable=all ${CMAKE_SOURCE_DIR}/path/to/source --xml 2>cppcheck_result.txt
    DEPENDS ${CMAKE_SOURCE_DIR}/path/to/source
)

# Define a custom command to run Clang-tidy
add_custom_command(
    OUTPUT clang_tidy_result.txt
    COMMAND clang-tidy ${CMAKE_SOURCE_DIR}/path/to/source -checks=* -- -I${CMAKE_SOURCE_DIR}/include -I${CMAKE_BINARY_DIR}
    DEPENDS ${CMAKE_SOURCE_DIR}/path/to/source
)

# Add a custom target to run both Cppcheck and Clang-tidy
add_custom_target(lint DEPENDS cppcheck_result.txt clang_tidy_result.txt)

# Add a function to parse the output of Cppcheck and Clang-tidy
function(parse_lint_results)
    # Run Cppcheck and Clang-tidy
    execute_process(
        COMMAND cppcheck --error-exitcode=1 --enable=all --suppress=missingIncludeSystem ${CMAKE_SOURCE_DIR} 2>cppcheck_result.txt
        COMMAND clang-tidy -p=${CMAKE_BINARY_DIR} ${CMAKE_SOURCE_DIR} -quiet -checks=* 2>clang_tidy_result.txt
        RESULT_VARIABLE LINT_RESULT
        ERROR_QUIET
    )
    if(LINT_RESULT)
        message(FATAL_ERROR "Linting failed")
    endif()
    # Parse the output of Cppcheck and Clang-tidy
    file(STRINGS "cppcheck_result.txt" CPPCHECK_OUTPUT REGEX "<error.*>")
    file(STRINGS "clang_tidy_result.txt" CLANG_TIDY_OUTPUT REGEX "<error>")
    # Use regex to parse the output and set the appropriate SWIG flags
    set(SWIG_FLAGS "-extern-c-header")
    if(CPPCHECK_OUTPUT OR CLANG_TIDY_OUTPUT)
        # Set flags to handle errors/warnings found by Cppcheck and Clang-tidy
        set(SWIG_FLAGS "${SWIG_FLAGS} -D SWIG_CSHARP_INCLUDE_CONST")
        set(SWIG_FLAGS "${SWIG_FLAGS} -includetypesafe")
    endif()
    if(CLANG_TIDY_OUTPUT)
        set(SWIG_FLAGS "${SWIG_FLAGS} -includeexcept")
        set(SWIG_FLAGS "${SWIG_FLAGS} -includeall")
    endif()
    # Set SWIG flags for the project
    set(CMAKE_SWIG_FLAGS ${SWIG_FLAGS})
endfunction()


# Call the parse_lint_results function after running Cppcheck and Clang-tidy
add_dependencies(lint cppcheck_result.txt clang_tidy_result.txt)
add_custom_command(TARGET lint POST_BUILD COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_SOURCE_DIR}/parse_lint_results.cmake)

# ex: Set swig flags for second project
set_source_files_properties(project2.i PROPERTIES CPLUSPLUS ON)
set_source_files_properties(project2.i PROPERTIES SWIG_FLAGS "-includetypesafe -DPROJECT2")
