# Project-04-Advanced-Lane-Lines


## Calibrating the Camera

One of the calibration images that was a .png was converted to .jpg using an online converter. Also two of the images for some reason had an extra row and column of pixels that seemed to be pure noise added to the top and left side. This was removed when reading in the image.

## Transform into Binary Image

### Color Threshold

#### RGB

Assume all lane lines are either yellow or white. Pure white is RGB = (255, 255, 255) and pure yellow is RGB = (255,255,0). So really a high threshold on both the R AND the G should force the remaining pixels to either be yellow or white.

Thresholding for yellow and white colors only seems to work on dark pavement. On the light pavement colors it doesn't really pick anything up until the very high thresholds which remove other information.

However using a low threshold (like 130) just on the red threshold could be used as an AND with other transforms since it could potentially help on the dark pavement sections and essentially have no effect on the light pavement sections.

#### HLS

Saturation channel seems to clearly be the best but still am not happy with the results on their own. Looking at the last photo in the bottom right (see html notebook) you can see that on the white pavement section the right lane lines hardly shows up at all.

For saturation a minimum threshold of 100 seems to clearly identify the lane lines without removing too much of the lane lines themselves.


### Perspective Transform

The most difficult part of the perspective transform is selecting how far up you should try to look for the lane lines. However the perspective transform really helps in visualizing not only where the horizon is (and thus that above this point there is no longer any lane line data) but also it helps finding the point where there isn't enough data (because the resolution of the camera) to make out small details of the lane lines. By setting the top perspective transform source points far away, you can see how blurry it is and how hard it is to make out the lane lines at points that are just before the horizon. I used this blurryness as a rough guide for how far up the road to look.

Based on the plots seen in the HTML under the heading of perspective transform, it can be seen that looking any higher than pixel 450, the image is so blurry that there is almost no discernable information in it.

### Gradient Transform
