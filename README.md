#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_rom_sys.h"

#define REQUEST_GPIO 4  // GPIO for request signal
#define GRANT_GPIO 5    // GPIO for grant signal
#define DELAY_US 1      // Minimal delay

static const char *TAG = "GPIO_Latency";

// Function prototype
void request_grant_task(void *pvParameter);

void app_main(void) {
    gpio_reset_pin(REQUEST_GPIO);
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);

    ESP_LOGI(TAG, "Starting Request-Grant signal test");

    // Small startup delay
    vTaskDelay(pdMS_TO_TICKS(5));

    // Create high-priority task pinned to core 0
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, NULL, configMAX_PRIORITIES - 1, NULL, 0);
}

void request_grant_task(void *pvParameter) {
    while (1) {
        // Set request GPIO HIGH
        gpio_set_level(REQUEST_GPIO, 1);
        esp_rom_delay_us(DELAY_US);  // Minimal delay

        // Generate random number (0-99)
        int rand_value = esp_random() % 100;

        // Grant access with 90% probability, deny with 10% probability
        if (rand_value < 90) {
            gpio_set_level(GRANT_GPIO, 1);  // Grant access
            ESP_LOGI(TAG, "Access granted (Rand: %d)", rand_value);
        } else {
            gpio_set_level(GRANT_GPIO, 0);  // Deny access
            ESP_LOGW(TAG, "Access denied (Rand: %d)", rand_value);
        }

        // Hold signal for a short time
        esp_rom_delay_us(DELAY_US);

        // Clear both signals
        gpio_set_level(REQUEST_GPIO, 0);
        gpio_set_level(GRANT_GPIO, 0);

        vTaskDelay(pdMS_TO_TICKS(100));  // Adjust delay for testing
    }
}
