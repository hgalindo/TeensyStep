# TeensyStep
## Problem to be solved 
A lot of interesting projects require the movement of things. An easy way to implement such a transport system is the use of stepper motors driven by readily available stepper drivers. Those drivers usally expect simple **step and direction signals** to move the motors. However, due to the motor intertia they can not be started at full speed but need a smooth **acceleration** or deceleration to the final velocity. Otherwise the step counters maintained by the application and the real motor positions can run out of sync (steploss errors). Practically all modern stepper drivers can operate in a so called **microstepping** mode where the mechanically fixed step positions (usually 200 per rev) are subdivided into a number of microsteps. While microstepping is great to reduce vibration at low velocity it requires  **high pulse rates** at even moderate motor speeds. 
Lets look at a typical example: Assume we have a standard 1.8° stepper (200 steps/rev) and  want to  run it at a speed of 1200 rpm in 1/16 micro stepping mode. The required pulse rate *r*  can be calculated by: 

&emsp;&emsp;&emsp; *r* = (16 * 200 stp/rev * 1'200 rev/min) / (60 s/min) = **64'000 steps/s**

Of course there are other Arduino stepper libraries available, e.g. the well known [AccelStepper](http://www.airspayce.com/mikem/arduino/AccelStepper/),&ensp; the standard [Arduino Stepper Library](https://www.arduino.cc/en/Reference/Stepper) or [Laurentiu Badeas Stepper Driver](https://github.com/laurb9/StepperDriver) to name a few. However, I found none which is able to handle accelerated synchronous and independent moves of more than one motor at the high pulse rates required for microstepping drivers. 

## Purpose of the Library
**TeensyDelay** is an efficient Arduino library compatible with Teensy 3.0, 3.1, 3.2, 3.5 and 3.6. The library is able to handle synchronous and independent movement of steppers with pulse rates of up to 300'000 steps per second. The following table shows a summary of the **TeensyDelay** specification:

| Description                                | Specification             |
|:-------------------------------------------|--------------------------:|
| Motor speed / pulse rate                   | 1 - 300'000 stp/s         |
| Acceleration                               | 0 - 500'000 stp/s^2       |
| Pull in speed (speed up to which no acceleration is required)| 50-10'000 stp/s|
| Synchronous movement of motors             | up to 10                  |
| Independend movement of motors             | 4 groups of 1 to 10 motors|
| Settable step pulse polarity               | Active high or low        |
| Settable step pulse width                  | 1-100µs                   |
| Settable direction signal polarity         | cw / ccw                  |

## Usage
TeensyStep basically uses the following two classes to control the steppers

### Stepper Class
The *Stepper* class encapsulates the physical properties of a stepper motor and its driver:
- Teensy pin numbers the STEP and DIR signals are connected to.
- Maximium speed and pull-in speed of the motor (pull-in speed is the speed up to which the motor can be started without the need for acceleration).
- Step pulse polarity, i.e., if your driver requires active high or active low step pulses.
- Setting for inverted rotation direction

Here a short snippet showing the usage of the stepper class
```c++
...
// define a few motors
Stepper motor_1(2,3); // STEP at pin 2, DIR at pin 3
Stepper motor_2(4,5); // STEP at pin 4, DIR at pin 5
Stepper motor_3(6,7); // STEP at pin 6, DIR at pin 7
...
void setup() {
    // Setup some motor properties
    motor_1.setMaxSpeed(64000);         // stp/s
    motor_1.setAcceleration(200000);    // stp/s^2
    
    // you can also use a 'fluent interface' to set the properties
    motor_2
        .setMaxSpeed(30000)             // stp/s
        .setPullInSpeed(800)            // stp/s
        .setAcceleration(150000);       // stp/s^2
    motor_3
        .setStepPulsePolarity(LOW)      //motor connected to an active low driver
        .setInvertedDir(true)
        .setMaxSpeed(45000)             // stp/s
        .setAcceleration(200000);       // stp/s^2      .
        ...
}
```
The stepper class also keeps track of the current position and is used to set a new target position. Target positions can be set absolutely or relative to the current position. 
```c++
...
uint pos = motor_1.getPosition();   // get current position of the motor
motor_1.setTargetAbs(pos + 100);    // sets the target position 100 steps ahead of the current position
motor_1.setTargetRel(100);          // does the same thing directly
...
```
### StepControl Class
The *StepControl* class is responsible for actually moving the motors
```c++
...
StepControl<> controller;                    // construct a controller object

controller.move(motor_1);                    // This will move motor_1 to its target position
controller.move(motor_2, motor_3);           // Moves motor_2 and motor_3 to their target positions

motor_1.setTargetAbs(0);
motor_2.setTargetAbs(0);
motor_3.setTargetAbs(0);
controller.move(motor_1, motor_2, motor_3);  // Move all motors back to the starting position
...
```
The *move* command is blocking. I.e., it will not return until the movement is finished. If you need to do some work while the motors are moving you can use the non blocking *moveAsync* command.

```c++
...
controller.moveAsync(motor_1, motor_2);     // This will start the movement and return immediately
doSomeWork();

while(controller.isRunning()){              // wait until the movement is finished
    delay(10);                     
}
```


## Movement Modes

If you want to control more than one motor there are three different posibilities to do so. Lets assume the standard belt driven x/y transport shown in the figures below. There are two motors (M1 and M2) which move a load (symbolized by the blue ring) on linear guides. M1 moves the load in y-direction and M2 in x-direction. 

## Sequential Movement
The simplest thing you can do is to move the motors one after the other:
```c++
...
StepControl<> controller;               

M1.setTargetAbs(targetY);
M2.setTargetAbs(targetX);

controller.move(M1);           // Move M1 to the target position (targetY)
controller.move(M2);           // Move M2_to the target position (targetX) 
...
```
The resulting movement is shown in the figure below.


![Sequential Movement](/media/seqMove.png?raw=true "Sequential Movement")


## Synchronous Movement
If you want to move on a straight line between the two positions StepControl needs to adjust the step rate of one of the motors depending on the other. Of course, this adjustment is necessary during the complete movement including acceleration and deceleration phases. StepControl uses [Bresenham's line algorithm](https://en.wikipedia.org/wiki/Bresenham%27s_line_algorithm) to do this adjustment.

Using synchronous movements is easy: the controller accepts up to 10 motors in its move and moveAsync commands. All motors passsed to the commands will be moved synchronously.

```c++
...
StepControl<> controller;               

M1.setTargetAbs(targetY);
M2.setTargetAbs(targetX);

controller.move(M1, M1);        // Move M1 _and_ M2 to the target position 
...
```
The resulting movement is shown in the figure below.


![Synchronous Movement](/media/sycMove.png?raw=true "Sequential Movement")


## Independend Movement
Sometimes it is necessary to move one or more motors independently but at the same time. Lets extend our example by another transport module (motor M3) which can move left / right only. This transport is supposed to bring fresh samples (red dot) to a transfer area on the right of the x/y transport. Independently of the the x/y transport which is still  moving around we want to bring a new sample to the transfer station. We can do this by creating a second controller:
```c++
StepControl<> controller_1;
StepControl<> controller_2;
...
controller_1.moveAsync(M1, M2);   // synchronous movement of M1 and M2
// ... more controller_1 movements....

M3.setTargetAbs(transferAreaPos);
controller_2.moveAsync(motor_3);  // move M3 to the transfer station, independent of M1/M2 movements which run in parallel
...
while(controller_1.isRunning() || controller_2.isRunning()){  // wait until both controllers finished their movements
    delay(10);
}
// do the sample transfer...
...
```

![Independend Movement](/media/indMove.png?raw=true "Sequential Movement")


## Used Ressources
### StepControl
Each *StepControl* object requires **one PIT Timer** (aka IntervallTimer) and **two channels of a FTM timer**. Since all supported Teensies implement four PIT timers and a FTM0 module which has 8 timer channels the usage is limited to 4 *StepControl* objects existing at the same time. In case you use **TeensyDelay** together with other libraries reserving one ore more IntervallTimer the number of available *StepControl* objects is reduced accordingly. You can use the *isOK()* member to check if the *Controller* object was able to reserve a PIT timer. 



```c++
StepControl<> cnt1;
StepControl<> cnt2;
StepControl<> cnt3;

if(!(cnt1.isOK() && cnt2.isOK() && cnt3.isOK()))
{
    panic();
}
```
A *StepControl* object requires 96 bytes of stack or static memory.
### Stepper
You can define as many *Stepper* objects as you like. *Stepper* objects do not use any system ressources except 56 bytes of stack or static memory.
## Advanced Usage
TBD

## Performance
The table in the figure below shows some information about the performance of the library. To  estimate the generated processor load we need to  know the time it takes for handling acceleration, Bresenham algorithm and switching the STEP and DIR signals. The experiment was done by setting a digital pin to HIGH when the processor enters the corresponding ISR and back to LOW when it leaves it. The actual times were measured with a logic analyzer, the processor load was calculated for various conditions and processors. (The .xlsx spreadsheet can be downloaded  [here](/media/load_calculation.xlsx)). 

![Alt text](/media/load_calculation.PNG?raw=true "Load calculation")

**Summarizing:**
Moving one motor at 100kHz generates a processor load of about 10% for a T3.6 and some 28% for a T3.2. Handling 3 motors at that speed generates a load of 13% and 34% respectively. The results roughly fit to the difference in the clock frequency of both boards. Numbers in the table are linear in motor speed, i.e. half the step rate generates half of the load. 
