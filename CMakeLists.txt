cmake_minimum_required(VERSION 3.5)
project(lpslam)

# Default to C99
if(NOT CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD 99)
endif()

# Default to C++17
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(tf2_geometry_msgs REQUIRED)
find_package(nav_msgs REQUIRED)
find_package(yaml_cpp_vendor REQUIRED)
if (USE_OPENVSLAM_DIRECTLY)
  find_package(message_filters REQUIRED)
  find_package(builtin_interfaces REQUIRED)
  find_package(cv_bridge REQUIRED)
  if (NOT LPSLAM_BUILD_OPENVSLAM)
    # This means that openvslam is built separately
    find_package(openvslam REQUIRED)
  endif()
else()
  find_package(lpslam_interfaces REQUIRED)
endif()
find_package(rclcpp_components REQUIRED)

SET(LPSLAM_BUILD_ROS2 ON CACHE BOOL "build with ROS2 support")
SET(LPSLAM_BUILD_LPSLAM ON CACHE BOOL "Build LPSLAM module")
SET(LPSLAM_BUILD_OPENVSLAM ON CACHE BOOL "enable")
SET(LPSLAM_BUILD_OPENVSLAM_PANGOLIN ON CACHE BOOL "enable")
SET(USE_PANGOLIN_VIEWER OFF CACHE BOOL "Use Pangolin Viewer")
SET(USE_OPENVSLAM_DIRECTLY OFF CACHE BOOL "Build with direct usage of OpenVSLAM (w/o lpslam module)")

if (LPSLAM_BUILD_LPSLAM)
  add_subdirectory(lpslam)
endif()

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  # the following line skips the linter which checks for copyrights
  # uncomment the line when a copyright and license is not present in all source files
  #set(ament_cmake_copyright_FOUND TRUE)
  # the following line skips cpplint (only works in a git repo)
  # uncomment the line when this package is not in a git repo
  #set(ament_cmake_cpplint_FOUND TRUE)
  ament_lint_auto_find_test_dependencies()
endif()

## Build
SET(LPSLAM_COMPONENTS_SRC src/LpBaseNode.cpp)
if(USE_OPENVSLAM_DIRECTLY)
  LIST(APPEND LPSLAM_COMPONENTS_SRC src/OpenVSLAMNode.cpp)
else()
  LIST(APPEND LPSLAM_COMPONENTS_SRC src/LpSlamNode.cpp)
endif()

add_library(lpslam_components SHARED
  ${LPSLAM_COMPONENTS_SRC}
)

SET(LPSLAM_NODE_LIBS
  ${OPENZEN_TARGET_NAME}
)

if(USE_OPENVSLAM_DIRECTLY)
  LIST(APPEND LPSLAM_NODE_LIBS openvslam)
  target_include_directories(lpslam_components
    PRIVATE
    lpslam/src/Interface)

  if(USE_PANGOLIN_VIEWER)
    LIST(APPEND LPSLAM_NODE_LIBS pangolin_viewer)
    if (NOT LPSLAM_BUILD_LPSLAM)
      # If build lpslam with conan, it will add pangolin dependency.
      # Otherwise, adding it manually
      LIST(APPEND LPSLAM_NODE_LIBS pangolin)
    endif()
  endif()
else()
  LIST(APPEND LPSLAM_NODE_LIBS lpslam)
endif()

target_link_libraries(lpslam_components
  ${LPSLAM_NODE_LIBS}
)

SET(LPSLAM_NODE_DEPS
  rclcpp
  std_msgs
  sensor_msgs
  nav_msgs
  tf2
  tf2_ros
  tf2_geometry_msgs
  yaml_cpp_vendor
  rclcpp_components
)

if(USE_OPENVSLAM_DIRECTLY)
  LIST(APPEND LPSLAM_NODE_DEPS message_filters builtin_interfaces cv_bridge)
  target_compile_definitions(lpslam_components PRIVATE -DUSE_OPENVSLAM_DIRECTLY)
else()
  LIST(APPEND LPSLAM_NODE_DEPS lpslam_interfaces)
endif()

if(USE_PANGOLIN_VIEWER)
  target_compile_definitions(lpslam_components PRIVATE -DUSE_PANGOLIN_VIEWER)
endif()

ament_target_dependencies(lpslam_components
  ${LPSLAM_NODE_DEPS})

if(USE_OPENVSLAM_DIRECTLY)
  rclcpp_components_register_nodes(lpslam_components "lpslam_components::OpenVSLAMNode")
else()
  rclcpp_components_register_nodes(lpslam_components "lpslam_components::LpSlamNode")
endif()


add_executable(lpslam_node
  src/main.cpp)


target_link_libraries(lpslam_node
  ${LPSLAM_NODE_LIBS}
  lpslam_components)
if(USE_OPENVSLAM_DIRECTLY)
  target_include_directories(lpslam_node
    PRIVATE
    lpslam/src/Interface)
  target_compile_definitions(lpslam_node PRIVATE -DUSE_OPENVSLAM_DIRECTLY)
endif()

if(USE_PANGOLIN_VIEWER)
  target_compile_definitions(lpslam_node PRIVATE -DUSE_PANGOLIN_VIEWER)
endif()

ament_target_dependencies(lpslam_node
  ${LPSLAM_NODE_DEPS})
install(TARGETS
  lpslam_components
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)  
install(TARGETS
  lpslam_node
  DESTINATION lib/${PROJECT_NAME})

if (LPSLAM_BUILD_LPSLAM)
  install(TARGETS lpslam
    DESTINATION lib/${PROJECT_NAME})
endif()

ament_package()
