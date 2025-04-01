#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_rom_sys.h"

#define REQUEST_GPIO   GPIO_NUM_4
#define GRANT_GPIO     GPIO_NUM_5
#define PRIORITY_GPIO  GPIO_NUM_6
#define GRANT_PULSE_US 100000  // 100 ms

static const char *TAG = "GPIO_GRANT_LOGIC";

void request_grant_task(void *pvParameter) {
    while (1) {
        int priority = gpio_get_level(PRIORITY_GPIO);
        int request  = gpio_get_level(REQUEST_GPIO);

        if (request) {
            if (priority == 1) {
                gpio_set_level(GRANT_GPIO, 1);
                esp_rom_delay_us(GRANT_PULSE_US);
                gpio_set_level(GRANT_GPIO, 0);
            } else {
                gpio_set_level(GRANT_GPIO, 0);
            }
        } else {
            gpio_set_level(GRANT_GPIO, 0);
        }

        vTaskDelay(pdMS_TO_TICKS(1));  // check every 1 ms
    }
}

void app_main(void) {
    ESP_LOGI(TAG, "Initializing GPIOs...");

    // Configure REQUEST and PRIORITY as inputs
    gpio_reset_pin(REQUEST_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_INPUT);

    gpio_reset_pin(PRIORITY_GPIO);
    gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);

    // Configure GRANT as output
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(GRANT_GPIO, 0);

    // Start task
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, NULL, 1, NULL, 0);
}
