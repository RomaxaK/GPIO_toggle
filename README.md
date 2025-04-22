#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "driver/uart.h"
#include "esp_log.h"
#include "esp_rom_sys.h"
#include "esp_timer.h"
#include "esp_system.h"
#include "esp_random.h"

#define REQUEST_GPIO    GPIO_NUM_6
#define GRANT_GPIO      GPIO_NUM_7
#define PRIORITY_GPIO   GPIO_NUM_10
#define GRANT_SWITCH    GPIO_NUM_11

#define UART_PORT       UART_NUM_0
#define UART_BUF_SIZE   128

static const char *TAG = "GPIO_latency";

static bool use_random = true;
static bool priority_gpio_is_output = false;

void uart_command_task(void *arg) {
    uint8_t data[UART_BUF_SIZE];

    while (1) {
        int len = uart_read_bytes(UART_PORT, data, sizeof(data) - 1, pdMS_TO_TICKS(100));
        if (len > 0) {
            data[len] = '\0';
            char *cmd = (char *)data;

            if (strncmp(cmd, "CMD,PRIO_MODE,INPUT", 19) == 0) {
                gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);
                priority_gpio_is_output = false;
                printf("PRIO_MODE set to INPUT\n");
            } else if (strncmp(cmd, "CMD,PRIO_MODE,OUTPUT", 20) == 0) {
                gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_OUTPUT);
                priority_gpio_is_output = true;
                printf("PRIO_MODE set to OUTPUT\n");
            } else if (strncmp(cmd, "CMD,PRIO_SET,HIGH", 17) == 0 && priority_gpio_is_output) {
                gpio_set_level(PRIORITY_GPIO, 1);
                printf("PRIO set to HIGH\n");
            } else if (strncmp(cmd, "CMD,PRIO_SET,LOW", 16) == 0 && priority_gpio_is_output) {
                gpio_set_level(PRIORITY_GPIO, 0);
                printf("PRIO set to LOW\n");
            } else if (strncmp(cmd, "CMD,USE_RANDOM,ON", 17) == 0) {
                use_random = true;
                printf("Randomness ENABLED\n");
            } else if (strncmp(cmd, "CMD,USE_RANDOM,OFF", 18) == 0) {
                use_random = false;
                printf("Randomness DISABLED\n");
            } else {
                printf("Unknown command: %s\n", cmd);
            }
        }
    }
}

void request_grant_task(void *pvParameter) {
    uint32_t request_count = 0;
    uint32_t grant_count = 0;

    while (1) {
        if (gpio_get_level(REQUEST_GPIO)) {
            request_count++;

            esp_rom_delay_us(2);

            int priority = gpio_get_level(PRIORITY_GPIO);
            uint32_t rand_value = esp_random() % 100;
            bool grant_low = (priority == 1 || (!priority_gpio_is_output && use_random && rand_value < 10));

            if (grant_low) {
                grant_count++;

                GPIO.out_w1tc.val = (1 << GRANT_GPIO) | (1 << GRANT_SWITCH);

                while (gpio_get_level(REQUEST_GPIO)) {
                    ;
                }

                GPIO.out_w1ts.val = (1 << GRANT_GPIO) | (1 << GRANT_SWITCH);

                // Send stats only when a grant is made
                printf("STATS,%lu,%lu\n", request_count, grant_count);
            } else {
                while (gpio_get_level(REQUEST_GPIO)) {
                    ;
                }
            }
        }
    }
}

void app_main(void) {
    // Set up GRANT output pins
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_GPIO);

    gpio_reset_pin(GRANT_SWITCH);
    gpio_set_direction(GRANT_SWITCH, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_SWITCH);

    // Configure PRIORITY as input by default
    gpio_reset_pin(PRIORITY_GPIO);
    gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);
    gpio_set_pull_mode(PRIORITY_GPIO, GPIO_FLOATING);

    // Configure REQUEST input
    gpio_config_t request_input = {
        .pin_bit_mask = (1ULL << REQUEST_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_ENABLE,
        .intr_type = GPIO_INTR_DISABLE
    };
    gpio_config(&request_input);

    // Set up UART for command reception
    const uart_config_t uart_config = {
        .baud_rate = 115200,
        .data_bits = UART_DATA_8_BITS,
        .parity    = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE
    };
    uart_param_config(UART_PORT, &uart_config);
    uart_driver_install(UART_PORT, UART_BUF_SIZE * 2, 0, 0, NULL, 0);

    // Start main tasks
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, NULL, configMAX_PRIORITIES - 1, NULL, tskNO_AFFINITY);
    xTaskCreatePinnedToCore(uart_command_task, "uart_command_task", 2048, NULL, 5, NULL, tskNO_AFFINITY);
}
