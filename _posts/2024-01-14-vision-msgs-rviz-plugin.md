---
title: 'Introducing a New RVIZ2 Plugin for Displaying Vision_msgs in ROS 2'
date: 2024-01-14
permalink: /posts/2024/01/1/
tags:
  - Gazebo
  - ROS2
  - Docker
---


Are you looking for an easy and efficient way to display object detection data in ROS 2 humble[^1]? If so, I have some exciting news for you! We have just released a new RVIZ2 plugin that can help you visualize [vision_msgs](https://github.com/ros-perception/vision_msgs) in a visually appealing and informative way.

My [RVIZ2 plugin](https://github.com/NovoG93/vision_msgs_rviz_plugins) is easy to use and comes with several useful features that can help ROS 2 users save time and effort when working with object detection data. Here's a brief overview of the features you can expect:

- [x] Detection3DArray
  - [x] Display ObjectHypothesisWithPose/score
  - [x] Change color based on ObjectHypothesisWithPose/id [car: orange, person: blue, cyclist: yellow, motorcycle: purple, other: grey]
  - [x] Visualization properties
    - [x] Alpha
    - [x] Line or Box
    - [x] Linewidth
    - [x] Change color map based on provided yaml file
- [x] Detection3D
  - [x] Display ObjectHypothesisWithPose/score
  - [x] Change color based on ObjectHypothesisWithPose/id [car: orange, person: blue, cyclist: yellow, motorcycle: purple, other: grey]
  - [x] Visualization properties
    - [x] Alpha
    - [x] Line or Box
    - [x] Linewidth
    - [x] Change color map based on provided yaml file
- [x] BoundingBox3D
    - [x] Alpha
    - [x] Line or Box    
        <span style="color:red">**Since no header in vision_msgs/BonudingBox visualizations uses rviz fixed frame for tf transformation**</span>
    - [x] Linewidth
- [x] BoundingBox3DArray
    - [x] Alpha
    - [x] Line or Box
    - [x] Linewidth

## Overview

To help you understand how the plugin works, I've included some screenshots below:


| [Gif](https://github.com/NovoG93/vision_msgs_rviz_plugins/blob/humble/assets/BBoxArray.gif?) | Image |
| --- | --- |
| ![Gif](https://global.discourse-cdn.com/business7/uploads/ros/original/3X/e/c/ec1da5217561a09e8415bb4d100b2efb5437dfcc.gif) | <p align="center"><img src="https://global.discourse-cdn.com/business7/uploads/ros/original/3X/0/0/00530d22151ea21cda9221006c9c0105400f15d8.png" height="auto" width="500"></p> |

As you can see, the RVIZ2 plugin can help you analyze object detection data with ease. The ability to adjust the display properties, change color maps, and choose between line or box displays can help you gain better insights into your algorithms and streamline your workflow.


## Install and Testing

__Install:__
1. Via apt:
```bash
sudo apt update && sudo apt install ros-$ROS_DISTRO-vision-msgs-rviz-plugins
```

2. From source:
```bash
$ cd ros2_ws/src && git clone https://github.com/NovoG93/vision_msgs_rviz_plugins -b humble
$ cd ros2_ws && rosdep install --from src --ignore-src -r -y \
  && colcon build --symlink-install --packages-select vision_msgs_rviz_plugins
```

__Testing:__
```bash
$ ros2 launch vision_msgs_rviz_plugins test_all.launch.py 
```


## Conclusion
I hope you find my RVIZ2 plugin useful in your work with ROS 2 humble. If you have any questions or feedback, please feel free to leave a comment below. Thank you for your support!

[^1]: As pointed out [here](https://github.com/NovoG93/vision_msgs_rviz_plugins/issues/3#issuecomment-1442682145) it should work starting from Galactic.