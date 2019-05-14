# GraspingInPointCloud
Details on implementing a robot manipulator to grasp objects using point cloud data. Read the report for more details.

## Hardware
- Aubo-i5 robot
- Asus Xtion Pro RGB-D camera

## Software Used
- All is done in Ubuntu 16.04
- OpenRave
- ([GPD](https://github.com/atenpas/gpd)) package for generating grasp poses point cloud data
- [TrajOpt](https://github.com/joschu/trajopt)
- Some installation [notes](https://github.com/hhn1n15/BugFixingHistory)

## Steps:
- Camera calibration:
  - RGB calibration: [tutorial](http://wiki.ros.org/camera_calibration/Tutorials/MonocularCalibration)
  - Depth calibration: [jsk_pcl_ros](https://jsk-recognition.readthedocs.io/en/latest/jsk_pcl_ros/calibration.html) <br /> The depth sensor of the Asus camera has significant error (5 cm at 50 cm distance), therefore it is very important to calibrate this sensor properly. I detected the error when using the Aruco tag, the coordinate of the Aruco tag is 5 cm behind the point cloud data of the tag. After calibration, the distance is about 1 cm which is sufficiently good enough. The above package tried to align depth estimation from RGB images and the depth output from the sensor. It can be installed through ```apt-get install```. 
  
    <p align="center">
    <img src="/images/after_calibration_2.png" width="600" />
    </p>
  
  It is also important to move the checkerboard to cover as much space as possible. You also should focus on taking samples in your working space as well. Improperly calibration will result in skewed point cloud which is not useful.
  
- Coordinate transformation: <br /> For planning the robot to go to the desired location to grasp, we need to transform whatever the camera sees to the robot coordinate. That is the transformation matrix from the camera_optical_coordinate to the robot_base_coordinate. I used Aruco tag as the intermediate step.
    
    <p align="center">
    <img src="/images/Aruco_Calibration.png" width="500" />
    </p>
    
    - The Aruco_coordinate (displayed in the above figure) can be computed easily compared to the camera_optical_coordinate. 
    - As the tag is sticked on the end-effector in a known way, the relationship between Aruco_coordinate and the end_effector_coordinate is also determined.
    - The forward kinematics gives you the last missing piece, the relationship between the end_effector_coordinate and the robot_base_coordinate.
    - I found it very useful for error detection to publish all the coordinates in rviz using tf_static_publisher from ROS. I created a launch file for that purpose. An example is like the below code when I can determine the robot_base_frame from the end_effector_frame using the robot kinematics.

      ```
      <launch>
      <node pkg="tf" type="static_transform_publisher" name="robot_base_broadcaster" args=" -0.214593534602693  -0.629495407290422  -0.215495156750654 1.616801510011362  -1.303983886960539   1.570780925228566 end_effector_frame robot_base_frame  100" />
      </launch>      
      ```
    - When having all the coordinates, to compute the transformation matrix from the optical_camera_coordinate to the robot_base_frame, we can just use the lookupTransform function. There is also a github [repo](https://github.com/IFL-CAMP/easy_handeye) doing the same thing.
      
    <p align="center">
    <img src="/images/calibration_1.png" width="500" />
    </p>      
      
- Prepare OpenRave environment: OpenRave is the environment that the motion planner (TrajOpt) used to plan the collision-free trajectories. It will contain: the robot model, the environment model (the point cloud data that the camera sees, transformed to the robot coordinate).
  - Aubo-i5 robot model: In this [link](https://github.com/hhn1n15/aubo_i5_full), an urdf model of the robot is stored. However, the model needs to be converted to .dae file to be used by OpenRave. Example command for the conversion (the ros workspace needs to be compiled first so that the system knows where the file is) ```rosrun collada_urdf urdf_to_collada <input-urdf> <output.dae>```. The output .dae file can be loaded in OpenRave using the command ```openrave aubo_i5_full.dae```.
  - The transformed point cloud data: As we already have the transformation matrix from the camera_optical_coordinate to the robot_base_coordinate. It is quite straight-forward to convert the point cloud data in the robot_base_coordinate. The example code is in this [line](https://github.com/hhn1n15/GraspingInPointCloud/blob/master/test_trajectory_no_pedestal.py#L2590). In this, the transformation matrix is stored in a [text file](https://github.com/hhn1n15/GraspingInPointCloud/blob/master/xtion_frame.txt). In this script, the point cloud data is saved in .pcd format as an input.
  - Environment [file](https://github.com/hhn1n15/GraspingInPointCloud/blob/master/environment.xml): This follows OpenRave xml format. Basically, it will load the robot and do some camera view settings. It is important however, to define a manipulator in this file, with the defined base and end-effector as the manipulator name will be used in the [constraints](https://github.com/hhn1n15/GraspingInPointCloud/blob/master/test_trajectory_no_pedestal.py#L2724) of OpenRave.
  - Loading everything together:
  <p align="center">
  <img src="/images/OpenRave_1.png" width="700" />
  </p>
  - Object selection: Color images and point cloud data are combined to select a specific object in a scene of multiple objects. First, a YOLOv3 network is trained to put a bounding box around the selected objects. Details on how to train YOLO-v3 with custom objects can be seen in https://github.com/AlexeyAB. For instance, when we want to grasp the coke in the below image, YOLO can put a bounding box around the object. As we need to generate grasp on the object using point cloud data, we need to know which is the corresponding point cloud of the selected object. Here are steps to do that:
    - From the point cloud data of objects, we first separate each point cloud data of each object, using filters such as statistical filter, voxel filter, and Euclidean cluster extraction.
    - Calculating the centroid of these clusters.
    - Convert these centroids to image plane using camera calibration matrices.
    - Calculate the distance to the center of the bounding box of the selected object, calculated from YOLO. The smallest distance belong to the cluster of the selected object in 2D. The whole process is illustrated in following images. Note that yellow squares are the projections of 3D centroids, the white dot (on the coke can) is the center of the bounding box from YOLO.
   
  <p align="center">
  <img src="/images/predictions.jpg" width="300" />
  <img src="/images/pcl.png" width="300" /> 
  <img src="/images/four_objects.png" width="300" />
  </p>


  - Grasp generation:
Given the point cloud data of the selected object is determined, GPD can be used to generated grasp on the object. Filtering is also applied to give priority of the poses that are closer to the orientation of the gripper.
- Grasp execution:
Given the point cloud data of objects, the desired grasp pose, and the model of the robot, TrajOpt can generate collision-free trajectories guiding the robot to the desired gripper. The trajectory is then replayed in the real robot.
- Results:
<p align="center">
<img src="/images/output.gif" width="450" />
<img src="/images/robot.gif" width="450" />  
</p>
