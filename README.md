[![Udacity - Robotics NanoDegree Program](https://s3-us-west-1.amazonaws.com/udacity-robotics/Extra+Images/RoboND_flag.png)](https://www.udacity.com/robotics)
# 3D Perception Project

**Writeup by Roberto Zegers**   
I programmed a PR2 robot that uses data from a RGB-D sensor to identify objects on a cluttered tabletop. The information about detected objects is than used by the robot to pick up the objects that belong to a “Pick-List” and place them in corresponding dropboxes.  
The multiple stages involved in the creation of this perception process are described bellow in a step by step manner.

![](https://github.com/digitalgroove/RoboND-Perception-Project/blob/master/writeup_images/2018-09-25-042815_1920x958_scrot.png)  
**Image 1: The completed pick and place process**

### Table of Contents
**Part 1: Tabletop Segmentation**
1. Create ROS node and subscribe to data from RGB-D camera
2. Remove noise using a statistical outlier filter
3. Downsample point cloud by applying a Voxel Grid Filter
4. Apply a Pass Through Filter to isolate the table and objects
5. Perform RANSAC plane fitting to identify the table
6. Create new point clouds containing the table and objects separately  

**Part 2: Euclidean Clustering for Object Segmentation**
1. Create separate clusters for individual items
2. Cluster Visualization
3. Publish point clouds of individual items

**Part 3: Implement Object Recognition**
1. Extract features of all objects
2. Train SVM model on all objects
3. Perform object recognition on objects

**Part 4: Write output as yaml file**
1. Object's centroid
2. Pick Pose
3. Name of the object
4. Position of a target dropbox
5. Robot arm to be used
6. Test scene number
7. Populate ROS messages and call the service 'pick_place_routine'

**Part 5: Future Improvements**

**Part 6: Setup Instructions**  
1. Installation Guide
2. Running the Project

## Part 1: Tabletop Segmentation 

### 1. Create ROS node and subscribe to data from RGB-D camera
I used the _project_template.py_ file as starting point.
As first step, inside the python main block, I added code to perform:
- ROS node initialization
- Create Subscribers and subscribe to `/pr2/world/points` topic and use pcl_callback() as callback function
- Create Publishers
- Load Model From disk _(provided by the template file)_
- Spin script while node is not shutdown


All steps described below were implemented inside the pcl_callback() function:  
``` 
def pcl_callback(pcl_msg): 

```

### 2. Remove noise using a statistical outlier filter  
The `/pr2/world/points` topic contains noisy point cloud data that must be filtered to reduce noise.  
To clean up the noise I implemented the statistical outlier filter found in python-pcl.  

| ![](https://github.com/digitalgroove/RoboND-Perception-Project/blob/master/writeup_images/perception-project-noisy-cloud.png)     |  ![](https://github.com/digitalgroove/RoboND-Perception-Project/blob/master/writeup_images/perception-project-filtered-cloud.png) 
:-------------------------:|:-------------------------:
**Image: Point cloud before statistical outlier filtering**         |  **Image: Point cloud after statistical outlier filtering** |

In addition to that I republished the filtered cloud to the topic `/pcl_stat_outlier_filter`  
It was crucial to fine tune the parameters to get the statistical outlier filter right:
- Set the number of neighboring points to analyze for any given point
- Set threshold scale factor

```python
    ## Statistical Outlier Filtering
    # Much like the previous filters, we start by creating a filter object: 
    outlier_filter = pcl_data.make_statistical_outlier_filter()

    # Set the number of neighboring points to analyze for any given point
    outlier_filter.set_mean_k(10)

    # Set threshold scale factor
    # All points who have a distance larger than X standard deviation of the mean distance
    # to the query point will be marked as outliers and removed
    x = 0.01

    # Any point with a mean distance larger than global (mean distance+x*std_dev) will be considered outlier
    outlier_filter.set_std_dev_mul_thresh(x)

    # Finally call the filter function for magic
    cloud_outlier_filtered = outlier_filter.filter()
```

### 3. Downsample point cloud by applying a Voxel Grid Filter
In this step I implemented a VoxelGrid Downsampling Filter to derive a point cloud that has fewer points than the original but that still does a good job representing the input point cloud as a whole.  
Note: This block of code was developed and fine tuned as part of [Perception Exercise 2](https://github.com/udacity/RoboND-Perception-Exercises).

```python
     ## Voxel Grid filter
     # A voxel grid filter allows to downsample the data
 
     # Create a VoxelGrid filter object for our input point cloud
     vox = cloud_outlier_filtered.make_voxel_grid_filter()
 
     # Choose a voxel (also known as leaf) size
     # Note: using 1 is a poor choice of leaf size
     # Units of the voxel size (or leaf size) are in meters
     # Experiment and find the appropriate size!
     LEAF_SIZE = 0.005
 
     # Set the voxel (or leaf) size  
     vox.set_leaf_size(LEAF_SIZE, LEAF_SIZE, LEAF_SIZE)
 
     # Call the filter function to obtain the resultant downsampled point cloud
     cloud_filtered = vox.filter()
```

### 4. Apply a Pass Through Filter to isolate the table and objects
In this step I apply a Pass Through Filter to remove useless data from the point cloud (like cropping the point cloud).  
Note: This block of code was also developed and fine tuned as part of [Perception Exercise 2](https://github.com/udacity/RoboND-Perception-Exercises).

```python
     ## PassThrough filter (was TODO)
     # It allows to crop a part by specifying an axis and cut-off values
 
     # Create a PassThrough filter object.
     passthrough = cloud_filtered.make_passthrough_filter()
 
     # Assign axis and range to the passthrough filter object.
     # Here axis_min and max is the height with respect to the ground
     filter_axis = 'z'
     passthrough.set_filter_field_name(filter_axis)
     axis_min = 0.6 # to retain only the tabletop and the objects sitting on the table
     axis_max = 1.0 # to filter out the upper part of the cloud
     passthrough.set_filter_limits(axis_min, axis_max)
 
     # Finally use the filter function to obtain the resultant point cloud. 
     cloud_passthrough_1 = passthrough.filter()
```

**In addition to the filtering of values based on the Z axis I had to add another Pass Through Filter based on the X axis values of the point cloud to filter out the edges of the dropboxes:**
```python
    # Create a PassThrough filter object.
    passthrough_x = cloud_passthrough_1.make_passthrough_filter()
    # Assign axis and range to the passthrough filter object.
    # Here axis_min and max is the height with respect to the ground
    filter_axis = 'x'
    passthrough_x.set_filter_field_name(filter_axis)
    x_axis_min = 0.35 # to filter out the closest part of the cloud
    x_axis_max = 1.5 # to filter out the farest part of the cloud
    passthrough_x.set_filter_limits(x_axis_min, x_axis_max)
    # Finally use the filter function to obtain the resultant point cloud. 
    cloud_passthrough_2 = passthrough_x.filter()
```

### 5. Perform RANSAC plane fitting to identify the table
In this step I apply a RANSAC algorithm provided by the PCL library to identify points that belong to the table (a plane) and separate them from other points (the objects).  
Note: This part of the code was mainly developed during [Perception Exercise 2](https://github.com/udacity/RoboND-Perception-Exercises).
```python 
    ## RANSAC plane segmentation
    # Identifies points that belong to a particular model (plane, cylinder, box, etc.)

    # Create the segmentation object
    seg = cloud_passthrough_2.make_segmenter()

    # Set the model you wish to fit 
    seg.set_model_type(pcl.SACMODEL_PLANE)
    seg.set_method_type(pcl.SAC_RANSAC)

    # Max distance for a point to be considered fitting the model
    # Experiment with different values for max_distance 
    # for segmenting the table
    max_distance = 0.01
    seg.set_distance_threshold(max_distance)

    # Call the segment function to obtain set of inlier indices and model coefficients
    inliers, coefficients = seg.segment()
```

### 6. Create new point clouds containing the table and objects separately
With RANSAC I identified which points in the cloud correspond to the table (inlier indices). The indices not corresponding to the table/plane are those representing the objects on the table (outliers).  
Note: This part of the code was also developed during [Perception Exercise 2](https://github.com/udacity/RoboND-Perception-Exercises):
```python 
     ## Extract inliers and outliers
     # Allows to extract points from a point cloud by providing a list of indices
     cloud_table = cloud_passthrough_2.extract(inliers, negative=False)
     # With the negative flag True we retrieve the points that do not fit the RANSAC model
     cloud_objects = cloud_passthrough_2.extract(inliers, negative=True)
```

## Part 2: Euclidean Clustering for Object Segmentation
### 1. Create separate clusters for individual items
Here I use a use a PCL library function called EuclideanClusterExtraction() to segment the points representing the objects on the table into individual objects.  
At the end of the code block _cluster_indices_ contains a list of indices for each cluster. In the next step, I will create a new point cloud to visualize the clusters by assigning a color to each of them.
```python
    ## Euclidean Clustering
    # Apply function to convert XYZRGB to XYZ
    white_cloud = XYZRGB_to_XYZ(cloud_objects) 
    tree = white_cloud.make_kdtree() # returns a kd-tree

    # Perform the cluster extraction
    # Create a cluster extraction object
    ec = white_cloud.make_EuclideanClusterExtraction()
    # Set tolerances for distance threshold 
    # as well as minimum and maximum cluster size (in points)
    ec.set_ClusterTolerance(0.05)
    ec.set_MinClusterSize(10)
    ec.set_MaxClusterSize(5000)
    # Search the k-d tree for clusters
    ec.set_SearchMethod(tree)
    # Extract indices for each of the discovered clusters
    cluster_indices = ec.Extract()
```

### 2. Cluster Visualization
Here I apply a unique color to each object's point cloud in order to be able to visualize the results in RViz!
```python
    ## Create Cluster-Mask Point Cloud to visualize each cluster separately
    # Assign a color corresponding to each segmented object in scene
    cluster_color = get_color_list(len(cluster_indices))

    color_cluster_point_list = []

    for j, indices in enumerate(cluster_indices):
        for i, indice in enumerate(indices):
            color_cluster_point_list.append([white_cloud[indice][0],
                                            white_cloud[indice][1],
                                            white_cloud[indice][2],
                                             rgb_to_float(cluster_color[j])])

    # Create new cloud containing all clusters, each with unique color
    cluster_cloud = pcl.PointCloud_PointXYZRGB()
    cluster_cloud.from_list(color_cluster_point_list)
```

### 3. Publish point clouds of individual items 
Before publishing the cloud of individual items on a separate topic I have to convert it to ROS' PointCloud2 type:

```python
    ## Convert PCL data to ROS messages (was TODO)
    ros_cloud_objects = pcl_to_ros(cloud_objects)
    ros_cloud_table = pcl_to_ros(cloud_table)
    # Cloud containing all clusters (objects), each with unique color:
    ros_cluster_cloud = pcl_to_ros(cluster_cloud) 
    ros_outlier_filtered_cloud = pcl_to_ros(cloud_outlier_filtered)

    ## Publish ROS messages (was TODO)
    pcl_objects_pub.publish(ros_cloud_objects)
    pcl_table_pub.publish(ros_cloud_table)
    # Cloud containing all clusters (objects), each with unique color:
    pcl_cluster_pub.publish(ros_cluster_cloud)
    stat_outlier_filter_pub.publish(ros_outlier_filtered_cloud)
```

## Part 3: Implement Object Recognition

### 1. Extract features of all objects (one time task)   
In order to be able to recognize objects, it is neccesary to first generate the object's features.  
Note: This task has to be executed one time, and is not run during the perception process.  
To extract the features of the objects I launched the `training.launch` file to bring up the Gazebo environment that I used to capture RGB-D point clouds of the objects:  
```$ roslaunch sensor_stick training.launch```  
Next, in a new terminal, I ran the `capture_features.py` script to capture and save features for each of the objects in the environment:  
```$ rosrun sensor_stick capture_features.py```  
As a result, the training_set.sav file was saved in the current directory, where the script was executed.  

Note: I modified the **capture_features.py** script to include all object models:  
_(code displayed shortened for brevity)_

```python
...
if __name__ == '__main__':
    rospy.init_node('capture_node')

    models = [\
       'biscuits',
       'book',
       'eraser',
       'glue',
       'snacks',
       'soap',
       'soap2',
       'sticky_notes']
...
```
I also modified the script to spawn each object in 20 random orientations to obtain more data for training the SVM model.

### 2. Train an SVM model on all objects (one time task)   
After that I ran the `train_svm.py` script to train an SVM classifier on my labeled set of features.  
``` $ rosrun sensor_stick train_svm.py ```

The above command creates a trained model that is saved in a model.sav file.
**Note:** This model.sav file is saved in the current directory, where the script is executed.

**I obtained following training results:**
``` 
Features in Training Set: 160  
Invalid Features in Training set: 0  
Scores: [ 0.9375   0.96875  0.84375  0.90625  0.9375 ]  
Accuracy: 0.92 (+/- 0.08)  
accuracy score: 0.91875  
```

An this is the relative accuracy of the classifier that I got:

![demo-1](https://github.com/digitalgroove/RoboND-Perception-Project/blob/master/writeup_images/SVM-Confusion-Matrix-Perception-Project.png)
**Image: Confusion matrix**

Next I copied the model.sav file to the directory where the project_template.py script is located.  
**Note:** This is because the model.sav file has to be in the same directory where the perception pipeline script is run!


### 3. Perform object recognition on objects
Continuing with the perception pipeline script, I wrote a for loop to cycle through each of the segmented clusters.  
The code then extracts the **color histograms** and **normal histograms** of the current object point cloud and uses the trained SVM model to find a match to an object.  
Note: This part of the code was mainly developed during [Perception Exercise 3](https://github.com/udacity/RoboND-Perception-Exercises).
```python
    for index, pts_list in enumerate(cluster_indices):
        # Grab the points for the cluster from the extracted objects (cloud_objects)
        pcl_cluster = cloud_objects.extract(pts_list)
        # Convert the cluster from pcl to ROS using helper function (was TODO)
        ros_cluster = pcl_to_ros(pcl_cluster)

        # Extract histogram features (Compute the associated feature vector)
        # Complete this step just as is covered in capture_features.py (was TODO)
        chists = compute_color_histograms(ros_cluster, using_hsv=True)
        normals = get_normals(ros_cluster)
        nhists = compute_normal_histograms(normals)
        feature = np.concatenate((chists, nhists))

        # Make the prediction, retrieve the label for the result
        # and add it to detected_objects_labels list
        prediction = clf.predict(scaler.transform(feature.reshape(1,-1)))
        label = encoder.inverse_transform(prediction)[0]
        detected_objects_labels.append(label)
```

To finish this step I created a label that can be read by Rviz to display it on top of the object (see image below):

![demo-1](https://github.com/digitalgroove/RoboND-Perception-Project/blob/master/writeup_images/perception-project-world3-results.png)
**Image: Point cloud after object recognition and object labeling tested in World 3**


```python
        # Publish a label into RViz
        label_pos = list(white_cloud[pts_list[0]])
        label_pos[2] += .4
        object_markers_pub.publish(make_label(label,label_pos, index))

        # Add the detected object to the list of detected objects
        # Declare a object of message type DetectedObject:
        do = DetectedObject() 
        do.label = label
        do.cloud = ros_cluster
        # A list of detected objects (of message type DetectedObject)
        detected_objects.append(do)

    ## end of for loop ##
    # Prints list of detected objects:
    rospy.loginfo('Detected {} objects: {}'.format(len(detected_objects_labels), detected_objects_labels))

    # Publish the list of detected objects
    detected_objects_pub.publish(detected_objects)
```

## Part 4: Write output as yaml file
The script template file defines a function called pr2_mover() to load parameters and request the PickPlace service

In this step I complete the pr2_mover() function so that the appropriate fields that the “pick_place” rosservice requires to operate are filled.  
Then I save that information as a yaml file that can be read by the ROS service.  

### 1.  Object's centroid
In this step I calculated the average in x, y and z of the set of points belonging to the current object.
But first, I verified that the predicted object's name was inside the pick up list.If the detected object is not in the pick-up list, the algorithm continues with other objects in the for loop.  
In the case it is part of the pick-up list, the centroid is calculated and the script continues to prepare the data needed to write an output file as yaml file.

```python
    ## Loop through the pick list 
    # Loop over our list using a plain for-in loop
    for index, item in enumerate(object_list_param):

        print "========== NEW ITERATION OVER PICK UP LIST ==========" # for debugging      
        ## Parse parameters into individual variables (was TODO)
        # object_list_param can be parsed to obtain object names and associated group
        object_name = object_list_param[index]['name']
        object_group = object_list_param[index]['group'] # will be either green or red
        # print object_name # for debugging

        ## Get the PointCloud for a given object and obtain it's centroid (was TODO)
        try:
            # Find the position of the current object from the pick up list inside the list of detected_objects
            # Get index in a list of objects by attribute            
            idx = [ x.label for x in object_list ].index(object_name)
        except ValueError:
            print "Object in pick-up list was not detected: {}".format(object_name)
            print "Continue with other objects in pick up list."
            continue
        print "Object to pick up: {}".format(object_name) # for debugging
        print "Index in list of detected objects (object_list): {}".format(idx) # for debugging
        # Use that position to retrieve the associated point cloud of the object
        pcl = object_list[idx].cloud
        labels.append(object_name)
        points_arr = ros_to_pcl(pcl).to_array()
        centroids.append(np.mean(points_arr, axis=0)[:3])
```



### 2. Pick Pose
Calculated pose of the recognized object's centroid.
```python
        ## Pick pose
        # Initialize an empty pose message
        pick_pose = Pose()

        # Fill in appropriate fields, access last object in centroids list
        # Recast to native Python float type using np.asscalar()
        pick_pose.position.x = np.asscalar(centroids[-1][0])
        pick_pose.position.y = np.asscalar(centroids[-1][1])
        pick_pose.position.z = np.asscalar(centroids[-1][2])
```
### 3. Name of the object
Create and fill in a message variable containing the name of the object.
```python
        # Initialize a variable
        msg_object_name = String()
        # Populate the data field
        msg_object_name.data = object_name
```
### 4. Position of a target dropbox
Get the position for a given dropbox and its coordinates as placement pose for the object
```python
        # Search a list of dictionaries and return a selected value in selected dictionary 
        selected_entry = [item for item in dropbox_obj_param if item['group'] == object_group][0]
        dropbox_position = selected_entry.get('position')
        # print "Position extracted from yaml file: {}".format(dropbox_position) # for debugging
      
        ## Create 'place_pose' or object placement pose for the object (was TODO)
        # Initialize an empty pose message
        place_pose = Pose()
        # Fill in appropriate fields
        place_pose.position.x = dropbox_position[0]
        place_pose.position.y = dropbox_position[1]
        place_pose.position.z = dropbox_position[2]
```
### 5. Robot arm to be used
Assign the arm to be used for pick and place operation
```python
        # Name of the arm can be either right/green or left/red
        # Initialize a variable
        which_arm = String()
        # Populate the data field
        if object_group == 'green':
            which_arm.data = 'right' 
        else:
            which_arm.data = 'left'
        print "Arm: {}".format(which_arm.data) # for debugging
```
### 6. Test scene number
Get the test scene number (either 1, 2 or 3)
```python
        # Initialize the test_scene_num variable
        test_scene_num = Int32()
        # Get/Read parameters of dropbox positions
        test_scene = rospy.get_param('/test_scene_num') # parameter name
        # Populate the data field of that variable:
        test_scene_num.data = test_scene
        # print "Test Scene Number: {}".format(test_scene_num.data) # for debugging


```
### 7. Populate ROS messages and call the service 'pick_place_routine'
Using the provided make_yaml_dict() helper function I converted the messages to dictionaries.  
Next I call rospy.wait_for_service() to block the code execution until the service 'pick_place_routine' is available.  
Then I call the service by creating a rospy.ServiceProxy with 'pick_place_routine' as the name of the service to call.  

```python
        ## Populate various ROS messages
        yaml_dict = make_yaml_dict(test_scene_num, which_arm, msg_object_name, pick_pose, place_pose)
        dict_list.append(yaml_dict)

        # Wait for 'pick_place_routine' service to come up
        rospy.wait_for_service('pick_place_routine')


        try:
            pick_place_routine = rospy.ServiceProxy('pick_place_routine', PickPlace)

            ## Insert your message variables to be sent as a service request (was TODO)
            resp = pick_place_routine(test_scene_num, msg_object_name, which_arm, pick_pose, place_pose)
            print ("Response: ",resp.success)

        except rospy.ServiceException, e:
            print "Service call failed: %s"%e

    ## end of for loop ## 
```

Finally I output the request parameters into a output yaml file:
```python
    # yaml filenames: output_1.yaml, output_2.yaml, and output_3.yaml
    try:
        yaml_filename = 'output_'+str(test_scene)+'.yaml'
        send_to_yaml(yaml_filename, dict_list) # list of dictionaries
        print "Saved output yaml file."
    except:
        print "No objects detected, no information to save on yaml file."
```

The block below shows how the output file (output_1.yaml) looks like:  

```
object_list:
- arm_name: right
  object_name: biscuits
  pick_pose:
    orientation:
      w: 0.0
      x: 0.0
      y: 0.0
      z: 0.0
    position:
      x: 0.5414684414863586
      y: -0.24128590524196625
      z: 0.7050934433937073
  place_pose:
    orientation:
      w: 0.0
      x: 0.0
      y: 0.0
      z: 0.0
    position:
      x: 0
      y: -0.71
      z: 0.605
  test_scene_num: 1
- arm_name: right
  object_name: soap
  pick_pose:
    orientation:
      w: 0.0
      x: 0.0
      y: 0.0
      z: 0.0
    position:
      x: 0.5446743965148926
      y: -0.018615946173667908
      z: 0.6766468286514282
  place_pose:
    orientation:
      w: 0.0
      x: 0.0
      y: 0.0
      z: 0.0
    position:
      x: 0
      y: -0.71
      z: 0.605
  test_scene_num: 1
- arm_name: left
  object_name: soap2
  pick_pose:
    orientation:
      w: 0.0
      x: 0.0
      y: 0.0
      z: 0.0
    position:
      x: 0.44491374492645264
      y: 0.2209254503250122
      z: 0.676362931728363
  place_pose:
    orientation:
      w: 0.0
      x: 0.0
      y: 0.0
      z: 0.0
    position:
      x: 0
      y: 0.71
      z: 0.605
  test_scene_num: 1

```

## Part 5: Future Improvements
- On the current version, the code has to loop over the complete pick list and execute a pick/place routine for each item before the yaml output file is written.
- The inverse kinematics could be improved to avoid the swirling of the robot arm.
- The logic of the gripper could be improved (e.g. make use of the orientation data).
- If an object is erroneously detected and this can be confirmed because the detected object is not part of the pick up list, one could try to use the object with the second highest class score instead.
- What did not work well was to modify the SVM parameters to use a _rbf_ kernel function instead of a _linear_ kernel function. Maybe with some more time to tweak it better one could obtain better results.

## Part 6: Setup Instructions
### Installation Guide
For this setup, I asume that catkin_ws is the name of your ROS Workspace.

Clone or download this repo into the src directory of your workspace:
```sh
$ cd ~/catkin_ws/src
$ git clone https://github.com/digitalgroove/RoboND-Perception-Project.git
```
**Note: If you have the Kinematics Pick and Place project in the same ROS Workspace as this project, please remove the 'gazebo_grasp_plugin' directory from the `RoboND-Perception-Project/` directory otherwise ignore this note.**

Now install missing dependencies using rosdep install:
```sh
$ cd ~/catkin_ws
$ rosdep install --from-paths src --ignore-src --rosdistro=kinetic -y
```
Build the project:
```sh
$ cd ~/catkin_ws
$ catkin_make
```
Add following to your .bashrc file
```
export GAZEBO_MODEL_PATH=~/catkin_ws/src/RoboND-Perception-Project/pr2_robot/models:$GAZEBO_MODEL_PATH
```

If you haven’t already, following line can be added to your .bashrc to auto-source all new terminals
```sh
source ~/catkin_ws/devel/setup.bash
```

### Running the Project

You can run the project like this:
```sh
$ cd ~/catkin_ws/src/RoboND-Perception-Project/pr2_robot/scripts
$ roslaunch pr2_robot pick_place_project.launch
```
Once Gazebo is up and running, make sure you see following in the gazebo world:
- Robot
- Table arrangement
- Three target objects on the table
- Dropboxes on either sides of the robot

Then run the perception pipeline algorithm:

```sh
$ rosrun pr2_robot project_template.py
```
Note:  If the following error message appers `No such file or directory: 'model.sav'` make sure you run the launch file from inside the `/scripts` directory as stated above.

Proceed through the pick and place operation by pressing the ‘Next’ or ‘Continue’ button on the RViz window called "RvizVisualToolsGuide".

To change the world number edit this file: **pick_place_project.launch**
And change the argument "test_scene_num" to a value of 1, 2 or 3.
```
  <!--TODO:Change the test number based on the scene you want loaded-->
  <arg name="test_scene_num" value="1"/>
```
Then run again as described above.
