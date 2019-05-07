# GraspingInPointCloud
Details on implementing a robot manipulator to grasp objects using point cloud data.

## Hardware
- Aubo-i5 robot
- Asus Xtion Pro RGB-D camera

## Major Software
- All is done in Ubuntu 16.04
- OpenRave
- Grasp Pose Detect (GPD) package for generating grasp poses point cloud data

## Steps:
- Camera calibration:
  - RGB calibration: http://wiki.ros.org/camera_calibration/Tutorials/MonocularCalibration
  - Depth calibration: https://jsk-recognition.readthedocs.io/en/latest/jsk_pcl_ros/calibration.html <br /> The depth sensor of the Asus camera has significant error (5 cm at 50 cm distance), therefore it is very important to calibrate this sensor properly. I detected the error when using the Aruco tag, the coordinate of the Aruco tag is 5 cm behind the point cloud data. After calibration, the distance is about 1 cm which is sufficiently good enough. The above package tried to align depth estimation from RGB images and the depth output from the sensor. It can be installed through ```apt-get install```. 
  
  ![Depth after calibration](https://github.com/hhn1n15/GraspingInPointCloud/blob/master/images/after_calibration_2.png)
  
  It is also important to move the checkerboard to cover as much space as possible. You also should focus on taking samples in your working space as well. Improperly calibration will result in skewed point cloud which is not useful.
  
- Coordinate transformation: <br /> For planning the robot to go to the desired location to grasp, we need to transform whatever the camera sees to the robot coordinate. That is the transformation matrix from the camera_optical_coordinate to the robot_base_coordinate. I used Aruco tag as the middle step.
  
    ![Aruco Tag](https://github.com/hhn1n15/GraspingInPointCloud/blob/master/images/Aruco_Calibration.png)
    
    - The Aruco_coordinate (displayed in the above figure) can be computed easily compared to the camera_optical_coordinate. 
    - As the tag is sticked on the end-effector in a known way, the relationship between Aruco_coordinate and the end_effector_coordinate is also determined.
    - The forward kinematics gives you the last missing piece, the relationship between the end_effector_coordinate and the robot_base_coordinate.
    - I found it very useful for error detection to publish all the coordinates in rviz using tf_static_publisher from ROS. I created a launch file for that purpose. An example is like the below code when I can determine the robot_base_frame from the end_effector_frame using the robot kinematics.

      ```
      <launch>
      <node pkg="tf" type="static_transform_publisher" name="robot_base_broadcaster" args=" -0.214593534602693  -0.629495407290422  -0.215495156750654 1.616801510011362  -1.303983886960539   1.570780925228566 end_effector_frame robot_base_frame  100" />
      </launch>      
      ```
    - When having all the coordinates, to compute the transformation matrix from the optical_camera_coordinate to the robot_base_frame, we can just use the lookupTransform function. There is also a github repo doing the same thing https://github.com/IFL-CAMP/easy_handeye.
      
    ![Rviz coordinates](https://github.com/hhn1n15/GraspingInPointCloud/blob/master/images/calibration_1.png)
- Prepare OpenRave environment: OpenRave is used to 
