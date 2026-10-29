cmake_minimum_required(VERSION 3.8)
project(rm_custom_controller_imu)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(generate_parameter_library REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(rm_message REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(rclcpp_components REQUIRED)
find_package(pluginlib REQUIRED)

include_directories(
  include
)

generate_parameter_library(rm_custom_controller_imu_parameters
  src/rm_custom_controller_imu.yaml
)

add_library(rm_custom_controller_imu_component SHARED
  src/rm_custom_controller_imu.cpp
)

add_executable(rm_custom_controller_imu
  src/main.cpp
)

ament_target_dependencies(rm_custom_controller_imu_component
  rclcpp
  std_msgs
  rm_message
  geometry_msgs
  tf2_ros
  rclcpp_components
)

target_link_libraries(rm_custom_controller_imu_component
  rm_custom_controller_imu_parameters
)

target_link_libraries(rm_custom_controller_imu
  rm_custom_controller_imu_component
)

rclcpp_components_register_nodes(rm_custom_controller_imu_component 
  "rm_custom_controller_imu::RmCustomControllerImu"
)
pluginlib_export_plugin_description_file(rclcpp_components rm_custom_controller_imu.xml)

install(TARGETS rm_custom_controller_imu
  DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY include/
  DESTINATION include/
)

install(TARGETS rm_custom_controller_imu_component
  rm_custom_controller_imu_parameters
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
  INCLUDES DESTINATION include
)

install(FILES rm_custom_controller_imu.xml
  DESTINATION share/${PROJECT_NAME}
)

install(DIRECTORY test/
  DESTINATION share/${PROJECT_NAME}/test
)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  # the following line skips the linter which checks for copyrights
  # comment the line when a copyright and license is added to all source files
  set(ament_cmake_copyright_FOUND TRUE)
  # the following line skips cpplint (only works in a git repo)
  # comment the line when this package is in a git repo and when
  # a copyright and license is added to all source files
  set(ament_cmake_cpplint_FOUND TRUE)
  ament_lint_auto_find_test_dependencies()
endif()

ament_package()
