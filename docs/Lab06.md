# Lab 06 – ESP-NOW Applied Exercises with Peripherals
## Activity Goals
-Implement data structures (structs) to format and send complex payloads over the wireless link.
-Establish reliable peer-to-peer communication between ESP32 boards using ESP-NOW.

## Materials
- 6 ESP32 Development Board
- 1x Push Button
- 1x Potentiometer
- 1x LED (Internal or External)
- Resistors and Jumper wires
-1x LED and current-limiting resistor
-1 sensor ultrasonic 
-1 servo motor
-1 buzzer 


---

## Exercise 1 -  One-to-one: wireless button to LED
* Goal: When the student presses a button on Board A, Board B turns an LED on. When the button is released, the LED turns off.


### Code 6.1
```c
// Pega aquí el código correspondiente a la tarea 6.1
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

// Ejemplo de struct
typedef struct {
    int sensor_id;
    float value;
} SensorData_t;

// ... resto de tu código ...
```
[LAB 2.4](https://youtu.be/hpEJd5n-EJY)


## Exercise 2 -  One-to-one: potentiometer-controlled dimmer
* Goal: Board A reads an analog potentiometer and sends the value to Board B. Board B adjusts the brightness of an LED using PWM.



### Code 6.1
```c
// Pega aquí el código correspondiente a la tarea 6.1
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

// Ejemplo de struct
typedef struct {
    int sensor_id;
    float value;
} SensorData_t;

// ... resto de tu código ...
```
[exercise 2](https://youtu.be/hpEJd5n-EJY)

## Exercise 3 -  Two-way: ping and acknowledgment with buzzer
* Goal: When the user presses a button on Board A, it sends a command to Board B. Board B activates a buzzer briefly and returns an ACK. Board A lights a status LED only if the ACK arrives before timeout.

### Code 3
```c
// Pega aquí el código correspondiente a la tarea 6.1
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

// Ejemplo de struct
typedef struct {
    int sensor_id;
    float value;
} SensorData_t;

// ... resto de tu código ...
```
[exercise 3](https://youtu.be/hpEJd5n-EJY)


## Exercise 4 -  One-to-many: one controller, three actuators
* Goal: One master board sends commands to three different boards. Each button triggers a different remote actuator.
### Code 4
```c
// Pega aquí el código correspondiente a la tarea 6.1
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

// Ejemplo de struct
typedef struct {
    int sensor_id;
    float value;
} SensorData_t;

// ... resto de tu código ...
```
[exercise 4](https://youtu.be/hpEJd5n-EJY)



## Exercise 5 - One-to-many with targeted NeoPixel control from terminal
* Goal: 
-The sender board reads commands from the serial terminal and sends them to a selected ESP node so it changes one pixel on its NeoPixel ring.
### Code 5
```c
// Pega aquí el código correspondiente a la tarea 6.1
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

// Ejemplo de struct
typedef struct {
    int sensor_id;
    float value;
} SensorData_t;

// ... resto de tu código ...
```
[exercise 5](https://youtu.be/hpEJd5n-EJY)


## Exercise 6 - Many-to-one: sensor hub printed in terminal
* Goal: 
Three sensor nodes send measurements to one gateway. The gateway prints the latest data from all nodes in the terminal.

### Code 6
```c
// Pega aquí el código correspondiente a la tarea 6.1
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

// Ejemplo de struct
typedef struct {
    int sensor_id;
    float value;
} SensorData_t;

// ... resto de tu código ...
```
[exercise 6](https://youtu.be/hpEJd5n-EJY)


## Exercise 7 - Everyone-to-everyone: distributed quiz buzzer
* Goal: 
Three sensor nodes send measurements to one gateway. The gateway prints the latest data from all nodes in the terminal.

### Code 7
```c
// Pega aquí el código correspondiente a la tarea 6.1
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

// Ejemplo de struct
typedef struct {
    int sensor_id;
    float value;
} SensorData_t;

// ... resto de tu código ...
```
[exercise 7](https://youtu.be/hpEJd5n-EJY)



## Exercise 8 - Group Assignment: range-extended chat system 
* Goal: 
Each board can send and receive messages to/from every other board. The user types a message in the terminal of any board and it is broadcast to all other boards, which print it in their terminals. Boards that are out of direct range of the sender can still receive the message because intermediate boards repeat it, extending the network's reach.

### Code 8
```c
// Pega aquí el código correspondiente a la tarea 6.1
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

// Ejemplo de struct
typedef struct {
    int sensor_id;
    float value;
} SensorData_t;

// ... resto de tu código ...
```
[exercise 8](https://youtu.be/hpEJd5n-EJY)


