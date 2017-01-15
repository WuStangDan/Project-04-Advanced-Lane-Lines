# Project-04-Advanced-Lane-Lines


## Calibrating the Camera and Undistorting the Images

One of the calibration images that was a .png was converted to .jpg using an online converter. Also two of the images for some reason had an extra row and column of pixels that seemed to be pure noise added to the top and left side. This was removed when reading in the image.

The camera was calibrated using the cv2.calibrateCamera function and the images were undistorted using the calibrated camera object and image points on all of the test images provided.

## Transform into Binary Image

### RGB Color Threshold
Assume all lane lines are either yellow or white. Pure white is RGB = (255, 255, 255) and pure yellow is RGB = (255,255,0). So really a high threshold on both (or either) of the R AND the G should force the remaining pixels to either be yellow or white. This could remove some noise that still happens to make it past the the other thresholds that will be used.

Thresholding for yellow and white colors only seems to work on dark pavement. On the light pavement colors it doesn't really pick anything up until the very high thresholds which would remove other information. Please see the accompanying pictures in the color transform section.

However using a low threshold (like 90 or less) just on the red threshold could be used as an AND with other transforms since it could potentially help on the dark pavement sections and essentially have no effect on the light pavement sections.



### HLS Color Threshold

Saturation channel seems to clearly be the best but still am not happy with the results on their own. Looking at the last photo in the bottom right (see html notebook) you can see that on the white pavement section the right lane lines hardly shows up at all.

For saturation a minimum threshold of 100 seems to clearly identify the lane lines without removing too much of the lane lines themselves.


### Perspective Transform

The most difficult part of the perspective transform is selecting how far up you should try to look for the lane lines. However the perspective transform really helps in visualizing not only where the horizon is (and thus that above this point there is no longer any lane line data) but also it helps finding the point where there isn't enough data (because the resolution of the camera) to make out small details of the lane lines. By setting the top perspective transform source points far away, you can see how blurry it is and how hard it is to make out the lane lines at points that are just before the horizon. I used this blurryness as a rough guide for how far up the road to look.

Based on the plots seen in the HTML under the heading of perspective transform, it can be seen that looking any higher than pixel 465 (thus top cords of 460 or less), the image is so blurry at the top that there is almost no discernable information in it.

### Gradient Threshold

A gradient output that is a combination of mutliple thresholds (absolute, magnitude, direction) is required to help find the lane lines in an image. While the output doesn't have to be perfect, because I will end up combining what I find with the HLS and RGB color thresholds, it does have to be fairly good at seeing the lane lines since it is most likely that this threshold will do most of the work.

Once I tune the parameters for a very good gradient that works on all of the images (1,4,5,6) I'll then fine tune the parameters so it can work better when I combine it with the HLS threshold and the R color threshold.

No amount of fine tuning seems to have the y gradient yield any information that is not in the x gradient. This would negate any benefit from using a y gradient in an AND with the x gradient. Also the y gradient is far too noisy for things outside the lane lines to be any use as a standalone addition as an OR. So I'm not going to use the y gradient for the final gradient combination.

The magnitude direction gradient on its own is very good at finding the lane lines however, it also pickups a great deal of noise. My main goal when selecting a direction gradient is to have it pick up the lane lines, but not the pieces of noise that can be seen in the magnitude gradient.

In the picture below it can be seen that the direction gardient picks up almost everything in the image, except for the noise seen in pixels 0-100 in the Y axis of the magnitude gradient. This means that using the direction gradient in an AND operator with the magnitude gradient should help filter out most of its noise.

![Magnitude and Direction Gradient](/screenshots/mag-dir.png)

### Combining Color, HLS, and Gradient

The first threshold I performed only on the red color channel I believe is still beneficial. A very low threshold (easily passed) will be used that will essentially filter out any object that doesn't have a strong red component (which the white and yellow lines will have plenty of). Thus I will use the red color channel threshold as an AND with both the HLS and gradient.

The HLS and gradient aspects will be used in an OR operator since they both have different strengths. The gradient is really good at dark road surfaces (where there is a large difference between the light lines and the dark road) and the HLS is really good at picking up any deep colors, regardless of the light or darkness of the road surface. However inorder to avoid most of the noise the HLS threshold has to be set quite aggressive, which is why the gardient threshold is needed.

Looking at test image 5 (shown below), my decision to include the red threshold as an AND operator on both saturation and gradient really pays off. As seen in the saturation threshold for test image 5, there is a great deal of noise on the Y axis around 300 pixels which isn't included in the final combined binary image. This is because the very weak red threshold (which allows almost everything in the image) clearly doesn't allow this dark shadow to pass through since it contains a very low red channel number.

Also using both gradient and saturation can be seen as necessary since there are some images where the saturation threshold is contributing almost all lane line information and there are other images where the majority of the lane line information is caputred by the gradient.

![Combined](/screenshots/combined.png)

## Estimate Lane Line Pixels

The find_lane_lines function takes in a undistorted image and outputs an image array where the pixels identified as belonging to the left lane are given a value of 1 and the right lane pixels are given a 2. This is accomplished by running the undistorted image through the perspective transform and the combined color, HLS, and gradient thresholds talked about in the previous section. This resulting binary image contains all pixels in the image that meet the various thresholds.

When there is no previous lane line information feed into the function, a sliding window histogram approach is used to find the horizontal location of where there are the most pixels for a given vertical window. The function is able to divide the image into any number of vertical slices but I found that 4 consistently yielded the best results. Once the horizontal location of the most pixels is found for each section (and one for each lane line) the max value is given a +- 80 pixel margin that within which, all of the pixels passed through the combined threshold are indicated as being part of either the left or right lane line.

The entire process from raw image to histogram with margin to final image with lef tand right lane lines can be seen below.

![Finding Left and Right Lane Lines](/screenshots/lanelines.png)


Once the lane lines have been found, the histogram margins are then output by the function so they can be used in the next iteration of the function. This saves the function from having to recompute the historgram for each sliding window at every frame.

When given this previous lane line information the function will calculate for the next frame which side of the window when using the old line line information (either the 80 pixels to the left or the 80 pixels to the right of the center) contains more pixels. The window will then be adjusted 5 pixels in that direction which will allow the window to follow the lane lines as they move without having to recompute the histogram. This not only increases speed of the calculation but also allow for error rejection as seen in the image below comparing the function with previous frame information vs one with no starting information.

![Starting Info vs No Info](/screenshots/startinginfo.png)

If at any point both the left or right side of the window contain 0 pixels passed from the thresholds (this indicates the window has lost the lane lines, the histogram will be recomputed for that section of the window and only for that lane line so that the lane line can be found again.

This whole process is shown in the videos in the top right hand corner of the final project video.

## Measuring Radius of Curvature

The radius of curvature is measured by fitting a 2nd order polynomial to the lane line pixels identified using the find lane lines function. This polynomial is converted from the unit of pixels to meters by estimating the lane width and the length of a lane line on the perspective transformed image. Based on the perspective transform and knowning the average width of a US lane and the average lenght of a line line, it is estimated that in the y direction there are 3 meters for every 100 pixels and in the x direction there is 3.7 meters covered for every 800 pixels. The radius of curvature is estimated for the midpoint between the front of the car and the maximum distance the car is looking ahead.

The center point of the car was determined to be the center point of the image based on measuring locations of distinct symmetrical markings on the hood.

## Video Generation

The generation of video just uses all the previously talked about functions that were meant to operate on still images and puts them in a function called process_image which takes video information one frame at a time. A class called Lines() is used to carry information over from one frame to the next. The polynomails which draw the estimated lane line position on the road, the radius of curvature, and the distance from center, are all put through a 5 frame moving average filter before being displayed on the video to help reduce the impact of noise.

The window information from find lane lines function is also continously fed forward so that every frame calculation has starting point information.

The final project video can be seen in the repository under the name "total-pipeline.mp4".

## Future Improvement

One area of future improvement that could be implemented to allow for better detection of the lane lines would be having the distance the moving window in the find lane lines function moves be a function of the pixel count between the left and right side of the window. There is one point in the project video where the top moving window (farthest from the car) in the left lane is behind the lane by about a second due to a rapid change in the position of the lane lines. This would better help the sliding window keep up with the lane lines while they move.

