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
* Apply full pipeline to project video and detect lanes throughout video
* [STRETCH] Also detect lanes reliably on challenge videos

[//]: # (Image References)
[image1]: ./output_images/DistortionCalibration.png "Distortion Correction"
[image2]: ./output_images/PerspectiveTransform.png "Perspective Transform"
[image3]: ./test_images/test5.jpg "Original Test"
[image4]: ./output_images/test5_undistorted.jpg "Undistort"
[image5]: ./output_images/test5_gradient_thresholds.jpg "Threshold"
[image6]: ./output_images/test5_perspective_warp.jpg "Warp"
[image7]: ./output_images/test5_found_lanes.jpg "Histogram"
[image8]: ./output_images/test5_found_lanes_centoids.jpg "Convolution"
[image8]: ./output_images/test5_final_result.jpg "Result"


## Source code

All source code referenced below is contained in the IPython notebook located in "./LaneLines.ipynb".  See that notebook to follow along with the code changes described here. 

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Camera Calibration and Transformation

All of the calibration code is contained in the third code cell of the notebook. I setup a class to contain all the calibration data so that it can be used later.  

#### 1. Calculate distortion correction

For distortion correction, I followed the instructions from the course materials to use the OpenCV findChessboardCorners() method on each chessboard image provided after converting them to greyscale.  The corners are appended to an image points array, and then the OpenCV calibrateCamera() method is used to calculate distortion coefficients and the camera matrix.   

I setup a method to apply this distortion correction to an input image image using the OpenCV undistort() function.  I tried this out on straight_lines2.jpg and saw this result:

![alt text][image1]

#### 2. Calculate perspective transform matrix

I also considered the perspective transformation a part of image calibration, since it depends on the mount point of the camera.  Thus, I added this to the CameraCalibration class in code cell 3.  To figure out src and destination points, I copied in my old lane finding pipeline from project 1 (see https://github.com/dylanbrandtner/CarND-LaneLines-P1-Dylan) into code block 2 of my notebook, and used it to detect the lane lines in straight_lines2.jpg.  I then straightened these lines into vertical lines as such: 
```python
src = np.float32([lines[0][0],
                  lines[0][1],
                  lines[1][0],
                  lines[1][1]])
dst = np.float32([[lines[0][0][0]-left_cutoff,imshape[0]],
                  [lines[0][0][0]-left_cutoff,0],
                  [lines[1][1][0]+right_cutoff,0],
                  [lines[1][1][0]+right_cutoff,imshape[0]]])
```
Note: left_cutoff and right_cutoff were 50 pixels, to remove the noisy edge pixels from the image.  

This resulted in the following source and destination points:

| Source      | Destination  | 
|:-----------:|:------------:| 
| 287, 670    |  187, 720    | 
| 543, 4850   |  187,   0    |
| 743, 485    | 1082,   0    |
| 1032,670    | 1082, 720    |

I stored this information in a transform matrix and inverse transform matrix using OpenCV's getPerspectiveTransform() so we can move between perspectives.  Applying this to the undistorted image above resulted in this:

![alt text][image2]

### Pipeline (single images)

At this point I had the calibration data to start my image pipeline.   In  code block 7 of my notebook, I setup a 'CameraImage' class to store each image and apply various parts of the image pipeline to it. 

I used the following test image to tune my pipeline:
![alt text][image3]

#### 1. Distortion correction

I applied the distortion correction calculated in the calibration phase to the test images and got the following result:
![alt text][image4]

#### 2. Thresholding

I used a combination of thresholds to generate a binary image. I went through two stages of tuning for these thresholds, but my first attempt based on this test image used the following:
* Sobel gradients on greyscale image:
	* A Sobel gradient in the x direction with thresholds of 20 to 100
	* A directional threshold with a sobel kernel size of 15, and thresholds of .5 and 1.3
* Channel selections on an HLS converted image:
	* A Hue channel max of 30
	* A Saturation channel minimum of 100

Applying these thresholds to the image resulted in the following output:

![alt text][image5]

My second tuning phase will be described later in the "Improving the Pipeline" section.  

#### 3. Perspective transform 

I then applied the perspective transform from my calibration data on the thresholded image:

![alt text][image6]

#### 4. Lane finding

In code blocks 9-11 I copy/modified the source code from the course materials to find lane pixels in an image using sliding windows and fit a polynomial to this.  I tried both the histogram and convolution approaches to see what was best.

Histogram and sliding windows result:
![alt text][image7]

Convolution and centroids result:
![alt text][image8]

The sliding window result was more reliable when tested on various images, and ended up being the method I used long term.  Also, the convolution method was more complex for me to understand and difficult for me to tinker with. 

I also pulled in the search_around_poly() method for later use in the video pipeline to search for new pixels only within a certain margin of the input polynomial.

#### 5. Overlaying a found lane image

In code block 12, I updated the combine_images() method from the course materials to take the found lane information and replot it on the image.  I also took the found lane image from above and overlaid it in the top right corner for reference. 

Here is my result on the test image:
![alt text][image9]

#### 6. Calculating radius and distance from center 

In code block 16, I used the instructions from the lesson to calculate the radius of the found polynomials. I then calculated the offset from lane center by subtracting the left and right x values at the bottom of each lane to get the lane size, then finding the lane center by adding half the lane size to the bottom pixel of the left lane.  The distance from lane center was then the image center minus the lane center.  This gave me values in pixels. 

I would convert to meters later in my video processing pipeline (block 17). For distance from lane center, I simply multiplied by x meters per pixel.  For calculating radius, I had perform a more complex calculating that was taken from the course materials.  See code block 17 for full details. 

I used y meters per pixel of 30/720 and x meters per pixel of 3.7/700.  These were given the the course materials.  I could have calculated them myself using the input calibration images and known data about US lane size and lane line lengths, however, this did not seem worthwhile since the course material as part of this exercise as the examples values were based on the camera used in this exercise.  This would not work if the camera type or mount point changed. 

---

### Pipeline (video)

In code block 17, I setup a video processing pipeline to be used with moviepy's "fl_image()" function that applies a function to each frame.  It basically applied all the previous steps to each image, added some handling to use the search_around_poly() function after the first frame, and drop frames if no lane pixels were found (to avoid errors when numpy's polyfit() method was applied to an empty array of pixels).  

#### Project Video First Result

Here's a [link to my project video result](./project_video_out.mp4)

The result was quite good for the most part, but at around 40 seconds, the car drives over a different colored patch of road, and the lane finding veers off course and cannot recover.   

#### Challenge Video First Result

Here's a [link to my challenge video result](./challenge_video_out.mp4)

In this video, the lane finding immediately picks up the wrong lines.  It sees some blacktop residue and shadows as the histogram peaks as the lanes, and then completely drops the lanes under the bridge. 

---

### Improving the Pipeline

#### 1. Tuning the thresholds

#### 2. Keeping a history

#### 3. Identifying outliers

#### 4.  Pipeline reset

### Improved results

#### Project Video Improved Result

Here's a [link to my project video result](./project_video_out_improved.mp4)

#### Challenge Video Improved Result

Here's a [link to my challenge video result](./challenge_video_out_improved.mp4)

#### Harder Challenge Video Result

Here's a [link to my challenge video result](./harder_challenge_video_out.mp4)
