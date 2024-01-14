---
title: 'Introducing a ROS2-Gazebo Drone Simulation Plugin: A New Frontier in ease of use for UAV Simulation'
date: 2024-01-14
permalink: /posts/2024/01/1/
tags:
  - Gazebo
  - ROS2
  - Docker
---



As a fervent enthusiast and professional in the realm of robotics and autonomous systems, I am excited to share with the community: the release of a my ROS2-Gazebo drone simulation plugin. This project, which can be found on GitHub ([sjtu_drone](https://github.com/NovoG93/sjtu_drone)), represents a UAV (Unmanned Aerial Vehicle) simulation.

## What is the ROS2-Gazebo Drone Simulation Plugin?

The ROS2-Gazebo drone simulation plugin is a tool designed for the simulation of quadcopter drones within the Gazebo environment, integrated with ROS2. This plugin is a ROS2 port, with modifications, of the [sjtu_drone](https://github.com/tahsinkose/sjtu-drone) plugin developed by [Shanghai Jiao Tong University](https://www.sjtu.edu.cn/). The plugin is designed to be user-friendly and highly customizable.

### Key Features:

- **Advanced Simulation Capabilities**: The plugin offers high-fidelity simulation of quadcopter drones, incorporating realistic physics and environmental interactions.
- **ROS2 Integration**: Seamless integration with ROS2 ensures compatibility with the latest developments in robotic software.
- **Customizability**: Users can customize various aspects of the simulation, including drone parameters and environmental conditions, to suit their specific needs.

### ROS2 Packages:

The ROS2-Gazebo drone simulation plugin consists of the following ROS2 packages:

- **sjtu_drone_description**: [This package](https://github.com/NovoG93/sjtu_drone/tree/ros2/sjtu_drone_description) contains the core plugin, which is responsible for the simulation of the drone in gazebo.
- **sjtu_drone_bringup**: [This package](https://github.com/NovoG93/sjtu_drone/tree/ros2/sjtu_drone_bringup) contains the launch files and configuration files for starting and configuring the simulation.
- **sjtu_drone_control**: [This package](https://github.com/NovoG93/sjtu_drone/tree/ros2/sjtu_drone_control) contains simple examples to steer the drone using topics and teleop.

<br/>

## How to Use the Plugin

To use the ROS2-Gazebo drone simulation plugin, users need to have a basic understanding of ROS2 and Gazebo. Please refer to the ROS2 wiki for a [basic tutorial of ROS2](https://docs.ros.org/en/iron/Tutorials.html). The plugin is designed to be user-friendly, and the following instructions will guide you through the process of installing and using the plugin.

The drone simulation plugin can be installed by compiling the ROS2 packages from source, or by using the provided Docker image. The following sections will guide you through the installation process for both methods.

### Docker:

1. **Installation**: The docker image can be installed by running the following command:

```bash
docker pull georgno/sjtu_drone:ros2-iron
```

2. **Running Simulations**: Once the docker image is pulled, users can start the container by running [run_docker.sh](https://github.com/NovoG93/sjtu_drone/blob/ros2/run_docker.sh):

```bash
curl -s https://raw.githubusercontent.com/NovoG93/sjtu_drone/ros2/run_docker.sh | bash -s
```
### Source Installation:

1. **Installation**: The ROS2 packages can be installed by cloning the repository into your workspace and building the packages using colcon:

```bash
git clone https://github.com/NovoG93/sjtu_drone/ -b ros2 <your_workspace>/src/sjtu_drone
cd <your_workspace>
rosdep install -r -y --from-paths src --ignore-src --rosdistro $ROS_DISTRO 
colcon build --symlink-install --packages-select-regex sjtu*
```

2. **Running Simulations**: Once the packages are built, users can start the simulation by launching the [sjtu_drone_bringup.launch.py]() file:

```bash
ros2 launch sjtu_drone_bringup sjtu_drone_bringup.launch.py
```

<br/>

## Contributing to the Project

As an open-source project, contributions from the community are highly encouraged. Whether it's through code contributions, bug reports, or feature suggestions, your input is invaluable in improving and advancing this tool.


## Conclusion

I invite you to explore the [GitHub repository](https://github.com/NovoG93/sjtu_drone), try out the plugin, and provide feedback. I look forward to seeing how this project will evolve and grow with the help of the community.
