# **Advanced Lane Finding**
## Writeup for Project 4

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./examples/cal_undistort.PNG "Calibration Image Distorted/Undistorted"
[image2]: ./examples/lane_undistort.PNG "Lane Distorted/Undistorted"
[image3]: ./examples/binary_threshold.PNG "Binary Example"
[image4]: ./examples/birdseye.PNG "Warp Example"
[image5]: ./examples/line_fits.PNG "Fit Visual"
[image6]: ./examples/final_output.PNG "Output"
[video1]: ./project_video_output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

This is the writeup file for the advanced lane finding project and is modified from the writeup template that was provided. The code is contained in a Jupyter Notebook and is largely copied from the examples from the lessons and then modified for use in this project.

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first few code cells of the IPython notebook located in `lane_finding.ipynb` under the heading of "Camera Calibration". This code is almost entirely duplicated from the camera calibration examples from the lesson.

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I used OpenCV to automatically find chessboard corners for each of the images in the camera_cal folder. I defined the number of chessboard corners to be 9x6, which meant that for a portion of the images that had corners outside the frame, the points would not be found and those images would not contribute to the calibration.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to one of the calibration images using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images. I took the saved camera calibration matrix and distortion coefficients from the calibration step and applied them to the test images using the `cv2.undistort()` function as before.

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image (see the "Create binary lane image pipeline" section of `lane_finding.ipynb`).  Here's an example of my output for this step. 

![alt text][image3]

The thresholding is an `OR` function of three types of threshold pairs. The first pair is of Sobel X and Y functions where the X should be high and the Y should be low in order to favor vertical lines. The second pair is similar in that it uses the Sobel X and Y functions to determine the gradient magnitude and direction to again favor vertical lines. Both of these Sobel threshold pairs used a default kernel size of only 3 pixels. It may be useful for me to increase that kernal size to improve my performance on the challenge videos in the future.

The final threshold pair is a color threshold for two different color transforms. It requires that the HLS saturation is high while the HSV value is also high in order to favor bright, saturated lines. This color threshold is particularly important for identifying lane pixels on far away, curved, and shadowed sections of road.

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `birdseye()`, which appears in the "Perspective Transform" section of `lane_finding.ipynb`.  The `birdseye()` function takes as inputs an image (`image`) and an offset (`offset`) that is used to determine the size of the transformed image margins. Source (`src`) are hard-coded into this function based on my measurements from a straight lane test image and destination (`dst`) points are calculated based on the image size and selected `offset`.  I chose the hard-coded source and destination points in the following manner:

```python
src = np.float32(
      [[590,450],
      [686,450],
      [1050,675],
      [255,675]])
dst = np.float32(
      [[offset,0],
      [img.shape[1]-offset,0],
      [img.shape[1]-offset,img.shape[0]],
      [offset,img.shape[0]]])
```

This resulted in the following source and destination points for my default offset of 300 pixels:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 590, 450      | 300, 0        | 
| 686, 450      | 940, 0        |
| 1050, 675     | 940, 720      |
| 255, 675      | 300, 730      |

I then used the `cv2.getPerspectiveTransform` and `cv2.warpPerspective` functions to warp the image based on these source and destination points. I also calculated and returned the inverse transformation matrix for use in plotting the lane lines later in the project.

I verified that my perspective transform was working as expected by applying it to a series of binary test images and confirmed that the lane could always be seen in the birdseye view.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

To determine the lane-line pixels, I used both the sliding window and polynomial window techniques from the lesson. The code for both of these was largely taken directly from the lesson. The code for this step can be found in the "Find Lane Lines" section of `lane_finding.ipynb`.

If a best fit for the lane line polynomial is not yet known (or if an insufficient number of points were found with the other method), then I would use the sliding window search (`slidingWindowLines()`). This method takes a histogram of the bottom half of a binary image and finds the column with the maximum number of filled pixels on both the left and right sides of the image. These two columns are considered the center of each lane line. All filled pixels within `margin` pixels (in this case 100 pixels) around this central location and within `window_height` from the bottom of the image are then considered part of the lane line. `window_height` is automatically calculated based on the number of windows (`nwindows`). The window is then advanced sequentially by `window_height` towards the top of the image and its center is recalculated to be the mean location of filled pixels in the window if not enough filled pixels are found. At each new window location, more pixels are added to the full list of lane line pixels.

If a best fit for the lane line polynomial is already known, then `targetedLines()` is used instead. This function simply looks for filled pixels within `margin` pixels of the polynomial fit. If not enough pixels are found, then it calls `slidingWindowLines()` to recalculate a best fit from the beginning.

In either case, once pixels have been identified for both the left an the right lines, a 2nd order polynomial fit is found using numpy's `polyfit` function. Here is an example with both types of search and their resulting fits:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of curvature and lane offset calculations can be found at the end of the `process_image()` function in the "Apply Lane Finding Pipeline to Video" section of `lane_finding.ipynb`.

First, I defined scaling factors from pixels to meters that were based on the recommendations in the lesson. I then converted the polynomial fits from pixel units to scaled units using the conversion equations that were highlighted in the lesson.

These converted coefficients were then inserted into the radius of curvature equation and the curvature was evaluated for the bottom of the image.

Offset was then calculated by evaluating the pixel-unit fit at the bottom of the image to find where the left and right lanes intersected the car. The center point of the image was subtracted from the average of these two points to find the offset in pixels and then multiplied by the scaling factor for the x direction of the image.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in the `process_image()` function in the "Apply Lane Finding Pipeline to Video" section of `lane_finding.ipynb`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

The major challenge in this project was determining the proper thresholding combination that would accurately determine the lane lines in every frame of the video. I started by building utility functions for multiple threshold types to facilitate experimentation on the example pictures. My first attempts used the magnitude and direction thrshold pairs along with an HSV saturation filter, since these were the combinations that seemed to have the best performance from the lessons. I tweaked the threshold values for each of these until I had good performance on the majority of the pictures.

When trying my test image optimized thresholds on the videos, I hit two trouble spots. One was the section with a light background color on the pavement and the other was the section with a tree shadow. In both cases, the saturation threshold allowed background details to pass through and disturb the line fit. When I tried to increase the threshold value to avoid that problem, it eliminated too much of the line in light-colored pavement section. I next tried to add an HSV value filter to suplement this, but I couldn't find a good balance. A high saturation filter would eliminate white lines, but let in some dark lines. Ultimately, I took HSV saturation and HSL lightness as a pair so that I could get both dark, saturated yellow lines and bright white lines, while rejecting the road's background.

The problem with my thresholding technique is that it was optimized for the project video. You can see in the challenge videos that it still fails with difficult lighting conditions. Much of the bright light is still passed through my thresholds, so the background bleeds through in many of the challenge frames.

The other problem seen in the challenge videos is that any strong vertical lines will pass through the gradient filter. Since I used an `or` operator between threshold pairs, a tar line appears to be part of the lane even if it is the wrong color. I could go back and require some level of thresholding with an `and` combination of these pairs to eliminate vertical tar or shadow lines.

If I want to improve the lane finding further, then I need to go back and adjust the line averaging. Right now it is simply the mean of the last few lines. A trailing geometrically weighted mean would be better. I also do not reject outliers or check for dopped frames. These outliers then throw off the mean significantly. Error checking and rejection would make the whole algorithm much smoother.
