#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_rom_sys.h"
#include "esp_random.h"

// GPIO definitions
#define REQUEST_GPIO       GPIO_NUM_4
#define GRANT_SWITCH_GPIO  GPIO_NUM_5
#define PRIORITY_GPIO      GPIO_NUM_6
#define GRANT_7015M_GPIO   GPIO_NUM_7

#define GRANT_PULSE_US     100  // Pulse duration in microseconds

void request_grant_polling_task(void *arg) {
    while (1) {
        if (gpio_get_level(REQUEST_GPIO)) {
            int priority = gpio_get_level(PRIORITY_GPIO);
            uint32_t rand_value = esp_random() % 100;

            if (priority == 1 || (priority == 0 && rand_value < 10)) {
                GPIO.out_w1tc.val = (1 << GRANT_SWITCH_GPIO);
                GPIO.out_w1tc.val = (1 << GRANT_7015M_GPIO);

                esp_rom_delay_us(GRANT_PULSE_US);

                GPIO.out_w1ts.val = (1 << GRANT_SWITCH_GPIO);
                GPIO.out_w1ts.val = (1 << GRANT_7015M_GPIO);
            }

            // Wait until request signal goes low again
            while (gpio_get_level(REQUEST_GPIO)) {
                ; // Spin
            }
        }
    }
}

void app_main(void) {
    // Configure outputs
    gpio_reset_pin(GRANT_SWITCH_GPIO);
    gpio_set_direction(GRANT_SWITCH_GPIO, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_SWITCH_GPIO); // Set high (inactive)

    gpio_reset_pin(GRANT_7015M_GPIO);
    gpio_set_direction(GRANT_7015M_GPIO, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_7015M_GPIO); // Set high (inactive)

    // Configure inputs
    gpio_reset_pin(PRIORITY_GPIO);
    gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);

    gpio_reset_pin(REQUEST_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_INPUT);

    // Start polling task
    xTaskCreatePinnedToCore(request_grant_polling_task, "poll_task", 2048, NULL, 10, NULL, 0);
}
