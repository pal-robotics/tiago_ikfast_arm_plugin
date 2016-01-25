# TIAGo IKfast MoveIt! Inverse Kinematics plugin
This repository contains the autogenerated IKfast MoveIt! plugin for TIAGo's arm (`master` branch is equivalent to the `freindex0` branch). The results were unusable. All the instructions to do it by yourself (maybe for other robot) are here.


# TIAGo IKfast setup and generation

## Requirements

1) Install [ROS Indigo](http://wiki.ros.org/indigo/Installation/Ubuntu).

2) Install [TIAGo simulation](http://wiki.ros.org/Robots/TIAGo) (click on Indigo).

You can download all necessary repositories using the rosinstall `workspace_rosinstall`. Just do:
````
mkdir -p ~/ikfast_ws/src
cd ~/ikfast_ws/src
catkin_init_workspace
wget https://raw.githubusercontent.com/pal-robotics/tiago_ikfast_arm_plugin/master/workspace_rosinstall .rosinstall
wstool update -j4
````
Or follow the step after step instructions you'll find nextly.

# Let's do an IKfast .cpp file

This tutorial is based in the [baxter tutorial](http://sdk.rethinkrobotics.com/wiki/Custom_IKFast_for_your_Baxter).

## Install OpenRAVE
    git clone --branch latest_stable https://github.com/rdiankov/openrave.git

It's a big repo (400~MB), takes a while.

Then (adjust -jX to your number of available CPU cores)

    make -j4

This took several minutes on a i7.

    sudo make install


## Get the tiago_ikfast repository

    git clone https://github.com/pal-robotics/tiago_ikfast.git

It's a package with the URDF of the arm of TIAGo having removed any link/joint not in the kinematic chain between the base and end-effector.

## Check the indices of the arm
You may need:

    sudo apt-get install python-scipy

Links:
````
$ openrave0.9-robot.py arm.rounded.dae --info links
name            index parents        
-------------------------------------
torso_lift_link 0                    
arm_1_link      1     torso_lift_link
arm_2_link      2     arm_1_link     
arm_3_link      3     arm_2_link     
arm_4_link      4     arm_3_link     
arm_5_link      5     arm_4_link     
arm_6_link      6     arm_5_link     
arm_7_link      7     arm_6_link     
arm_tool_link   8     arm_7_link     
-------------------------------------
name            index parents        
````
Joints:
````
$ openrave0.9-robot.py arm.rounded.dae --info joints
name           joint_index dof_index parent_link     child_link    mimic
------------------------------------------------------------------------
arm_1_joint    0           0         torso_lift_link arm_1_link         
arm_2_joint    1           1         arm_1_link      arm_2_link         
arm_3_joint    2           2         arm_2_link      arm_3_link         
arm_4_joint    3           3         arm_3_link      arm_4_link         
arm_5_joint    4           4         arm_4_link      arm_5_link         
arm_6_joint    5           5         arm_5_link      arm_6_link         
arm_7_joint    6           6         arm_6_link      arm_7_link         
arm_tool_joint -1          -1        arm_7_link      arm_tool_link      
------------------------------------------------------------------------
name           joint_index dof_index parent_link     child_link    mimic
````


## Generate the IKfast solver
Locate your `ikfast.py`

````
sam@tosh:~/ikfast_ws/src/tiago_ikfast$ sudo updatedb
sam@tosh:~/ikfast_ws/src/tiago_ikfast$ locate ikfast.py
/home/sam/ikfast_ws/src/openrave/python/ikfast.py
/home/sam/ikfast_ws/src/openrave/test/test_ikfast.py
/usr/local/lib/python2.7/dist-packages/openravepy/_openravepy_0_9/ikfast.py
````
And execute:

    python /usr/local/lib/python2.7/dist-packages/openravepy/_openravepy_0_9/ikfast.py --robot=arm.rounded.dae --iktype=transform6d --baselink=0 --eelink=8 --freeindex=0 --savefile=tiago_arm_ikfast_solver.cpp

`--freeindex=X` is pending research for reasoning the ideal value. Seems like it tries by default every 0.1 radians as can be seen in the [source code](https://github.com/rdiankov/openrave/blob/master/python/databases/inversekinematics.py#L832) it can also be tuned with the parameter `--freeinc=X.YY`.

It took 12 minutes on a `Intel® Core™ i7 CPU Q 720 @ 1.60GHz × 8` as said by Ubuntu's About this computer.

**Note**: `--freeindex=0` gave bad results, it was giving not many good IKs as when dragging the MoveIt! Rviz plugin marker to move the arm, as can be seen in [this video](https://youtu.be/gmp73K5fC6Y).

At the end of the document you'll find the results of trying the rest of `--freeindex=X`.

## Generate MoveIt IKfast plugin
Following instructions from the [moveit_ikfast tutorial](http://docs.ros.org/indigo/api/moveit_ikfast/html/doc/ikfast_tutorial.html).

Create a package for our plugin

    catkin_create_pkg tiago_ikfast_arm_plugin

Compile so ROS notices it exists

    cd ~/your_ws
    catkin_make

Also, we need the source of `tiago_moveit_config` as we'll modify a couple of things nextly:

    git clone https://github.com/pal-robotics/tiago_moveit_config.git

In order to create the plugin with `moveit_ikfast` it expects the `.srdf` of the robot to be called `tiago.srdf` which is not our case, so we'll do a tiny hack:

    roscd tiago_moveit_config/config
    ln -s ~/ikfast_ws/src/tiago_moveit_config/config/tiago_steel.srdf tiago.srdf

Also, as of writting at 24/01/2016 moveit_ikfast has not been released with [the needed fix](https://github.com/ros-planning/moveit_ikfast/issues/49) that solves the error:
````
Traceback (most recent call last):
  File "/opt/ros/indigo/lib/moveit_ikfast/create_ikfast_moveit_plugin.py", line 178, in <module>
    solver_version = int(line_search.group(1))
ValueError: invalid literal for int() with base 10: '0x10000048'
````
So we need to checkout and compile the indigo version:

    git clone https://github.com/ros-planning/moveit_ikfast.git
    cd moveit_ikfast
    git checkout indigo-devel
    cd ~/ikfast_ws
    catkin_make

Create the source code of the plugin:

    rosrun moveit_ikfast create_ikfast_moveit_plugin.py tiago arm tiago_ikfast_arm_plugin ~/ikfast_ws/src/tiago_ikfast/tiago_arm_ikfast_solver.cpp

The output looked like:
````
Warning: The default search has changed from OPTIMIZE_FREE_JOINT to now OPTIMIZE_MAX_JOINT!

IKFast Plugin Generator
Loading robot from 'tiago_moveit_config' package ... 
Creating plugin in 'tiago_ikfast_arm_plugin' package ... 
  found 3 planning groups: arm, arm_torso, gripper
  found group 'arm'
  found source code generated by IKFast version 268435528

Created plugin file at '/home/sam/ikfast_ws/src/tiago_ikfast_arm_plugin/src/tiago_arm_ikfast_moveit_plugin.cpp'

Created plugin definition at: '/home/sam/ikfast_ws/src/tiago_ikfast_arm_plugin/tiago_arm_moveit_ikfast_plugin_description.xml'

Overwrote CMakeLists file at '/home/sam/ikfast_ws/src/tiago_ikfast_arm_plugin/CMakeLists.txt'

Modified kinematics.yaml at /home/sam/ikfast_ws/src/tiago_moveit_config/config/kinematics.yaml

Created update plugin script at /home/sam/ikfast_ws/src/tiago_ikfast_arm_plugin/update_ikfast_plugin.sh
````

Now just check that it compiles going to the workspace and `catkin_make` it.

And then... try it!

    roslaunch tiago_moveit_config demo.launch robot:=steel

You can change the solver at `tiago_moveit_config/config/kinematics.yaml`.

## Computing all possible --freeindex=X

### --freeindex=1
    python /usr/local/lib/python2.7/dist-packages/openravepy/_openravepy_0_9/ikfast.py --robot=arm.rounded.dae --iktype=transform6d --baselink=0 --eelink=8 --freeindex=1 --savefile=tiago_arm_freeindex_1_ikfast_solver.cpp
    cd ~/ikfast_ws/src
    rosrun moveit_ikfast create_ikfast_moveit_plugin.py tiago arm tiago_ikfast_arm_plugin /home/sam/ikfast_ws/src/tiago_ikfast/tiago_arm_freeindex_1_ikfast_solver.cpp
    cd ..
    catkin_make
Worked as bad as `--freeindex=0`.

### --freeindex=2
    python /usr/local/lib/python2.7/dist-packages/openravepy/_openravepy_0_9/ikfast.py --robot=arm.rounded.dae --iktype=transform6d --baselink=0 --eelink=8 --freeindex=2 --savefile=tiago_arm_freeindex_2_ikfast_solver.cpp
    cd ~/ikfast_ws/src
    rosrun moveit_ikfast create_ikfast_moveit_plugin.py tiago arm tiago_ikfast_arm_plugin /home/sam/ikfast_ws/src/tiago_ikfast/tiago_arm_freeindex_2_ikfast_solver.cpp
    cd ..
    catkin_make
Worked as bad, or worse, than `--freeindex=0`.

### --freeindex=3
    python /usr/local/lib/python2.7/dist-packages/openravepy/_openravepy_0_9/ikfast.py --robot=arm.rounded.dae --iktype=transform6d --baselink=0 --eelink=8 --freeindex=3 --savefile=tiago_arm_freeindex_3_ikfast_solver.cpp
    cd ~/ikfast_ws/src
    rosrun moveit_ikfast create_ikfast_moveit_plugin.py tiago arm tiago_ikfast_arm_plugin /home/sam/ikfast_ws/src/tiago_ikfast/tiago_arm_freeindex_3_ikfast_solver.cpp
    cd ..
    catkin_make
Worked as bad than `--freeindex=0`.

### --freeindex=4
    python /usr/local/lib/python2.7/dist-packages/openravepy/_openravepy_0_9/ikfast.py --robot=arm.rounded.dae --iktype=transform6d --baselink=0 --eelink=8 --freeindex=4 --savefile=tiago_arm_freeindex_4_ikfast_solver.cpp
Didn't finish, last output (was stuck here for more than 10minutes) before stopping the process was:
````
INFO: matrix has 104 symbols
INFO: skipping dependent index 3
INFO: skipping dependent index 7
INFO: computed non-singular AU matrix
INFO: reducing 12 equations
WARNING: equation way too complex (1982), looking for another solution
WARNING: equation way too complex (1979), looking for another solution
WARNING: equation way too complex (1969), looking for another solution
WARNING: equation way too complex (2001), looking for another solution
WARNING: equation way too complex (3131), looking for another solution
WARNING: equation way too complex (3185), looking for another solution
WARNING: equation way too complex (2023), looking for another solution
WARNING: equation way too complex (2056), looking for another solution
WARNING: equation way too complex (4070), looking for another solution
WARNING: equation way too complex (4071), looking for another solution
````

### --freeindex=5
    python /usr/local/lib/python2.7/dist-packages/openravepy/_openravepy_0_9/ikfast.py --robot=arm.rounded.dae --iktype=transform6d --baselink=0 --eelink=8 --freeindex=5 --savefile=tiago_arm_freeindex_5_ikfast_solver.cpp
Didn't finish, last output (was stuck here for more than 10minutes) before stopping the process was:
````
INFO: matrix has 96 symbols
INFO: skipping dependent index 3
INFO: skipping dependent index 7
INFO: computed non-singular AU matrix
INFO: reducing 12 equations
WARNING: equation way too complex (2212), looking for another solution
WARNING: equation way too complex (2209), looking for another solution
WARNING: equation way too complex (2158), looking for another solution
WARNING: equation way too complex (2368), looking for another solution
WARNING: equation way too complex (3546), looking for another solution
WARNING: equation way too complex (3751), looking for another solution
WARNING: equation way too complex (2210), looking for another solution
WARNING: equation way too complex (2421), looking for another solution
WARNING: equation way too complex (4626), looking for another solution
WARNING: equation way too complex (4631), looking for another solution
````

### --freeindex=6
Still computing after 1h...

    python /usr/local/lib/python2.7/dist-packages/openravepy/_openravepy_0_9/ikfast.py --robot=arm.rounded.dae --iktype=transform6d --baselink=0 --eelink=8 --freeindex=6 --savefile=tiago_arm_freeindex_6_ikfast_solver.cpp



## Possible extra work
Try to do it with the torso lift joint, that would be 8 DOF, but `moveit_ikfast` does not support the creation of the plugin. Some handmade work may be needed. And well, after seeing the results we got, it's probably not going to be very useful.

