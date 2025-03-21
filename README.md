#include <stdio.h>
#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_timer.h"
#include "soc/gpio_struct.h"
#include "hal/gpio_hal.h"
#include "freertos/queue.h"
#include "esp_rom_sys.h"
#include "esp_system.h"

#define REQUEST_GPIO 4
#define GRANT_GPIO 5
#define DELAY_US 5        
#define REQUEST_PERIOD_MS 100  

static const char *TAG = "GPIO_Latency";
static volatile bool grant_high = false;
static volatile bool grant_pending = false;

void IRAM_ATTR grant_timer_callback(void *arg) {
    if (grant_pending) {
        ESP_DRAM_LOGI(TAG, "Grant timer triggered: grant_high = %d", grant_high);
        if (grant_high) {
            GPIO.out_w1ts.val = (1 << GRANT_GPIO);  
            esp_rom_delay_us(10);  
            GPIO.out_w1tc.val = (1 << GRANT_GPIO);  
        }
        grant_pending = false;  
    }
}

void request_grant_task(void *pvParameter) {
    int priority = *(int*)pvParameter;  
    esp_timer_handle_t grant_timer;

    const esp_timer_create_args_t grant_timer_args = {
        .callback = &grant_timer_callback,
        .name = "grant_timer"
    };
    esp_timer_create(&grant_timer_args, &grant_timer);

    while (1) {
        uint32_t rand_value = esp_random() % 100;
        grant_high = (priority == 1 || rand_value < 90);

        GPIO.out_w1ts.val = (1 << REQUEST_GPIO);
        esp_rom_delay_us(DELAY_US);  

        if (!grant_pending) {
            grant_pending = true;  
            esp_timer_stop(grant_timer);  
            esp_err_t err = esp_timer_start_once(grant_timer, DELAY_US);
            if (err != ESP_OK) {
                ESP_LOGE(TAG, "Failed to start grant timer: %d", err);
            }
        }

        GPIO.out_w1tc.val = (1 << REQUEST_GPIO);
        vTaskDelay(pdMS_TO_TICKS(REQUEST_PERIOD_MS));  
    }
}

void app_main(void) {
    ESP_LOGI(TAG, "Initializing GPIOs...");
    gpio_reset_pin(REQUEST_GPIO);
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);

    static int priority = 0;
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, &priority, configMAX_PRIORITIES - 1, NULL, tskNO_AFFINITY);
}
