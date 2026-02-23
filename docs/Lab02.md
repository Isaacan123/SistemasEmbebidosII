# Lab 02 – FreeRTOS Multi-Tasking: Queues, Mutexes and Structs

## Activity Goals
- Implement a complex system with **7 concurrent tasks**.
- Manage data communication using **Queues** with custom `struct` types.
- Protect shared hardware resources (GPIO) using **Mutexes**.
- Centralize system events and errors through a dedicated **Logging Task**.

## Materials
- ESP32 Development Board
- 1x Push Button
- 1x LED (Internal or External)
- Resistors and Jumper wires

---

## Code Of exercise LAB02

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
#define BUTTON_GPIO 4 

static const char *TAG = "LAB02";

// Estructura para la comunicación por cola
typedef struct {
    int id;
    int counter;
    char name[10];
} patient_t;

// Handles globales de FreeRTOS
static QueueHandle_t q_patient;
static QueueHandle_t q_logs;
static SemaphoreHandle_t button_mutex;

// 1. Heartbeat Task: Indica que el sistema está corriendo
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

// 2. Alive Task: Mensaje periódico en consola
static void alive_task(void *pv) {
    int n = 0;
    while (1) {
        ESP_LOGI(TAG, "System is Alive, count=%d", n++);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}

// 3. Queue Struct Send (Producer)
static void producer_task(void *pv) {
    patient_t p;
    p.id = 101;
    p.counter = 0;
    strcpy(p.name, "Isaac");
    while (1) {
        p.counter++;
        if (xQueueSend(q_patient, &p, pdMS_TO_TICKS(10)) == pdPASS) {
            ESP_LOGI(TAG, "Data sent to queue for: %s", p.name);
        }
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

// 4. Queue Struct Receive (Consumer)
static void consumer_task(void *pv) {
    patient_t p_rec;
    while (1) {
        if (xQueueReceive(q_patient, &p_rec, portMAX_DELAY)) {
            ESP_LOGI(TAG, "Queue Recv -> Name: %s, ID: %d, Counter: %d", p_rec.name, p_rec.id, p_rec.counter);
        }
    }
}

// 5 & 6. Mutex Reading Tasks: Dos tareas acceden al mismo botón
static void button_task(void *pv) {
    char *task_id = (char *)pv;
    while (1) {
        if (xSemaphoreTake(button_mutex, portMAX_DELAY)) {
            if (gpio_get_level(BUTTON_GPIO) == 0) {
                char *msg = "Event: Button Pressed detected";
                xQueueSend(q_logs, &msg, 0);
            }
            xSemaphoreGive(button_mutex);
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}

// 7. Error Logging Task: Centraliza los mensajes de log
static void logger_task(void *pv) {
    char *msg;
    while (1) {
        if (xQueueReceive(q_logs, &msg, portMAX_DELAY)) {
            ESP_LOGW("LOGGER_SERVICE", "%s", msg);
        }
    }
}

void app_main(void) {
    // Inicialización de colas y semáforos
    q_patient = xQueueCreate(5, sizeof(patient_t));
    q_logs = xQueueCreate(10, sizeof(char *));
    button_mutex = xSemaphoreCreateMutex();

    // Configuración de Hardware
    gpio_reset_pin(BUTTON_GPIO);
    gpio_set_direction(BUTTON_GPIO, GPIO_MODE_INPUT);
    gpio_pullup_en(BUTTON_GPIO);

    // Despliegue de las 7 tareas con diferentes prioridades
    xTaskCreate(heartbeat_task, "Heart", 2048, NULL, 1, NULL);
    xTaskCreate(alive_task, "Alive", 2048, NULL, 1, NULL);
    xTaskCreate(producer_task, "Prod", 2048, NULL, 2, NULL);
    xTaskCreate(consumer_task, "Cons", 2048, NULL, 2, NULL);
    xTaskCreate(button_task, "Btn_T5", 2048, "Task 5", 3, NULL);
    xTaskCreate(button_task, "Btn_T6", 2048, "Task 6", 3, NULL);
    xTaskCreate(logger_task, "Log_T7", 2048, NULL, 4, NULL);
}
```

---

## Exercises

**Mutex Necessity:** ¿Qué pasaría si las Tareas 5 y 6 no usaran un Mutex para leer el botón?
- **Observación:** Aunque el hardware permite la lectura simultánea, en periféricos más complejos (como un sensor I2C), el sistema colapsaría. 
- **Conclusión:** El Mutex asegura que la sección crítica del código sea accedida por un solo proceso a la vez, manteniendo la integridad de los datos.

**Data Structs in Queues:** ¿Cuál es la ventaja de usar una `struct` en la cola en lugar de solo un entero?
- **Respuesta:** Permite enviar paquetes de información completos (ID, Nombres, Valores).
- **Efecto:** El sistema se vuelve más robusto y organizado, facilitando el procesamiento de datos complejos entre el productor y el consumidor.

**Centralized Logging:** ¿Cómo beneficia la Tarea 7 al rendimiento del sistema?
- **Razón:** La escritura en consola (Serial) es una operación lenta. 
- **Efecto:** Al enviar los mensajes a una cola, las tareas principales no se bloquean esperando a que el texto se imprima, delegando esa carga a la Tarea 7.