cmake_minimum_required(VERSION 3.0.2)

project(vio_homing)

find_package(catkin REQUIRED COMPONENTS
  rospy
  sensor_msgs
  geometry_msgs
  nav_msgs
  std_msgs
)

catkin_package(
  CATKIN_DEPENDS
    rospy
    sensor_msgs
    geometry_msgs
    nav_msgs
    std_msgs
)

catkin_install_python(
  PROGRAMS
    scripts/mavlink_imu_bridge.py
    scripts/test_odometry.py
    scripts/check_imu_timing.py
  DESTINATION
    ${CATKIN_PACKAGE_BIN_DESTINATION}
)
