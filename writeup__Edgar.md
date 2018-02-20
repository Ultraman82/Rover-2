## Project: Search and Sample Return


### Notebook Analysis

 1. Import rover camera images with the matplotlib.image library.
  Images are saved in an array with each pixel's X and Y postition, and it's RGB (3 channel) value. 

 2. Import an example grid and the rock sample images.

 3. To convert the rover's view to a 'top-down' map image, use cv2.getPerspectiveTransform, and cv2.warpPerspective which perform a geometric calculation using 'perspect_transform()'. Then, input the coordinates of the grid vertices manually.
 To avoid errors of perception, I used a mask which shows only the range of the camera's view, not the blank area from the warped images.

![image](output/warped_example.jpg)
 
 
 4. To define the navigable terrain, I used the color threshold method. After creating an empty array the same size of the original image, I applied a brightness threshold of 160 on each RGB channel. With that, we can make a binary image of the navigable terrain. And as for the rock sample, I used the specific RGB boundries of 110, 110, and 50.
 These are used in 'color_thresh()' and 'find_rocks()'.
 
 
 ![image](output/warped_threshed.jpg)
 
 
 ![image](output/rocks_threshed.png)
  

#### 1. Populate the `process_image()` function with the appropriate analytical steps to map the pixels identifying navigable terrain, obstacles, and rock samples into a worldmap.  Run `process_image()` on your test data using the `moviepy` function(s?) provided to create a video output of your result. 

Describe in your writeup how you modified the process_image() to demonstrate your analysis and how you created a worldmap. Include your video output with your submission.


1. I needed to change the images from the rover to 'top-down' map images. To do this, I used the cv2.getPerspectiveTransform, and cv2.warpPerspective modules to calculate the images with calibration for  'perspect_transform()'.

  
2. With the warped images and color threshold images, I marked the navigable terrain on the map using
threshed = color_thresh(warped)
  
3. On the worldmap, I gave a binary signal 'on' to all the rover's view on the red channel, and overlayed the blue channel on the navigable terrain.
  
    data.worldmap[y_world, x_world, 2] = 255
    data.worldmap[obs_y_world, obs_x_world, 0] = 255
    
4. I converted the map image pixel values to rover-centric coordinates.    
    xpix, ypix = rover_coords(threshed)
    
5. Then I converted the rover-centric pixel values to world coordinates using def pix_to_world.

6. The next step was to change rover-centric pixel positions to the polar coordinates.
    dist, angles = to_polar_coords(xpix, ypix)

7. If the rover finds a rock, it marks it on the map with the 'find_rock()' function and this code:
     rock_map = find_rocks(warped, levels = (110,110,50))
     if rock_map.any():
 
 ![image](output/processed.png)

- Notebook Analysis Output Video (https://github.com/Ultraman82/Rover-2/tree/master/output/test_mapping.mp4)

### Autonomous Navigation and Mapping

#### 1. Fill in the `perception_step()` (at the bottom of the `perception.py` script) and `decision_step()` (in `decision.py`) functions in the autonomous mapping scripts and an explanation is provided in the writeup of how and why these functions were modified as they were.
perception_step() and decision_step() functions have been filled in and their functionality explained in the writeup.

## Perception_step()

 - Apply perspective transform 
    warped,mask = perspect_transform(Rover.img, source, destination)
 
 - Apply color threshold to identify navigable terrain/obstacles
    threshed = color_thresh(warped)
    obs_map = np.absolute(np.float32(threshed) - 1)  *  mask
    
 - Update Rover.vision_image (this will be displayed on left side of screen)
    Rover.vision_image[:,:,2] = threshed * 255
    Rover.vision_image[:,:,0] = obs_map * 255
    
 - Convert map image pixel values to rover-centric coords    
    xpix, ypix = rover_coords(threshed)
    
 - Convert rover-centric pixel values to world coordinates
    world_size = Rover.worldmap.shape[0]
    scale = 2 * dst_size
    xpos = Rover.pos[0]
    ypos = Rover.pos[1]
    yaw = Rover.yaw
    x_world, y_world = pix_to_world(xpix, ypix, xpos, ypos,
                                yaw, world_size, scale)
    obsxpix, obsypix = rover_coords(obs_map)
    obs_x_world, obs_y_world = pix_to_world(obsxpix, obsypix, xpos, ypos,
                                           yaw, world_size, scale)
                                           
 - Update Rover worldmap (to be displayed on right side of screen)         
    Rover.worldmap[y_world, x_world, 2] += 10
    Rover.worldmap[obs_y_world, obs_x_world, 0] += 1
    
 - Convert rover-centric pixel positions to polar coordinates.   Update Rover pixel distances and angles
    dist, angles = to_polar_coords(xpix, ypix)
    Rover.nav_angles = angles
    
 - Dectect rocks and mark the location on the map.
    rock_map = find_rocks(warped, levels = (110,110,50))
    if rock_map.any():
      rock_x, rock_y = rover_coords(rock_map)
      rock_x_world, rock_y_world = pix_to_world(rock_x, rock_y, xpos, ypos,
                                                yaw, world_size, scale)
      rock_dist, rock_ang = to_polar_coords(rock_x, rock_y)
      rock_idx = np.argmin(rock_dist)
      rock_xcen = rock_x_world[rock_idx]
      rock_ycen = rock_y_world[rock_idx]
      
      Rover.worldmap[rock_ycen, rock_xcen, 1] = 255
      Rover.vision_image[:,:,1] = rock_map = 255
      
## Decision Step()

 First, it checks if there is data from the vision.
 
  Yes = It checks the rover is forward mode
  
      Yes = It checks there is enough navigable terrain
      
          Yes = It checks if the speed is max
          
              Yes = Keep moving(throttle = 0) within +_ 15 angle adjustment
              
              No  = Set throttle value to throttle setting.
              
          No = Stop
          
      No = It checks the velocity of the rover is faster than 0.2
      
          Yes = Keep trying to stop
          
          No = It checks if there's a path forward
          
              Yes = Start to go forward
              
              No = Turn around
              
#### 2. By launching in autonomous mode, the rover can navigate and map autonomously. Explain your results and how you might improve them in your writeup. 

- There are some cases in which the rover gets stuck but it isn't detected by our pipeline, I would investigate these cases and check how to detect them.

- In the case of the rover getting stuck but being unable to get out using a turning method, I will try to make a new mode.

- If the rover wanders around without considering if it has already gone down that path before, I need to make the rover consider whether the path has already been taken.

- Sometimes the rover loops around a large area several times before going to a new path, I would try to avoid this scenario.

- This course was my very first course in Robotics. Not just Robotics, but also in an English based course that requires writing.
 So, lots of things were quite confusing, where I could almost do nothing but follow the course work to pass through the videos. Even with those simple tasks, I faced many obstacles to make the code have the same result as the video. I put tons of time but the deadline is near, so it was inevitable I submit the writeup like this. But still, it was quite a fun process and I hope to get better on the next project and develop this project to pick up the rocks.

![video](output/out.ogv)

