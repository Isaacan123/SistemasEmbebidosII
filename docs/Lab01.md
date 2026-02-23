# Lab 01 – FreeRTOS Task Creation and Scheduling

## Activity Goals
- Understand FreeRTOS task creation and scheduling
- Implement GPIO control for LED blinking on ESP32
- Manage time delays using `vTaskDelay`
- Configure task priorities and stack sizes correctly

## Materials
No materials required

## Code Example

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO GPIO_NUM_2

static const char *TAG = "LAB1";

static void blink_task(void *pvParameters)
{
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
    while (1) {
        gpio_set_level(LED_GPIO, 1);
        vTaskDelay(pdMS_TO_TICKS(300));
        gpio_set_level(LED_GPIO, 0);
        vTaskDelay(pdMS_TO_TICKS(300));
    }
}

static void hello_task(void *pvParameters)
{
    int n = 0;
    while (1) {
        ESP_LOGI(TAG, "hello_task says hi, n=%d", n++);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 1 (two tasks)");
    xTaskCreate(blink_task, "blink_task", 2048, NULL, 5, NULL);
    xTaskCreate(hello_task, "hello_task", 2048, NULL, 2, NULL);
}
```

## Exercises

**Priority Experiment:** Change `hello_task` priority from 5 to 2. Does behavior change? Why?
- Yes. Lower priority numbers execute less frequently, allowing the higher-priority blink task to run more often.

**Starvation Demo:** Remove `vTaskDelay(...)` from `hello_task`. What happens?
- The LED stops blinking because `hello_task` monopolizes CPU time, starving other tasks.
- **Fix:** Blocking tasks with `vTaskDelay` allows the scheduler to handle other tasks fairly.

## Lab 02 

## Code Of exercise LAB02 

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "esp_log.h"

static const char *TAG = "LAB2";
static QueueHandle_t q_numbers;

static void producer_task(void *pvParameters)
{
    int value = 0;
    while (1) {
        value++;
        if (xQueueSend(q_numbers, &value, pdMS_TO_TICKS(50)) == pdPASS) {
            ESP_LOGI(TAG, "Produced %d", value);
        } else {
            ESP_LOGW(TAG, "Queue full, dropped %d", value);
        }
        vTaskDelay(pdMS_TO_TICKS(20));
    }
}

static void consumer_task(void *pvParameters)
{
    int rx = 0;
    while (1) {
        if (xQueueReceive(q_numbers, &rx, pdMS_TO_TICKS(1000)) == pdPASS) {
            ESP_LOGI(TAG, "Consumed %d", rx);
            vTaskDelay(pdMS_TO_TICKS(300));
        } else {
            ESP_LOGW(TAG, "No data in 1s");
        }
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 2 (queue)");
    q_numbers = xQueueCreate(20, sizeof(int));
    if (q_numbers == NULL) {
        ESP_LOGE(TAG, "Queue create failed");
        return;
    }
    xTaskCreate(producer_task, "producer_task", 2048, NULL, 5, NULL);
    xTaskCreate(consumer_task, "consumer_task", 2048, NULL, 5, NULL);
}
```

## Exercises

## 1. Make the producer faster
- **Change:** Producer delay `200ms → 20ms`.
- **Effect:**  
  With this change, the producer generates data every 20 ms, but the consumer does not keep up.  
  - **Observation:** You will see **“Queue full”** when the producer tries to send data faster than the consumer can process.  
  - **Reason:** The consumer is slower than the producer, so the queue fills up quickly.

---

## 2. Increase the queue length
- **Change:** Queue length `5 → 20`.
- **Effect:**  
  - The consumer now has more buffer space to store data.  
  - After filling those 20 slots, the consumer will start losing data again if it cannot process fast enough.  
  - **Conclusion:** Increasing queue length delays data loss but does not solve the imbalance between producer and consumer speeds.

---

## 3. Make the consumer “slow”
- **Change:** After a successful receive, add:  
  ```c
  vTaskDelay(pdMS_TO_TICKS(300));
/

  ## What pattern is happening now (buffering / backlog)?

With this code we can see that the system in the first seconds the producer will load the 20 spaces of the queue.  
The consumer is slower, so it will be processing data item 1 while the producer has already sent items 2, 3, 4, ...  

Therefore, our queue will **always be full**, because the consumer takes 300 ms to process a single piece of data, and in that time the producer tried to send 15 new pieces of data (`300 / 20 = 15`).  

## Lab 03

## Code of exercise LAB03A

```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"

static const char *TAG = "LAB3A";

static volatile int shared_counter = 0;

static void increment_task(void *pvParameters)
{
    const char *name = (const char *)pvParameters;

    while (1) {
        // NOT safe: read-modify-write without protection
        int local = shared_counter;
        local++;
        shared_counter = local;

        if ((shared_counter % 1000) == 0) {
            ESP_LOGI(TAG, "%s sees counter=%d", name, shared_counter);
        }

        vTaskDelay(pdMS_TO_TICKS(1));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 3A (race demo)");

    xTaskCreate(increment_task, "incA", 2048, "TaskA", 5, NULL);
    xTaskCreate(increment_task, "incB", 2048, "TaskB", 5, NULL);
}
```

# Part A: Why can the counter be wrong?

The counter can be wrong because Task A copies the counter value into its local memory, but before finishing the update the system pauses Task A and gives control to Task B, which reads the same value (100), increments it to 101, and stores it; when Task A resumes it still believes the counter is 100, increments it to 101, and overwrites Task B’s update, so although two increments were made the counter only increased by 1 and information was lost due to a race condition.


```c
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#define LED_GPIO GPIO_NUM_2   
#define BUTTON5_GPIO  GPIO_NUM_3
int Read_button5 = 0; // Aquí guardamos la lectura

void read_button_task5(void *pvParameters)
{
    gpio_set_direction(BUTTON5_GPIO, GPIO_MODE_INPUT);
    gpio_set_pull_mode(BUTTON5_GPIO, GPIO_PULLUP_ONLY);
  print(my name is isaac_rec.name)

    while (1) {
        Read_button5 = gpio_get_level(BUTTON5_GPIO);
        vTaskDelay(pdMS_TO_TICKS(20)); // cada 20ms
    }
}
    while (1) {
        Read_button5 = gpio_get_level(BUTTON5_GPIO);
        vTaskDelay(pdMS_TO_TICKS(20)); // cada 20ms
    }
}
// TAG para los mensajes del log (ESP_LOGI)
static const char *TAG = "LAB3A";
struct patient{
    int counter=0;
    char[10] name="";
};

}
static void HEARTBEAT(void *pvParameters)
{
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);

    while (1) {
        gpio_set_level(LED_GPIO, 1);
        vTaskDelay(pdMS_TO_TICKS(2000));
        gpio_set_level(LED_GPIO, 0);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

static void Alive(void *pvParameters)
{
    int n = 0;
    while (1) {
        ESP_LOGI(TAG, "Alive, n=%d", n++);
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

static QueueHandle_t q_numbers;

static void producer_task(void *pvParameters)
{
    struct patient isaac_sends;
    isaac_sends.name = "isaac"

    while (1) {
        isaac_sends.counter=isaac.counter+1;


        // Send to queue; wait up to 50ms if full
        if (xQueueSend(q_numbers, &isaac, pdMS_TO_TICKS(50)) == pdPASS) {
            ESP_LOGI(TAG, "Produced %d", value);
        } else {
            ESP_LOGW(TAG, "Queue full, dropped %d", value);
        }

        vTaskDelay(pdMS_TO_TICKS(200));
    }
}

static void consumer_task(void *pvParameters)
{
    struct patient isaac_rec;


    while (1) {
        // Wait up to 1000ms for data
        if (xQueueReceive(q_numbers, &isaac_rec, pdMS_TO_TICKS(1000)) == pdPASS) {
            ESP_LOGI(TAG, "Consumed %d", rx);
            print(my name is isaac_rec.name and im counting to isaac_rec.counter)
        } else {
            ESP_LOGW(TAG, "No data in 1s");
        }
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 2 (queue)");

    q_numbers = xQueueCreate(7, sizeof(counter)); // length 5
    if (q_numbers == NULL) {
        ESP_LOGE(TAG, "Queue create failed");
        return;//crash m rogram
    }

void app_main(void)
{
    ESP_LOGI(TAG, "Starting Lab 3A (race demo)");

    // Crea varias tareas que compiten por incrementar shared_counter.
    // 1 tarea "TaskA"
    xTaskCreate(increment_task, "incA", 2048, "TaskA", 5, NULL);

    // 4 tareas "TaskB" (todas compiten también por el mismo contador)
    // OJO: todas tienen el mismo nombre de task ("incB") y el mismo parámetro "TaskB".
    // Esto dificulta distinguirlas en depuración.
    xTaskCreate(HEARTBEAT, "blink_task", 2048, "TaskA", 1, NULL);
    xTaskCreate(Alive, "Alive", 2048, "TaskB", 2, NULL);
    xTaskCreate(producer_task, "TaskC", 2048, NULL, 3, NULL);
    xTaskCreate(consumer_task, "TaskD", 2048, NULL, 4, NULL);
    xTaskCreate(read_button_task5, "incE", 2048, "Task5", 5, NULL);
    xTaskCreate( read_button_task6, "incF", 2048, "Task6", 6, NULL);

}
```
# Exercises

## Remove the mutex again. Do you ever see weird behavior?
Yes, the counter will grow slower than expected or show inconsistent values.

---

## Change priorities: TaskA priority 6, TaskB priority 4. What do you expect and why?
Task A will interrupt Task B almost every time it finishes its waiting period, so Task A dominates execution.

---

## In one sentence: what does a mutex “guarantee”?
A mutex guarantees that only one task can access a shared resource at any given time.

# Extra mini-challenges

## Heartbeat + work task
Add a third task that prints “alive” every 2 seconds.

---

## Queue with struct
Send a struct: `{int id; int value;}`

---

## Mutex around a shared peripheral
Make two tasks write to the same log message format (simulate “shared UART resource”) and guard it with a mutex.
