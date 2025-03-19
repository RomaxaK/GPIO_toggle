#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_timer.h"

#define REQUEST_GPIO 4
#define GRANT_GPIO 5
#define DELAY_US 5  // Fine-tuned for minimal latency

static volatile bool grant_high = false;

void IRAM_ATTR grant_timer_callback(void *arg) {
    if (grant_high) {
        REG_WRITE(GPIO_OUT_W1TS_REG, (1 << GRANT_GPIO));  // Set GRANT high
    } else {
        REG_WRITE(GPIO_OUT_W1TC_REG, (1 << GRANT_GPIO));  // Set GRANT low
    }
}

void request_grant_task(void *pvParameter) {
    int priority = *(int*)pvParameter;  // Retrieve priority value
    esp_timer_handle_t grant_timer;
    
    // Configure High-Resolution Timer for precise control
    const esp_timer_create_args_t grant_timer_args = {
        .callback = &grant_timer_callback,
        .name = "grant_timer"
    };
    esp_timer_create(&grant_timer_args, &grant_timer);
    
    while (1) {
        uint32_t rand_value = esp_random() % 100;  // Precompute randomness

        // Determine grant signal state before GPIO toggling
        grant_high = (priority == 1 || rand_value < 90);

        // Set REQUEST high
        REG_WRITE(GPIO_OUT_W1TS_REG, (1 << REQUEST_GPIO));

        // Start the grant timer to toggle GRANT_GPIO with minimal delay
        esp_timer_start_once(grant_timer, DELAY_US);

        // Wait for a short delay before clearing REQUEST
        esp_rom_delay_us(DELAY_US);
        REG_WRITE(GPIO_OUT_W1TC_REG, (1 << REQUEST_GPIO));  // Clear REQUEST

        // Short sleep to simulate main loop iteration
        vTaskDelay(pdMS_TO_TICKS(1));  // Keep FreeRTOS happy
    }
}

void app_main(void) {
    gpio_reset_pin(REQUEST_GPIO);
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);

    static int priority = 0; // Change to 1 to test priority effects
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, &priority, configMAX_PRIORITIES - 1, NULL, 0);
}
