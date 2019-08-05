# Advanced Lane Finding
[![Udacity - Self-Driving Car NanoDegree](https://s3.amazonaws.com/udacity-sdc/github/shield-carnd.svg)](http://www.udacity.com/drive)
Anil Kumar
  

The Project
---

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

[im01]: ./output_images/camera_cal/distorted_camera_cal.png "Distorted Chessboard"
[im02]: ./output_images/camera_cal/undistorted_camera_cal.png "Undistorted Chessboard"

[im03]: ./output_images/color_channels/distortedImage5.png "Distorted Image"
[im04]: ./output_images/color_channels/undistortedImage5.png "Undistorted Image"

[im05]: ./output_images/color_channels/warpedImage5.png "Hue Image"
[im06]: ./output_images/color_channels/hChannelImage5.png "Hue Image"
[im07]: ./output_images/color_channels/lChannelImage5.png "Lightness Image"
[im08]: ./output_images/color_channels/sChannelImage5.png "Saturation Image"
[im09]: ./output_images/color_channels/combinedBinaryImage5.png "Combined Binary Thresholded Image"

[im10]: ./output_images/sliding_window.png "Sliding Window Image"
[im11]: ./output_images/lane_fitting.png "Lane fitting on the binary image"
[im12]: ./output_images/lane_draw_on_original.png "Lane drawn on the original image"
[im13]: ./output_images/lane_draw_on_original_with_roc.png "Lane drawn on the original image with radius of curvature and distance from center"

## Computing the Camera Calibration Co-efficients

The code for computing camera calibration matrix can be found in the file : 'cameraCalibration.ipynb'.

The images in the camera_cal folder serve as a basis here to compute calibration co-efficients. For every image in the folder, the chess board corners are found using the opencv function 'findChessBoardCorners'. The corresponding object_points (real world points in 3d) and image_points (image plane points in 2d) are stored in an array for all images.

Then, the opencv function 'calibrateCamera' can be used to obtain the calibration and distortion co-efficients on any image with the stored arrays.Using the co-efficients calculated, a distorted image is then passed to another opencv function 'undistort' to get an undistorted image.

The below figure shows a distorted and undistorted chessboard

| Distorted Chessboard | Undistorted Chessboard |
|:---:|:---:|
| ![alt text][im01] | [alt text][im02] |



## Image Pipeline for lane lines extraction

The code for an image pipeline for lane finding can be found in the file 'AdvancedLaneDetection.ipynb' from cells 1-4. 

The goal of this pipeline is to create a warped binary thresholded image given a raw distorted image. The following steps discusses the techniques used.

### Applying the distortion correction to raw images

The computed camera and distortion co-efficients can now be used to correct the raw images. The same opencv function 'undistort' is used here too. Input here is the distorted images and output is the undistorted images.

The figures below show distorted and undistorted images.

|Distorted Image | Undistorted Image |
|:---:|:---:|
| ![alt text][im03] | ![alt text][im04] |

### Creating a warped perspective

Once the image has been undistorted, it is used to get a better view of the lane in the image. Since the front view of the image gives a poor representation of the lane markings, a top-view is necessary in order to get a better view. The opencv functions, 'warpPerspective' and 'getPerspectiveTransform' are used to achieve this.

The figure below shows an undistorted image and its warped perspective image

|Undistorted Image | Warped Perspective Image |
|:---:|:---:|
| ![alt text][im04] | ![alt text][im05] |

### Evaluation of color channels

The top-view image or the warped perspective image now holds a good view of the lane markings. But, it still needs to be extracted. Although, the traditional edge detection techniques like Sobel and Canny work relatively well on detecting edges on the images which could point to the lane markings, they fail to give good results when the conditions are challenging. Due to this reason, various color channels are evaluated from the image to see which channel gives us a better representation of lane markings.

In this work, I have evaluated Hue, Lightness and Saturation channels of the undistorted image. These can be obtained by using the following function 

```
def getHlsColorChannels(warpedTopViewImage):
    hlsImage = cv2.cvtColor(warpedTopViewImage, cv2.COLOR_RGB2HLS)
    hChannel = hlsImage[:,:,0]
    lChannel = hlsImage[:,:,1]
    sChannel = hlsImage[:,:,2]   
    return hChannel, lChannel, sChannel
    
```

Upon evaluation, it could be found that the saturation channels can extract yellow lines and lightness channel can extract white lines. These two channels are now combined and thresholded to be able to extract both yellow and white lines. The following function combines and threshold the two channels to output a thresholded binary image.

```
def getCombinedBinaryImage(lChannel, sChannel):
    # Combine L and S channels
    combined = np.zeros_like(lChannel)
    combined[(lChannel == 1) | (sChannel == 1)] = 1  
    return combined    
```   

The figures below show multiple channels of an image and the combined thresholded binary image.

|Hue Channel | Lightness Channel | Saturation Channel | Combined Binary Threshld |
|:---:|:---:|:---:|:---:|
| ![alt text][im06] | ![alt text][im07] | ![alt text][im08] | ![alt text][im09] |


## Polynomial fitting

With the evaluation of color channels, the yellow and white lines are extracted from the image. The accurate representation of lane-line pixels cannot be achieved just by this because, the color channels extraction method do not work under all circumstances  and for the complete image. To get an accurate representation, a polynomial is fitted to the thresholded binary image from bottom to the top of the image. This is explained in the sections below.

### Polynomial fitting using a sliding window

The code for this can be found in cell block 5.

From the thresholded binary image, which is the output of image pipeline, a histogram is calculated of the bottom half of the image. The peak of left and right halves of the histogram indirectly points us to left and right lane areas. 

Now, total number of windows to be fit is defined. In this example, I have defined it to be 10 which was found to be satisfactory. Different properties like height, minimum pixels inside the windows are also defined at this stage. After this, non-zero pixels within this window is identified in both X and Y directions and are appended to the lane line indices. The window is now moved all over the image from bottom to top. Once all the non-zero pixels (lane line indices) are appended, the left and right lane pixel positions can be extracted using

```
    leftx = nonzerox[left_lane_inds]
    lefty = nonzeroy[left_lane_inds] 
    rightx = nonzerox[right_lane_inds]
    righty = nonzeroy[right_lane_inds]    
```

To the extracted left and right lane pixel positions, a second order polynomial is now fit using

```
    left_fit = np.polyfit(lefty, leftx, 2)
    right_fit = np.polyfit(righty, rightx, 2) 
```    
The figure below shows stacked windows in the search area for lane line pixels
![alt text][im10]


### Ploynomial fitting using a previous fit

Code for this can be found in code block 6.

Suppose, we are considering a sequence of images as in a video, we need not expect a big change in the scene from one frame to another. Utilizing this, the search area for lane-line pixels can be reduced if we know where the lane lines were detected in the last frame. This function only searches in the area where the lane-line pixels was found in the previous frame which reduces computation time and accuracy.

The figure below shows a lane drawn onto a warped binary image.

![alt text][im11]

The figure below shows the lane drawn onto the original image.

![alt text][im12]

## Calculating Radius of Curvature and distance of the vehicle from the center

Since the image is measured in pixels and the radius of curvature, distance need to be measured in meters, the assumed conversion between pixels and meters is as shown below.

```
    # Define conversions in x and y from pixels space to meters
    ym_per_pix = 3.048/100 # meters per pixel in y dimension
    xm_per_pix = 3.7/378 # meters per pixel in x dimension
```


Now, new polynomials are fitted to each of left and right lanes in x,y which are in meters.

```
    left_fit_cr = np.polyfit(lefty*ym_per_pix, leftx*xm_per_pix, 2)
    right_fit_cr = np.polyfit(righty*ym_per_pix, rightx*xm_per_pix, 2)
``` 

After the polynomials are fit, the radius of curvature can be calculated as shown below for each lane

```
    left_curverad = ((1 + (2*left_fit_cr[0]*y_eval*ym_per_pix + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
    right_curverad = ((1 + (2*right_fit_cr[0]*y_eval*ym_per_pix + right_fit_cr[1])**2)**1.5) / np.absolute(2*right_fit_cr[0])
```

The distance of the vehicle from the center is calculated by the difference between midpoint in X direction and mean of left and line lane fit intercepts.

```
    center_dist = (car_position - lane_center_position) * xm_per_pix
```

The figure below shows the lane drawn onto the original image and displaying the radius of curvature, distance of the vehicle from the center on top.

![alt text][im13]

## Video Pipeline for Advanced Lane Finding

The video pipeline utilizes the same techniques as in the image pipeline discussed before. In addition, the succesive frames are checked if a suitable lane fit already exists from previous frames so that it can be used to reduce the search area for polynomial fitting. To achieve this, a class Line() is defined as updated every frame which can be found in code block 27.

A function `process_image` defined in code block 28 is executed for each frame which runs the pipeline, fits a polynomial, calculates radius of curvature, distance from center and visualizes them on the original image.

The output video can be found here. 