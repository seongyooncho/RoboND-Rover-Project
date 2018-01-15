## Project: Search and Sample Return

---


**The goals / steps of this project are the following:**  

**Training / Calibration**  

* Download the simulator and take data in "Training Mode"
* Test out the functions in the Jupyter Notebook provided
* Add functions to detect obstacles and samples of interest (golden rocks)
* Fill in the `process_image()` function with the appropriate image processing steps (perspective transform, color threshold etc.) to get from raw images to a map.  The `output_image` you create in this step should demonstrate that your mapping pipeline works.
* Use `moviepy` to process the images in your saved dataset with the `process_image()` function.  Include the video you produce as part of your submission.

**Autonomous Navigation / Mapping**

* Fill in the `perception_step()` function within the `perception.py` script with the appropriate image processing functions to create a map and update `Rover()` data (similar to what you did with `process_image()` in the notebook).
* Fill in the `decision_step()` function within the `decision.py` script with conditional statements that take into consideration the outputs of the `perception_step()` in deciding how to issue throttle, brake and steering commands.
* Iterate on your perception and decision function until your rover does a reasonable (need to define metric) job of navigating and mapping.  

[//]: # (Image References)

[image1]: ./misc/nb_perspect_transform.jpg
[image2]: ./misc/nb_output.png
[image3]: ./misc/nb_find_rocks.png
[image10]: ./calibration_images/example_rock1.jpg

## [Rubric](https://review.udacity.com/#!/rubrics/916/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it!

### Notebook Analysis
#### 1. Run the functions provided in the notebook on test images (first with the test data provided, next on data you have recorded). Add/modify functions to allow for color selection of obstacles and rock samples.

##### Perspective transform
I have modified `perspect_trasform()` to return mask along with warped image. Mask is needed in later steps to specify effective area.

Following **the project walkthrough**, I have calculated effective area by applying same perspective warp function to image filled with ones.

```python
# mask with angle
mask = cv2.warpPerspective(np.ones_like(img[:, :, 0]), M, (img.shape[1], img.shape[0]))
```

Because precision of warped image is reduced in proportion to distance, I've decided to clip it with radius of 120. It gives higher fidelity afterwards.

```python
# mask with radius
radius = 120
Y, X = np.ogrid[:img.shape[0], :img.shape[1]]
dist_from_center = np.sqrt((X - img.shape[1]/2)**2 + (Y-img.shape[0])**2)
mask = mask & (dist_from_center <= radius)
```

Final example of warped image and mask is as below.

![alt text][image1]

##### Coordinate transformation
I've only used navigable angles with shorter than 40 meters in distance to decide where to head. It gives better steering in case of avoiding obstacles.
```python
# Extract angles with shorter than 40 meters in distance
angles = np.extract(dist < 40, angles)
mean_dir = np.mean(angles)
```

##### Find rocks
I have added additional `find_rocks()` function following `the project walkthrough`.  It's used to find rocks from image.

```python
def find_rocks(img, levels=(110, 110, 50)):
    rockpix = ((img[:, :, 0] > levels[0]) \
              & (img[:, :, 1] > levels[1]) \
              & (img[:, :, 2] < levels[2]))
    color_select = np.zeros_like(img[:, :, 0])
    color_select[rockpix] = 1

    return color_select
```

You can see the result of `find_rocks()` from calibration image.

![][image3]


#### 1. Populate the `process_image()` function with the appropriate analysis steps to map pixels identifying navigable terrain, obstacles and rock samples into a worldmap.  Run `process_image()` on your test data using the `moviepy` functions provided to create video output of your result.

I've followed **the project walkthrough** here.

First, using current `img`, get warped image and mask for effective area.
```python
# Perspective Transform and mask
warped, mask = perspect_transform(img, source, destination)
```

Then apply color threshold to find navigable terrain and multiply it mask to exclude ineffective area.
```python
# Apply color threshold to identify navigable terrain/obstacles/rock samples
threshed = color_thresh(warped)
threshed = threshed * mask
```

By inverting this result, we can get map of obstacles.
```python
obs_map = np.absolute(np.float32(threshed) - 1) * mask
```

Then we calculate world coordinates of navigable terrain and obstacles.
```python
xpix, ypix = rover_coords(threshed)
# Convert rover-centric pixel values to world coordinates
world_size = data.worldmap.shape[0]
scale = 2 * dst_size
xpos = data.xpos[data.count]
ypos = data.ypos[data.count]
yaw = data.yaw[data.count]
x_world, y_world = pix_to_world(xpix, ypix, xpos, ypos,
                                yaw, world_size, scale)
obsxpix, obsypix = rover_coords(obs_map)
obs_x_world, obs_y_world = pix_to_world(obsxpix, obsypix, xpos, ypos,
                                        yaw, world_size, scale)
```

After all, we update worldmap. Navigable terrain is colored with blue and obstacles are colored with red. If some area is found navigable but previously thought obstacles, the it is overridden to navigable.
```python
# 6) Update worldmap (to be displayed on right side of screen)
data.worldmap[y_world, x_world, 2] = 255
data.worldmap[obs_y_world, obs_x_world, 0] = 255
nav_pix = data.worldmap[:, :, 2] > 0

data.worldmap[nav_pix, 0] = 0
```

If there's any rock found, it's colored to white.
```python
# See if we can find some rocks
rock_map = find_rocks(warped, levels=(110, 110, 50))
if rock_map.any():
    rock_x, rock_y = rover_coords(rock_map)
    rock_x_world, rock_y_world = pix_to_world(rock_x, rock_y, xpos, ypos,
                                             yaw, world_size, scale)
    data.worldmap[rock_y_world, rock_x_world, :] = 255
```

The final result of map generated with recorded data is shown below.
![alt text][image2]


### Autonomous Navigation and Mapping

#### 1. Fill in the `perception_step()` (at the bottom of the `perception.py` script) and `decision_step()` (in `decision.py`) functions in the autonomous mapping scripts and an explanation is provided in the writeup of how and why these functions were modified as they were.

#### `perception_step()`
First, I have defined `dst_size` and `bottom_offset` to calculate `source` and `destination`. Then, `perspect_transform` the `image` to get `warped` and `mask`
```python
dst_size = 5
bottom_offset = 6
image = Rover.img
source = np.float32([[14, 140], [301 ,140],[200, 96], [118, 96]])
destination = np.float32([[image.shape[1]/2 - dst_size, image.shape[0] - bottom_offset],
              [image.shape[1]/2 + dst_size, image.shape[0] - bottom_offset],
              [image.shape[1]/2 + dst_size, image.shape[0] - 2*dst_size - bottom_offset],
              [image.shape[1]/2 - dst_size, image.shape[0] - 2*dst_size - bottom_offset],
              ])
warped, mask = perspect_transform(image, source, destination)
```

Then I have multiplied mask to `color_thresh()` result to include pixels within 120 meters only. And by inverting `threshed`, I got map of obstacles.
```python
threshed = color_thresh(warped)
threshed = threshed * mask
obs_map = np.absolute(np.float32(threshed) - 1) * mask
```

I have updated `Rover.vision_image` with navigable terrains and obstacles.
```python
Rover.vision_image[:, :, 2] = threshed * 255
Rover.vision_image[:, :, 0] = obs_map * 255
```

I have converted image pixels of navigable terrain and obstacles to world coordinates.
```python
# 5) Convert map image pixel values to rover-centric coords
xpix, ypix = rover_coords(threshed)
# 6) Convert rover-centric pixel values to world coordinates
world_size = Rover.worldmap.shape[0]
scale = 2 * dst_size
x_world, y_world = pix_to_world(xpix, ypix, Rover.pos[0], Rover.pos[1],
    Rover.yaw, world_size, scale)
obsxpix, obsypix = rover_coords(obs_map)
obs_x_world, obs_y_world = pix_to_world(obsxpix, obsypix, Rover.pos[0], Rover.pos[1],
    Rover.yaw, world_size, scale)
```

And then, drew this result to worldmap.
```python
Rover.worldmap[y_world, x_world, 2] += 10
Rover.worldmap[obs_y_world, obs_x_world, 0] += 1
```

Before updating rover angles, I have extracted angles within 40 meters. This makes the rover focus on near field and keep it from running into small obstacles.
```python
dist, angles = to_polar_coords(xpix, ypix)
# Extract angles shorter than 40 meters in distance
angles = np.extract(dist < 40, angles)
# Update Rover pixel distances and angles
Rover.nav_angles = angles
```

If we find any rocks, then we mark it to `worldmap` and `vision_image`.
```python
rock_map = find_rocks(warped, levels=(110, 110, 50))
if rock_map.any():
    rock_x, rock_y = rover_coords(rock_map)

    rock_x_world, rock_y_world = pix_to_world(rock_x, rock_y, Rover.pos[0],
        Rover.pos[1], Rover.yaw, world_size, scale)
    rock_dist, rock_ang = to_polar_coords(rock_x, rock_y)
    rock_idx = np.argmin(rock_dist)
    rock_xcen = rock_x_world[rock_idx]
    rock_ycen = rock_y_world[rock_idx]

    Rover.worldmap[rock_ycen, rock_xcen, 1] = 255
    Rover.vision_image[:, :, 1] = rock_map * 255
else:
    Rover.vision_image[:, :, 1] = 0
```

### `decision_step()`

I haven't changed anything on `decision_step()`

#### 2. Launching in autonomous mode your rover can navigate and map autonomously.  Explain your results and how you might improve them in your writeup.  

**Note: running the simulator with different choices of resolution and graphics quality may produce different results, particularly on different machines!  Make a note of your simulator settings (resolution and graphics quality set on launch) and frames per second (FPS output to terminal by `drive_rover.py`) in your writeup when you submit the project so your reviewer can reproduce your results.**

With following **the walkthrough**, I could achieve around 60% of fidelity. Here are some issues I've tried to improve.

#### Increase fidelity
Fidelity of `color_thresh()`ed result is decreased in proportion to distance. So I have decided to set threshold at appropriate distance. After several simulations, 120 meters seemed fine. If the threshold is to low, the rover couldn't find any rock samples. I think fidelity can be increased if I could apply weights by distance, rather just applying threshold.  

#### Prevent from sticking into rocks
Most of the time, the rover could go well. But if there are small rocks just in front of it, the rover sometimes just go straight and get stuck. To prevent this from happening, the rover should focus on near view. So I've filtered out all the navigable angles farther than 40 meters.
