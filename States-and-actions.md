Hi @Boilevin,

I have spend the last week going through your code and I am really impressed on the motor control with odometry.
I do not fully grasp the inner workings of the PID controller and some of the calculations you make, but that will come eventually 😉 

I have seen that it has become a really complex state-machine, because of the old Azurit code remains and that you have created a new state for every operation/target. In my opinion this makes everything over-complex and hard to improve.

I am proposing a new/revised way of coding the Ardumower, that I want to contribute to your current code. 

I would like to know your point of view on this method and would really like to accomplish this together, instead of creating my own standalone fork of your code. So please reply here and discuss.


# Change the way the states are used
My thoughts would be that we use the following items:
- **A machine state** defines/shows the robot action it is performing. 
  - This would be, driving forward, reversing, turning, charging, etc
  - These simpeler robot actions should be no more than the robot can do  and would re-used on a lot of statuses/tasks
- **A status** would be the job or main action the robot is performing
  - This would be; back to station, mowing, charging, etc
- **A task** would be a task within the status or job that the robot needs to perform to accomplish the job (or status)
  - This would be: turning at the end of a lane or on the perimeter, or avoid an obstacle
  - A task would be a combination of several states, like reverse when hit an obstacle and drive a circle around is to avoid it.

In the code it would be wise (to my opinion) to honour the following rules:
- every check on sensors would only return a variable about their status
  - and not change the state directly
- in the loop section of the state these variables would be evalutated and change a task according according to their inputs
- in the setState section for the new state, the output/motor settings would be applied/calculated
  - and preferably nothing more. 

# My recommendations on the state improvements would be the following

## States (these would be the actual action the robot is doing; the outputs)

- OFF
  - Do nothing, except monitor battery
  - Set all motors to off
- DRIVE_FORWARD
  - Keep a straight heading, using the IMU and odometry
  - motorControlOdo();
  - Sonar distance ≤ sonarTriggerBelow would cause slower speed
  - near perimeter would cause slower speed if set in config
- STOP_HARD
  - Need to brute stop
  - Triggers:
    - Bumper
    - Sonar (at distance ≤ sonarToFrontDist*1.2)
    - Drive Motor Current ≥ motorPowerMax*.8
  - Next action based on trigger and status/task
- STOP_ROLLOUT
  - Gradually roll out the defined cm untill a stop.
  - Independed of the forward or reverse status
  - Triggers:
    - Perimeter found
    - Sonar (at distance ≤ (sonarToFrontDist+DistPeriOutStop)*1.2)
    - NeedToCalibrateIMU
    - BatteryLow
  - Next action, based on trigger
- ROLL
  - Turn the defined angle, number of degrees
  - If outside perimeter, retry (or revert) the roll
- ROLL_TO_HEADING
  - Turn untill the defined heading has been reached
- DRIVE_BACKWARDS
  - Drive back the defined cm
  - Based on previous state?
- PERI_FIND --> Should not be implemented. This would be FORWARD with task PERI_FIND and status BACK_TO_STATION
  - Drive straight untill the perimeter is detected
  - Direction possibly by GPS location
- PERI_TRACK
  - Track the perimeter using motorControlPerimeter();
- DRIVE_CIRCLE (better ARC)
  - this would be roll while driving forward (maybe backwards also?)
  - reason is to avoid a obstacle.
  - circle has a variable radius
  - End state depends on originating reason (task)
- MANUAL
  - uses motorControl(); with direct rpm settings from PFOD
- REMOTE --> Unable to find code that supports this
  - uses motorControl(); with direct steering/driving settings from RC
- MOTOR_CALIB_SPEED
- MOTOR_TEST

## Statuses (what major job am I doing)

- NORMAL_MOWING
- SPIRAL_MOWING (High Grass circular cutting)
- WIRE_MOWING
- BACK_TO_STATION
- CHARGING
- WAITING
- TESTING
  - MOTOR TEST
  - Calibration from menu
- MANUAL
  - In RC or Manual
- DRIVE_TO_START
  - Leave station and track to start
  - Can include drive to other area

## Task (current task/substatus)

This task would consist of a few predefined STATES.

For example, avoid obstacle would be reverse, roll x degrees, circles around it for x meter, roll back to original heading and resume status.

- DRIVE
  - Go forward and keep the heading
- ESCAPE_SMALL_AREA
  - Used to find an escape route from an small corner or corridor.
  - Go forward/backwards small distance
  - Is triggered by motorCurrent at reverse (could also be back bumper/sonar) or hit perimeter/obstacle really quick after turning
  - Rotate small angle and try to move forward a greater distance than the mower has reversed.
    - repeat if unable to travel a greater distance
- PERI_FIND
  - Heading can be random or set by heading to GPS coördinate.
  - Possibly rotate (ROLL) to find stronger perimeter signal and drive into that direction
- PERI_TRACK
  - Drive along the perimeter until charge detected
    - or Bumper if bumper at station
- PERI_READ
  - Turn small angles (45°) left and right, to find out if we can continue driving with a small (temporary) angle correction/ ARC driving
    - After peri out roll, reverse DistPeriOutStop*.75 and then check
  - Returns PERI location at left, right or center
- TURN
  - Change heading when reached the perimeter or unable to avoid obstacle
  - Angle would depend on mowing pattern
    - LANES: change lane (eventually 180°)
    - RANDOM: angle based on driven distance since last state change (see https://www.facebook.com/groups/319588508137220/permalink/4311102302319134)
      - Longer distance, greater angle. Set for the next 20 times
      - Short distance, smaller angle. Set for the next 20 times
- AVOID_OBSTACLE
  - Try to avoid obstacle and continue course
    - Try to circle around the object and continue
  - If bump on object or perimeter the second time, goto TURN and continue the original heading minus 180° (opposite direction)
- LEAVE_STATION
  - Need support for non-blocking station?
- TRACK_TO_START
  - Find and follow perimeter wire until the specified distance has been travelled
- DRIVE_TO_NEW_AREA
  - Drive into the specified heading for the specified distance
- WAIT_FOR_SIG
  - Stand still and wait until the perimeter signal is detected (greater than minimum smag big area)
  - If found, start PERI_FIND
- CALIBRATE
  - Stop and calibrate the IMU/COMPASS/GPS heading...
  - After success continue
        