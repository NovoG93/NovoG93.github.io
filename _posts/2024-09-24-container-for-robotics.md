---
title: 'How to Use Docker Containers and systemd to Manage Robotics Applications'
date: 2024-01-14
permalink: /posts/2024/09/25/
tags:
  - ROS2
  - ROS
  - Docker
  - systemd
---


# How to Use Docker Containers and systemd to Manage Robotics Applications

Managing robotics applications, especially in complex environments like search and rescue (SAR) or disaster response, requires robust, scalable, and efficient systems. Docker containers and systemd provide a powerful solution for achieving these goals, enabling portability, consistency, and ease of management. In this guide, The following blob post will discuss challenges in managing robotics applications and how container technologies like Docker, along with Linux service management tools like systemd, can streamline the deployment and operation of robotic systems as described in my paper[^footnote-Enrich23-1] and implemented at [https://github.com/TW-Robotics/search-and-rescue-robot-2024](https://github.com/TW-Robotics/search-and-rescue-robot-2024).

## Challenges when Deploying Robotics Applications
When deploying robotics applications, several challenges arise that can complicate the development, deployment, and maintenance of the system. These challenges include among others:

### **Dependency Management**
Robotic applications often require complex software stacks with many dependencies. Docker solves this by packaging all necessary components into a container, so developers don’t need to worry about managing dependencies on the host machine.

### **Consistent Environments**
Robots might be deployed on various platforms, and ensuring consistent software behavior across different hardware is challenging. Docker containers ensure that the application behaves the same way in any environment, as the dependencies and configurations are pre-defined.

### **Reliability and Uptime**
State management and service recovery are crucial for robotic applications but robotic frameworks like ROS do not provide built-in mechanisms for this.


## Why Use Docker Containers?

Docker is a containerization technology that packages your application and its dependencies into a portable and isolated environment. For robotics applications, especially those running on different hardware and operating systems, Docker simplifies deployment, reduces system conflicts, and allows for seamless scaling.

**Key Benefits of Docker Containers**:
- **Portability**: Docker containers can be easily moved between development, testing, and production environments, allowing robotic applications to run consistently across different systems.
- **Isolation**: Each application is sandboxed within its container, ensuring that dependencies and libraries do not conflict with other containers or the host system.
- **Simplified Deployment**: Using Docker images, applications can be deployed with a single command, including all necessary libraries, frameworks, and drivers, such as those required by Robot Operating System (ROS) nodes.

### Installing Docker on Ubuntu

The first necessary step is to install the required programs as shown below[^footnote-DockerInstall].

```bash
$ curl https://get.docker.com | sh  && sudo systemctl --now enable docker
```

### Using Docker for Robotics Applications


#### Running a Simple ROS Application in a Docker Container
To demonstrate the use of Docker for robotics applications, let’s consider a simple example of running a ROS master node in a Docker container.


**Creating a Dockerfile**:
Each container is defined by a Dockerfile, which specifies the base image, dependencies, and commands to run the application. For example, to create a Dockerfile to start the `rosmaster` (the core of ROS 1 based applications, enabling communication between different nodes in the system), you can use the following commands:

```bash
cat << EOF > Dockerfile.rosmaster
FROM ros:noetic-ros-base

SHELL ["/bin/bash", "-o", "pipefail", "-c"]
RUN apt-get update && rosdep update && rm -rf /var/lib/apt/lists/* 
ENV ROS_MASTER_URI="http://10.0.0.100:11311"
ENV ROS_IP="10.0.0.100"
ENV ROS_HOSTNAME="10.0.0.100"

CMD ["roscore"]
EOF
```

**Building and Running the Docker Container**:
To build the Docker container, run the following command:

```bash
docker build -t myrobot/rosmaster:v1 -f Dockerfile.rosmaster .
```

To run the container, use the following command:

```bash
docker run --rm --net=host --name rosmaster myrobot/rosmaster:v1
```

This setup allows you to run the ROS master node in a Docker container, ensuring that the application is isolated and portable across different systems. By providing the `--net=host` flag, the container shares the host network, enabling communication with other ROS nodes running on the host machine.

#### Running a Stack of ROS Nodes in Docker Containers
Rather frequently, robotic applications involve multiple ROS nodes that work together within a specific domain, e.g. `sensing`, `mapping`, `control`, ... .   
To manage these nodes efficiently, Docker Compose can be used to define and run multiple containers simultaneously and enforce dependencies between them.


**Managing Multiple Containers with Docker Compose**:

When your application involves multiple services (e.g., sensor processing, mapping, and control), Docker Compose allows you to manage them efficiently. A `docker-compose.yml` file can define multiple containers and their relationships, such as ensuring that the mapping service waits for the sensor service to start:

```bash
cat << EOF > docker-compose.yml
services:
  talker:
    image: osrf/ros:humble-desktop
    container_name: talker
    hostname: talker
    command: ros2 run demo_nodes_cpp talker
  listener:
    image: osrf/ros:humble-desktop
    container_name: listener
    hostname: listener
    command: ros2 run demo_nodes_cpp listener
    depends_on:
      - talker
EOF
```

##### Running the Docker Compose Stack
To start the Docker Compose stack, run the following command:

```bash
docker compose up
```

You should see output similar to the following:

```shell
[+] Running 14/14
 ✔ listener Pulled                                                       106.4s 
   ✔ 6414378b6477 Pull complete                                            4.7s 
   ✔ 5e134650a585 Pull complete                                            4.9s 
   ✔ 64c1e367d2c4 Pull complete                                            5.0s 
   ✔ b33f588ed24b Pull complete                                            5.0s 
   ✔ e6852c2b80f6 Pull complete                                            5.0s 
   ✔ 41d91075c9ed Pull complete                                           44.5s 
   ✔ 6410a8272a83 Pull complete                                           44.5s 
   ✔ 543a0131e0c7 Pull complete                                           45.9s 
   ✔ 319799ae1afd Pull complete                                           45.9s 
   ✔ d92d07b72e74 Pull complete                                           45.9s 
   ✔ bd82b3fc9b82 Pull complete                                           46.6s 
   ✔ 2d647f3395ef Pull complete                                          104.1s 
 ✔ talker Pulled                                                         106.4s 
[+] Running 3/3
 ✔ Network desktop_default  Created                                        0.2s 
 ✔ Container talker         Created                                        2.0s 
 ✔ Container listener       Created                                        0.1s 
Attaching to listener, talker
talker    | [INFO] [1727201540.565084273] [talker]: Publishing: 'Hello World: 1'
listener  | [INFO] [1727201540.565474261] [listener]: I heard: [Hello World: 1]
talker    | [INFO] [1727201541.565085441] [talker]: Publishing: 'Hello World: 2'
listener  | [INFO] [1727201541.565341856] [listener]: I heard: [Hello World: 2]
talker    | [INFO] [1727201542.565070940] [talker]: Publishing: 'Hello World: 3'
listener  | [INFO] [1727201542.565364976] [listener]: I heard: [Hello World: 3]
```

This command will start the `talker` and `listener` nodes in separate containers, ensuring that the `listener` node waits for the `talker` node to be ready before starting. This way, you can manage complex robotic applications with multiple nodes using Docker Compose.

## Using systemd for Service Management

**Systemd** is a Linux-based system and service manager that ensures that services (such as Docker containers) are automatically started, stopped, or restarted as needed. It also handles dependencies between services and provides detailed logging, making it easier to manage the lifecycle of your applications.

**Advantages of systemd**:
- **Automatic Restarts**: systemd can restart applications if they fail, ensuring that critical robotic functions (like sensor processing) remain operational.
- **Resource Management**: It allows the allocation of resources like CPU and memory to ensure that each container runs efficiently.
- **Service Templates**: systemd allows you to create template unit files to manage multiple containers with similar configurations.

**Example systemd Unit File**:
Here’s an example of a systemd unit file that controls a Docker container running a ROS node:

```bash
cat << EOF > $HOME/.config/systemd/user/roscore.service
[Unit]
Description=My ROS Docker Container

[Service]
Restart=always
ExecStart=/usr/bin/docker run --rm --net=host --name ros_container ros:noetic-ros-base roscore
ExecStop=/usr/bin/docker stop ros_container

[Install]
WantedBy=default.target
EOF
```

This unit file ensures that the ROS container is automatically started on system boot and restarted if it crashes. The `ExecStart` command specifies the Docker command to run the container, while `ExecStop` stops the container when the service is stopped.


### Enabling and Starting the systemd Service
To enable and start the systemd service, follow these steps:

```bash
systemctl --user enable roscore.service
systemctl --user start roscore.service
```


We can check the status of the service using the `journalctl` and/or the `systemd` command:

**journalctl**:

```bash
journalctl --user-unit -b0 -f roscore.service
```
Here we use the `-b0` flag to show logs from the current boot and `-f` to follow the logs in real-time.


**systemctl**:
```bash
systemctl --user status roscore.service
```

You should see similar output to the following:

```shell
● roscore.service - My ROS Docker Container
     Loaded: loaded (/home/user/.config/systemd/user/roscore.service; enabled; preset: disabled)
    Drop-In: /usr/lib/systemd/user/service.d
             └─10-timeout-abort.conf
     Active: active (running) since Wed 2024-09-25 06:01:18 CEST; 1min 48s ago
   Main PID: 151153 (docker)
      Tasks: 11 (limit: 27986)
     Memory: 8.7M (peak: 10.2M)
        CPU: 41ms
     CGroup: /user.slice/user-1000.slice/user@1000.service/app.slice/roscore.service
             └─151153 /usr/bin/docker run --rm --net=host --name ros_container ros:noetic-ros-base roscore

Sep 25 06:01:18 fedora systemd[3596]: Started roscore.service - My ROS Docker Container.
```

<br/>

To verify that the ROS master node is running, you can start a new docker container and check the ROS master status:

```bash
docker run --rm --net=host --name rostest -it ros:noetic-ros-base rosnode list
```

You should see the following output:

```shell
/rosout
```

This output confirms that the ROS master node is running and ready to communicate with other ROS nodes.


## Stopping and disabling the systemd service

To stop and disable the systemd service, use the following commands:

```bash
systemctl --user stop roscore.service
systemctl --user disable roscore.service
```


## Starting Multiple ROS Nodes with systemd

In my paper "On the Applicability of Docker Containers and systemd Services for Search and Rescue Applications" [^footnote-Enrich23-1], I describe how to use systemd to manage multiple ROS nodes using Docker Compose. This approach allows you to define and manage complex robotic applications with multiple nodes, ensuring that they start, stop, and restart as needed.

My setup relies on a specific folder structure which is depicted below:

```shell
.
├── domains
│   ├── exploration
│   ├── mapping
│   ├── perception
│   ├── Readme.md
│   ├── sensing
│   └── systemd -> ../systemd
└── systemd
    ├── check_ros.sh
    ├── Readme.md
    └── robot@.service
```

Let's break down the content of the folders:

* **systemd**: Contains scripts and systemd unit files for managing ROS nodes with Docker Compose.
* **domains**: Contains folders for different robotic domains (e.g., sensing, mapping, perception). Each domain contains one or more Dockerfiles that are started via a docker-compose.yaml file.


### systemd Unit Files for Docker Compose

Here’s an example of a systemd unit file that controls a Docker Compose stack with multiple ROS nodes[^footnote-Code1]:

```bash
[Unit]
Description=%i service with docker compose
PartOf=docker.service
After=docker.service
After=network-online.target

[Service]
Type=oneshot
RemainAfterExit=true
WorkingDirectory=/home/user/git/robot/domains/%i
ExecStartPre=/bin/bash check_ros.sh %i
ExecStart=/usr/bin/docker-compose -p %i up -d --remove-orphans
ExecStop=/usr/bin/docker-compose down

[Install]
WantedBy=multi-user.target
```

Let's break down the key components of this unit file:

* `%i`: This is a placeholder for the service name, which will be replaced with the name of the Docker Compose stack.
* `PartOf=docker.service`: This line ensures that the service is stopped when the `docker.service` is stopped.
* `After=docker.service`: This line specifies that the service should start after the `docker.service`.
* `After=network-online.target`: This line ensures that the service starts after the network is online.
* `ExecStartPre=/bin/bash check_ros.sh %i`: This line runs a [script](https://github.com/TW-Robotics/search-and-rescue-robot-2024/blob/main/robot/systemd/check_ros.sh) to check if the ROS master node is running before starting the Docker Compose stack.
* `ExecStart=/usr/bin/docker-compose -p %i up -d --remove-orphans`: This line starts the Docker Compose stack with the specified project name.
* `ExecStop=/usr/bin/docker-compose down`: This line stops the Docker Compose stack when the service is stopped.
* `WantedBy=multi-user.target`: This line specifies that the service should be started at boot time.


### Docker Compose Files for Robotic Domains
Below is an example of a Docker Compose setup file for the mapping domain:

```yaml
services:
  slam_toolbox:
    image: robot/slam_toolbox
    build:
      context: ./
      dockerfile: Dockerfile.slam_toolbox
    command: roslaunch ./slam_toolbox_lifelong.launch --wait
    network_mode: host

  octomap:
    image: robot/octomap
    build:
      context: ./
      dockerfile: Dockerfile.octomap
    command: roslaunch ./octomap.launch --wait
    network_mode: host
```

This file defines two services (`slam_toolbox` and `octomap`) that run ROS nodes for [2D](https://github.com/SteveMacenski/slam_toolbox) and [3D](https://github.com/OctoMap/octomap_mapping) mapping and  generation. The `network_mode: host` line ensures that the containers share the host network, allowing them to communicate with each other and other ROS nodes running on the host machine.

### Enabling and Starting the systemd Service for the Mapping Domain

To enable and start the systemd service for the mapping domain, follow these steps:

```bash
cp ~/git/robot/systemd/robot@.service ~/.config/systemd/user/  # Provide the template Unit file
systemctl --user enable robot@mapping.service  # Enable the service
systemctl --user start robot@mapping.service  # Start the service
```
Here the `%i` placeholder in the `robot@.service` file is replaced with the domain name (`mapping` in this case) to create a specific service for the mapping domain. Upon starting the service, the Docker Compose stack for the mapping domain will be launched, running the `slam_toolbox` and `octomap` nodes.   

Note how the `--wait` flag is provided to the `roslaunch` command to ensure that the nodes are waiting for the ROS master to be ready before starting. Therefore, ensuring that none of the mapping nodes take over the task of serving as the ROS master.



## Conclusion
Using Docker and systemd to manage robotic applications provides a modular, scalable, and robust framework for deploying, monitoring, and maintaining robots. The combination of isolated Docker containers, reliable service management with systemd, and centralized monitoring with journalctl enables more efficient and resilient operations, particularly in high-risk environments like search and rescue missions.

<br/>
<br/>

[^footnote-Enrich23-1]: Novotny, G., Schwaiger, S., Muster, L., Aburaia, M., & Wöber, W. (2023). On the Applicability of Docker Containers and systemd Services for Search and Rescue Applications. In A. Müller, M. Nader, & H. Gattringer (Eds.), Proceedings of the Austrian Robotics Workshop (pp. 19–24). Johannes Kepler University. [https://www.joanneum.at/fileadmin/ROBOTICS/ARW_Proceedings/2023_ARW_Proceedings.pdf](https://www.joanneum.at/fileadmin/ROBOTICS/ARW_Proceedings/2023_ARW_Proceedings.pdf)

[^footnote-DockerInstall]: For more detailed instructions on installing Docker on Ubuntu, refer to the official Docker documentation: https://docs.docker.com/engine/install/

[^footnote-Code1]: [https://github.com/TW-Robotics/search-and-rescue-robot-2024/blob/main/robot/systemd/taurob%40.service](https://github.com/TW-Robotics/search-and-rescue-robot-2024/blob/main/robot/systemd/taurob%40.service)