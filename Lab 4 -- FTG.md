/scan [sensor_msgs/msg/LaserScan]
/drive [ackermann_msgs/msg/AckermannDriveStamped]

Testing the `/drive` topic manually, we can send:
```
ros2 topic pub --once /drive ackermann_msgs/msg/AckermannDriveStamped "{
  'header': {'stamp': {'sec': 0, 'nanosec': 0}, 'frame_id': 'base_link'},
  'drive': {'steering_angle': 0.0, 'speed': 0.0}
}"
```
with the desired steering angle and speed.

Available Maps:
![[levine.png|400]]
![[levine_blocked.png|400]]
![[levine_obs.png|400]]
![[Spielberg_map.png|400]]
