# imc_ros_bridge
![CI](https://github.com/smarc-project/imc_ros_bridge/workflows/CI/badge.svg?branch=noetic-devel) [![license](https://img.shields.io/badge/License-BSD%203--Clause-blue.svg)](https://opensource.org/licenses/BSD-3-Clause)

Minimal library for bridging ROS and IMC messages

## Building

A simple `catkin_make` should do.

## Launching

The command
```
roslaunch imc_ros_bridge bridge.launch server_addr:=127.0.0.1 server_port:=6002
```
will launch the `imc_bridge` node.

## Trying with Neptus

These instructions have been tested for Ubuntu 16.04 and 18.04. For 18.04, you
may have to comment a line in a jdk file to get it running (TODO: check path to file).

First, install dependencies: `sudo apt install openjdk-8-jdk ant-optional`.

Then clone and build neptus:
```
git clone https://github.com/LSTS/neptus.git
cd neptus
ant
```
If successful, run it with `./neptus.sh`. Open the comm monitor by clicking `Communications > IMC Comm. Monitor`.
In the status panel, click the button with the IMC logo (with hover text `Start IMC comms`) to start up IMC.
Go to the all messages panel to see the messages that you publish from ROS.

You can publish e.g. the `/heartbeat` message in ROS and see that it appears in the all messages panel (see "existing conversions" below):
```
rostopic pub /heartbeat std_msgs/Empty "{}" --once
```

If you go to the systems list panel, you should also see the ROS auv displayed as a cyan panel.
This indicates that neptus can communicate with the auv. Any other color indicates som problem,
see [this link](https://www.lsts.pt/neptus/manual/trunk/elements.html#systems-list).

### Troubleshooting
On Ubuntu 18.04, you might have openjdk-11-jre-headless package, remove it manually and make sure that `apt list --installed | grep jre` only shows one jre.
Additionally, if you get a "No VTK Java Packages Found" message accompanied by an error, do this: 

`sudo vim /etc/java-8-openjdk/accessibility.properties`
Comment out the following line:
`assistive_technologies=org.GNOME.Accessibility.AtkWrapper`

If you get an error that involves iced-tea in the Neptus terminal window when you try to open a console, check that you have openjdk-8-jre. Not the headless version of it, Neptus needs the head.

## Existing conversions

### ros_to_imc

* `std_msgs::Empty` on topic `/heartbeat` -> `IMC::Heartbeat`
* `sensor_msgs::NavSatFix` on topic `/gps_fix` -> `IMC::GpsFix`
* `geometry_msgs::Pose` on topic `/goto_input` -> `IMC::Goto`
* `sensor_msgs::NavSatFix` on topic `/gps_nav_data` -> `IMC::GpsNavData`
* `sensor_msgs::NavSatFix` on topic `/estimated_state` -> `IMC::EstimatedState`
* `imc_ros_bridge::PlanControlState` on topic `plan_control_state` -> `IMC::PlanControlState`

### imc_to_ros

* `IMC::Goto` -> `geometry_msgs::Pose` on topic `/goto_waypoint`
* `IMC::Abort` -> `std_msgs::Empty` on topic `/abort`
* `IMC::Heartbeat` -> `std_msgs::Empty` on topic `/imc_heartbeat`
* `IMC::PlanDB` -> `std_msgs::String` on topic `/plan_db` (JSON)
* `IMC::Plancontrol` -> `imc_ros_bridge::PlanControl` on topic `/plan_control`

## Creating new conversions

For each conversion in either direction, you need to create a new library
that contains a specialized `convert` function for the types you want to convert.
You then need to add a `BridgeServer` to the node, see details below.

### ros_to_imc

Check out examples in the `include/imc_ros_bridge/ros_to_imc` and `src/ros_to_imc` folders
for more examples. All conversions specialize the `convert` function like this:

```cpp
namespace ros_to_imc {
template <>
bool convert(const sensor_msgs::NavSatFix& ros_msg, IMC::GpsFix& imc_msg)
{
    imc_msg.lat = ros_msg.latitude;
    imc_msg.lon = ros_msg.longitude;
    imc_msg.height = ros_msg.altitude;

    return true;
}
}
```

And they also add the bridge server to the `bridge_node` like this:
```cpp
ros_to_imc::BridgeServer<sensor_msgs::NavSatFix, IMC::GpsFix> gpsfix_server(ros_node, imc_handle, "/gps_fix");
```
And link the convert libary into `ros_to_imc_node` in the `CMakeLists.txt` file.

### imc_to_ros

Check out examples in the `include/imc_ros_bridge/imc_to_ros` and `src/imc_to_ros` folders
for more examples. All conversions specialize the `convert` function like this:

```cpp
namespace imc_to_ros {
template <>
bool convert(const IMC::Goto& imc_msg, geometry_msgs::Pose& ros_msg)
{
    ros_msg.position.x = imc_msg.lon;
    ros_msg.position.y = imc_msg.lat;
    ros_msg.position.z = imc_msg.z;
    return true;
}
}
```

And they also add the bridge server to the `bridge_node` like this:
```cpp
imc_to_ros::BridgeServer<IMC::Goto, geometry_msgs::Pose> goto_server(imc_handle, ros_node, "/goto_waypoint");
```
And link the convert libary into `imc_to_ros_node` in the `CMakeLists.txt` file.

## Neptus-AUV Integration
For some vehicle named `$VEHICLE`.
In this repo, we have three vehicles available: `sam-auv`, `lolo-auv` and `imc_ros_bridge`.


### IMPORTANT NOTE

The bridge should be running on the same physical device as Neptus. This is a current limitation due to IP address shenanigans that is planned to be fixed eventually. Both for physical and simulated vehicles, Neptus and bridge must run on the same device.


### Adding a vehicle to Neptus

An arbitrary integer `$NUMBER` is used by Neptus as a convention to group together similar vehicles. 

Copy `neptus_vehicle_definitions/$VEHICLE_files/$NUMBER-$VEHICLE.nvcl` into `.../neptus/vehicle-defs/` and `neptus_vehicle_definitions/$VEHICLE_files/$VEHICLE` folder into `.../neptus/vehicle_files/`.
This will add `$VEHICLE` to the list of vehicles available in Neptus' list, with `$VEHICLE`'s visuals. 

Do the same thing with `imc_ros_bridge` to get a generic vehicle. The `imc_ros_bridge` fake vehicle has id 4.

When starting the `imc_ros_brdige` node, be careful to set the `imc_id` parameter to the correct one, otherwise the bridge will not receive data from the respective console window. Do not forget to select the correct vehicle in the console window (top right, drop down menu) either. 

The `.nvcl` file contains the id of the vehicle you are adding. You can find the `imc-id` field under `<communication-means> -> <comm-mean> -> <protocols-args> -> <imc>`. This field should be changed to whatever imc id you might want to use. It also must match the `imc_id` parameter used by the bridge node.

For advanced uses of the `.nvcl` file, consult Neptus documentation.

### Moving a vehicle around

Using the console (opened by going to Vehicles -> `$VEHICLE` -> click console button) generate a plan (In the console window: Tools -> Generate plan or use the plan tool on the left). Select the plan on the right section of the console and click the blue arrow (upload button) towards the top. This will create and send out a JSON formatted plan to the ros topic `/plan_db`. Parse this JSON on the vehicle, and control `$VEHICLE`.

In order to see the updated pose of $VEHICLE in the Neptus console, publish to the ros topic `/estimated_state`. Currently only lat, lon, altitude are used. The update rate on the Neptus console can be about once every 1-2 seconds, be patient.

See [this tutorial](https://github.com/smarc-project/smarc_scenarios/tree/master/bts_tutorial) for a demo that uses this bridge as part of a full planning -> execution pipeline in simulation. 

### Emergency

From the Neptus console, the big red ABORT button can be used to send an empty message to the ros topic `/abort`. Probably a good idea to subscribe to this topic. Disregard the errors that pop out in the Neptus console due to missing acoustic transponders and such.






