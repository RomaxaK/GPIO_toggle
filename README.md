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

static const char *TAG = "GPIO_latency";

void request_grant_task(void *pvParameter) {
    int priority = *(int *)pvParameter;

    while (1) {
        if (gpio_get_level(REQUEST_GPIO)) {
            uint32_t rand_value = esp_random() % 100;
            bool grant_low = (priority == 1 || rand_value < 10);

            if (grant_low) {
                GPIO.out_w1tc.val = (1 << GRANT_GPIO);
                while (gpio_get_level(REQUEST_GPIO)) {
                    ;
                }
                GPIO.out_w1ts.val = (1 << GRANT_GPIO);
            } else {
                while (gpio_get_level(REQUEST_GPIO)) {
                    ;
                }
            }
        }
        esp_rom_delay_us(10);
    }
}

void app_main(void) {
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_GPIO);

    gpio_config_t io_conf = {
        .pin_bit_mask = (1ULL << REQUEST_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_ENABLE,
        .intr_type = GPIO_INTR_DISABLE
    };
    gpio_config(&io_conf);

    static int priority = 0;
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, &priority, configMAX_PRIORITIES - 1, NULL, tskNO_AFFINITY);
}
