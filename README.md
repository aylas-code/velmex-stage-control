# velmex-stage-control
A MATLAB class to control the Velmex motorized stages using serial communications

The serialport_VMX class is meant to abstract away the serial communications protocol used to control the Velmex linear and rotary motorized stages.
Arranging the commands and structures into a single class has two advantages: 1) Code is reused efficiently and 2) Only one object can have an open serial port to the motor controller which is consolidated in this class.
A consequence of the design of this class is that all stages *must* be controlled through a single instance of this class. Attempting to create multiple instances through a *single serial port* will fail as only one instance is permitted to open a serial port.
Multiple stage controllers can be controlled at once by creating multiple classes of different names, e.g. `serialport_VMX_1`, `serialport_VMX_2` ... etc.

The current commands are limited to setting the motor speed and moving the stages at a constant velocity. For additional commands and alternate programming options refer to the user manual included in this repository.

> WARNINGS: Does not support MATLAB < R2008b; Partially supports MATLAB R2008b - R2013a; Fully supports MATLAB > R2013b. To use with MATLAB R2008b - R2013a comment out the first line after "handle" and before the ampersand. You will not be able to use the custom display but the rest of the code will work perfectly well.

## Class contents

Examples in this readme will use the following syntax:  
`stage = serialport_VMX(11,1000,1,2,3)`  
where `stage` is an instance of the `serialport_VMX` class.

### Properties
The public properties of the class are 

* `vmx`
* `port`
* `motorResponse`
* `x`
* `y`
* `z`
* `theta`

`vmx` holds the serial communication to the motor controller.
`port` is the COM port used to connect to the motor controller (on Windows use device manager to locate the correct port). 
`motorResponse` is the last byte received from the motor controller.
The motor response can generally be ignored and is meant for debugging if the stages become unresponsive. After a successful move, the response will be a caret '^'.

`x`, `y`, `z`, and `theta` hold structures that define the status of various stages. The contents of these structures are described in the following subsection. Not all axes are required to be invoked or populated to use this class. (In fact, we only have three stages in total, two linear and one rotary). However, all stages that will be used must be declared during the construction of the class ([see Public Methods](#public-methods))

There are two private properties, `baud` and `motorsPresent`.
The baud is set to a default speed of 9600 as the communication speed is rarely a bottleneck in operation.
With Velmex' VXC Controller, a baud rate of 57600 is default. With this change made, the class works as it does with a VMX unit. 
In the unlikely event that higher speeds are needed, refer to page 13 of the users manual.
`motorsPresent` is used internally and should not be modified.

### Structure of stage variables
The motorized stages are defined using a MATLAB structure object. All of the stages use the following generic structure:

* `axis`
* `motorNum`
* `speed`
* `units`
* `unitsPerStep`
* `displacementSteps`
* `displacementUnits`

Many of the parameters are set during class construction (`axis`,`motorNum`,`speed`,`units`,`unitsPerStep`) while the displacement parameters are updated after each stage movement.
The `axis` and `motorNum` parameters should not be modified after invoking the class.
If a mistake is made during the call to VMX, delete the instance and start again.

The default value for `speed` is 1000 steps per second (2.5 revolutions per second).
Simply changing the value for the speed parameter is insufficient to modify the speed of the motor. First, the speed parameter for a given axis must be changed followed by a call to the `setSpeed` method.

The default value for `units` and `unitsPerStep` are 'mm'/0.00635 and 'degrees'/0.0125 for the linear and rotary stages respectively.
It is possible to change the units from millimeters to inches and from degrees to radians. Please use the `toggleUnits` method to ensure proper conversion and compatibility.

Displacement values give only relative information, they do not encode the absolute position of the stage! Every time an instance of the class is created the displacement parameters are reset to 0. This mirrors the behavior of the physical system, as the stepper motors do not retain a memory of their previous position. 

After maneuvering the stages into their starting position, the displacements may be reset manually or by invoking the `setCurrentPositionAsHome` method.


### Public Methods
The Public methods are:

* `obj = serialport_VMX(portNum,motorSpeed,xNum,yNum,thetaNum,zNum)`
* `setSpeed(obj)`
* `moveMotorRelative(obj,motorToMove,moveAmount)`
* `moveMotorAbsolute(obj,motorToMove,position)`
* `setCurrentPositionAsHome(obj,axisToZero)`
* `toggleUnits(obj,axisToToggle)`
* `delete(obj)`

`obj = serialport_VMX(portNum,motorSpeed,xNum,yNum,thetaNum,zNum)` is the class constructor and is crucial to using this class effectively. Not all variables are required. If access is needed to variables following an unnecessary parameter pass an empty variable `[]`.  
`portNum` is the COM port used to communicate with the motor controller. This value is REQUIRED.  
`motorSpeed` is the number of steps per second the motor will turn. The default value is 1000 and is set for _all_ of the motors that are initialized during construction. This value is OPTIONAL.  
`xNum`, `yNum`, `thetaNum`, and `zNum` are the motor numbers for each stage as the are connected to the motor controller. This is NOT the desired numbering system and care must be taken when supplying these values. E.g. stage 1 is the stage connected using the plugs with a 1 written on the side of the connectors. If an axis is not used pass an empty variable `[]`. You may also omit trailing arguments if the axis will not be used. This value is OPTIONAL _but_ all axis must be declared during construction if they are to be used.

`setSpeed(obj)` is a method to set the speed of each motor connected to the motor controller. To vary the speed first modify the speed parameter of the axis and then invoke `setSpeed`. Note: the speed to all axes are updated when calling this method, but the speeds need not be the same. See [Example Usage](#example-usage) for more details.

`moveMotorRelative(obj,motorToMove,moveAmount)` will move the specified stage a distance relative to the current position. `motorToMove` may either be the axis name (e.g. 'y') or the structure for the given axis (e.g. stage.x). The movement amount is in the units of the specified axis. The units can be determined by inspecting the axis structure (e.g. stage.theta.units will return 'degrees' by default). Positive values move away from the motor (linear stage)/counter-clockwise (rotary stage) while negative values produce the opposite motions.

`moveMotorAbsolute(obj,motorToMove,position)` will move the specified stage to a position relative to the "home" position for the axis. The method follows the conventions of the relative move method above.

`setCurrentPositionAsHome(obj,axisToZero)` will reset the current displacement values for the specified axis. As with the movement methods, the `axisToZero` may either be the axis name or the structure to the axis.

`toggleUnits(obj,axisToToggle)` will toggle the units between millimeters and inches for linear stages and degrees and radians for rotary stages. As with the movement methods, the `axisToToggle` may either be the axis name or the structure to the axis.

`delete(obj)` will safely close the serial port and return the motor controller to local mode so that the stages may be manipulated using the front panel of the motor controller.


### Private Methods
The private methods are:

* `connectMotor(obj)`
* `moveMotor(obj)`

`connectMotor(obj)` is called by the `VMX` constructor method during class creation. This method configures communication protocols with the motor controller and sets the timeout value to 30 seconds. If a longer timeout is required (low motor speed + large travel distances) increase the timeout value accordingly.

`moveMotor(obj)` is the backbone of the motion commands. It formats the requested motion into the correct format for transmission to the motor controller.

### Protected Methods
The protected methods are used to format the output of the information displayed in the command window of MATLAB. The information is displayed when the instance is called without a semicolon or when explicitly requested using `disp`. The custom information ensures all of the pertinent information is made available at a glance without having to remember to call a particular method, everything is handled automatically.

* `displayScalarObject(obj)`
* `propgrp = getPropertyGroups(obj)`

`displayScalarObject(obj)` formats the header information for the display, calls getPropertyGroups, and displays the custom information.

`propgrp = getPropertyGroups(obj)` formats the information for each axis being controlled by the current class instance. 

### Static (and private) Methods

`motor = createEmptyStruct(movementAxis,motorNumber)` is a static method used to instantiate the structure used to store parameters for each axis being controlled. This method ensures uniform structure field names and automatically provides the units and conversion factors for linear and rotary stages based on the axis name (`x`,`y`, and `z` are linear while `theta` is rotary). The method is called by the `serialport_VMX` constructor during initialization.


## Example usage

Examples are provided to illustrate the usage of the `serialport_VMX` class. The first case involves using a linear stage in the x-axis and a rotary stage. The second case involves a y-z-theta configuration. The use cases are hypothetical and movement patterns are kept simple.

In both cases, the motor controller will connect using COM port 11.

### Case 1 X-theta
In this case, the linear stage will be moved into a starting position and reset. The rotary stage will be toggled into radians and rotate to between [pi/8,-pi/8] at two locations along the x-axis. After visual inspection, it is determined that the x-stage is connected with the number 2 cable and the rotary stage is connected with the number 1 cable.

```matlab
% position stages
stage = serialport_VMX(11,1000,2,[],1); % linear stage is connected using cable 2, rotary with cable 1
stage.moveMotorRelative('x',12); % move linear stage 12 mm away from the motor
stage.setCurrentPositionAsHome('x'); % the x axis is in the correct starting position, reset to zero
stage.toggleUnits('theta'); % toggle the rotation units to radians

% move stages
stage.moveMotorRelative('theta',pi/8); % starting from 0 move to pi/8
stage.moveMotorRelative('theta',-pi/4); % starting from pi/8 move pi/4 radians to -pi/8 using a relative move

stage.moveMotorRelative('x',10); % move the stage forward 10 mm
stage.moveMotorAbsolute('theta',pi/8); % move using absolute coordinates
stage.moveMotorAbsolute('theta',-pi/8);

stage.moveMotorAbsolute('x',0); % reset the stages to their starting position
stage.moveMotorAbsolute('theta',0);
delete(stage); clear stage; % delete the VMX instance to return control to the front panel of the motor controller
```

Notice that absolute moves are often easier to work with than relative moves (compare lines 9 and 13), though both are available. Also notice that the axes specified as arguments to `moveMotorRelative` etc. could be given as
```matlab
stage.setCurrentPositionAsHome(stage.x);
stage.toggleUnits(stage.theta);
stage.moveMotorRelative(stage.theta,pi/8);
stage.moveMotorAbsolute(stage.x,0);
```
without passing the name of the axis as a string. 


### Case 2 Y-Z-theta
In this case, the linear stages will be used in inches. The rotary stage will have to initially rotate 60 degrees. The speed of the linear stages will be set to 500 steps per second. The stages will complete a square rotating 90 degrees at each corner. After visual inspection, it is determined that the y, z, and theta stages are connected with the 1, 2, and 3 cables respectively.

```matlab
stage = serialport_VMX(11,1000,[],1,3,2); % order is port, speed for all motors, x number, y number, theta number, and z number
stage.toggleUnits('y'); % toggle to inches
stage.toggleUnits('z'); % toggle to inches
stage.moveMotorRelative('theta',60);

% set the speed of the linear stages
stage.y.speed = 500;
stage.z.speed = 500; 
stage.setSpeed(); % only AFTER calling setSpeed will the speeds be updated. Note stage.theta.speed is still equal to 1000

% move the stages
stage.moveMotorAbsolute('y',4);
stage.moveMotorRelative(stage.theta,90); % could replace stage.theta with 'theta'

stage.moveMotorAbsolute('z',4);
stage.moveMotorRelative(stage.theta,90);

stage.moveMotorAbsolute('y',0);
stage.moveMotorRelative(stage.theta,90);

stage.moveMotorAbsolute('z',0);
stage.moveMotorRelative(stage.theta,90);

delete(stage); clear stage;
```
