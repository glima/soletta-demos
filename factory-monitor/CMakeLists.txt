cmake_minimum_required(VERSION 2.8)

project(soletta-demos)

add_definitions(-DSOL_FLOW_NODE_TYPE_MODULE_EXTERNAL)

set(NODE_TYPES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/node_types)
set(MONITOR_RESOURCE ${NODE_TYPES_DIR}/oic.r.monitor.json)
set(MODULES_DIR ${CMAKE_BINARY_DIR}/module_descriptions)
set(STAGE_DIR ${CMAKE_BINARY_DIR}/stage)
set(STAGE_DIR ${CMAKE_BINARY_DIR}/stage)
set(MONITOR_GEN_H ${STAGE_DIR}/monitor-gen.h)
set(MONITOR_GEN_C ${STAGE_DIR}/monitor-gen.c)
set(MONITOR_JSON ${STAGE_DIR}/monitor.json)
set(MONITOR_C ${STAGE_DIR}/monitor.c)
set(MONITOR_GEN_JSON ${MODULES_DIR}/monitor.json)
set(CUSTOM_NODE_JSON ${NODE_TYPES_DIR}/custom-node.json)
set(CUSTOM_NODE_C ${NODE_TYPES_DIR}/custom-node.c)
set(CUSTOM_NODE_GEN_H ${STAGE_DIR}/custom-node-gen.h)
set(CUSTOM_NODE_GEN_C ${STAGE_DIR}/custom-node-gen.c)
set(CUSTOM_NODE_GEN_JSON ${STAGE_DIR}/custom-node-gen.json)

find_package(PkgConfig REQUIRED)
pkg_check_modules(SOLETTA soletta>=0.0.1)

function(pkg_check_variable _pkg _name)
    string(TOUPPER ${_pkg} _pkg_upper)
    string(TOUPPER ${_name} _name_upper)
    string(REPLACE "-" "_" _pkg_upper ${_pkg_upper})
    string(REPLACE "-" "_" _name_upper ${_name_upper})
    set(_output_name "${_pkg_upper}_${_name_upper}")

    execute_process(COMMAND ${PKG_CONFIG_EXECUTABLE} --variable=${_name} ${_pkg}
        OUTPUT_VARIABLE _pkg_result
        OUTPUT_STRIP_TRAILING_WHITESPACE)

    set("${_output_name}" "${_pkg_result}" CACHE STRING "pkg-config variable ${_name} of ${_pkg}")
endfunction()

pkg_check_variable(soletta modulesdir)
pkg_check_variable(soletta nodeschemapath)
pkg_check_variable(soletta pkgdatadir)

add_custom_command(OUTPUT ${MONITOR_C} ${MONITOR_GEN_C} ${MONITOR_GEN_H} ${MONITOR_JSON}
    DEPENDS ${MONITOR_RESOURCE}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${STAGE_DIR}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${MODULES_DIR}
    COMMAND ${CMAKE_FIND_ROOT_PATH}/${SOLETTA_PREFIX}/bin/sol-oic-gen.py --schema-dir=${NODE_TYPES_DIR} --node-type-json=${MONITOR_JSON} --node-type-impl=${MONITOR_C} --node-type-gen-c=${MONITOR_GEN_C} --node-type-gen-h=${MONITOR_GEN_H}
    COMMAND ${CMAKE_FIND_ROOT_PATH}/${SOLETTA_PREFIX}/bin/sol-flow-node-type-validate.py ${CMAKE_FIND_ROOT_PATH}/${SOLETTA_NODESCHEMAPATH} ${MONITOR_JSON}
    COMMAND ${CMAKE_FIND_ROOT_PATH}/${SOLETTA_PREFIX}/bin/sol-flow-node-type-gen.py --prefix=sol-flow-node-type --genspec-schema=${CMAKE_FIND_ROOT_PATH}/${SOLETTA_NODESCHEMAPATH} ${MONITOR_JSON} ${MONITOR_GEN_H} ${MONITOR_GEN_C} ${MONITOR_GEN_JSON}
    )

add_custom_target(custom-node-gen
    BYPRODUCTS ${CUSTOM_NODE_GEN_C} ${CUSTOM_NODE_GEN_H} ${CUSTOM_NODE_GEN_JSON}
    DEPENDS ${CUSTOM_NODE_JSON} ${CUSTOM_NODE_C} ${MONITOR_GEN_H}
    COMMAND echo "Generating custom node code..."
    COMMAND ${CMAKE_COMMAND} -E make_directory ${STAGE_DIR}
    COMMAND ${CMAKE_COMMAND} -E make_directory ${MODULES_DIR}
    COMMAND ${CMAKE_FIND_ROOT_PATH}/${SOLETTA_PREFIX}/bin/sol-flow-node-type-validate.py ${CMAKE_FIND_ROOT_PATH}/${SOLETTA_NODESCHEMAPATH} ${CUSTOM_NODE_JSON}
    COMMAND ${CMAKE_FIND_ROOT_PATH}/${SOLETTA_PREFIX}/bin/sol-flow-node-type-gen.py --prefix=sol-flow-node-type --genspec-schema=${CMAKE_FIND_ROOT_PATH}/${SOLETTA_NODESCHEMAPATH} ${CUSTOM_NODE_JSON} ${CUSTOM_NODE_GEN_H} ${CUSTOM_NODE_GEN_C} ${CUSTOM_NODE_GEN_JSON}
    COMMAND echo "Done."
    )

set(BUILT_SOURCES
    ${MONITOR_GEN_H}
    ${MONITOR_GEN_C}
    ${MONITOR_C}
    ${CUSTOM_NODE_GEN_H}
    ${CUSTOM_NODE_GEN_C}
    )
SET_SOURCE_FILES_PROPERTIES(${BUILT_SOURCES} PROPERTIES
    GENERATED true
    )

include_directories(
    ${SOLETTA_INCLUDE_DIRS}
    ${STAGE_DIR} #Check if ok
    )

link_directories(
    ${SOLETTA_LIBRARY_DIRS}
    )

add_library(monitor SHARED
    ${MONITOR_C}
    )
set_target_properties(monitor
    PROPERTIES PREFIX ""
    )
target_link_libraries(monitor
    ${SOLETTA_LIBRARIES}
    )

add_library(custom-node SHARED
    ${CUSTOM_NODE_C}
    )
add_dependencies(custom-node
    custom-node-gen
    )
set_target_properties(custom-node
    PROPERTIES PREFIX ""
    )
target_link_libraries(custom-node
    ${SOLETTA_LIBRARIES}
    )

install(TARGETS monitor custom-node
    LIBRARY DESTINATION ${SOLETTA_MODULESDIR}/flow/
    )

install(FILES ${MONITOR_GEN_JSON} ${CUSTOM_NODE_GEN_JSON}
    DESTINATION ${SOLETTA_PKGDATADIR}/flow/descriptions
    )
