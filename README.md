#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_rom_sys.h"  // Needed for esp_rom_delay_us()

#define REQUEST_GPIO 8  // GPIO for request signal
#define GRANT_GPIO 9    // GPIO for grant signal
#define DELAY_US 10     // Microsecond delay

static const char *TAG = "GPIO_Latency";

// Function prototype
void request_grant_task(void *pvParameter);

void app_main(void) {
    // Configure GPIOs
    gpio_reset_pin(REQUEST_GPIO);
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);

    ESP_LOGI(TAG, "Starting Request-Grant signal test");

    // Start the task
    xTaskCreate(request_grant_task, "request_grant_task", 2048, NULL, 5, NULL);
}

void request_grant_task(void *pvParameter) {
    while (1) {
        // Set request GPIO HIGH
        gpio_set_level(REQUEST_GPIO, 1);
        ESP_LOGI(TAG, "REQUEST_GPIO SET TO HIGH");
        esp_rom_delay_us(DELAY_US);

        // Read the actual GPIO state after setting it HIGH
        ESP_LOGI(TAG, "REQUEST_GPIO: %d", gpio_get_level(REQUEST_GPIO));

        // Set grant GPIO HIGH
        gpio_set_level(GRANT_GPIO, 1);
        ESP_LOGI(TAG, "GRANT_GPIO SET TO HIGH");
        esp_rom_delay_us(DELAY_US);

        // Read the actual GPIO state after setting it HIGH
        ESP_LOGI(TAG, "GRANT_GPIO: %d", gpio_get_level(GRANT_GPIO));

        // Set both GPIOs LOW
        gpio_set_level(REQUEST_GPIO, 0);
        gpio_set_level(GRANT_GPIO, 0);
        ESP_LOGI(TAG, "Both GPIOs set to LOW");

        ESP_LOGI(TAG, "Request sent, Grant given");

        // Delay for monitoring
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
