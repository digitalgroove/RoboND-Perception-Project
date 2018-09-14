[![Udacity - Robotics NanoDegree Program](https://s3-us-west-1.amazonaws.com/udacity-robotics/Extra+Images/RoboND_flag.png)](https://www.udacity.com/robotics)
# 3D Perception
Before starting any work on this project, please complete all steps for [Exercise 1, 2 and 3](https://github.com/udacity/RoboND-Perception-Exercises). At the end of Exercise-3 you have a pipeline that can identify points that belong to a specific object.

In this project, you must assimilate your work from previous exercises to successfully complete a tabletop pick and place operation using PR2.

The PR2 has been outfitted with an RGB-D sensor much like the one you used in previous exercises. This sensor however is a bit noisy, much like real sensors.

Given the cluttered tabletop scenario, you must implement a perception pipeline using your work from Exercises 1,2 and 3 to identify target objects from a so-called “Pick-List” in that particular order, pick up those objects and place them in corresponding dropboxes.

# Project Setup
For this setup, catkin_ws is the name of active ROS Workspace, if your workspace name is different, change the commands accordingly
If you do not have an active ROS workspace, you can create one by:

```sh
$ mkdir -p ~/catkin_ws/src
$ cd ~/catkin_ws/
$ catkin_make
```

Now that you have a workspace, clone or download this repo into the src directory of your workspace:
```sh
$ cd ~/catkin_ws/src
$ git clone https://github.com/udacity/RoboND-Perception-Project.git
```
### Note: If you have the Kinematics Pick and Place project in the same ROS Workspace as this project, please remove the 'gazebo_grasp_plugin' directory from the `RoboND-Perception-Project/` directory otherwise ignore this note. 

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
```
source ~/catkin_ws/devel/setup.bash
```

To run the demo:
```sh
$ cd ~/catkin_ws/src/RoboND-Perception-Project/pr2_robot/scripts
$ chmod u+x pr2_safe_spawner.sh
$ ./pr2_safe_spawner.sh
```
![demo-1](https://user-images.githubusercontent.com/20687560/28748231-46b5b912-7467-11e7-8778-3095172b7b19.png)



Once Gazebo is up and running, make sure you see following in the gazebo world:
- Robot

- Table arrangement

- Three target objects on the table

- Dropboxes on either sides of the robot


If any of these items are missing, please report as an issue on [the waffle board](https://waffle.io/udacity/robotics-nanodegree-issues).

In your RViz window, you should see the robot and a partial collision map displayed:

![demo-2](https://user-images.githubusercontent.com/20687560/28748286-9f65680e-7468-11e7-83dc-f1a32380b89c.png)

Proceed through the demo by pressing the ‘Next’ button on the RViz window when a prompt appears in your active terminal

The demo ends when the robot has successfully picked and placed all objects into respective dropboxes (though sometimes the robot gets excited and throws objects across the room!)

Close all active terminal windows using **ctrl+c** before restarting the demo.

You can launch the project scenario like this:
```sh
$ roslaunch pr2_robot pick_place_project.launch
```
# Required Steps for a Passing Submission:
1. Extract features and train an SVM model on new objects (see `pick_list_*.yaml` in `/pr2_robot/config/` for the list of models you'll be trying to identify).
** Generate Features: **
To generate features, first launch the training.launch file to bring up the Gazebo environment:
```$ roslaunch sensor_stick training.launch```
Next, in a new terminal, run the capture_features.py script to capture and save features for each of the objects in the environment.
```$ rosrun sensor_stick capture_features.py```
**Note:** The training_set.sav file will be saved in the current directory, where the script was executed.
This script spawns each object in 20 random orientations.
The script was modified to include following models:
- biscuits
- book
- eraser
- glue
- snacks
- soap
- soap2
- sticky_notes

After that you run the train_svm.py script to train an SVM classifier on your labeled set of features.
``` $ rosrun sensor_stick train_svm.py ```

**Training Results:**
``` 
Features in Training Set: 160  
Invalid Features in Training set: 0  
Scores: [ 0.9375   0.96875  0.84375  0.90625  0.9375 ]  
Accuracy: 0.92 (+/- 0.08)  
accuracy score: 0.91875  
```

The following plots show the relative accuracy of the classifier for the various objects:

![demo-1](https://github.com/digitalgroove/RoboND-Perception-Project/blob/master/writeup_images/SVM-Confusion-Matrix-Perception-Project.png)
**Image: Confusion matrix**

Running the above command will also result in your trained model being saved in a model.sav file.
**Note:** This model.sav file will be saved in the current directory, where the script was executed.

**Note:** keep in mind that the model.sav file needs to be in the same directory where you run this!
**Note2:** chmod +x project_template.py

Copy the model.sav to the directory where the project_template.py script is located.

To test with the development so far, first run:
``` $ roslaunch pr2_robot pick_place_project.launch ```

and then in another terminal, run your object recognition node:
**IMPORTANT:** RUN THE NEXT SCRIPT FROM THE DIRECTORY WHERE THE FILE MODEL.SAV IS LOCATED:
e.g first $ cd ~/catkin_ws/src/RoboND-Perception-Project/pr2_robot/scripts
``` $ rosrun pr2_robot project_template.py ```


2. Write a ROS node and subscribe to `/pr2/world/points` topic. This topic contains noisy point cloud data that you must work with.
3. Remove outliers using a statistical outlier filter

To clean up the noise of the camera image we use the statistical outlier filter found in python-pcl:
![demo-1](https://github.com/digitalgroove/RoboND-Perception-Project/blob/master/writeup_images/perception-project-noisy-cloud.png)
**Image: Point cloud before statistical outlier filtering**

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

![demo-1](https://github.com/digitalgroove/RoboND-Perception-Project/blob/master/writeup_images/perception-project-filtered-cloud.png)
**Image: Point cloud after statistical outlier filtering**

4. Use filtering and RANSAC plane fitting to isolate the objects of interest from the rest of the scene.
5. Apply Euclidean clustering to create separate clusters for individual items.
6. Perform object recognition on these objects and assign them labels (markers in RViz).
7. Calculate the centroid (average in x, y and z) of the set of points belonging to that each object.
8. Create ROS messages containing the details of each object (name, pick_pose, etc.) and write these messages out to `.yaml` files, one for each of the 3 scenarios (`test1-3.world` in `/pr2_robot/worlds/`).  See the example `output.yaml` for details on what the output should look like.  
9. Submit a link to your GitHub repo for the project or the Python code for your perception pipeline and your output `.yaml` files (3 `.yaml` files, one for each test world).  You must have correctly identified 100% of objects from `pick_list_1.yaml` for `test1.world`, 80% of items from `pick_list_2.yaml` for `test2.world` and 75% of items from `pick_list_3.yaml` in `test3.world`.

# Extra Challenges: Complete the Pick & Place
7. To create a collision map, publish a point cloud to the `/pr2/3d_map/points` topic and make sure you change the `point_cloud_topic` to `/pr2/3d_map/points` in `sensors.yaml` in the `/pr2_robot/config/` directory. This topic is read by Moveit!, which uses this point cloud input to generate a collision map, allowing the robot to plan its trajectory.  Keep in mind that later when you go to pick up an object, you must first remove it from this point cloud so it is removed from the collision map!
8. Rotate the robot to generate collision map of table sides. This can be accomplished by publishing joint angle value(in radians) to `/pr2/world_joint_controller/command`
9. Rotate the robot back to its original state.
10. Create a ROS Client for the “pick_place_routine” rosservice.  In the required steps above, you already created the messages you need to use this service. Checkout the [PickPlace.srv](https://github.com/udacity/RoboND-Perception-Project/tree/master/pr2_robot/srv) file to find out what arguments you must pass to this service.
11. If everything was done correctly, when you pass the appropriate messages to the `pick_place_routine` service, the selected arm will perform pick and place operation and display trajectory in the RViz window
12. Place all the objects from your pick list in their respective dropoff box and you have completed the challenge!
13. Looking for a bigger challenge?  Load up the `challenge.world` scenario and see if you can get your perception pipeline working there!

For all the step-by-step details on how to complete this project see the [RoboND 3D Perception Project Lesson](https://classroom.udacity.com/nanodegrees/nd209/parts/586e8e81-fc68-4f71-9cab-98ccd4766cfe/modules/e5bfcfbd-3f7d-43fe-8248-0c65d910345a/lessons/e3e5fd8e-2f76-4169-a5bc-5a128d380155/concepts/802deabb-7dbb-46be-bf21-6cb0a39a1961)
Note: The robot is a bit moody at times and might leave objects on the table or fling them across the room :D
As long as your pipeline performs succesful recognition, your project will be considered successful even if the robot feels otherwise!
