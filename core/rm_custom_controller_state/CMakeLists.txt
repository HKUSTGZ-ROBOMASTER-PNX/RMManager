cmake_minimum_required(VERSION 3.8)
project(rm_custom_controller_state)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(generate_parameter_library REQUIRED)
find_package(geometry_msgs REQUIRED)
find_package(rclcpp REQUIRED)
find_package(std_msgs REQUIRED)
find_package(rm_message REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(rclcpp_components REQUIRED)
find_package(pluginlib REQUIRED)

include_directories(
  include
)

generate_parameter_library(rm_custom_controller_state_parameters
  src/rm_custom_controller_state.yaml
)

add_library(rm_custom_controller_state_component SHARED
  src/rm_custom_controller_state.cpp
)

add_executable(rm_custom_controller_state
  src/main.cpp
)

ament_target_dependencies(rm_custom_controller_state_component
  geometry_msgs
  rclcpp
  std_msgs
  rm_message
  sensor_msgs
  rclcpp_components
)

target_link_libraries(rm_custom_controller_state_component
  rm_custom_controller_state_parameters
)

target_link_libraries(rm_custom_controller_state
  rm_custom_controller_state_component
)

rclcpp_components_register_nodes(rm_custom_controller_state_component
  "rm_custom_controller_state::RmCustomControllerState"
)
pluginlib_export_plugin_description_file(rclcpp_components rm_custom_controller_state.xml)

install(TARGETS rm_custom_controller_state
  DESTINATION lib/${PROJECT_NAME}
)

install(DIRECTORY include/
  DESTINATION include/
)

install(TARGETS rm_custom_controller_state_component
  rm_custom_controller_state_parameters
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
  INCLUDES DESTINATION include
)

install(FILES rm_custom_controller_state.xml
  DESTINATION share/${PROJECT_NAME}
)

install(DIRECTORY test/
  DESTINATION share/${PROJECT_NAME}/test
)

if(BUILD_TESTING)
  find_package(ament_cmake_gtest REQUIRED)
  find_package(ament_lint_auto REQUIRED)

  ament_add_gtest(test_rm_custom_controller_state
    test/test_rm_custom_controller_state.cpp
  )

  if(TARGET test_rm_custom_controller_state)
    target_link_libraries(test_rm_custom_controller_state
      rm_custom_controller_state_component
    )
    ament_target_dependencies(test_rm_custom_controller_state
      geometry_msgs
      rclcpp
      rm_message
      sensor_msgs
      std_msgs
    )
  endif()

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
