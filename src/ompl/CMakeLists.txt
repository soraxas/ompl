configure_file("${CMAKE_CURRENT_SOURCE_DIR}/config.h.in" "${CMAKE_CURRENT_BINARY_DIR}/config.h")
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/config.h"
    DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/ompl"
    COMPONENT ompl)

file(GLOB_RECURSE OMPL_SOURCE_CODE datastructures/*.cpp util/*.cpp base/*.cpp geometric/*.cpp control/*.cpp tools/*.cpp)
file(GLOB_RECURSE OMPL_HEADERS datastructures/*.h util/*.h base/*.h geometric/*.h control/*.h tools/*.h)

#############################################
# Add enabled extensions to the source code #
#############################################

## replace XXX by the extension
#if (OMPL_EXTENSION_XXX)
#    file(GLOB_RECURSE OMPL_XXX_EXTENSION_SOURCE_CODE extensions/XXX/*.cpp)
#    set(OMPL_SOURCE_CODE ${OMPL_SOURCE_CODE} ${OMPL_XXX_EXTENSION_SOURCE_CODE})
#
##    if additional libraries need to be linked, they are added to manifest.xml for ROS
##    or they are added to the list of OMPL_LINK_LIBRARIES
#
#endif()

if (OMPL_EXTENSION_MORSE)
    file(GLOB_RECURSE OMPL_MORSE_EXTENSION_SOURCE_CODE extensions/morse/*.cpp)
    file(GLOB_RECURSE OMPL_MORSE_EXTENSION_HEADERS extensions/morse/*.h)
    set(OMPL_SOURCE_CODE ${OMPL_SOURCE_CODE} ${OMPL_MORSE_EXTENSION_SOURCE_CODE})
    set(OMPL_HEADERS ${OMPL_HEADERS} ${OMPL_MORSE_EXTENSION_HEADERS})
endif()

if (OMPL_EXTENSION_ODE)
    file(GLOB_RECURSE OMPL_ODE_EXTENSION_SOURCE_CODE extensions/ode/*.cpp)
    file(GLOB_RECURSE OMPL_ODE_EXTENSION_HEADERS extensions/ode/*.h)
    set(OMPL_SOURCE_CODE ${OMPL_SOURCE_CODE} ${OMPL_ODE_EXTENSION_SOURCE_CODE})
    set(OMPL_HEADERS ${OMPL_HEADERS} ${OMPL_ODE_EXTENSION_HEADERS})
    list(APPEND OMPL_LINK_LIBRARIES "${ODE_LIBRARIES}")
endif()

if (OMPL_EXTENSION_TRIANGLE)
    file(GLOB_RECURSE OMPL_TRIANGLE_EXTENSION_SOURCE_CODE extensions/triangle/*.cpp)
    file(GLOB_RECURSE OMPL_TRIANGLE_EXTENSION_HEADERS extensions/triangle/*.h)
    set(OMPL_SOURCE_CODE ${OMPL_SOURCE_CODE} ${OMPL_TRIANGLE_EXTENSION_SOURCE_CODE})
    set(OMPL_HEADERS ${OMPL_HEADERS} ${OMPL_TRIANGLE_EXTENSION_HEADERS})
    list(APPEND OMPL_LINK_LIBRARIES "${TRIANGLE_LIBRARY}")
endif()

source_group("OMPL Source" FILES "${OMPL_SOURCE_CODE}")
source_group("OMPL Headers" FILES "${OMPL_HEADERS}")

# build the library
if(MSVC)
    add_library(ompl STATIC ${OMPL_SOURCE_CODE})
else(MSVC)
    add_library(ompl SHARED ${OMPL_SOURCE_CODE})
endif(MSVC)
target_link_libraries(ompl
    PUBLIC
    ${OMPL_LINK_LIBRARIES}
    ${Boost_SERIALIZATION_LIBRARY}
    ${Boost_FILESYSTEM_LIBRARY}
    ${Boost_SYSTEM_LIBRARY}
    ${CMAKE_THREAD_LIBS_INIT})

if (OMPL_EXTENSION_ODE)
    if (NOT CMAKE_VERSION VERSION_LESS 3.13)
        target_link_directories(ompl PUBLIC ${ODE_LIBRARY_DIRS})
    else()
        link_directories(ompl ${ODE_LIBRARY_DIRS})
    endif()
endif()

if (OMPL_HAVE_SPOT)
    target_link_libraries(ompl PUBLIC ${SPOT_LIBRARIES})
endif()

if (MSVC)
    set_target_properties(ompl PROPERTIES VERSION "${PROJECT_VERSION}" STATIC_LIBRARY_FLAGS "psapi.lib ws2_32.lib")
else (MSVC)
    if (MINGW)
        target_link_libraries(ompl PUBLIC psapi ws2_32)
        set_target_properties(ompl PROPERTIES LINK_FLAGS "-Wl,--export-all-symbols")
    endif (MINGW)
    set_target_properties(ompl PROPERTIES VERSION "${PROJECT_VERSION}" SOVERSION "${OMPL_ABI_VERSION}")
endif (MSVC)

##################################################################
find_package(catkin REQUIRED)
## NECESSARY DIR SETTINGS by user
set(DIFF_SAMP_DIR $ENV{HOME}/research/diffeomorphic_sampler)
#set(MOVEIT_SRC_ROOT $ENV{HOME}/ROS/ws_jaco-diff/src/moveit)
set(MOVEIT_SRC_ROOT ${CMAKE_SOURCE_DIR}/../moveit)
# typical var catkin_DIR=/opt/ros/noetic/share/catkin/cmake
# we will walk backward to find system include path
set(CATKIN_SYSTEM_INCLUDE "${catkin_DIR}/../../../include")
##################################################################
function(ensure_path_exists _dir)
  if(NOT IS_DIRECTORY "${_dir}" OR NOT EXISTS "${_dir}")
    message(FATAL_ERROR "The given directory '${_dir}' does not exists!")
  endif()
endfunction(ensure_path_exists)
## Ensure path exists, sanity check!
ensure_path_exists("${DIFF_SAMP_DIR}")
ensure_path_exists("${MOVEIT_SRC_ROOT}")
ensure_path_exists("${CATKIN_SYSTEM_INCLUDE}")
##################################################################
## add the actual source for diff sampler
add_subdirectory(${DIFF_SAMP_DIR} diff-samp)
#find_package(catkin REQUIRED COMPONENTS diffeomorphic_state_sampler)
target_link_libraries(ompl PUBLIC diffeomorphic_state_sampler)
##################################################################
## Link to libtorch
find_package(Torch REQUIRED)
target_compile_features(ompl PRIVATE cxx_range_for)
target_link_libraries(ompl PRIVATE ${TORCH_LIBRARIES})
##################################################################
## Include necessary headers from MoveIt
target_include_directories(
        ompl PRIVATE
        ${MOVEIT_SRC_ROOT}/moveit_planners/ompl/ompl_interface/include
        ${MOVEIT_SRC_ROOT}/moveit_core/robot_model/include
        ${MOVEIT_SRC_ROOT}/moveit_core/macros/include
        ${MOVEIT_SRC_ROOT}/moveit_core/exceptions/include
        #
        ${MOVEIT_SRC_ROOT}/../geometric_shapes/include
        #/opt/ros/melodic/include
        ${CATKIN_SYSTEM_INCLUDE}
        ${CMAKE_INSTALL_FULL_INCLUDEDIR}
)
list(APPEND CMAKE_PREFIX_PATH ${CMAKE_INSTALL_PREFIX})  # the install prefix (devel) allows us to find self-built moveit
find_package(
  catkin REQUIRED
  COMPONENTS
  moveit_core
)
include_directories(SYSTEM ${catkin_INCLUDE_DIRS})
##################################################################
## Link to MoveIt interface for robot model
#find_package(moveit_planners_ompl REQUIRED)
#target_link_libraries(ompl PRIVATE moveit_ompl_interface)
find_library(
        MOVEIT_OMPL_INTERFACE_LIB moveit_ompl_interface
        HINTS ${CATKIN_DEVEL_PREFIX}/lib)
target_link_libraries(ompl PRIVATE ${MOVEIT_OMPL_INTERFACE_LIB})
##################################################################

# install the library
install(TARGETS ompl
    DESTINATION ${CMAKE_INSTALL_LIBDIR}
    COMPONENT ompl)
if (NOT MSVC)
    add_custom_command(TARGET ompl POST_BUILD
        COMMAND "${CMAKE_COMMAND}" -E copy "$<TARGET_FILE:ompl>"
        "${CMAKE_CURRENT_SOURCE_DIR}/../../py-bindings/ompl/util/libompl${CMAKE_SHARED_LIBRARY_SUFFIX}"
        WORKING_DIRECTORY "${LIBRARY_OUTPUT_PATH}")
endif (NOT MSVC)
