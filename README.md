# My first *MOOS* project
[![Build Status](https://travis-ci.org/moos-tutorials/00-my-first-moos-project.svg?branch=master)](https://travis-ci.org/moos-tutorials/00-my-first-moos-project)

In this tutorial, you will learn how to install and use *MOOS-IvP* on your machine.

## Introduction
[*MOOS*](http://themoos.org) is an open-source [middleware](http://en.wikipedia.org/wiki/Middleware) for inter-process communication ([IPC](http://en.wikipedia.org/wiki/Inter-process_communication)) based on the [publish-subscribe](http://en.wikipedia.org/wiki/Publish–subscribe_pattern) pattern.
Such middleware are important for complex systems, including robotic systems, where processes (or applications) need to share the data to make decisions better understand the environment with ease.
Having independent applications also allow the reuse of such applications in different scenarios without the modification of the source code.

## System requirement
### Disclaimer
[*MOOS*](http://themoos.org) is [cross-platform](http://en.wikipedia.org/wiki/Cross-platform), however, in this series, we will use a set of extra-tools [*MOOS-IvP*](http://moos-ivp.org) that compile only using [GCC](https://gcc.gnu.org) or [CLang](clang.llvm.org) for now.
Therefore, *Windows* users are required to install one of these compilers or use *Linux* or *macOS*.

In these series also, it is supposed that a *Linux Debian* system will be used. The command line instructions provided are for *Ubuntu 16.04* but will work on other system through some small adaptations too.

### Tools
- `git` and `subversion`: [version control systems](en.wikipedia.org/wiki/Version_control) to manage the code that will be written.
  - `sudo apt install git subversion`
- C/C++ compiler and other essential tools: compile the source code to executables.
  - `sudo apt install build-essential`
- CMake :cross-platform, open-source make system
  - `sudo apt install cmake`
- FLTK libraries for the GUI
  - `sudo apt install libfltk1.3-dev`
- TIFF libraries to visualize the maps:
  - `sudo apt install libtiff5-dev`

## Install *MOOS-IvP*
It is recommended to get the latest version of *MOOS-IvP* from the [download page](http://oceanai.mit.edu/moos-ivp/pmwiki/pmwiki.php?n=Site.Download) :
```shell
svn co https://oceanai.mit.edu/svn/moos-ivp-aro/trunk/ moos-ivp
```

Once the source code has been downloaded, go to the new `moos-ivp` directory and launch the build script:
```shell
cd moos-ivp
./build.sh
sudo ./build-moos.sh install
sudo ./build-ivp.sh install
```

## Test the installation
To test the success of the installation, the output of `which MOOSDB` should be similar to:
```shell
$ which MOOSDB
/usr/local/bin/MOOSDB
$ which pMarineViewer
/usr/local/bin/pMarineViewer
```

## Running an example mission `s1_alpha`
*MOOS-IvP* comes with many example missions to get you started and help understand the logic behind it.
These missions are located in `ivp/missions/`.

Let's start with the `s1_alpha` mission:
```shell
$ cd ivp/missions/s1_alpha
$ ls
alpha.bhv  alpha.moos  clean.sh  launch.sh
```
### The scripts `launch.sh` & `clean.sh`
The `bash` scripts `launch.sh` and `clean.sh` are optional to run a mission but very useful for advanced use as we will see in the future.

In this example, `launch.sh` is used to configure and launch the mission.
The reader is invited to open and read the script to better understand what it does.

The `clean.sh` script is used here to clean the generated files and logs.

### The `alpha.moos` file
Mission configuration files have the `.moos` extension.
Configuration for *MOOS* applications go here, but also more general configuration like the **host** IP and what port to use.

#### General configuration
For *MOOS* to know what IP and port to use, but also what to call the community it is running, the following lines are required
```c++
ServerHost = localhost // Can be any valid IP address
ServerPort = 9000      // The port number to use
Community  = alpha     // The name of the community
```

On the same machine, multiple communities can coexist as long as ports are different.


When simulating missions, it is useful to be able to run the simulations at higher speed than 1x.
The following line is useful for that purpose:
```c++
MOOSTimeWarp = 1
```

For the geodesic projections to work correctly, we set up the latitude and longitude coordinates of the origin of the map:
```c++
LatOrigin  = 43.825300
LongOrigin = -70.330400
```

#### `pAntler` configuration
`pAntler` is a *MOOS* app that an launch applications, *MOOS* or not, in an ordered sequence.
An application configuration block for `pAntler` looks like following:
```c++
//------------------------------------------
// Antler configuration  block
ProcessConfig = ANTLER
{
  MSBetweenLaunches = 200

  Run = MOOSDB          @ NewConsole = false, ExtraProcessParams=db
  Run = pLogger         @ NewConsole = false
  Run = uSimMarine      @ NewConsole = false
  Run = pMarinePID      @ NewConsole = false
  Run = pHelmIvP        @ NewConsole = false
  Run = pMarineViewer   @ NewConsole = false, ExtraProcessParams=one
  Run = uProcessWatch   @ NewConsole = false
  Run = pNodeReporter   @ NewConsole = false

  one = --size=1400x800
  db = --event_log=eventlog
}
```

What `pAntler` will do is launch each application listed in the same order with a delay of `MSBetweenLaunches` in milliseconds between launches with the option listed as `ExtraProcessParams`.
For instance, let's take the first two launches:
```shell
MOOSDB --event_log=eventlog alpha.moos
sleep 0.200
pLogger alpha.moos
```

#### *MOOS* app configuration
All *MOOS* applications can be configured using the `ProcessConfig` block.
Here's an example with `pHelmIvP`, the heart of *MOOS-IvP*:
```c++
ProcessConfig = pHelmIvP
{
  AppTick    = 4
  CommsTick  = 4

  behaviors  = alpha.bhv
  domain     = course:0:359:360
  domain     = speed:0:4:41
}
```

- The `AppTick` variable sets the beat on which the iteration loop of the application is called, here at least 4 time per second.
- The `CommsTick` variable sets the communication frequency with the `MOOSDB` to send and receive messages.
- The rest of the configuration is specific to each application. `pHelmIvP` requires extra configuration options to operate:
  - `domain` defines the domain in which the solver will look for a decision:
    - `course:0:359:360` : here the solver will generate 360 points between 0 and 359
    - `speed:0:4:41` : here the solver will generate 41 points between 0 and 4
  - `behaviors` sets the behavior file that will considered.

### The `alpha.bhv` file
The behavior file is specific to *MOOS-IvP* and is not required by *MOOS*.
This file defines the behaviors that will generate the decision functions that will then generate an output in a form of `DESIRED_SPEED` and `DESIRED_HEADING` for example.

The file usually has some local variable initialization:
```c++
initialize   DEPLOY = false
initialize   RETURN = false
```
These lines set two variables `DEPLOY` and `RETURN` and initialize then to `false`. (in *MOOS*, variables are either binary data, floating point or strings, there's no boolean type.)

A behavior configuration block looks like the following:
```c++
Behavior=BHV_Waypoint
{
  name       = waypt_return
  pwt        = 100
  condition  = RETURN = true
  condition  = DEPLOY = true
  perpetual  = true
  updates    = RETURN_UPDATE
  endflag    = RETURN = false
  endflag    = DEPLOY = false
  endflag    = MISSION = complete

           speed = 2.0
  capture_radius = 2.0
     slip_radius = 8.0
          points = 0,-20
 }
```

### Launch the mission
To launch the mission, simply call the launch script:
```shell
./launch.sh
```

A `pMarineViewer` window will show an aerial view and a vehicle.
The vehicle can be deployed using the `DEPLOY` button in the lower right corner.

## Conclusion
In this tutorial, you learned about how to install and run a very basic mission.
In the upcoming tutorials you will be writing your very first *MOOS* application.

## Read more
- [MOOS-IvP documentation](http://oceanai.mit.edu/ivpman/pmwiki/pmwiki.php)
- [Unmanned Marine Vehicle Autonomy, Sensing and Communications (MIT 2.680 class)](http://oceanai.mit.edu/2.680/pmwiki/pmwiki.php)

## License
This tutorial is released under an [MIT license](LICENSE.md).
