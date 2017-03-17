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

[image1]: ./examples/undistortion_example.jpeg "Undistorted"
[image7]: ./examples/Undistorted_real.JPG "Undistorted Real"
[image8]: ./examples/binary_filters.JPG "Binary Filters"
[image3]: ./examples/binary_combo.JPG "Binary Example"
[image4]: ./examples/warp_example.jpg "Warp Example"
[image5]: ./examples/lane_lines.jpg "Fit Visual"
[image6]: ./examples/textSample.jpg "Output"
[image9]: ./examples/histogram.jpg "Histogram"
[image10]: ./examples/radius_formula.jpg "Radius Formula"
[video1]: ./project_video.mp4 "Video"
  

---
###Writeup / README

###Camera Calibration

*Camera Calibration*

The code for this step is contained in the first code cell of the IPython notebook located in "./P4_Notebook.ipynb". The function that performs the undistortion is called "cal_undistort" and is located under the heading "Image Pipeline Functions" 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

Then I applied the distortion correction to an image of the real world, and the result is below:

![alt text][image7]

I find it easiest to notice the correction when looking at the trees in the corners of the images.

*Line Identification*

I used a combination of color and gradient thresholds to generate binary images, the functions are visible in the section labeled 'Binary Functions' in the jupyter notebook.  Here's an example of the various functions in action:

![alt text][image8]

I then combined different binary outputs to create a combination image that takes the best parts of various binary images. This "combined binary" function is called "binary_pipeline" and is visible under the "Image Pipeline Functions" heading.
An example of this combined binary image:

![alt text][image3]

*Perspective Transform*

The code for my perspective transform includes is found in the first cell of the jupyter notebook. I initialize the values under the comment "Perspective Transform". I store the result of cv2.getPerspectiveTransform (M, and Minv (which does the opposite)) for later use in the image pipeline. It probably would have been more efficient to make a function, but I didn't need to at the time. The cv2.getPerspectiveTransform function takes in source and destination points to create a mapping matrix which is used later on in the image pipeline when I call cv2.warpPerspective. I chose to hardcode the source and destination points in the following manner:

```
src = np.array([[688,450],[1040,673],[271,673],[594,450]], np.float32)
dst_points = np.array([[1000,0],[1000,720],[250,720],[250,0]], np.float32)

This severely affects the re-usability of this code as the dimesions are tied to the image size of one particular camera, and I could make it more robust by using ratios of image size.

```
This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 688, 450      | 1000, 0        | 
| 1040, 673      | 1000, 720      |
| 271, 673     | 250, 720      |
| 594, 450      | 250, 0        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

*Line Fitting*

Once I had a bird's eye view of the lane, I was able to calculate a second order polynomial that would fit each line. The code responsible is found in the notebook under the "Fitting Lines" heading.
In order to identify the line, I created a histogram of the base of the line which showed the hotspots for detected pixels in the bottom half of the picture:

![alt text][image9]

Once I knew where the base of the line was likely to be found, I implemented a sliding box approach to detect line pixels that would pick out all the pixels that were likely to make up the lane lines. In the 
following image, the left lane pixels are highlighted in red, and the right lane pixels are highlighted in blue. The sliding box is visible in green, and the code only searches for lane line pixels within that box.
The box can move left or right on the average of detected pixels if enough are found to satisfy a threshold that I specified "minpix".

![alt text][image5]

Once I knew the general whereabouts of the lane line, I searched within the vicinity instead of using sliding boxes. The method that performs this search is called "focused_search".

*Radius of curvature*

I calculated the radius of curvature using the established formula:

![alt text][image10]

The code calculated the radii of the lines individually, then averaged the result to display in the video. The function is called get_line_radius in the notebook.

*Offset from center*

I was able to calculate the distance from center of the lane by using the fitted lines as a guide, and finding the y-intercept of the lines which results in the base of the line. Assuming the camera is located
on the center of the car, and that we know the real world distance between lane lines, we can determine real world distances using the number of pixels. Then you just need to calculate the difference in distance the center of the image is from the center of the lane lines.
The function responsible for the calculation is called get_vehicle_position.

*Painting the lane*
We can paint all the pixels between our two lane lines and then perform a reverse perspective transform to show this painted lane on the original undistorted photos. The function responsible is called "paint_lines".
This function is also responsible for displaying the radius and offset.

![alt text][image6]

---

*Pipeline video result*

Here's a [link to my video result](https://www.youtube.com/watch?v=wdI46GCf6Ts&feature=youtu.be)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?
I spent hours trying to find the best combination of binary filters that would only pick up the lane lines so I wouldn't have to do any smoothing on the video (saving/averaging previous values) but that proved to be a fruitless
endeavour. The HLS binary was amazing at picking up the left lane, but really struggled with dark shadows. I wanted to combine it with the hsv binary to remedy this, but then the pipeline struggled to pick up the left line in the bright patches.
I ultimately decided to perform some smoothing by averaging a the line fit over a certain amount of previous frames and adding some sanity testing to see if a line was found. If no line was found, then the previous values are used.
The pipeline will definitely fail on images that are of a different size, or on images that have lots of bright and dark spots. I could easily modify the perspective transform points to accomodate different images to help make the pipeline more robust.
I feel as though I could also play around with the gradient/color thresholding values a lot longer to really filter out only the lane lines in any condition. 
One unique attempt I made was to add a "ghost" of the previous frame's lane line to the new image to help in places where there are no lane lines. I was going to save some pixels of the line, then add a random subset (was using 50%) of them to the new line
which would add a ghost of the old line to the new line. Ultimately I abandoned this approach as I thought it might create an overwhelming number of pixels in the case of a solid line. I think it is an idea worth revisiting, especially if I can identify whether a line is solid or not.
In cases where it is dashed, the ghosting method could prove useful.

