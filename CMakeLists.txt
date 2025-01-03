cmake_minimum_required(VERSION 3.8)
project(slam)

# Make sure to set this path before building the  
set(ENV{PYTHONPATH} "/opt/ros/humble/lib/python3.10/site-packages/") # Must be set to match your installation

# Must use C++17 to make it compatible with rclcpp
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -O3")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -O3")

# Check C++17 support
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++17" COMPILER_SUPPORTS_CXX17)

if(COMPILER_SUPPORTS_CXX17)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++17")
  add_definitions(-DCOMPILEDWITHC17)
  message(STATUS "Using flag -std=c++17.")
else()
  message(FATAL_ERROR "The compiler ${CMAKE_CXX_COMPILER} has no C++17 support. Please use a different C++ compiler.")
endif()

LIST(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake_modules) # REDUNDANT?

# find dependencies
find_package(ament_cmake REQUIRED)
find_package(ament_cmake_python REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rclpy REQUIRED)
find_package(std_msgs REQUIRED)
find_package(visualization_msgs REQUIRED)
find_package(std_srvs REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(geometry_msgs REQUIRED)
# find_package(your_custom_msg_interface REQUIRED)
find_package(cv_bridge REQUIRED)
find_package(OpenCV 4.2 REQUIRED)
find_package(Eigen3 3.3.0 REQUIRED) # Matched with Sophus
find_package(Pangolin REQUIRED)
find_package(image_transport REQUIRED)
find_package(custom_interfaces REQUIRED)
find_package(tf2 REQUIRED)
find_package(tf2_ros REQUIRED)
find_package(tf2_geometry_msgs REQUIRED)
find_package(Yaml-cpp REQUIRED)
find_package(MORB_SLAM REQUIRED COMPONENTS MORB_SLAM DBoW2 g2o sophus)
find_package(octomap REQUIRED)
find_package(octomap_ros REQUIRED)
find_package(octomap_msgs REQUIRED)

# Header file locations [C++ node]
include_directories(
	${ament_INCLUDE_DIRS}
	$<INSTALL_INTERFACE:include>
	include 
	${EIGEN3_INCLUDE_DIRS}
	${Pangolin_INCLUDE_DIRS}
	${OpenCV_INCLUDE_DIRS}
	${YAMLCPP_INCLUDE_DIR}
	${OCTOMAP_INCLUDE_DIRS})



set(THIS_PACKAGE_INCLUDE_DEPENDS
  rclcpp
  rclpy
  std_msgs
  visualization_msgs
  std_srvs
  sensor_msgs
  geometry_msgs
  # your_custom_msg_interface
  cv_bridge
  image_transport
  # OpenCV
  # Eigen3
  # Pangolin
  custom_interfaces
  tf2
  tf2_ros
  tf2_geometry_msgs
  geometry_msgs
)

#* ORB SLAM3 as a shared library
# Whenever you add a new .h, .hpp or .cc, .cpp file in the ros2_orb_slam3/orb_slam3/include and os2_orb_slam3/orb_slam3/src directories, make sure to add them here as shown below
add_library(orb_slam3_lib SHARED
  orb_slam3/src/System.cc
  orb_slam3/src/Tracking.cc
  orb_slam3/src/LocalMapping.cc
  orb_slam3/src/LoopClosing.cc
  orb_slam3/src/ORBextractor.cc
  orb_slam3/src/ORBmatcher.cc
  orb_slam3/src/OctoMapBuilder.cpp
  orb_slam3/src/FrameDrawer.cc
  orb_slam3/src/Converter.cc
  orb_slam3/src/MapPoint.cc
  orb_slam3/src/KeyFrame.cc
  orb_slam3/src/Atlas.cc
  orb_slam3/src/Map.cc
  orb_slam3/src/MapDrawer.cc
  orb_slam3/src/Optimizer.cc
  orb_slam3/src/Frame.cc
  orb_slam3/src/KeyFrameDatabase.cc
  orb_slam3/src/Sim3Solver.cc
  orb_slam3/src/Viewer.cc
  orb_slam3/src/ImuTypes.cc
  orb_slam3/src/G2oTypes.cc
  orb_slam3/src/CameraModels/Pinhole.cpp
  orb_slam3/src/CameraModels/KannalaBrandt8.cpp
  orb_slam3/src/OptimizableTypes.cpp
  orb_slam3/src/MLPnPsolver.cpp
  orb_slam3/src/GeometricTools.cc
  orb_slam3/src/TwoViewReconstruction.cc
  orb_slam3/src/Config.cc
  orb_slam3/src/Settings.cc
)

# ORB-SLAM3 includes [VSLAM library]
target_include_directories(
	orb_slam3_lib
	PUBLIC ${PROJECT_SOURCE_DIR}
	PUBLIC ${PROJECT_SOURCE_DIR}/orb_slam3
	PUBLIC ${PROJECT_SOURCE_DIR}/orb_slam3/include
	PUBLIC ${PROJECT_SOURCE_DIR}/orb_slam3/include/CameraModels
	PUBLIC ${PROJECT_SOURCE_DIR}/orb_slam3/Thirdparty
	PUBLIC ${PROJECT_SOURCE_DIR}/orb_slam3/Thirdparty/Sophus
)


set_target_properties(orb_slam3_lib PROPERTIES VERSION "${orb_slam3_lib_VERSION}") # TODO need to findout why this is required

# ament_target_dependencies(orb_slam3_lib
#   ${THIS_PACKAGE_INCLUDE_DEPENDS}
# )


# Link libraries
target_link_libraries(orb_slam3_lib
  ${OpenCV_LIBS}
  ${EIGEN3_LIBS}
  ${Pangolin_LIBRARIES}
  ${PROJECT_SOURCE_DIR}/orb_slam3/Thirdparty/DBoW2/lib/libDBoW2.so
  ${PROJECT_SOURCE_DIR}/orb_slam3/Thirdparty/g2o/lib/libg2o.so
  -lboost_system
  -lboost_serialization
  -lcrypto 
)


# Find the .so files provided in ros2_orb_slam3/orb_slam3/Thirdparty projects
set(DBoW2_PATH "${PROJECT_SOURCE_DIR}/orb_slam3/Thirdparty/DBoW2/lib/libDBoW2.so")
set(g2o_PATH "${PROJECT_SOURCE_DIR}/orb_slam3/Thirdparty/g2o/lib/libg2o.so")

# Install .so files to the lib directory in the install space
install(FILES ${DBoW2_PATH} DESTINATION lib)
install(FILES ${g2o_PATH} DESTINATION lib)

# Add executable
add_executable(orbslam3_mono_node
	src/node.cpp
	src/slam.cpp
	src/monocular.cpp
	src/orbslam3/monocular.cpp
	src/monocular_driver.cpp
)
set(USE_ORBLAM3 1)
target_compile_definitions(orbslam3_mono_node PRIVATE USE_ORBSLAM3=${USE_ORBLAM3})
target_include_directories(
	orbslam3_mono_node
	PUBLIC ${PROJECT_SOURCE_DIR}
	PUBLIC ${PROJECT_SOURCE_DIR}/orb_slam3
	PUBLIC ${PROJECT_SOURCE_DIR}/orb_slam3/include
	PUBLIC ${PROJECT_SOURCE_DIR}/orb_slam3/include/CameraModels
	PUBLIC ${PROJECT_SOURCE_DIR}/orb_slam3/Thirdparty
	PUBLIC ${PROJECT_SOURCE_DIR}/orb_slam3/Thirdparty/Sophus
)
add_executable(morbslam_mono_node
	src/node.cpp
	src/slam.cpp
	src/monocular.cpp
	src/morbslam/monocular.cpp
	src/monocular_driver.cpp
)
# target_include_directories()
set(USE_MORBLAM 1)
target_compile_definitions(morbslam_mono_node PRIVATE USE_MORBSLAM=${USE_MORBSLAM})

ament_target_dependencies(orbslam3_mono_node
  PUBLIC ${THIS_PACKAGE_INCLUDE_DEPENDS}
)
ament_target_dependencies(morbslam_mono_node
  PUBLIC ${THIS_PACKAGE_INCLUDE_DEPENDS}
)

target_link_libraries(orbslam3_mono_node 
	PUBLIC orb_slam3_lib
  	PUBLIC ${YAMLCPP_LIBRARY}
	PUBLIC yaml-cpp) # Link a node with the internal shared library
target_link_libraries(morbslam_mono_node
	PUBLIC MORB_SLAM::MORB_SLAM
	PUBLIC MORB_SLAM::DBoW2
	PUBLIC MORB_SLAM::g2o
	PUBLIC MORB_SLAM::sophus
  	PUBLIC ${YAMLCPP_LIBRARY}
	PUBLIC yaml-cpp
	PUBLIC ${OCTOMAP_LIBRARIES})

# Install all the header files in package/package/include
install (DIRECTORY include/ 
        DESTINATION include
)

# Install our node and library
install(TARGETS orbslam3_mono_node orb_slam3_lib
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION lib/${PROJECT_NAME}
  INCLUDES DESTINATION include/${PROJECT_NAME}
)

# Tell downstream packages where to find the headers
ament_export_include_directories(include)

# Help downstream packages to find transitive dependencies
ament_export_dependencies(
  orb_slam3_lib
  ${THIS_PACKAGE_INCLUDE_DEPENDS}
)


# Python node
# Install Python modules, submodules
ament_python_install_package(${PROJECT_NAME}) # Install the modules in ros2_orb_slam3/ros2_orb_slam3 folder

ament_package()
