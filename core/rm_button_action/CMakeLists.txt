cmake_minimum_required(VERSION 3.8)
project(rm_button_action)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(generate_parameter_library REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclcpp_components REQUIRED)
find_package(pluginlib REQUIRED)
find_package(std_msgs REQUIRED)

include_directories(
  include
)

generate_parameter_library(rm_button_action_parameters
  src/rm_button_action.yaml
)

add_library(rm_button_action_component SHARED
  src/rm_button_action.cpp
)

add_executable(rm_button_action
  src/main.cpp
)

ament_target_dependencies(rm_button_action_component
  rclcpp
  rclcpp_components
  std_msgs
)

target_link_libraries(rm_button_action_component
  rm_button_action_parameters
)

target_link_libraries(rm_button_action
  rm_button_action_component
)

rclcpp_components_register_nodes(rm_button_action_component 
  "rm_button_action::RmButtonAction"
)
pluginlib_export_plugin_description_file(rclcpp_components rm_button_action.xml)

install(TARGETS rm_button_action
  DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY include/
  DESTINATION include/
)

install(TARGETS rm_button_action_component
  rm_button_action_parameters
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
  INCLUDES DESTINATION include
)

install(FILES rm_button_action.xml
  DESTINATION share/${PROJECT_NAME}
)

install(DIRECTORY params/
  DESTINATION share/${PROJECT_NAME}/params
)

install(DIRECTORY launch/
  DESTINATION share/${PROJECT_NAME}/launch
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
