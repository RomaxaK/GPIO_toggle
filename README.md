#include <stdio.h>
#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_rom_sys.h"
#include "esp_random.h"
#include "esp_log.h"

// GPIO definitions
#define REQUEST_GPIO   GPIO_NUM_4      // Input from QPG7015M
#define GRANT_GPIO     GPIO_NUM_5      // Output to RF switch
#define DELAY_US       200000          // Delay for pulse width (e.g., 100ms)
#define REQUEST_PERIOD_MS 1000         // Loop cycle in ms (can be shorter if needed)

static const char *TAG = "GPIO_latency";

// Task that handles request and grants
void request_grant_task(void *pvParameter) {
    int priority = *(int *)pvParameter;

    while (1) {
        if (gpio_get_level(REQUEST_GPIO)) {
            uint32_t rand_value = esp_random() % 100;
            bool grant_low = (priority == 1 || rand_value < 10);

            if (grant_low) {
                GPIO.out_w1tc.val = (1 << GRANT_GPIO);   // Set GRANT LOW
                esp_rom_delay_us(100000);                // Pulse duration
                GPIO.out_w1ts.val = (1 << GRANT_GPIO);   // Set GRANT HIGH
            }
        }

        esp_rom_delay_us(10);  // Small CPU-friendly delay
    }
}

void app_main(void) {
    ESP_LOGI(TAG, "Initializing GPIOs...");

    // Set GRANT as output and initialize HIGH
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_GPIO);

    // Set REQUEST as input
    gpio_reset_pin(REQUEST_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_INPUT);

    // Optional: log priority externally or set manually
    static int priority = 0;

    // Start polling task
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, &priority, configMAX_PRIORITIES - 1, NULL, tskNO_AFFINITY);
}
