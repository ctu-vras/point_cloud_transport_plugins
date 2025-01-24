cmake_minimum_required(VERSION 3.10.2)
project(draco_point_cloud_transport)

# Use Draco as a standalone cmake project, it gets built thanks to catkin
# package.xml.
find_package(catkin REQUIRED COMPONENTS
  cras_cpp_common
  dynamic_reconfigure
  message_generation
  pluginlib
  point_cloud_transport
  sensor_msgs
  std_msgs)

# There are some find_package/naming issues due to name draco/Draco mismatch.
# Currently, with caktin tools only DracoConfig.cmake is found with just two
# variables defined, draco_INCLUDE_DIR and draco_LIBRARY_DIR. In that case we
# try to reconstruct the rest from FindDraco.cmake (module mode).
find_package(Draco REQUIRED)
if(draco_INCLUDE_DIR)
  set(draco_FOUND TRUE)
  set(draco_LIBRARY_DIRS "${draco_LIBRARY_DIR}")
  find_path(draco_INCLUDE_DIRS draco/point_cloud/point_cloud.h PATHS "${draco_INCLUDE_DIR}/..")
  find_library(draco_LIBRARIES NAMES draco PATHS "${draco_LIBRARY_DIR}")
  message("Draco found: ${draco_INCLUDE_DIRS}, ${draco_LIBRARIES}")
endif()

add_message_files(FILES
  CompressedPointCloud2.msg
)

generate_messages(DEPENDENCIES
  sensor_msgs
  std_msgs
)

generate_dynamic_reconfigure_options(cfg/DracoPublisher.cfg cfg/DracoSubscriber.cfg)

catkin_package(
  INCLUDE_DIRS include
  LIBRARIES ${PROJECT_NAME}
  CATKIN_DEPENDS message_runtime point_cloud_transport sensor_msgs std_msgs
)

include_directories(include ${catkin_INCLUDE_DIRS} SYSTEM ${draco_INCLUDE_DIRS})

set(SOURCE_FILES
  src/draco_publisher.cpp
  src/draco_subscriber.cpp
)

add_library(${PROJECT_NAME}
  ${SOURCE_FILES}
)
add_dependencies(${PROJECT_NAME} ${${PROJECT_NAME}_EXPORTED_TARGETS})
target_link_libraries(${PROJECT_NAME} PUBLIC ${catkin_LIBRARIES} PRIVATE ${draco_LIBRARIES})

add_library(${PROJECT_NAME}_plugin
  ${SOURCE_FILES}
  src/manifest.cpp
)
add_dependencies(${PROJECT_NAME}_plugin ${${PROJECT_NAME}_EXPORTED_TARGETS})
target_link_libraries(${PROJECT_NAME}_plugin PUBLIC ${catkin_LIBRARIES} PRIVATE ${draco_LIBRARIES})
class_loader_hide_library_symbols(${PROJECT_NAME}_plugin)

install(TARGETS ${PROJECT_NAME} ${PROJECT_NAME}_plugin
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_GLOBAL_BIN_DESTINATION}
)

install(DIRECTORY include/${PROJECT_NAME}/
  DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
  FILES_MATCHING PATTERN "*.h"
)

install(FILES draco_plugins.xml
  DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
)

if (CATKIN_ENABLE_TESTING)
  find_package(roslint REQUIRED)

  # catkin_lint - checks validity of package.xml and CMakeLists.txt
  # ROS buildfarm calls this without any environment and with empty rosdep cache,
  # so we have problems reading the list of packages from env
  # see https://github.com/ros-infrastructure/ros_buildfarm/issues/923
  if(DEFINED ENV{ROS_HOME})
    #catkin_lint: ignore_once env_var
    set(ROS_HOME "$ENV{ROS_HOME}")
  else()
    #catkin_lint: ignore_once env_var
    set(ROS_HOME "$ENV{HOME}/.ros")
  endif()
  #catkin_lint: ignore_once env_var
  if(DEFINED ENV{ROS_ROOT} AND EXISTS "${ROS_HOME}/rosdep/sources.cache")
    roslint_custom(catkin_lint "-W2" .)
  endif()

  # Roslint C++ - checks formatting and some other rules for C++ files

  file(GLOB_RECURSE ROSLINT_INCLUDE include/*.h include/*.hpp)
  file(GLOB_RECURSE ROSLINT_SRC src/*.cpp src/*.hpp src/*.h)
  #file(GLOB_RECURSE ROSLINT_TEST test/*.cpp)

  set(ROSLINT_CPP_OPTS "--extensions=h,hpp,hh,c,cpp,cc;--linelength=120;--filter=\
    -build/header_guard,-readability/namespace,-whitespace/braces,-runtime/references,\
    -build/c++11,-readability/nolint,-readability/todo,-legal/copyright,-build/namespaces")
  roslint_cpp(${ROSLINT_INCLUDE} ${ROSLINT_SRC})

  roslint_add_test()
endif()
