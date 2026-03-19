RoboRacer Autonomous Racing is a semi-regular competition organized by an international community of researchers, engineers, and autonomous systems enthusiasts. The teams participating in the 27th RoboRacer Autonomous Racing Competition at ICRA 2026 will write software for a 1:10 scaled autonomous racecar to fulfill the objectives of the competition: drive fast but don’t crash!


These vehicles are simulated as a combination of rigid body and sprung mass representations with adequate attention to rigid body dynamics, suspension dynamics, actuator dynamics, and tire dynamics. Additionally, the simulator detects mesh-mesh interference and computes contact forces, frictional forces, momentum transfer, as well as linear and angular drag acting on the vehicle. Finally, being an autonomy-oriented digital twin, the simulator offers physically-based sensor simulation for proprioceptive as well as exteroceptive sensors on-board the virtual vehicle.


Shape of the track:
Size and Structure
- The simulated racetrack will be designed in accordance with the physical RoboRacer racetracks, using similar materials and spanning similar dimensions.
    
- The entire racetrack will be designed to fit within an area of around 3010 m2, much like the physical RoboRacer racetracks.
    
- The racetrack border will be constructed from air ducts of about 33 cm in diameter, making sure that it is perceivable to exteroceptive sensors.
    
- The racetrack will be at least 3 car widths (90 cm) wide throughout to allow safe vehicle traversal, while giving an opportunity to optimize the raceline.
    

Design and Features
- The road surface will be simulated with properties of polished concrete, which is flat and reflective. Therefore, exteroceptive perception may become challenging at times, much like in the real world.
    
- There may be a gap(s) between the ducts through which the LiDAR beams can pass, making it appear as an obstacle-free space. Therefore, motion planning may become challenging at times, much like in the real world.
    
- The racetrack may consist of variabilities such as straight stretch, chicane, bifurcation, obstacles, etc. to make the course challenging.