#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_timer.h"
#include "esp_log.h"
#include "soc/gpio_struct.h"
#include "hal/gpio_hal.h"

#define REQUEST_GPIO 4
#define GRANT_GPIO 5
#define DELAY_US 5  // Fine-tuned for minimal latency

static const char *TAG = "GPIO_Latency";
static volatile bool grant_high = false;

void IRAM_ATTR grant_timer_callback(void *arg) {
    ESP_DRAM_LOGI(TAG, "Grant timer triggered: grant_high = %d", grant_high);
    if (grant_high) {
        GPIO.out_w1ts = (1 << GRANT_GPIO);  // Set GRANT high
    } else {
        GPIO.out_w1tc = (1 << GRANT_GPIO);  // Set GRANT low
    }
}

void request_grant_task(void *pvParameter) {
    int priority = *(int*)pvParameter;  
    ESP_LOGI(TAG, "Task started with priority: %d", priority);

    while (1) {
        uint32_t rand_value = esp_random() % 100;
        grant_high = (priority == 1 || rand_value < 90);

        ESP_LOGI(TAG, "Setting REQUEST_GPIO high");
        GPIO.out_w1ts = (1 << REQUEST_GPIO);  // Set REQUEST_GPIO high
        esp_rom_delay_us(DELAY_US);

        // Directly set GRANT_GPIO instead of using a timer for testing
        if (grant_high) {
            ESP_LOGI(TAG, "Setting GRANT_GPIO high");
            GPIO.out_w1ts = (1 << GRANT_GPIO);
        } else {
            ESP_LOGI(TAG, "Setting GRANT_GPIO low");
            GPIO.out_w1tc = (1 << GRANT_GPIO);
        }

        ESP_LOGI(TAG, "Clearing REQUEST_GPIO and GRANT_GPIO");
        GPIO.out_w1tc = (1 << REQUEST_GPIO); // Clear REQUEST
        GPIO.out_w1tc = (1 << GRANT_GPIO);   // Ensure GRANT is reset

        vTaskDelay(pdMS_TO_TICKS(1));  // Short delay to prevent watchdog reset
    }
}

void app_main(void) {
    ESP_LOGI(TAG, "Initializing GPIOs...");
    gpio_reset_pin(REQUEST_GPIO);
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);

    static int priority = 0;
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, &priority, configMAX_PRIORITIES - 1, NULL, 0);
}
