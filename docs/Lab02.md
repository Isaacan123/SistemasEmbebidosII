
## Lab 02 

## Code Of exercise LAB02 

### Activity goals
Create the 7 task code
--
-Task 1 Heartbeat
-Task 2 Alive task
-Task 3 Queue Struct Send
-Task 4 Queue Struct Receive
-Task 5 and 6 Mutex reading a button
-Task 7 Error loggin for task 1-6

 ### Materials
 --
ESP32 Development Board
--
1x Push Button
--
1x LED (Internal or External)
--
Resistors and Jumper wires

### Code
```c
#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"
#include "freertos/semphr.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define LED_GPIO 2
#define BUTTON_GPIO 4 // Cambiado a 4 por ser común en ESP32

static const char *TAG = "LAB04";

// Estructura para la tarea de la cola
typedef struct {
    int id;
    int counter;
    char name[10];
} patient_t;

// Handles globales
static QueueHandle_t q_patient;
static QueueHandle_t q_logs;
static SemaphoreHandle_t button_mutex;

// 1. Heartbeat
static void heartbeat_task(void *pv) {
    gpio_reset_pin(LED_GPIO);
    gpio_set_direction(LED_GPIO, GPIO_MODE_OUTPUT);
    while (1) {
        gpio_set_level(LED_GPIO, 1);
        vTaskDelay(pdMS_TO_TICKS(500));
        gpio_set_level(LED_GPIO, 0);
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

// 2. Alive
static void alive_task(void *pv) {
    int n = 0;
    while (1) {
        ESP_LOGI(TAG, "System is Alive, count=%d", n++);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

// 3. Producer (Queue Struct Send)
static void producer_task(void *pv) {
    patient_t p;
    p.id = 101;
    p.counter = 0;
    strcpy(p.name, "Isaac");
    while (1) {
        p.counter++;
        if (xQueueSend(q_patient, &p, pdMS_TO_TICKS(10)) == pdPASS) {
            ESP_LOGI(TAG, "Produced data for: %s", p.name);
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

// 4. Consumer (Queue Struct Receive)
static void consumer_task(void *pv) {
    patient_t p_rec;
    while (1) {
        if (xQueueReceive(q_patient, &p_rec, portMAX_DELAY)) {
            ESP_LOGI(TAG, "Consumed: Name=%s, Counter=%d", p_rec.name, p_rec.counter);
        }
    }
}

// 5 & 6. Mutex Reading Tasks
static void button_task(void *pv) {
    char *task_name = (char *)pv;
    while (1) {
        if (xSemaphoreTake(button_mutex, portMAX_DELAY)) {
            int level = gpio_get_level(BUTTON_GPIO);
            if (level == 0) {
                char *msg = "Event: Button Pressed";
                xQueueSend(q_logs, &msg, 0);
            }
            xSemaphoreGive(button_mutex);
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

// 7. Error Logging
static void logger_task(void *pv) {
    char *msg;
    while (1) {
        if (xQueueReceive(q_logs, &msg, portMAX_DELAY)) {
            ESP_LOGW("LOGGER", "%s", msg);
        }
    }
}

void app_main(void) {
    // Inicialización
    q_patient = xQueueCreate(5, sizeof(patient_t));
    q_logs = xQueueCreate(10, sizeof(char *));
    button_mutex = xSemaphoreCreateMutex();

    gpio_reset_pin(BUTTON_GPIO);
    gpio_set_direction(BUTTON_GPIO, GPIO_MODE_INPUT);
    gpio_pullup_en(BUTTON_GPIO);

    // Creación de las 7 tareas
    xTaskCreate(heartbeat_task, "Heart", 2048, NULL, 1, NULL);
    xTaskCreate(alive_task, "Alive", 2048, NULL, 1, NULL);
    xTaskCreate(producer_task, "Prod", 2048, NULL, 2, NULL);
    xTaskCreate(consumer_task, "Cons", 2048, NULL, 2, NULL);
    xTaskCreate(button_task, "Btn5", 2048, "Task 5", 3, NULL);
    xTaskCreate(button_task, "Btn6", 2048, "Task 6", 3, NULL);
    xTaskCreate(logger_task, "Log", 2048, NULL, 4, NULL);
}