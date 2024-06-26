<h1 align='center' style="text-align:center; font-weight:bold; font-size:2.0em;letter-spacing:2.0px;"> FSM: Correspondenceless scan-matching of panoramic 2D range scans </h1>

[![ieeexplore.ieee.org](https://img.shields.io/badge/IEEE/RSJ_IROS_2022_paper-00629B)](https://ieeexplore.ieee.org/abstract/document/9981228)
[![youtube.com](https://img.shields.io/badge/1'_presentation-YouTube-FF0000)](https://www.youtube.com/watch?v=hB4qsHCEXGI)
[![github.com](https://img.shields.io/badge/pdf_presentation-333333)](https://github.com/phd-li9i/fsm_presentation_iros22/blob/master/main.pdf)

`fsm_lidom_ros` provides LIDAR odometry from measurements of a single panoramic 2D LIDAR sensor, a.k.a. a sensor whose field of view is 360 degrees. `fsm_lidom_ros` is the ROS wrapper of [`fsm`](https://github.com/li9i/fsm).

<p align="center">
  <img src="https://i.imgur.com/hUsBImy.png">
</p>

Lidar odometry is achieved via scan-matching _but without establishing correspondences_ by leveraging the range signal's periodicity.
Hence FSM may exploit properties of the Discrete Fourier Transform.
These two pillars support the robustness of FSM's pose error to sensor noise and distance between consecutive poses, as you can see in the figure that summarises key experiments below.

## Why use FSM

![Experimental results at a glance](https://i.imgur.com/GvFlHgF.png)


Table of Contents
=================
* [Installation](#installation)
  * [Via Docker](#via-docker)
  * [Via traditional means](#via-traditional-means)
* [Run](#run)
  * [Launch](#launch)
  * [Call](#call)
* [Nodes](#nodes)
  * [`fsm_lidom_node`](#fsm_lidom_node)
    * [Subscribed topics](#subscribed-topics)
    * [Published topics](#published-topics)
    * [Services offered](#services-offered)
    * [Parameters](#parameters)
    * [Transforms published](#transforms-published)
* [Motivation and Under the hood](#motivation-and-under-the-hood)
  * [1 min summary video](#1-min-summary-video)
  * [IROS 2022 paper](#iros-2022-paper)



## Installation

### Via Docker

If this is your first time running docker then I happen to find [this](https://youtu.be/SAMPOK_lazw?t=67) docker installation guide very friendly and easy to follow. Then build the image with the most recent code of this repository using `compose` with

```sh
git clone git@github.com:li9i/fsm_lidom_ros.git
cd fsm_lidom_ros
docker compose build
```

or pull the docker image and run it with

```sh
docker pull li9i/fsm_lidom_ros:latest

docker run -it \
    --name=fsm_lidom_ros_container \
    --env="DISPLAY=$DISPLAY" \
    --net=host \
    --rm \
    li9i/fsm_lidom_ros:latest
```

### Via traditional means

Tested in Ubuntu 16.04 and ROS kinetic

#### Dependencies: `CGAL 4.7` `FFTW3` `boost/random`

```sh
cd ~/catkin_ws/src
git clone git@github.com:li9i/fsm_lidom_ros.git
cd fsm_lidom_ros; mv fsm/* $PWD; rmdir fsm; cd ../..
catkin build fsm_lidom_ros
```

## Run

### Launch

#### Via Docker

```sh
docker compose up
```

#### Via traditional means


```sh
roslaunch fsm_lidom_ros avanti_fsm_lidom.launch
```

### Call

Launching `fsm` simply makes it go into stand-by mode and does not actually execute anything. To do so simply call the provided service

#### Via Docker

```sh
docker exec -it fsm_lidom_ros_container sh -c "source ~/catkin_ws/devel/setup.bash; rosservice call /fsm_lidom/start"
```

#### Via traditional means

```sh
rosservice call /fsm_lidom/start
```


## Nodes

### `fsm_lidom_node`

#### Subscribed topics

| Topic                | Type                                     | Utility                                                                                |
| -------------------- | ---------------------------------------- | ---------------------------------------------------------------------------------------|
| `scan_topic`         | `sensor_msgs/LaserScan`                  | 2d panoramic scans are published here                                                  |
| `initial_pose_topic` | `geometry_msgs/PoseWithCovarianceStamped`| optional---for setting the very first pose estimate to something other than the origin |


#### Published topics

| Topic                 | Type                        | Utility                                                                       |
| --------------------- | ----------------------------| ------------------------------------------------                              |
| `pose_estimate_topic` | `geometry_msgs/PoseStamped` | the current pose estimate relative to the global frame is published here      |
| `path_estimate_topic` | `nav_msgs/Path`             | the total estimated trajectory relative to the global frame is published here |
| `lidom_topic`         | `nav_msgs/Odometry`         | the odometry is published here                                                |


#### Services offered

| Service                                | Type             | Utility                                                                                                                                          |
| -------------------------------------- | ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `fsm_lidom/clear_estimated_trajectory` | `std_srvs/Empty` | clears the vector of estimated poses                                                                                                             |
| `fsm_lidom/set_initial_pose`           | `std_srvs/Empty` | calling this service means: node subscribes to `initial_pose_topic`, obtains the latest pose estimate, sets fsm's initial pose, and unsubscribes |
| `fsm_lidom/start`                      | `std_srvs/Empty` | commences node functionality                                                                                                                     |
| `fsm_lidom/stop`                       | `std_srvs/Empty` | halts node functionality (node remains alive)                                                                                                    |


#### Parameters

Found in `config/params.yaml`:

| IO Topics                | Description                                                         |
| ------------------------ | ------------------------------------------------------------------- |
| `scan_topic`             | 2d panoramic scans are published here                               |
| `initial_pose_topic`     | (optional) the topic where an initial pose estimate may be provided |
| `pose_estimate_topic`    | `fsm_lidom_ros`'s pose estimates are published here                 |
| `path_estimate_topic`    | `fsm_lidom_ros`'s total trajectory estimate is published here       |
| `lidom_topic`            | `fsm_lidom_ros`'s odometry estimate is published here               |

| Frame ids         | Description                                                     |
| ----------------- | -------------------------------------                           |
| `global_frame_id` | the global frame id (e.g. `/map`)                               |
| `base_frame_id`   | the lidar sensor's reference frame id (e.g. `/base_laser_link`) |
| `lidom_frame_id`  | the (lidar) odometry's frame id                                 |

| FSM-specific parameters  | Description                                                                                                       |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------- |
| `size_scan`              | the size of scans that are matched (execution time is proportional to scan size, hence subsampling may be needed) |
| `min_magnification_size` | base angular oversampling                                                                                         |
| `max_magnification_size` | maximum angular oversampling                                                                                      |
| `num_iterations`         | Greater sensor velocity requires higher values                                                                    |
| `xy_bound`               | Axiswise radius for randomly generating a new initial position estimate in case of recovery                       |
| `t_bound`                | Angularwise radius for randomly generating a new initial orientation estimate in case of recovery                 |
| `max_counter`            | Lower values decrease execution time                                                                              |
| `max_recoveries`         | Ditto                                                                                                             |


#### Transforms published

```
lidom_frame_id <- base_frame_id
```

in other words `fsm_lidom_node` publishes the transform from `/base_laser_link`
(or equivalent) to the equivalent of `/odom` (in this case `lidom_frame_id`)


## Motivation and Under the hood

### 1 min summary video
[![IMAGE ALT TEXT](http://img.youtube.com/vi/hB4qsHCEXGI/0.jpg)](http://www.youtube.com/watch?v=hB4qsHCEXGI "1 min summary video")

<!-- ### IROS 2022 presentation slides -->
<!-- [PDF link](https://raw.githubusercontent.com/li9i/fsm_presentation_iros22/master/main.pdf) -->

### IROS 2022 paper

```bibtex
@INPROCEEDINGS{9981228,
  author={Filotheou, Alexandros and Sergiadis, Georgios D. and Dimitriou, Antonis G.}
  booktitle={2022 IEEE/RSJ International Conference on Intelligent Robots and Systems (IROS)},
  title={FSM: Correspondenceless scan-matching of panoramic 2D range scans},
  year={2022},
  pages={6968-6975},
  doi={10.1109/IROS47612.2022.9981228}}
