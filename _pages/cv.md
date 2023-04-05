---
layout: archive
title: "CV"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---

{% include base_path %}

Education
======
* B.S. in Mechatronics & Robotics, UAS Technikum Vienna, 2016
* M.S. in Mechatronics & Robotics with focus on Mobile Robotics, UAS Technikum Vienna, 2018
* Ph.D in System Identification & Parameter Estimation for Autonomous , GitHub University, 2024 (expected)

Work experience
======
* Sep 2018 - Current: Researcher / Lecturer - UAS Technikum Vienna
  * Collaboration on projects:
    * Autonomous Land Systems
      * Implementing/developing software modules for a Search and Rescue robot
      * Containerization of software modules
      * System maintenance
    * Engineering Goes International (ENGINE)
      * Software development
    * IoCESST (Internationalization of the Curricula in Engineering, Environmental, Smart Cities and Sport Technologies)
      * Development of the curriculum of the Mobile Robotics Lecture
      * Lectures in Mobile Robotics

* Oct 2022 - Mai 2023: Software Engineer - Ebe Mobility & Green Energy GmbH
  * Implementation Self-Hosted Gitlab Instance
    * Gitlab CI/CD based on self-hosted Docker container
    * Gitlab - Openproject implementation
  * Server monitoring and management
  * Development of applications and solutions in the field of e-mobility, charging controllers and charging station
  control

* Dec 2019 - Sep 2022: University Assistant - Johannes Kepler University
  * Lead on Last Mile Delivery Robot project
    * 2D and 3D SLAM
    * ROI Detection
    * Path Planner
  * Collaboration on Intelligent Car project
    * Communication Layer car CAN-PC implementation
    * Sensor mount design for various exteroceptive/proprioceptive sensors
  * Course Exercises
    * Python Programming for Economic and Business Analytics
    * Introduction to software development with Python
  
Skills
======
* Linux
* Programming (<img src="https://raw.githubusercontent.com/yurijserrano/Github-Profile-Readme-Logos/042e36c55d4d757621dedc4f03108213fbb57ec4/programming%20languages/bash.svg" height="25" alt="Bash"> <img src="https://raw.githubusercontent.com/yurijserrano/Github-Profile-Readme-Logos/042e36c55d4d757621dedc4f03108213fbb57ec4/programming%20languages/c.svg" height="25" alt="C"> <img src="https://raw.githubusercontent.com/yurijserrano/Github-Profile-Readme-Logos/042e36c55d4d757621dedc4f03108213fbb57ec4/programming%20languages/c%2B%2B.svg" height="25" alt="C++"> <img src="https://raw.githubusercontent.com/yurijserrano/Github-Profile-Readme-Logos/042e36c55d4d757621dedc4f03108213fbb57ec4/programming%20languages/python.svg" height="25" alt="Python">)
* CI/CD
  * Github Actions
  * Gitlab CI
  * Travis CI
  * Jemkins

Publications
======
  <ul>{% for post in site.publications reversed %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>
  
Talks
======
  <ul>{% for post in site.talks reversed %}
    {% include archive-single-talk-cv.html  %}
  {% endfor %}</ul>
  
Teaching
======
  <ul>{% for post in site.teaching reversed %}
    {% include archive-single-cv.html %}
  {% endfor %}</ul>
