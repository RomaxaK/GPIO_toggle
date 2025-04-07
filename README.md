#include <stdio.h>
#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_rom_sys.h"
#include "esp_random.h"
#include "esp_log.h"

#define REQUEST_GPIO   GPIO_NUM_4
#define GRANT_GPIO     GPIO_NUM_5
#define DELAY_US       200000
#define REQUEST_PERIOD_MS 1000

static const char *TAG = "GPIO_latency";

void request_grant_task(void *pvParameter) {
    int priority = *(int *)pvParameter;

    while (1) {
        if (gpio_get_level(REQUEST_GPIO)) {
            uint32_t rand_value = esp_random() % 100;
            bool grant_low = (priority == 1 || rand_value < 10);

            if (grant_low) {
                GPIO.out_w1tc.val = (1 << GRANT_GPIO);  // set GRANT LOW
                esp_rom_delay_us(100000);               // hold for 100 ms
                GPIO.out_w1ts.val = (1 << GRANT_GPIO);  // set GRANT HIGH
            }
        }

        esp_rom_delay_us(10);  // small CPU-friendly delay
    }
}

void app_main(void) {
    ESP_LOGI(TAG, "Initializing GPIOs...");

    // --- Configure GRANT_GPIO as OUTPUT ---
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_GPIO);  // initially HIGH

    // --- Configure REQUEST_GPIO as INPUT with pull-down ---
    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << REQUEST_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_ENABLE,  // <- essential!
        .intr_type = GPIO_INTR_DISABLE
    };
    gpio_config(&io_conf);

    // Optional priority value
    static int priority = 0;

    // Create task
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, &priority, configMAX_PRIORITIES - 1, NULL, tskNO_AFFINITY);
}
