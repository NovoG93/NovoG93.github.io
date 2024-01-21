---
title: "Autonomous Vehicles: Vehicle Parameter Estimation Using Variational Bayes and Kinematics"
collection: publications
permalink: /publication/2020-09-10-Autonomous-Vehicles-Vehicle-Parameter-Estimation-Using-Variational-Bayes-and-Kinematics
date: 2020-09-10
venue: 'Applied Sciences'
citation: 'Wöber, W., Novotny, G., Mehnen, L., & Olaverri-Monreal, C. (2020). Autonomous vehicles: Vehicle parameter estimation using variational bayes and kinematics. Applied Sciences (Switzerland), 10(18). https://doi.org/10.3390/APP10186317'
---

__Abstract:__ On-board sensory systems in autonomous vehicles make it possible to acquire information about the vehicle itself and about its relevant surroundings. With this information the vehicle actuators are able to follow the corresponding control commands and behave accordingly. Localization is thus a critical feature in autonomous driving to define trajectories to follow and enable maneuvers. Localization approaches using sensor data are mainly based on Bayes filters. Whitebox models that are used to this end use kinematics and vehicle parameters, such as wheel radii, to interfere the vehicle's movement. As a consequence, faulty vehicle parameters lead to poor localization results. On the other hand, blackbox models use motion data to model vehicle behavior without relying on vehicle parameters. Due to their high non-linearity, blackbox approaches outperform whitebox models but faulty behaviour such as overfitting is hardly identifiable without intensive experiments. In this paper, we extend blackbox models using kinematics, by inferring vehicle parameters and then transforming blackbox models into whitebox models. The probabilistic perspective of vehicle movement is extended using random variables representing vehicle parameters. We validated our approach, acquiring and analyzing simulated noisy movement data from mobile robots and vehicles. Results show that it is possible to estimate vehicle parameters with few kinematic assumptions.

[Access paper here](https://www.mdpi.com/2076-3417/10/18/6317){:target="_blank"} or get the pdf [here](files/paper/Autonomous_Vehicles_Vehicle_Parameter_Estimation_Using_Variational_Bayes_and_Kinematics.pdf){:target="_blank"}

__Bibtex:__ [download](files/bib/Woeber2020a.bib)

```bibtex
@article{,
  author    = {Wilfried W{\"o}ber and Georg Novotny and Lars Mehnen and Cristina Olaverri-Monreal},
  doi       = {10.3390/APP10186317},
  issn      = {20763417},
  issue     = {18},
  journal   = {Applied Sciences (Switzerland)},
  keywords  = {Probabilistic robotics,Variational bayes,Vehicle parameter estimation},
  month     = {9},
  publisher = {MDPI AG},
  title     = {Autonomous vehicles: Vehicle parameter estimation using variational bayes and kinematics},
  volume    = {10},
  year      = {2020}
}
```