FTG Stands for Follow The Gap which works by receiving the LiDAR array for distances, and turning it into a steering angle and speed

The file has initialization variables for the different speed option for the code to choose from depending on the situation.
##  Initialization     
```
    DEBUG = rospy.get_param('/state_machine/debug')
    
    #Lidar processing params
    
    PREPROCESS_CONV_SIZE = 3
    SAFETY_RADIUS = rospy.get_param('/state_machine/safety_radius')
    MAX_LIDAR_DIST = rospy.get_param('/state_machine/max_lidar_dist')
    
    # Speed params

    MAX_SPEED = rospy.get_param('/state_machine/max_speed', 1.5)
    scale = 0.6  # .575 is  max
    CORNERS_SPEED = 0.3 * MAX_SPEED * scale
    MILD_CORNERS_SPEED = 0.45 * MAX_SPEED * scale
    STRAIGHTS_SPEED = 0.8 * MAX_SPEED * scale
    ULTRASTRAIGHTS_SPEED = MAX_SPEED * scale

    #Steering params

    STRAIGHTS_STEERING_ANGLE = np.pi / 18  # 10 degrees
    MILD_CURVE_ANGLE = np.pi / 6  # 30 degrees
    ULTRASTRAIGHTS_ANGLE = np.pi / 60  # 3 deg

  

    def __init__(self, mapping=False) -> None:

        Initialize the FTG controller.

        Parameters:
        self.mapping = mapping
        self.radians_per_elem = None # used when calculating the angles of the LiDAR data
        self.range_offset = rospy.get_param('/state_machine/range_offset')
        self.track_width = rospy.get_param('/state_machine/track_width', 2.0)
        self.velocity = 0
        self.scan = None    
```

### **1. Vehicle Motion Limits**

- Parameters such as **maximum speed** and **maximum steering angle** (if configured) are set during initialization. They act as global constraints that remain unchanged while the algorithm runs.
    

### **2. Runtime Variables (Not Initialized Here)**

Values such as:

- processed ranges,
    
- closest obstacle index,
    
- gap start/end indices,
    
- selected steering angle,
    
- commanded speed  
    are **not** set at initialization. They are computed dynamically every time a ==**new scan arrives**== in `lidar_callback()`.
### **3. Parameters:
 mapping (bool): Flag indicating whether FTG is used for mapping or not.

self.range_offset = rospy.get_param('/state_machine/range_offset')
	get the offset from state_machine if it doesn't found the server it will return zero 

self.track_width = rospy.get_param('/state_machine/track_width', 2.0)
	This retrieves **track_width** from the `/state_machine` parameter server.  
	If the parameter is not found, it returns **2.0**, which is the default value you choose.  
	This can help us diagnose errors — if the function always returns **2.0**, we can conclude that **track_width** is not set on the parameter server.
## `_preprocess_lidar`

```
Preprocess the LiDAR scan array.

  
 
        This method performs preprocessing on the LiDAR scan array. The preprocessing steps include:

        1. Setting each value to the mean over a specified window.

        2. Rejecting high values (e.g., values greater than 3m).


        Parameters:

            ranges (numpy.ndarray): The LiDAR scan array.  

        Returns:

            numpy.ndarray: The preprocessed LiDAR scan array.
```

This function sets the radians per element in the array of the LiDAR as to know which index corresponds to what angle. This value is stored in the variable `self.radians_per_elem`

Then, an array is set from the 360 degrees, as we are not going to use the full array, only the parts that's Infront of us, this range is set to `-135 to 135` degrees

The function returns an array of the LiDAR elements from -135 to 135 degrees in order, and clipped between 0 and the max lidar distance.

==return proc_ranges[::-1]==
This is essential because most LiDAR drivers publish right → left, and FTG operates left → right.
## `_get_steer_angle`

```
Get the angle of a particular element in the LiDAR data and

        transform it into an appropriate steering angle.

        Parameters:

            point_x (float): The x-coordinate of the LiDAR data point

            point_y (float): The y-coordinate of the LiDAR data point

        Returns:

            float: The transformed steering angle
```

as the name entails, this function is a simple math equation to return the angle of the best point chosen.

## `_get_best_range_point`

```
Find the best point i.e. the middle of the largest gap within the bubble radius.

        Parameters:

            proc_ranges (list): List of processed ranges.

        Returns:

            tuple: The x and y coordinates of the best point.
```

The function starts by getting the Largest gap indexes, and returning the `x` and `y` cartesian points to follow, which is the middle of the gap that's been identified and chosen to follow.

## `_find_largest_gap`

```
Find the index of the starting and ending of the largest gap and its width

        Parameters:

            ranges (numpy.ndarray): Array of range values
            radius (float): Threshold radius value
  
        Returns:

            tuple: A tuple containing the index of the starting of the largest gap, the index of the ending of the largest gap, and the width of the largest gap.
```

This function binarizes the array of the LiDAR elements, it goes through each element and checks if it passes the threshold for it to be considered a 'gap', this comparison is now saved in a new array called `bin_ranges`

### Gemini explanation:
This line creates a new NumPy array called `bin_ranges`. It performs a **boolean comparison** element-wise on the input array `ranges` against the scalar `radius`:

- If an element in `ranges` is **greater than or equal to** `radius` ($\ge$), the corresponding element in `bin_ranges` is set to **1** (or `True`).
    
- If an element in `ranges` is **less than** `radius` ($<$), the corresponding element in `bin_ranges` is set to **0** (or `False`).

The function also has an array for identifying the transition between `No Gap` to `Gap`, by using 
`diff( )` we have an array that's always 1 when there's a transition and a 0 when the current state continues, this is used to know the boundaries of the gap

Then it iterates through the array to know the biggest gap possible to follow, and returns the index of the `Left boundary` and the `Right boundary`. 

## `_safety_border`

```
Add a safety bubble if there is a big increase in the range between two points.
  
        Parameters:
            ranges (list): List of range values.

        Returns:
            np.ndarray: Array of filtered range values.
```

this function checks the distance between each array element and the next, to place the safety bubble at the edges of the objects to not place them in our FTG decisions, we only choose the gap that's not super close to edges to avoid our car hitting the walls or scraping against them, this is done by two `For` loops that goes once in the positive direction and once in the other direction to cover the entire area of the LiDAR scan.

## `_get_radius`

```
        Calculate the radius based on the track width and velocity.

        Returns:
            float: The calculated radius.
```

simply calculates the radius of the safety bubble according to the track width

## `process_lidar`

```
        Process each LiDAR scan as per the Follow Gap algorithm &
        calculate the speed and steering angle.

        Parameters:
            ranges (list): List of LiDAR scan ranges

        Returns:
            tuple: A tuple containing the speed and steering angle
```

This function utilizes all the functions into one main function.
