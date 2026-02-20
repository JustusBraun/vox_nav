cmake_minimum_required(VERSION 3.5)
project(vox_nav_control C CXX)

# Default to C++17
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

# Add this repository's cmake modules to CMAKE_MODULE_PATH
list(INSERT CMAKE_MODULE_PATH 0 ${CMAKE_CURRENT_LIST_DIR}/cmake)

find_package(PkgConfig REQUIRED)
pkg_check_modules(LIBFCL REQUIRED "fcl>=0.7.0")
# replace LIBFCL_LIBRARIES with full paths to the libraries
set(LIBFCL_LIBRARIES_FULL "")
foreach(LIBFCL_LIBRARY ${LIBFCL_LIBRARIES})
  find_library(${LIBFCL_LIBRARY}_LIB ${LIBFCL_LIBRARY} ${LIBFCL_LIBRARY_DIRS})
  list(APPEND LIBFCL_LIBRARIES_FULL ${${LIBFCL_LIBRARY}_LIB})
endforeach()
set(LIBFCL_LIBRARIES "${LIBFCL_LIBRARIES_FULL}")

find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_action REQUIRED)
find_package(std_msgs REQUIRED)
find_package(visualization_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(builtin_interfaces REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(ompl REQUIRED)
find_package(OCTOMAP REQUIRED)
find_package(octomap_msgs REQUIRED)
find_package(tf2_geometry_msgs REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(trajectory_msgs REQUIRED)
find_package(vox_nav_msgs REQUIRED)
find_package(pcl_ros REQUIRED)
find_package(pluginlib REQUIRED)
find_package(vox_nav_msgs REQUIRED)
find_package(vox_nav_utilities REQUIRED)
find_package(Eigen3 REQUIRED)
find_package(tf2_eigen REQUIRED)
find_package(casadi REQUIRED)
find_package(vision_msgs REQUIRED)

set(dependencies
    rclcpp
    pluginlib
    geometry_msgs
    sensor_msgs
    nav_msgs
    std_msgs
    tf2_geometry_msgs
    tf2_ros
    trajectory_msgs
    visualization_msgs
    pcl_ros
    vox_nav_msgs
    vox_nav_utilities
    Eigen3
    tf2_eigen
    LIBFCL
    vision_msgs)

add_executable(vox_nav_controller_server src/controller_server.cpp)
target_include_directories(vox_nav_controller_server PUBLIC include ${OMPL_INCLUDE_DIRS})
ament_target_dependencies(vox_nav_controller_server ${dependencies})
target_link_libraries(vox_nav_controller_server ${OMPL_LIBRARIES} mosquittopp mosquitto)

add_library(mpc_controller_casadi SHARED               src/mpc_controller_casadi/mpc_controller_casadi_core.cpp 
                                                       src/mpc_controller_casadi/mpc_controller_casadi_ros.cpp)
target_include_directories(mpc_controller_casadi PUBLIC include ${casadi_INCLUDE_DIR} ${OMPL_INCLUDE_DIRS})
ament_target_dependencies(mpc_controller_casadi ${dependencies})
target_link_libraries(mpc_controller_casadi ${OMPL_LIBRARIES} casadi::casadi)

add_library(lyapunov_controller SHARED                 src/lyapunov_controller/lyapunov_controller_ros.cpp)
target_include_directories(lyapunov_controller PUBLIC include ${OMPL_INCLUDE_DIRS})
ament_target_dependencies(lyapunov_controller ${dependencies})
target_link_libraries(lyapunov_controller ${OMPL_LIBRARIES})

# Path Refiner plugins
add_library(traversability_based_plan_refiner SHARED src/plan_refiner_plugins/traversability_based_plan_refiner.cpp)
target_include_directories(traversability_based_plan_refiner PUBLIC include ${OMPL_INCLUDE_DIRS})
ament_target_dependencies(traversability_based_plan_refiner ${dependencies})
target_link_libraries(traversability_based_plan_refiner ${OMPL_LIBRARIES} ${PCL_LIBRARIES} ${LIBFCL_LIBRARIES})

install(TARGETS mpc_controller_casadi
                lyapunov_controller
                traversability_based_plan_refiner
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin)

install(TARGETS vox_nav_controller_server
  RUNTIME DESTINATION lib/${PROJECT_NAME})

install(DIRECTORY include/
  DESTINATION include/)

install(DIRECTORY cmake launch 
DESTINATION share/${PROJECT_NAME})

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()
endif()

ament_export_include_directories(include)
ament_export_libraries(mpc_controller_casadi)
ament_export_libraries(lyapunov_controller)
ament_export_libraries(traversability_based_plan_refiner)

ament_export_dependencies(${dependencies})
ament_export_include_directories(include)

pluginlib_export_plugin_description_file(${PROJECT_NAME} plugins.xml)
ament_package()
