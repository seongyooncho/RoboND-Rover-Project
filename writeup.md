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
[image3]: ./calibration_images/example_rock1.jpg

## [Rubric](https://review.udacity.com/#!/rubrics/916/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it!

### Notebook Analysis
#### 1. Run the functions provided in the notebook on test images (first with the test data provided, next on data you have recorded). Add/modify functions to allow for color selection of obstacles and rock samples.

##### Perspective transform
I have modified `perspect_trasform()` to return mask along with warped image following **the walkthrough**. Because precision of warped image is reduced in proportion to distance, I've decided to clip it with radius of 120. It gives higher fidelity afterwards.
```python
# mask with radius
radius = 120
Y, X = np.ogrid[:img.shape[0], :img.shape[1]]
dist_from_center = np.sqrt((X - img.shape[1]/2)**2 + (Y-img.shape[0])**2)
mask = mask & (dist_from_center <= radius)
```

![alt text][image1]

##### Coordinate transformation
I've only used navigable angles with shorter than 40 meters in distance to decide where to head. It gives better steering in case of avoiding obstacles.
```python
# Extract angles with shorter than 40 meters in distance
angles = np.extract(dist < 40, angles)
mean_dir = np.mean(angles)
```

For rest of the things, I've followed **the walkthrough** and it performed quite well.

#### 1. Populate the `process_image()` function with the appropriate analysis steps to map pixels identifying navigable terrain, obstacles and rock samples into a worldmap.  Run `process_image()` on your test data using the `moviepy` functions provided to create video output of your result.
I've followed **the walkthrough** and it performed quite well. The final result looks like this.

![alt text][image2]


### Autonomous Navigation and Mapping

#### 1. Fill in the `perception_step()` (at the bottom of the `perception.py` script) and `decision_step()` (in `decision.py`) functions in the autonomous mapping scripts and an explanation is provided in the writeup of how and why these functions were modified as they were.

#### `perception_step()`
I have multiplied mask to `color_thresh()` result to include pixels within 120 meters only.
```python
threshed = color_thresh(warped)
threshed = threshed * mask
```
Before updating rover angles, I have extracted angles within 40 meters.
```python
dist, angles = to_polar_coords(xpix, ypix)
# Extract angles shorter than 40 meters in distance
angles = np.extract(dist < 40, angles)
# Update Rover pixel distances and angles
Rover.nav_angles = angles
```
For the rest of the part, I followed steps on **the walkthrough**.

#### 2. Launching in autonomous mode your rover can navigate and map autonomously.  Explain your results and how you might improve them in your writeup.  

**Note: running the simulator with different choices of resolution and graphics quality may produce different results, particularly on different machines!  Make a note of your simulator settings (resolution and graphics quality set on launch) and frames per second (FPS output to terminal by `drive_rover.py`) in your writeup when you submit the project so your reviewer can reproduce your results.**

With following **the walkthrough**, I could achieve around 60% of fidelity. Here are some issues I've tried to improve.

#### Increase fidelity
Fidelity of `color_thresh()`ed result is decreased in proportion to distance. So I have decided to set threshold at appropriate distance. After several simulations, 120 meters seemed fine. If the threshold is to low, the rover couldn't find any rock samples. I think fidelity can be increased if I could apply weights by distance, rather just applying threshold.  

#### Prevent from sticking into rocks
Most of the time, the rover could go well. But if there are small rocks just in front of it, the rover sometimes just go straight and get stuck. To prevent this from happening, the rover should focus on near view. So I've filtered out all the navigable angles farther than 40 meters.
