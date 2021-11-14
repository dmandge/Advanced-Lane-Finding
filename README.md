# Advanced-Lane-Finding
Software pipeline to identify positions of the left and right lane lines on the road, in the image and video stream


https://user-images.githubusercontent.com/59345845/141694469-ee932220-a340-437a-8a0b-c57be885c3f4.mp4


## Overview
The goal of this project is to write a software pipeline, that will identify positions of the left and right lane lines on the road, in the images and video stream.
This is achieved in following steps:
1. ***Camera Calibration*** – Calculate calibration matrix and distortion coefficient for the installed camera from a set of chessboard images
2. ***Undistortion*** – Apply this distortion correction to raw road images
3. ***Color and gradient thresholds*** – To obtain a binary image containing maximum pixels belonging to lane lines
4. ***Perspective Transform*** – obtain birds eye view of each road image
5. ***Fit polynomials to lane lines***
6. ***Calculate radius of curvature and offset from center for each lane line***

### Camera Calibration
A camera mounted on the car (used to capture test images and videos) converts 3D image of the road into 2D. This conversion causes the generated images to become distorted – bent at the edges and lines may appear incorrectly skewed.
The distortion can be either radial or tangential and it needs to be corrected for an accurate lane lines detection in self driving cars

Radial distortion correction -

![Formula1](https://user-images.githubusercontent.com/59345845/141692568-37e727ef-591e-40e5-82eb-938a56b73fd2.JPG)

Tangential distortion correction -

![Formula2](https://user-images.githubusercontent.com/59345845/141692577-5e43f2aa-8bd5-4a64-9551-b32542b5e7b5.JPG)

Camera calibration for distortion correction can be performed using multiple images of a chessboard taken from different angles and applying functions from OpenCV library in python.

#### Steps for camera calibration - 
1. Create empty arrays of object points (3D coordinates of undistorted chessboard image) and image points (2D coordinates of distorted chessboard image)
2. Object and image point here correspond to 9X6 corners visible in the chessboard images
3. A for loop is created to load the images of the chessboard one by one and convert them to grayscale
4. Cv2.findChessboardCorners() function is used to find chessboard corners in the grayscaled image
5. Once the 9X6 corner points are detected at the end of one for loop, their coordinates are added to the object and image points array.
6. Copy of the original image – img is used to overlay these corner points using cv2.drawChessboardCorners() function

      ![drwchessvoardcorners](https://user-images.githubusercontent.com/59345845/141693289-ec461464-4a42-4914-8e59-94c1a1456bb3.jpg)

      *Figure 1: Draw Chesssboard Corners*
 
7. Cv2.CalibrateCamera() function is used to compute the calibration coefficients. This function returns camera matrix (mtx), distortion coefficients (dist) along with rotation and translation vectors. Mtx and dist from this function are used later in the software pipeline to undistort test images using cv2.undistort()
      ![imageundstoration](https://user-images.githubusercontent.com/59345845/141693184-995fd91b-a068-43ab-8e77-aff3ef423845.JPG)

      *Figure 2: Image Undistortion*

  ### Software Pipeline for images
  1.	Distortion Correction – 
  * Distortion correction coefficients and camera matrix calculated via cv2.calibrateCamera() function are applied to each image in the test image folder to undistort them, using cv2.undistort()
  ![imageundstoration_lanes](https://user-images.githubusercontent.com/59345845/141693521-3327060c-a7c4-4411-a66f-9b0ea60affa1.JPG)
  
      *Figure 4: image undistortion*

  2.	Gradient Threshold (Sobel Operator) – 
  * After images are undistorted, we convert it into a binary image in order to perform further filtering to highlight lane lines in the image
  * First step towards achieving that is applying Sobel operator. I defined a function abs_sobel_thresh() to apply sobel operator to each test image and take derivative in the x and y direction.
  * Keeping the kernel size at 3, threshold windows are chosen such that maximum pixels from both lane lines are highlighted with minimum noise from other objects
  * X gradient threshold – [15,255]
  * X gradient threshold – [25,255]
  
  3. HLS and HSV Threshold – 
  * In addition to gradient threshold, I have applied saturation threshold (HLS) to capture lower saturation lane lines and value saturation to reduce the effect of right light and shadows from the image
  * S threshold – [110,255]
  * V threshold – [60,255]
  * In the final software pipeline, gradient and SV threshold is applied with OR condition so that lane line detection failure with gradient threshold because of lighting is captured by SV thresholds
  * Example of final binary image after color and gradient threshold is shown below 
  ![binaryimageafterthreshhold](https://user-images.githubusercontent.com/59345845/141693632-e0f5eaf1-984b-433e-b6a6-42e85ff1e487.JPG)
  
     *Figure 5: Binary image after thresholds*

4.	Perspective Transform – 
* Once the binary image is created that highlights maximum pixels from lane lines, the next step is to perform perspective transform to generate a birds’ eye view of the road image. This is done because ultimately we want to measure curvature of the lines. To do this we first define source and destination points. Source points are coordinates of 4 points on the test image that are approximately in the same plane and adequately cover both lane lines. I have selected source points such that perspective transform only captures lane lines while avoiding any noise from surrounding objects. I used an image Interactive window to detect x and y coordinates of source points. These source points are mapped to destination points that represent a rectangle of the same size as test image
* Cv2.getPerspectiveTransform() function is used to generate a transform, M and cv2.warpedPerspective() function is used to create a warped image with bird’s eye view of the lanes. 
* Figure 7 shows an example of the road image after perspective transform. One of my criteria to define source points was that both lane lines should appear approximately parallel to each other after perspective transform

  ![warpedimage](https://user-images.githubusercontent.com/59345845/141693700-91ec01e0-fab0-432b-9eb4-1ca3d85df936.JPG)
  
    *Figure 7: Warped image with bird's eye view of the road*
    
5.	Finding Lane Lines (polynomial fit)
* Once the binary warped image of the lane lines is generated, the next step is to locate pixels belonging to left and right lane lines. For this I followed histogram technique as described in the course.
    * First, I took a histogram of the bottom half of the image (bottom half because we want to locate left and right lane pixels for the first frame starting at the bottom). Following plot shows histogram for all test images.
 
    ![histogram](https://user-images.githubusercontent.com/59345845/141693798-1446dcc7-567e-41ee-85f5-5bbfbb05b54a.jpg)

    * Start point of the left and right lane is easily noticed in the above image and can be calculated from x and y coordinates of the image
    * Using two highest peaks of the histogram as starting point, I used sliding window technique to determine location of frames moving further along the road
    * For this project I selected 9 frames in one image, with a margin of 100 and minimum pixel concentration is kept at 50. 
    * Looping through each window, I found boundaries of current window and number of activated pixels in it. If the number of pixels in the subsequent window is more than 50, I reset the window location to the new mean position. This is performed in the function find_lane_pixels() in my code. A n example of test image 1 shown below.
    ![lanelinestracked](https://user-images.githubusercontent.com/59345845/141693982-f2180ab6-db8e-4af5-8fb8-48a7b7ad571b.JPG)
        *Figure 8: Lane Lines Tracked*
        
* Once lane lines are tracked in the binary warped image, next step is to fit a polynomial through it to define left and right lanes. 
    * Using left and right pixel coordinates, I used np.polyfit to generate a polynomial for left and right lane
    * To plot these curves on the top of the test image, I created a different array of x and y coordinates of lanes, using polynomial equation and an array of y coordinates in ploty (starting from the bottom of the image in an interval of 1)
    * To visualized this on the binary image, left lane pixels are shown in blue and right lane in red. The polynomial fit is shown in green on the top of the lane lines
    
    ![lanelinevisual](https://user-images.githubusercontent.com/59345845/141694038-e026b43e-ccb2-462d-9a3a-46e4e918754a.JPG)
       
     *Figure 9: Lane Line Visualization*
* For running this pipeline on videos, we don’t need to do this process from the beginning for every frame, we can just search in the margin of previous lane lines position. For this I grabbed x and y pixel position from earlier frame along with the polynomial fit, and calculated location of pixels that fall into green shaded area shown above. 
This way all the relevant lane pixels are found and we fit a second order polynomial, again for both lanes, repeating the step above, I generated left and right lanes [x,y] coordinates to plot on the image. 
As a final step, lane lines are plotted on the original image using cv2.addWeighted().

![lanelinesonoriginalimages](https://user-images.githubusercontent.com/59345845/141694089-23b86646-6405-4425-80b5-6616a848692b.JPG)
  
  *Figure 10: Lane Lines on Original Image*
  
6.	Radius of Curvature and Offset Calculation
* For additional information, I calculated radius of curvature for both left and right lane lines and offset from center (assuming camera I mounted at the center of the image/car)
* Lane is assumed to be 30 m long and 3.7 m wide. This information is used ot convert pixels into meters. using np. Polyfit(), I fit a polynomial for lanes (this time in units of meters)
* Offset is distance by which center ofteh lane lines is off from center of the car/image. Center of the lane line is average of x coordinate of left and right lanes and center of the car is image width/2. Difference, offset is printed on the final output image, as shown below

![finaloutputimage](https://user-images.githubusercontent.com/59345845/141694141-747cd30e-c0c3-426b-a102-550d40f2a4af.JPG)
  
  *Figure 13: Final output image*


 
 
 
 
 

