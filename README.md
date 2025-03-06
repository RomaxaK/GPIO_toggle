#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_rom_sys.h"
#include "soc/gpio_struct.h"
#include "hal/gpio_hal.h"
#include "esp32c6/rom/gpio.h"

#define REQUEST_GPIO 4  // Optimized GPIOs
#define GRANT_GPIO 5
#define DELAY_US 1  // Reduced delay

static const char *TAG = "GPIO_Latency";

// Function prototype
void request_grant_task(void *pvParameter);

void app_main(void) {
    gpio_reset_pin(REQUEST_GPIO);
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);

    ESP_LOGI(TAG, "Starting Request-Grant signal test");

    // Use highest priority task
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, NULL, configMAX_PRIORITIES - 1, NULL, 0);
}

void request_grant_task(void *pvParameter) {
    while (1) {
        // Set request GPIO HIGH
        REG_WRITE(GPIO_OUT_W1TS_REG, (1 << REQUEST_GPIO));
        ESP_LOGI(TAG, "REQUEST_GPIO SET TO HIGH");

        // Read GPIO state
        ESP_LOGI(TAG, "REQUEST_GPIO: %d", gpio_get_level(REQUEST_GPIO));

        // Set grant GPIO HIGH
        REG_WRITE(GPIO_OUT_W1TS_REG, (1 << GRANT_GPIO));
        ESP_LOGI(TAG, "GRANT_GPIO SET TO HIGH");

        ESP_LOGI(TAG, "GRANT_GPIO: %d", gpio_get_level(GRANT_GPIO));

        // Clear GPIOs (Set to LOW)
        REG_WRITE(GPIO_OUT_W1TC_REG, (1 << REQUEST_GPIO));
        REG_WRITE(GPIO_OUT_W1TC_REG, (1 << GRANT_GPIO));
        ESP_LOGI(TAG, "Both GPIOs set to LOW");

        ESP_LOGI(TAG, "Request sent, Grant given");
    }
}
