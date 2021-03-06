## Project: 3D Motion Planning

### Writeup / README

### The Starter Code

#### 1. The functionality of what's provided in `motion_planning.py` and `planning_utils.py`

##### motion_planning.py

The file `motion_planning.py` provides a subclass of `Drone` with path planning encapsulated into a single method: `MotionPlanning.plan_path`. This method is invoked as part of a planning state, analogous to the landing state, arming state, waypoint state, etc., that we encountered in the backyard flyer project.

The path planning state is entered after the arming state and the manual state are completed, and before takeoff begins. When in the planning state, the drone is responsible for reading the map, identifying its start and end states relative to the map, identify an efficient path through the map, and then convert that path into a set of waypoints in its nevigation frame and store those waypoints for navigation during the waypoint state.

##### planning_utils.py

The file `planning_utils.py` contains files that facilitate the drone's planning phase. Specifically the file contains a `create_grid` function for parsing the provided map, a class of actions to represent the actions that may be performed to traverse the map, and an `a_star` algorithm and associated `heuristic` function for finding a path through the map.

### Implementing My Path Planning Algorithm

#### 1. Setting my global home position

Our home or starting position was included at the top of the `colliders.csv` file as a pair of latitude/longitude coordinates. I:

  1. Retrieved these coordinates from the file using basic file read operations and string manipulation.
  2. Set the home position to these coordinates using `self.set_home_position`.


```
  with open('colliders.csv') as f:
      lat0, lon0 = [float(coord.strip().split()[1]) for coord in f.readline().split(',')]
        
  self.set_home_position(lon0, lat0, 0)  # set the current location to be the home positionp
```

#### 2. Set your current local position

I identified my current local position using global to local, comparing my global_position to my global_home, producing a relative position in NED meters.

```
local_position = global_to_local(global_position, self.global_home)
```

#### 3. Set grid start position from local position

I was able to use my current local position to determine my current local position in the grid frame. Reading the map produced a set of North, East offsets that I could subtract from my local position to produce my position in the grid with the following code:

```
  grid_start = (int(local_position[0]) - north_offset, int(local_position[1]) - east_offset)

```

#### 4. Set grid goal position from geodetic coords

To produce a set of grid goal coordinates from geodetic coordinates, I:

  1. Choose/defined a set of geodetic coordinate goals.
  2. Transformed those into NED coordinates relative to my starting position using global_to_local
  3. Transformed those into grid coordinates by applying the `north_offset` and `east_offset`.

```
global_goal = (-122.399883, 37.793442, 0)

local_goal = global_to_local(global_goal, self.global_home)
grid_goal = (-north_offset + int(local_goal[0]), -east_offset + int(local_goal[1]))
```


#### 5. Modify A* to include diagonal motion (or replace A* altogether)

I modified A* to include diagonal motion by:

  1. Adding NORTHEAST, NORTWEST, SOUTHEAST, and SOUTHWEST actions, including the appropriate motion transformations and a cost of 2^(1/2).
  2. Adding rules to eliminate these new diagonal actions when in cases where they would move the drone into an obstacle or off the grid.

```
NORTHEAST = (-1, 1, 1.4)
SOUTHEAST = (1, 1, 1.4)
# ...

if y + 1 > m or grid[x, y + 1] == 1:
    valid_actions.remove(Action.EAST)
if (x + 1 > n and y + 1 > m) or grid[x + 1, y + 1] == 1:
    valid_actions.remove(Action.SOUTHEAST)
if (x + 1 > n and y - 1 < 0) or grid[x + 1, y - 1] == 1:
    valid_actions.remove(Action.SOUTHWEST)
# ...
```


#### 6. Cull waypoints 

I culled waypoints using the colinearity approach. Specifically, I retained the initial waypoint, the final waypoint, and tested each waypoint using the area method for colinearity with its neighbors: if the waypoint and its neighbors defined a triangle with area under a given threshold, then I threw pruned the waypoint out.

```
pruned_path = [path[0]]
for i in range(1, len(path) - 1):
    if not collinearity_float(path[i - 1], path[i], path[i + 1], epsilon=.6):
	pruned_path.append(path[i])
pruned_path.append(path[-1])
path = pruned_path
```



### Execute the flight


Aside from working through outright bugs, my first flights included some glitches: the drone would attempt to cut through the corners of buildings or would produce unnecessarily "zig-zaggy" paths.

To address these issues, I adjusted a couple of parameters:

1. To keep the drone from cutting through the corners of buildings, I adjusted the "safe distance" that a drone should try to keep from a building. Bumping this up from 5 to 7 solved my problems.
2. To reduce the "zig-zaggyness" of paths, I made path pruning more aggressive. Specifically, I adjusted the `epsilon` value that I used to determine whether three given points were colinear. By allowing an area of up to 0.7 in the triangle of colinear points, I was able to remove a satisfactory number of extra waypoints.

Following these adjustments, I was satisfied with the flight path of the drone.

