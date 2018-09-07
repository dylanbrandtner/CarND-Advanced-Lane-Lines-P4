## Advanced Lane Finding Project

[//]: # (Image References)
[image1]: ./output_images/DistortionCalibration.png "Distortion Correction"
[image2]: ./output_images/PerspectiveTransform.png "Perspective Transform"
[image3]: ./test_images/test5.jpg "Original Test"
[image4]: ./output_images/test5_undistorted.jpg "Undistort"
[image5]: ./output_images/test5_gradient_thresholds.jpg "Threshold"
[image6]: ./output_images/test5_perspective_warp.jpg "Warp"
[image7]: ./output_images/test5_found_lanes.jpg "Histogram"
[image8]: ./output_images/test5_found_lanes_centoids.jpg "Convolution"
[image9]: ./output_images/test5_final_result.jpg "Result"
[image10]: ./output_images/threshold_tune_start.png "Tuning frame"
[image11]: ./output_images/threshold_tune_original_pieces.png "tuning piece"
[image12]: ./output_images/threshold_tune_original_overhead.png "tuning overhead"
[image13]: ./output_images/threshold_tune_new_pieces.png "tuned piece"
[image14]: ./output_images/threshold_tune_new_overhead.png "tuned result"
[image15]: ./output_images/final_ex.png "final result"

The overall goal of this project was to overlay detected lane information onto a input video.  Here as an example frame from my final result:
![alt text][image15]

The specific goals / steps of this project are the following:

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

## Source code

All source code referenced below is contained in the [IPython notebook](./LaneLines.ipynb) in this repo.  See that notebook to follow along with the code changes described here. 

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Camera Calibration and Transformation

All of the calibration code is contained in the third code cell of the notebook. I setup a class to contain all the calibration data so that it can be used later.  

#### 1. Calculate distortion correction

For distortion correction, I followed the instructions from the course materials to use the OpenCV findChessboardCorners() method on each chessboard image provided after converting them to greyscale.  The corners are appended to an image points array, and then the OpenCV calibrateCamera() method is used to calculate distortion coefficients and the camera matrix.   

I setup a method to apply this distortion correction to an input image using the OpenCV undistort() function.  I tried this out on straight_lines2.jpg and saw this result:

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

I stored this information in a transform matrix and inverse transform matrix using OpenCV's getPerspectiveTransform() so I could move between perspectives.  Applying this to the undistorted image above resulted in the following image:

![alt text][image2]

### Pipeline (single images)

At this point I had the calibration data needed to start my image pipeline.   In  code block 7 of my notebook, I setup a 'CameraImage' class to store each image and apply various parts of the image pipeline to it. 

I used the following test image to tune my pipeline ("test5.jpg"):
![alt text][image3]

#### 1. Distortion correction

I applied the distortion correction calculated in the calibration phase to the test images and got the following result:
![alt text][image4]

#### 2. Thresholding

I used a combination of thresholds to generate a binary image. I went through two stages of tuning for these thresholds, but my first attempt based on this test image used the following:
* Sobel gradients on a greyscaled image:
	* A Sobel gradient in the x direction with thresholds of 20 to 100
	* A directional threshold with a sobel kernel size of 15, and thresholds of 0.5 and 1.3
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

In code blocks 9-11, I utilized the code from the course materials to find lane pixels in an image using sliding windows and then used numpy's polyfit() method to fit a polynomial to the found pixels.  I tried both the histogram and convolution approaches to see what was best.

Histogram and sliding windows result:
![alt text][image7]

Convolution and centroids result:
![alt text][image8]

The sliding window result was more reliable when tested on various images, and ended up being the method I used long term.  Also, the convolution method was more complex for me to understand and difficult for me to tinker with (ex. I added some logic to stop the detection if the sliding windows reached the edges of the image).

I also pulled in the search_around_poly() method for later use in the video pipeline to search for new pixels only within a certain margin of the input polynomials.

#### 5. Overlaying a found lane image

In code block 12, I updated the combine_images() method from the course materials to take the found lane information and replot it on the image.  I also took the found lane image from above and overlaid it in the top right corner so I could always see what the lane detection algorithm was doing. 

Here is my result on the test image:
![alt text][image9]

#### 6. Calculating radius and distance from center 

In code block 16, I used the instructions from the lesson to calculate the radius of the found polynomials. I then calculated the offset from lane center by subtracting the left and right x values at the bottom of each lane to get the lane size, then finding the lane center by adding half the lane size to the bottom pixel of the left lane.  The distance from lane center was then the image center minus the lane center.  This gave me values in pixels. 

I would convert to meters later in my video processing pipeline (block 17). For distance from lane center, I simply multiplied by x meters per pixel.  For calculating radius, I had perform a more complex calculation that was taken from the course materials.  See code block 17 for full details. For the conversion, I used a y meters per pixel value of 30/720 and a x meters per pixel value of 3.7/700.  These were given in the the course materials.  I could have calculated them myself using the input calibration images and known data about US lane size and lane line lengths, however, this did not seem worthwhile since the example values were based on the camera used in this exercise, and they seemed relatively consistent with my eyeball measurements.  This would not work if the camera mount point or camera itself was changed. 

---

### Pipeline (video)

In code block 17, I setup a video processing pipeline to be used with moviepy's "fl_image()" function that applies a function to each frame.  It basically applied all the previous steps to each image, added some handling to use the search_around_poly() function after the first frame, and drop frames if no lane pixels were found (to avoid errors when numpy's polyfit() method was applied to an empty array of pixels).  

#### Project Video First Result

Here's a [link to my first project video result](./project_video_out.mp4).

The result was quite good for the most part, but at around 40 seconds, the car drives over a different colored patch of road, and the lane finding veers off course and cannot recover.   

#### Challenge Video First Result

Here's a [link to my first challenge video result](./challenge_video_out.mp4).

In this video, the lane finding immediately picks up the wrong lines.  It sees some blacktop residue and shadows as the histogram peaks, and then completely drops the lanes under the bridge. 

---

### Improving the Pipeline

When viewing the above results, two major things became clear:  
1. The current thresholding operations resulted in too noisy of an image for the lane finding operation, and prominent road imperfections or vertical shadows were often mistaken for lanes.
2. Mistakes in the lane finding pipeline caused major disruptions in the pipeline, and it struggled to ever recover from these 

#### 1. Improving the thresholds

The road quality in the challenge video was much worse than the project video, so I used an early frame in that video to guide my threshold tuning.  I grabbed the frame from 4 seconds:

![alt text][image10]

Applying my thresholds and plotting the various pieces, I saw this:

![alt text][image11]

The resulting overhead lane finding was this:

![alt text][image12]

As you can see, the Sobel threshold picked up the shadow on the left and road imperfection in the middle. However, it also picked up the right lane line where the Saturation channel select failed. After thinking about what these had in common and examining additional images in more detail, I had a few other ideas:
1. The colors of lanes are always yellow or white
2. The Hue channel select consistently filtered out the shadows and road imperfections

Therefore, I updated my thresholds to the following (updates are in **bold**):
* Sobel gradients on greyscale image:
	* A Sobel gradient in the x direction with thresholds of 20 to 100
	* A directional threshold with a sobel kernel size of 15, and thresholds of .5 and 1.3
	* **Combine the above with a Hue channel max of 30**
* Channel selections on an HLS converted image:
	* A Hue channel max of 30
	* A Saturation channel minimum of 100
* **Color selections on the RGB image:**
	* **Selected yellow by looking for high R and G channel values (above 140) with a low B channel value (below 120)**
	* **Selected white by looking for high values in all three channels (above 200)**

I plotted these new thresholds, and saw these results:

![alt text][image13]

![alt text][image14]

Much better! The real lane lines are shown with hardly any other noise, and the lane detection is working well.  The results on other frames I tested were also improved significantly.   

#### 2. Keeping a history

As suggested in the course materials, in code block 31 of the notebook I setup a "LaneLine" class to store each detected lane, keep a history of previous lane detections, and average the result into a "best fit".  I setup a minimum of 5 detections before the lane would be considered "detected" and would be displayed.   

#### 3. Identifying outliers

As part of the LaneLine class, I also calculated the radius, position of the lane from the center of the image, and the difference of the current lane detections polynomial coefficients from the current "best fit" average.  Using this information, I could set thresholds for lane detections that were outside reasonable limits.  

For radius, anything with a radius below 100m, or above 100,000m would be rejected.  Any lanes detected more than 3 meters away from center would be rejected.  I then calculated the [Median absolute deviation](https://en.wikipedia.org/wiki/Median_absolute_deviation) of the current polynomial, and used this algorithm to give new polynomial's a "Z score". Typically in this methodology, Z scores above 1.0 are considered outliers, so I set that as my threshold.

#### 4.  Pipeline reset

After setting the above thresholds, I needed a way to reset the lane detection if the lanes were completely lost and we needed to restart the detection pipeline.  Each time a frame was rejected, I incremented a frame drop count, and if that count exceeded 10, I reset the entire pipeline by clearing the history of detections, and setting the detection states back to their original values. 

### Improved results

In code block 32, I combined the above improvements into a new video pipeline, and applied it to the videos.  I also added some updates to the text on screen so the viewer could see when frames were being dropped for the left and right lanes, and could see when the pipeline had been reset.

#### Project Video Improved Result

Here's a [link to my improved project video result](./project_video_out_improved.mp4).

In the trouble spot encountered before, the pipeline drops a couple bad frames, and the overall detection is generally very smooth. This should meet the minimum requirements for this project.

#### Challenge Video Improved Result

Here's a [link to my improved challenge video result](./challenge_video_out_improved.mp4).

The improved thresholding really shines on the poor quality roads and detects the proper lanes well.  Under the overpass, it loses the lanes after it drops 10 frames on the left lane, but is able to reset and quickly recover immediately afterwards.

#### Harder Challenge Video Result

At this point, I wanted to see how my improved pipeline would fare on the even harder challenge video.  

Here's a [link to my harder challenge video result](./harder_challenge_video_out.mp4).

Things start off fine, but as the lighting conditions begin to deteriorate and the curves get sharper, the lanes are quickly lost, and near the end, the detections completely overlap.  I saw this as a reasonable stopping point for the purposes of this exercise, but I had several other ideas for improvements if I had more time:
* I tried a few more rounds of threshold tuning, but never found a combination that was able to detect lanes reliably in all lighting conditions. My most reliable detection thresholds in light conditions were often the complete inverse in the dark.  Hue was most consistent channel I found, but using it exclusively resulted in far too much noise. I started to wonder if using the same thresholds for both light and dark was simply not feasible, and perhaps a better approach would be to detect the overall light condition using an image average, and then apply different thresholds for light vs. dark conditions. 
* Object detection (which I believe is the next set of lessons), could be useful to filter out noise from other vehicles, signs, etc.   
* The tight curves caused my overhead image to barely show the full lane in some cases.  My original region of interest cut off some of the left/right edges of the image to reduce noise, but I might need to bring these back to see the full curves.  
* Real lane size should be fairly consistent, but my detections often weren't, despite my lane base position thresholds and detection margins.  My lane base positions were an absolute value of the distance from the center of the image.  Instead, I could have added a variable to the LaneLine class that specified whether the lane was the left or the right one, and then the thresholds could be positive for the right lane, and negative for the left.  Alternatively, I could require the total size of the detected lane (i.e. the difference of the two) to be within a threshold instead of using distance from center.  I could also feed this threshold back into the lane finding pipeline to force the sliding windows to remain a certain distance apart.  I could also run multiple lane finding passes on the overhead image with different starting points to try and get the best result, as sometimes the first histogram peak was not the real lane line.