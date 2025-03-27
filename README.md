#include <stdio.h>
#include <stdlib.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_rom_sys.h"

#define REQUEST_GPIO        4
#define GRANT_GPIO          5
#define DELAY_US            1
#define REQUEST_PERIOD_MS   100
#define GRANT_ACTIVE_LOW    1

static const char *TAG = "GRANT_CTRL";

void request_grant_task(void *pvParameter)
{
    int priority = (int)pvParameter;

    while (1) {
        GPIO.out_wts.val = (1 << REQUEST_GPIO);
        esp_rom_delay_us(DELAY_US);

#if GRANT_ACTIVE_LOW
        GPIO.out_wtc.val = (1 << GRANT_GPIO);
#else
        GPIO.out_wts.val = (1 << GRANT_GPIO);
#endif
        esp_rom_delay_us(10);

#if GRANT_ACTIVE_LOW
        GPIO.out_wts.val = (1 << GRANT_GPIO);
#else
        GPIO.out_wtc.val = (1 << GRANT_GPIO);
#endif

        GPIO.out_wtc.val = (1 << REQUEST_GPIO);

        vTaskDelay(pdMS_TO_TICKS(REQUEST_PERIOD_MS));
    }
}

void app_main(void)
{
    ESP_LOGI(TAG, "Initializing GPIOs...");

    gpio_reset_pin(REQUEST_GPIO);
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);

    static int priority = 0;
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, &priority, configMAX_PRIORITIES - 1, NULL, tskNO_AFFINITY);
}
