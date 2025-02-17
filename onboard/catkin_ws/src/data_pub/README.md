# Data Pub

The `data_pub` package contains scripts that interface with various sensors, currently interfacing with the DVL (Doppler Velocity Log), pressure sensor (depth), voltage sensor, and temperature/humidity. You can launch all nodes in this package using the `pub_all.launch` file.

Additionally, the package allows control over the marker dropper servo. The servo is controlled by publishing a Bool message to the `servo_control` service.

## Config File
This package contains a config file for each robot, stored at `config/<ROBOT_NAME>.yaml`, where `<ROBOT_NAME>` is the value of the `ROBOT_NAME` enviornment variable corresponding to each robot. Config files for all robots must be structured as below:
```yaml
dvl:
  ftdi: DVL_FTDI_string
  negate_x_vel: true or false
  negate_y_vel: true or false
  negate_z_vel: true or false
```

There must be only one top-level key, `dvl`. Under it, there must be the following keys:
- `ftdi`: The FTDI string that uniquely identifies the DVL for that robot and can be used to find the port the DVL is connected to.
- `negate_x_vel`: Boolean indicating whether the DVL's X velocity should be negated by the `dvl_to_odom` script before being published.
- `negate_y_vel`: Boolean indicating whether the DVL's Y velocity should be negated by the `dvl_to_odom` script before being published.
- `negate_z_vel`: Boolean indicating whether the DVL's Z velocity should be negated by the `dvl_to_odom` script before being published.

## DVL

We use the [Teledyne Pathfinder DVL](https://www.teledynemarine.com/brands/rdi/pathfinder-dvl) for velocity measurements. The DVL is connected to the robot's main computer via a USB serial converter.

The `dvl_raw` script publishes the raw DVL data to the `/sensors/dvl/raw` topic with type `custom_msgs/DVLRaw`. It obtains the DVL's FTDI string from the config file mentioned above and uses it to find the DVL's port.

The `dvl_to_odom` script converts the raw DVL data and publishes it to `/sensors/dvl/odom` with type `nav_msgs/Odometry` for use in `sensor_fusion`. It obtains the DVL's negation values from the config file mentioned above and uses them to negate the DVL's X, Y, and Z velocities before publishing the odometry message.

You can launch both scripts using the `pub_dvl.launch` file.

## External Sensors

The pressure Arduino obtains data from the [Blue Robotics Bar02 Pressure Sensor](https://bluerobotics.com/store/sensors-cameras/sensors/bar02-sensor-r1-rp/) and a generic voltage sensor and sends the data as raw serial messages to the robot's main computer.

To connect to the pressure Arduino, the `pressure_voltage.py` script obtains its FTDI string from the config file for the current robot (as indicated by the `ROBOT_NAME` environment variable) in the `offboard_comms` package. The FTDI string is used to find the pressure Arduino's port.

The data obtained from serial contains tags `P:` and `V:` identifying pressure (depth) and voltage, respectively.

The `pressure_voltage.py` script can be launched using the `pub_pressure_voltage.launch` file.

### Pressure/Depth
The depth data obtained is filtered, converted into a `geometry_msgs/PoseWithCovarianceStamped` message, and published to the `/sensors/depth` topic, for use in `sensor_fusion`.

Two filters are applied to the depth:
1. Values with absolute value greater than 7 are ignored.
2. A median filter is applied to the 3 most recent values.

These filters are applied to eliminate noise in the data that would otherwise result in an inaccurate Z position in state.

All data in this `PoseWithCovarianceStamped` message is set to 0 except for the `pose.pose.position.z` value, which is set to the depth in meters. The `pose.pose.orientation` is set to the identity quaternion. Except for the `pose.pose.position.z` value, all other values are unused in `sensor_fusion`.

### Voltage
The same node also gets voltage data from the voltage sensor on the same arduino and is published as a Float64 to `/sensors/voltage`, without modification.

### Temperature/Humidity
On a different Arduino, both temperature and humidity readings are gathered by a [DHT11 sensor](https://www.adafruit.com/product/386), dumped over serial, and published to the `/sensors/temperature` and `/sensors/humidity` topics, respectively. The data is published as a Float64.

The data obtained from serial contains tags `T:` and `H:` identifying temperature and humidity, respectively.

Temperature is measured in degrees Fahrenheit and humidity is measured in percentage.

The `servo_sensors.py` script can be launched using the `pub_servo_sensors.launch` file. This Arduino also houses the servo for the marker dropper, and is designed to be a general-purpose sensor Arduino for future use.

### Servo
The `servo_control` service allows control over the marker dropper servo. The marker dropper holds two rounds that can be dropped individually. The service takes a `Bool` message with the following values:
- `True`: Drop the "left" round.
- `False`: Drop the "right" round.
