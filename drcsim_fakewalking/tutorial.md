# Overview

**IMPORTANT: This tutorial only works with ROS Groovy.**

This tutorial will explain how to drive simulated Atlas around as if it were a wheeled robot (i.e., without walking or balancing).

## Setup

We assume that you've already done the [installation step](http://gazebosim.org/tutorials/?tut=drcsim_install).

If you haven't done so, add the environment setup.sh files to your .bashrc.

~~~
echo 'source /usr/share/drcsim/setup.sh' >> ~/.bashrc
source ~/.bashrc
~~~

We're going to use the [pr2_teleop](http://ros.org/wiki/pr2_teleop) package to drive Atlas around with keyboard commands. Install the Ubuntu package that contains it:

~~~
sudo apt-get install ros-groovy-pr2-teleop-app
~~~

## Background

Note that this tutorial does not use the [walking controller](http://gazebosim.org/tutorials/?tut=drcsim_walking&cat=drcsim). It simply spawns Atlas with position controllers enabled that keep it standing upright, as shown in the following image:

[[file:files/Gazebo_with_drc_robot.png|640px]]

Note that all of the robot's joints, including the legs, are physically simulated and actively controlled.

So the simulated robot in this tutorial can't walk, but we still want to move it around in the world.  Fortunately, the simulated robot accepts velocity commands via ROS to translate and rotate in the plane, as if it were a wheeled robot.

## The code

1. Start the simulator:

    ~~~
    VRC_CHEATS_ENABLED=1 roslaunch drcsim_gazebo atlas.launch
    ~~~
    
    >**For drcsim < 3.1.0**: The package and launch file had a different name:
    
    >~~~
    VRC_CHEATS_ENABLED=1 roslaunch atlas_utils atlas.launch
    >~~~
    
    Note: Setting the variable `VRC_CHEATS_ENABLED=1` exposes several development aid topics including `/atlas/cmd_vel`, which are by default disabled for the VRC competition.

2. In another shell, start `pr2_teleop/pr2_teleop_keyboard`:

    ~~~
    rosrun pr2_teleop teleop_pr2_keyboard cmd_vel:=atlas/cmd_vel
    ~~~
    
    You should see something like:
    
            Reading from keyboard
            ---------------------------
            Use 'WASD' to translate
            Use 'QE' to yaw
            Press 'Shift' to run

3. Follow the instructions: get the robot moving with those keys.  Press any other key to stop. Control-C to stop the teleop utility.

## How does that work?


The simulated robot is awaiting [ROS Twist](http://ros.org/doc/api/geometry_msgs/html/msg/Twist.html) messages, which specify 6-D velocities, on the `atlas/cmd_vel` topic.  Check that with [rostopic](http://ros.org/wiki/rostopic):

~~~
rostopic info atlas/cmd_vel
~~~

You should see something like:

~~~
Type: geometry_msgs/Twist

Publishers:
 * /pr2_base_keyboard (http://osrf-Latitude-E6420:36506/)

Subscribers:
 * /gazebo (http://osrf-Latitude-E6420:35339/)
~~~

The teleop utility is simply converting your keyboard input to messages of that type and publishing them to that topic.  You can publish such messages from anywhere, including from the command line, using rostopic.  First, let's see what's in a Twist message, using [rosmsg](http://ros.org/wiki/rosmsg):

~~~
rosmsg show Twist
~~~

You should see:

~~~
[geometry_msgs/Twist]:
geometry_msgs/Vector3 linear
  float64 x
  float64 y
  float64 z
geometry_msgs/Vector3 angular
  float64 x
  float64 y
  float64 z
~~~

It's a 6-D velocity: 3 linear velocities (X, Y, and Z) and 3 angular velocities (rotations about X, Y, Z, also called roll, pitch, and yaw).  Our robot is constrained to move in the plane, so we only care about X, Y, and yaw (rotation about Z).  Make the robot drive counter-clockwise in a circle:

~~~
rostopic pub --once atlas/cmd_vel geometry_msgs/Twist '{ linear: { x: 0.5, y: 0.0, z: 0.0 }, angular: { x: 0.0, y: 0.0, z: 0.5 } }'
~~~

Note that the robot keeps moving after rostopic exits; that's because there's no watchdog that requires recent receipt of a velocity command (that may change in the future).  You can verify that no commands are being sent with this with the command:

~~~
rostopic echo atlas/cmd_vel
~~~

To stop the robot, send zero velocities:

~~~
rostopic pub --once atlas/cmd_vel geometry_msgs/Twist '{ linear: { x: 0.0, y: 0.0, z: 0.0 }, angular: { x: 0.0, y: 0.0, z: 0.0 } }'
~~~

From here, you're ready to write code that moves the robot around the world.