#include <stdio.h>
#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_rom_sys.h"
#include "soc/gpio_struct.h"
#include "esp_intr_alloc.h"
#include "hal/gpio_hal.h"
#include "esp32c6/rom/gpio.h"
#include "esp_timer.h"
#include "esp_system.h"
#include "esp_random.h"
#include "esp_mac.h"
#include "esp_task_wdt.h"

#define REQUEST_GPIO    GPIO_NUM_6
#define GRANT_GPIO      GPIO_NUM_7
#define PRIORITY_GPIO   GPIO_NUM_10
#define GRANT_SWITCH    GPIO_NUM_11

static const char *TAG = "GPIO_latency";

void request_grant_task(void *pvParameter) {
    uint32_t request_count = 0;
    uint32_t grant_count = 0;

    while (1) {
        if (gpio_get_level(REQUEST_GPIO)) {
            request_count++;

            esp_rom_delay_us(2);

            int priority = gpio_get_level(PRIORITY_GPIO);
            uint32_t rand_value = esp_random() % 100;
            bool grant_low = (priority == 1 || rand_value < 10);

            if (grant_low) {
                grant_count++;

                // Set both GRANT and GRANT_SWITCH LOW
                GPIO.out_w1tc.val = (1 << GRANT_GPIO) | (1 << GRANT_SWITCH);

                while (gpio_get_level(REQUEST_GPIO)) {
                    ;
                }

                // Set both GRANT and GRANT_SWITCH HIGH
                GPIO.out_w1ts.val = (1 << GRANT_GPIO) | (1 << GRANT_SWITCH);
            } else {
                while (gpio_get_level(REQUEST_GPIO)) {
                    ;
                }
            }

            if (request_count % 10 == 0) {
                ESP_LOGI(TAG, "Requests: %" PRIu32 ", Grants: %" PRIu32, request_count, grant_count);
            }
        }
    }
}

void app_main(void) {
    // Configure GRANT_GPIO as output and set HIGH
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_GPIO);

    // Configure GRANT_SWITCH as output and set HIGH
    gpio_reset_pin(GRANT_SWITCH);
    gpio_set_direction(GRANT_SWITCH, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_SWITCH);

    // Configure REQUEST input
    gpio_config_t request_input = {
        .pin_bit_mask = (1ULL << REQUEST_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_ENABLE,
        .intr_type = GPIO_INTR_DISABLE
    };
    gpio_config(&request_input);

    // Configure PRIORITY input
    gpio_config_t priority_input = {
        .pin_bit_mask = (1ULL << PRIORITY_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type = GPIO_INTR_DISABLE
    };
    gpio_config(&priority_input);

    // Start the main task
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, NULL, configMAX_PRIORITIES - 1, NULL, tskNO_AFFINITY);
}
