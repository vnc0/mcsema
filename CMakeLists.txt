# Copyright (c) 2020 Trail of Bits, Inc.
#
# This program is free software: you can redistribute it and/or modify
# it under the terms of the GNU Affero General Public License as
# published by the Free Software Foundation, either version 3 of the
# License, or (at your option) any later version.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU Affero General Public License for more details.
#
# You should have received a copy of the GNU Affero General Public License
# along with this program.  If not, see <http://www.gnu.org/licenses/>.

# Find remill first because its config has useful dependency-finding info that
# needs to be found before the CMake `project` declaration
find_package(remill COMPONENTS VCPKG_DEPS QUIET)

include(cmake/vcpkg_helper.cmake)

project(mcsema C CXX ASM)

include(GNUInstallDirs)
cmake_minimum_required(VERSION 3.14)


include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/settings.cmake")
include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/utils.cmake")
include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/BCCompiler.cmake")
include("${CMAKE_CURRENT_SOURCE_DIR}/cmake/ccache.cmake")

configureCcache()
FindAndSelectClangCompiler()

set(MCSEMA_SOURCE_DIR "${PROJECT_SOURCE_DIR}")

option(MCSEMA_ENABLE_RUNTIME "Should runtimes for re-compilation of bitcode be produced?" ON)

find_package(Python3 QUIET)
if(Python3_FOUND)
  option(MCSEMA_INSTALL_PYTHON3_LIBS "Install Python 3 libraries" ON)
else()
  option(MCSEMA_INSTALL_PYTHON3_LIBS "Install Python 3 libraries")
endif()

# warnings and compiler settings
if(NOT DEFINED WIN32)
  set(PROJECT_CXXFLAGS
    ${GLOBAL_CXXFLAGS}
    -Wconversion
    -pedantic
    -Wno-unreachable-code-return
  )
endif()

#
# libraries
#

find_package(remill CONFIG REQUIRED)
list(APPEND PROJECT_LIBRARIES remill_settings remill)
get_target_property(REMILL_INCLUDE_LOCATION remill_settings INTERFACE_INCLUDE_DIRECTORIES)

# protobuf
# Compatibility since we use older protobuf CMake functions
set(protobuf_MODULE_COMPATIBLE ON CACHE BOOL "" FORCE)
find_package(Protobuf REQUIRED)
list(APPEND PROJECT_LIBRARIES ${Protobuf_LIBRARIES})
list(APPEND PROJECT_INCLUDEDIRECTORIES ${Protobuf_INCLUDE_DIR})
list(APPEND PROJECT_DEFINITIONS "GOOGLE_PROTOBUF_NO_RTTI")

#
# protobuf file generation
#

# this function can't be told where to store the output files! we have to add the whole binary directory
# to the include directories (or change it and lose compatibility with the system libraries)
protobuf_generate_cpp(PROJECT_PROTOBUFSOURCEFILES
  PROJECT_PROTOBUFHEADERFILES "${CMAKE_CURRENT_SOURCE_DIR}/mcsema/CFG/CFG.proto"
)

list(APPEND PROJECT_INCLUDEDIRECTORIES ${CMAKE_CURRENT_BINARY_DIR})

protobuf_generate_python(PROJECT_PROTOBUFPYTHONMODULE
  "${CMAKE_CURRENT_SOURCE_DIR}/mcsema/CFG/CFG.proto"
)

add_custom_target(protobuf_python_module_ida
  DEPENDS ${PROJECT_PROTOBUFPYTHONMODULE}
)

# disable -Werror on these file since they have been generated
set_source_files_properties(${PROJECT_PROTOBUFSOURCEFILES} PROPERTIES
  COMPILE_FLAGS "-Wno-sign-conversion -Wno-shorten-64-to-32 -Wno-conversion"
)

set_source_files_properties(${PROJECT_PROTOBUFHEADERFILES} PROPERTIES
  COMPILE_FLAGS "-Wno-sign-conversion -Wno-shorten-64-to-32 -Wno-conversion"
)

#
# target settings
#

set(MCSEMA_LIFT mcsema-lift-${REMILL_LLVM_VERSION})

# for version information
add_subdirectory(mcsema/Version)

add_executable(${MCSEMA_LIFT}
  ${PROJECT_PROTOBUFSOURCEFILES}

  mcsema/Arch/Arch.cpp

  mcsema/CFG/CFG.cpp

  mcsema/BC/Callback.cpp
  mcsema/BC/External.cpp
  mcsema/BC/Function.cpp
  mcsema/BC/Instruction.cpp
  mcsema/BC/Legacy.cpp
  mcsema/BC/Lift.cpp
  mcsema/BC/Optimize.cpp
  mcsema/BC/Segment.cpp
  mcsema/BC/Util.cpp

  tools/mcsema_lift/Lift.cpp
)


# Copy mcsema-disass in
add_custom_command(
  TARGET protobuf_python_module_ida POST_BUILD
  DEPENDS {PROJECT_PROTOBUFPYTHONMODULE}
  COMMAND ${CMAKE_COMMAND} -E copy ${PROJECT_PROTOBUFPYTHONMODULE} ${CMAKE_CURRENT_SOURCE_DIR}/tools/mcsema_disass/ida7
)

# this is needed for the #include directives with absolutes paths to work correctly; it must
# also be set to PUBLIC since mcsema-lift includes some files directly
list(APPEND PROJECT_INCLUDEDIRECTORIES ${CMAKE_CURRENT_SOURCE_DIR})

add_dependencies(${MCSEMA_LIFT}
  protobuf_python_module_ida)

#
# libraries
#

# anvill
find_package(anvill CONFIG REQUIRED)
list(APPEND PROJECT_LIBRARIES anvill)

# mcsema-disass

set(MCSEMA_PYTHON_SOURCES
    tools/setup.py
    tools/mcsema_disass/__init__.py
    tools/mcsema_disass/__main__.py
    tools/mcsema_disass/ida7/__init__.py
    tools/mcsema_disass/ida7/anvill_compat.py
    tools/mcsema_disass/ida7/arm_util.py
    tools/mcsema_disass/ida7/CFG_pb2.py
    tools/mcsema_disass/ida7/disass.py
    tools/mcsema_disass/ida7/exception.py
    tools/mcsema_disass/ida7/flow.py
    tools/mcsema_disass/ida7/get_cfg.py
    tools/mcsema_disass/ida7/refs.py
    tools/mcsema_disass/ida7/segment.py
    tools/mcsema_disass/ida7/table.py
    tools/mcsema_disass/ida7/util.py
    tools/mcsema_disass/ida7/x86_util.py
)

if(MCSEMA_INSTALL_PYTHON3_LIBS)
  add_custom_target(build_mcsema_disass_python3
    DEPENDS ${MCSEMA_PYTHON_SOURCES})

  add_custom_command(
      TARGET build_mcsema_disass_python3 POST_BUILD
      COMMAND which python3 && python3 setup.py build --force
      COMMENT "Building McSema Python 3 mcsema-disass"
      WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/tools")

  add_dependencies(${MCSEMA_LIFT}
    build_mcsema_disass_python3)

  add_dependencies(build_mcsema_disass_python3
    protobuf_python_module_ida)
endif()

#
# target settings
#

target_link_libraries(${MCSEMA_LIFT} PRIVATE ${PROJECT_LIBRARIES} McSemaVersion)
target_include_directories(${MCSEMA_LIFT} SYSTEM PUBLIC ${PROJECT_INCLUDEDIRECTORIES})
target_compile_definitions(${MCSEMA_LIFT} PUBLIC ${PROJECT_DEFINITIONS})
target_compile_options(${MCSEMA_LIFT} PRIVATE ${PROJECT_CXXFLAGS})

if("${CMAKE_HOST_SYSTEM_PROCESSOR}" STREQUAL "x86_64" AND MCSEMA_ENABLE_RUNTIME)
  add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/mcsema/Arch/X86/Runtime)
endif()

#TODO(artem): this may need some logic to select only ABIs compatible with current os/arch
#TODO(kumarak): Disable the ABI libraries build till we don't have script to automate the generation of `ABI_libc.h`. Use pre-build library to test the --abi_libraries flag
function(GetABILibraryList)
  file(GLOB abi_library_list RELATIVE "${CMAKE_CURRENT_SOURCE_DIR}/mcsema/OS" "${CMAKE_CURRENT_SOURCE_DIR}/mcsema/OS/*")
  set(GetABILibraryList_Output ${abi_library_list} PARENT_SCOPE)
endfunction()

if(NOT MCSEMA_ABI_LIBRARY_LIST_INITIALIZED)
  GetABILibraryList()
  set(MCSEMA_DISABLED_ABI_LIBRARIES ${GetABILibraryList_Output} CACHE
    STRING "A semicolon-separated list of disabled ABI libraries"
  )

  set(MCSEMA_ABI_LIBRARY_LIST_INITIALIZED ON CACHE INTERNAL
    "Whether the abi library list has been initialized or not"
  )
endif()

message(STATUS "Disabled ABI libraries: ${MCSEMA_DISABLED_ABI_LIBRARIES}")
message(STATUS "You can change this setting with -DMCSEMA_DISABLED_ABI_LIBRARIES:STRING=\"Name1;Name2\"")

GetABILibraryList()
foreach(abi_library_name ${GetABILibraryList_Output})
  set(abi_library_path "mcsema/OS/${abi_library_name}")

  list(FIND MCSEMA_DISABLED_ABI_LIBRARIES "${abi_library_name}" abi_lib_index)
  if(NOT ${abi_lib_index} EQUAL -1)
    message(STATUS "Skipping ABI library: ${abi_library_path}") 
    continue()
  endif()

  message(STATUS "Adding ABI library: ${abi_library_path}")
  add_subdirectory("${abi_library_path}")
endforeach()

if(DEFINED WIN32)
  set(install_folder "${CMAKE_INSTALL_PREFIX}/mcsema")
else()
  set(install_folder "${CMAKE_INSTALL_PREFIX}")
endif()

install(
  TARGETS ${MCSEMA_LIFT}
  RUNTIME DESTINATION "${install_folder}/bin"
  LIBRARY DESTINATION "${install_folder}/lib"
)

# target for dyninst frontend
if (DEFINED BUILD_MCSEMA_DYNINST_DISASS)
    if(${BUILD_MCSEMA_DYNINST_DISASS} EQUAL 1)
        add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/tools/mcsema_disass/dyninst)
    endif()
endif()

set(python_package_installer "${CMAKE_CURRENT_SOURCE_DIR}/tools/setup_launcher")
if(DEFINED WIN32)
  string(REPLACE "/" "\\" python_package_install_path "${install_folder}")
  set(python_package_installer3 "${python_package_installer}.bat")
  set(optional_interpreter "cmd.exe /C")
else()
  set(python_package_install_path "${install_folder}")
  set(python_package_installer3 "${python_package_installer}_py3.sh")
endif()

if(MCSEMA_INSTALL_PYTHON3_LIBS)
  install(CODE
    "execute_process(COMMAND ${optional_interpreter} \"${python_package_installer3}\" \"${python_package_install_path}\"\nWORKING_DIRECTORY \"${PROJECT_SOURCE_DIR}/tools\"\nRESULT_VARIABLE exit_code3)\n   if(NOT exit_code3 EQUAL 0)\n       message(FATAL_ERROR \"Failed to install the Python 3 package\")\n     endif()")
endif()

add_subdirectory(examples)

