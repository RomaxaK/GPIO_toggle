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
#define DELAY_US 5  // Fine-tuned for minimal latency

static const char *TAG = "GPIO_Latency";
static volatile bool grant_high = false;
static volatile bool grant_pending = false;

/**
 * @brief High-resolution timer callback to toggle GRANT_GPIO
 */
void IRAM_ATTR grant_timer_callback(void *arg) {
    if (grant_pending) {
        ESP_DRAM_LOGI(TAG, "Grant timer triggered: grant_high = %d", grant_high);
        if (grant_high) {
            GPIO.out_w1ts.val = (1 << GRANT_GPIO);  // Set GRANT high
        } else {
            GPIO.out_w1tc.val = (1 << GRANT_GPIO);  // Set GRANT low
        }
        grant_pending = false;  // Mark the request as processed
    }
}

/**
 * @brief Task to handle REQUEST signal and trigger GRANT signal using a timer
 */
void request_grant_task(void *pvParameter) {
    int priority = *(int*)pvParameter;  
    esp_timer_handle_t grant_timer;

    // Initialize high-resolution timer for GRANT signal
    const esp_timer_create_args_t grant_timer_args = {
        .callback = &grant_timer_callback,
        .name = "grant_timer"
    };
    esp_timer_create(&grant_timer_args, &grant_timer);

    while (1) {
        // PRECOMPUTE GRANT SIGNAL STATUS
        uint32_t rand_value = esp_random() % 100;
        grant_high = (priority == 1 || rand_value < 90);

        // Set REQUEST_GPIO high (Start of a new request)
        GPIO.out_w1ts.val = (1 << REQUEST_GPIO);
        esp_rom_delay_us(DELAY_US);  // Microsecond delay for synchronization

        // Ensure grant is only given once per request
        if (!grant_pending) {
            grant_pending = true;  // Mark a new request as pending

            // Start the grant timer
            esp_timer_stop(grant_timer);  // Stop timer before restarting
            esp_err_t err = esp_timer_start_once(grant_timer, DELAY_US);
            if (err != ESP_OK) {
                ESP_LOGE(TAG, "Failed to start grant timer: %d", err);
            }
        }

        // Clear REQUEST_GPIO after processing
        GPIO.out_w1tc.val = (1 << REQUEST_GPIO);

        // Reduced vTaskDelay to minimize task switching issues
        vTaskDelay(pdMS_TO_TICKS(5));  
    }
}

/**
 * @brief ESP32 main function to initialize GPIOs and start FreeRTOS tasks
 */
void app_main(void) {
    ESP_LOGI(TAG, "Initializing GPIOs...");
    gpio_reset_pin(REQUEST_GPIO);
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);

    static int priority = 0;
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, &priority, configMAX_PRIORITIES - 1, NULL, tskNO_AFFINITY);
}
