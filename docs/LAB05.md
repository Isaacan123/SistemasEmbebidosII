# MQTT LAB pt2

## Team 
* Yahir Gil Mendoza
* Isaac Antonio Perez Aleman
* Pablo Eduardo López Manzano
* Juan David Garcia cortez 
* Sumie Arai Erazo

## 1) Exercise Goals
Connect 2 ESPS with MQTT to control on state and brightness

## 2) Materials & Setup
- **Tools/Software** - Editors: VS Code, Python 3.12, ESP32-C6

**Wiring/Safety:** - ESP32-C6 , LED, BUTTON, RESISTANCE OF  220 OHMS and 1KOHMS

## 3) Procedure 
The ESP32 works like a mini‑website that listens to commands and controls the LED, while making sure the responses are easy for apps or browsers to understand.

## Codes
### Main Code
```bash
#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "nvs_flash.h"
#include "esp_log.h"

// Tus cabeceras personalizadas
#include "wifi_sta.h"
#include "mqtt_client_app.h"

static const char *TAG = "MAIN_APP";

// Credenciales y Broker
#define MI_SSID "iPhone de Sumie"
#define MI_PASS "12345678"
#define MI_BROKER "mqtt://test.mosquitto.org:1883"

void app_main(void) {
    // 1. Inicializar NVS (indispensable para Wi-Fi)
    esp_err_t ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }
    ESP_ERROR_CHECK(ret);

    // 2. Conectar Wi-Fi usando tu función de laboratorio
    ESP_LOGI(TAG, "Iniciando Wi-Fi...");
    esp_err_t wifi_res = wifi_sta_connect(MI_SSID, MI_PASS, 30000);

    if (wifi_res == ESP_OK) {
        ESP_LOGI(TAG, "Wi-Fi Conectado. Iniciando MQTT...");
        
        // 3. Arrancar MQTT
        mqtt_app_start(MI_BROKER);
    } else {
        ESP_LOGE(TAG, "Error crítico: No se pudo conectar al Wi-Fi.");
    }

    // Heartbeat para saber que el sistema no se ha colgado
    while (1) {
        vTaskDelay(pdMS_TO_TICKS(10000));
        ESP_LOGI(TAG, "Sistema activo...");
    }
}
```
### MQTT CLIENT CODE FOR ESP1
```bash
#include <stdio.h>
#include <stdint.h>
#include <string.h>
#include <stdlib.h>
#include "esp_log.h"
#include "mqtt_client.h"
#include "driver/gpio.h"
#include "driver/adc.h"
#include "driver/ledc.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

static const char *TAG = "ESP2_TEAM_PAPU";

#define PIN_LED1 6  
#define PIN_LED2 7  
#define PIN_BTN1 5  
#define PIN_BTN2 4  
#define POT_CHANNEL ADC1_CHANNEL_0 

#define LEDC_MODE LEDC_LOW_SPEED_MODE
#define LEDC_RES LEDC_TIMER_12_BIT 

#define TEAM_ID "teamPapu"
#define TOPIC_SUB_POT   "ibero/" TEAM_ID "/esp1/potenciometer"
#define TOPIC_SUB_LED1  "ibero/" TEAM_ID "/esp1/led1"
#define TOPIC_SUB_LED2  "ibero/" TEAM_ID "/es1/led2"

#define TOPIC_PUB_POT   "ibero/" TEAM_ID "/esp2/potenciometer"
#define TOPIC_PUB_BTN1  "ibero/" TEAM_ID "/esp2/led1"
#define TOPIC_PUB_BTN2  "ibero/" TEAM_ID "/esp2/led2"

static esp_mqtt_client_handle_t client = NULL;

int estado_global_led1 = 0;
int estado_global_led2 = 0;
int brillo_pot_recibido = 0;

void refrescar_salidas_pwm() {
    ledc_set_duty(LEDC_MODE, LEDC_CHANNEL_0, estado_global_led1 * brillo_pot_recibido);
    ledc_update_duty(LEDC_MODE, LEDC_CHANNEL_0);
    ledc_set_duty(LEDC_MODE, LEDC_CHANNEL_1, estado_global_led2 * brillo_pot_recibido);
    ledc_update_duty(LEDC_MODE, LEDC_CHANNEL_1);
}

void init_hw() {
    gpio_config_t btn_cfg = {.pin_bit_mask=(1ULL<<PIN_BTN1)|(1ULL<<PIN_BTN2), .mode=GPIO_MODE_INPUT, .pull_up_en=GPIO_PULLUP_ENABLE};
    gpio_config(&btn_cfg);
    adc1_config_width(ADC_WIDTH_BIT_12);
    adc1_config_channel_atten(POT_CHANNEL, ADC_ATTEN_DB_12);
    ledc_timer_config_t timer = {.speed_mode=LEDC_MODE, .timer_num=LEDC_TIMER_0, .duty_resolution=LEDC_RES, .freq_hz=5000, .clk_cfg=LEDC_AUTO_CLK};
    ledc_timer_config(&timer);
    ledc_channel_config_t l1 = {.channel=LEDC_CHANNEL_0, .gpio_num=PIN_LED1, .speed_mode=LEDC_MODE, .timer_sel=LEDC_TIMER_0, .duty=0};
    ledc_channel_config(&l1);
    ledc_channel_config_t l2 = {.channel=LEDC_CHANNEL_1, .gpio_num=PIN_LED2, .speed_mode=LEDC_MODE, .timer_sel=LEDC_TIMER_0, .duty=0};
    ledc_channel_config(&l2);
}

void sensors_task(void *pvParameters) {
    int last_pot = -1, l_b1 = 1, l_b2 = 1;
    while (1) {
        if (client) {
            int b1 = gpio_get_level(PIN_BTN1), b2 = gpio_get_level(PIN_BTN2);
            if (b1 == 0 && l_b1 == 1) esp_mqtt_client_publish(client, TOPIC_PUB_BTN1, "toggle", 0, 0, 0);
            l_b1 = b1;
            if (b2 == 0 && l_b2 == 1) esp_mqtt_client_publish(client, TOPIC_PUB_BTN2, "toggle", 0, 0, 0);
            l_b2 = b2;
            int pot = adc1_get_raw(POT_CHANNEL);
            if (abs(pot - last_pot) > 60) {
                char s[10]; sprintf(s, "%d", pot);
                esp_mqtt_client_publish(client, TOPIC_PUB_POT, s, 0, 0, 0);
                last_pot = pot;
            }
        }
        vTaskDelay(pdMS_TO_TICKS(50));
    }
}

static void mqtt_event_handler(void *handler_args, esp_event_base_t base, int32_t event_id, void *event_data) {
    esp_mqtt_event_handle_t event = (esp_mqtt_event_handle_t) event_data;
    if (event_id == MQTT_EVENT_CONNECTED) {
        esp_mqtt_client_subscribe(client, TOPIC_SUB_POT, 0);
        esp_mqtt_client_subscribe(client, TOPIC_SUB_LED1, 0);
        esp_mqtt_client_subscribe(client, TOPIC_SUB_LED2, 0);
    } else if (event_id == MQTT_EVENT_DATA) {
        char *data = strndup(event->data, event->data_len);
        if (strncmp(event->topic, TOPIC_SUB_POT, event->topic_len) == 0) {
            brillo_pot_recibido = atoi(data);
        } else if (strncmp(event->topic, TOPIC_SUB_LED1, event->topic_len) == 0) {
            estado_global_led1 = !estado_global_led1;
        } else if (strncmp(event->topic, TOPIC_SUB_LED2, event->topic_len) == 0) {
            estado_global_led2 = !estado_global_led2;
        }
        refrescar_salidas_pwm();
        free(data);
    }
}

void mqtt_app_start(const char *broker_uri) {
    init_hw();
    xTaskCreate(sensors_task, "sensors", 4096, NULL, 5, NULL);
    esp_mqtt_client_config_t cfg = { .broker.address.uri = broker_uri };
    client = esp_mqtt_client_init(&cfg);
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, NULL);
    esp_mqtt_client_start(client);
}
```

### MQTT Change for ESP2:
```bash
#define TOPIC_SUB_POT   "ibero/" TEAM_ID "/esp2/potenciometer"
#define TOPIC_SUB_LED1  "ibero/" TEAM_ID "/esp2/led1"
#define TOPIC_SUB_LED2  "ibero/" TEAM_ID "/esp2/led2"

#define TOPIC_PUB_POT   "ibero/" TEAM_ID "/esp1/potenciometer"
#define TOPIC_PUB_BTN1  "ibero/" TEAM_ID "/esp1/led1"
#define TOPIC_PUB_BTN2  "ibero/" TEAM_ID "/esp1/led2"
```

#### Video of it working 
<!-- Ajuste para mostrar correctamente el video Shorts de YouTube como embed -->
<iframe width="560" height="315"
src="https://www.youtube.com/embed/j12o1w3M6sM"
title="YouTube video player"
frameborder="0"
allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
allowfullscreen>
</iframe>