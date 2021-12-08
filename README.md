# Udacity_Self-Driving Car
## Advanced Lane Finding Project

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

[im01]: ./output_images/01.camera_calibration.png "Chessboard Calibration"
[im02]: ./output_images/02.undistorted_chessboard.png "Undistorted Chessboard"
[im03]: ./output_images/03.undistort.png "Undistorted Dashcam Image"
[im04]: ./output_images/04.unwarp.png "Perspective Transform"
[im05]: ./output_images/05.colorspace.png "Colorspace Exploration"
[im06]: ./output_images/09.sobel_magnitude_and_direction.png "Sobel Magnitude & Direction"
[im07]: ./output_images/11.HLS_L_Channel.png "HLS L-Channel"
[im08]: ./output_images/06.sobel_absolute.png "Sobel Absolute"
[im09]: ./output_images/12.pipeline_all_test_images.png "Processing Pipeline for All Test Images"
[im10]: ./output_images/13.sliding_window_polyfit.png "Sliding Window Polyfit"
[im11]: ./output_images/14.sliding_window_histogram.png "Sliding Window Histogram"
[im12]: ./output_images/15.polyfit_from_previous_fit.png "Polyfit Using Previous Fit"
[im13]: ./output_images/16.draw_lane.png "Lane Drawn onto Original Image"
[im14]: ./output_images/17.draw_data.png "Data Drawn onto Original Image"

[video1]: ./project_video_output.mp4 "Video"

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first two code cells of the Jupyter notebook `Advanced Lane Finding Project.ipynb`.  

‘FindChessboardCorners’ and ‘calibrateCamera’ are OpenCV functions for image correction. Use as input multiple images of a chessboard taken from different angles with the same camera. The object point array (objpoints) corresponding to the position of the inside edge of the chessboard and the imagepoints (imgpoints), which is the pixel location of the inside chessboard edge determined by'findChessboardCorners', are input to the'calibrateCamera' which returns the camera correction and distortion coefficient.
Then, the ‘undistort’ function, the OpenCV function, cancels the distortion effect of the image created with the same camera. The image below shows the edges drawn on 20 chessboard images using the OpenCV function “drawChessboardCorners”:

![alt text][im01]

*Note: Some of the chessboard images don't appear because `findChessboardCorners` was unable to detect the desired number of internal corners.*

The image below depicts the results of applying `undistort`, using the calibration and distortion coefficients, to one of the chessboard images:

![alt text][im02]

### Pipeline (single images)

#### 1. Examples of distortion-corrected images.

The image below depicts the results of applying `undistort` to one of the project dashcam images:

![alt text][im03]

The effect of `undistort` is subtle, but can be perceived from the difference in shape of the hood of the car at the bottom corners of the image.

#### 2. Describes a variety of methods, including color transformation, gradients, and more, to create a thresholded binary image. Here is an example of a binary image result.

I explored several combinations of sobel gradient thresholds and color channel thresholds in multiple color spaces. These are labeled clearly in the Jupyter notebook. Below is an example of the combination of sobel magnitude and direction thresholds:

![alt text][im06]

The below image shows the various channels of three different color spaces for the same image:

![alt text][im05]

Ultimately I chose to separate the white and yellow lines using the L channel in the HLS color space and Sobel's Absolute. I did not use the slope threshold in the pipeline. However, we have fine-tuned the thresholds for each condition to be minimally acceptable to changes in lighting. Several trials and errors in this process have shown that the next threshold is perceived as the clearest lane. Here are example thresholds for the HLS L channel and Sobel Absolute:

![alt text][im07]
![alt text][im08]


Below are the results of applying the binary thresholding pipeline to various sample images:

![alt text][im09]

#### 3. Describe how you performed a perspective transform and provide an example of a transformed image.

The `unwarp()` function takes an image (`img`) and a source (`src`) and a destination (`dst`) point as inputs. I chose to hardcode the source and destination points in the following way:

```
src = np.float32([(575,464),
                  (707,464),
                  (258,682),
                  (1049,682)])
dst = np.float32([(450,0),
                  (w-450,0),
                  (450,h),
                  (w-450,h)])
```
I wrote the part where I want to transform the viewpoint of the image in the ‘src’ function with 4 coordinates. And I wrote the coordinates of the part where I want the picture to see the front in the ‘dst’ function:

![alt text][im04]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

`sliding_window_polyfit` and `polyfit_using_prev_fit` are functions that identify lanes and fit a quadratic polynomial to both the right and left lanes. The first of these computes the histogram for the bottom half of the image and finds the bottom x-position (or "baseline") of the left and right lanes. Originally these locations were identified at the local maxima of the left and right half of the histogram, but in the final implementation I changed it to 1/4 of the histogram to the left and right of the midpoint. This helped to reject routes coming from adjacent lanes. Then this function will identify 10 windows to identify lane pixels. Each window is centered at the midpoint of a pixel in the lower window. This effectively speeds up processing by "following" the lane all the way to the top of the binary image and searching for active pixels only for a small portion of the image. The pixels belonging to each lane are identified and the Numpy `polyfit()` method fits a quadratic polynomial on each set of pixels. The image below shows how this process works.

![alt text][im10]

The image below depicts the histogram generated by `sliding_window_polyfit`:

![alt text][im11]

The'polyfit_using_prev_fit' function basically does the same thing, but it alleviates a lot of the difficulty of the search process by taking advantage of the previous video frame and searching only the sub-optimal pixels within a certain range of that fit. The image below shows this. The green shaded area is the range of the previous alignment, and the yellow lines and red and blue pixels are from the current image.

![alt text][im12]

#### 5. Describe how you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

The position of the vehicle with respect to the center of the lane is calculated with the following lines of code:
```
lane_center_position = (r_fit_x_int + l_fit_x_int) /2
center_dist = (car_position - lane_center_position) * x_meters_per_pix
```
`r_fit_x_int` and `l_fit_x_int` are the x-intercepts of the right and left fits, respectively. This requires evaluating the fit at the maximum y value (719, in this case - the bottom of the image) because the minimum y value is actually at the top (otherwise, the constant coefficient of each fit would have sufficed). The car position is the difference between these intercept points and the image midpoint (assuming that the camera is mounted at the center of the vehicle).

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The trapezoid is created based on the plot of left and right alignment, and is warped back to the perspective of the original image using the inverse perspective matrix'Minv' and overlaid on the original image. The image below is an example of the result of the `draw_lane` function.

![alt text][im13]

Below is an example of the results of the `draw_data` function, which writes text identifying the curvature radius and vehicle position data onto the original image:

![alt text][im14]

---

### Pipeline (video)

#### 1. Provide a link to your final video output

Here's a [link to my video result](./project_video_output.mp4)

---
