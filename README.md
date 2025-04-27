#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_rom_sys.h"
#include "soc/gpio_struct.h"
#include "esp_intr_alloc.h"
#include "hal/gpio_hal.h"
#include "esp32c6/rom/gpio.h"
#include "esp_timer.h"
#include "esp_system.h"
#include "esp_random.h"
#include "esp_mac.h"
#include "esp_task_wdt.h"
#include "driver/uart.h"

#define REQUEST_GPIO    GPIO_NUM_6
#define GRANT_GPIO      GPIO_NUM_7
#define PRIORITY_GPIO   GPIO_NUM_10
#define GRANT_SWITCH    GPIO_NUM_11

#define UART_PORT       UART_NUM_0
#define UART_BUF_SIZE   128

static bool use_random = true;
static bool priority_gpio_is_output = false;
static int priority_value = 0;  

void set_priority_mode_input(void) {
    gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);
    gpio_set_pull_mode(PRIORITY_GPIO, GPIO_FLOATING);
    priority_gpio_is_output = false;
    printf("PRIO_MODE set to INPUT\n");
}

void set_priority_mode_output(void) {
    gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(PRIORITY_GPIO, priority_value);
    priority_gpio_is_output = true;
    printf("PRIO_MODE set to OUTPUT\n");
}

void uart_command_task(void *arg) {
    uint8_t data[UART_BUF_SIZE];

    printf("Waiting for UART input...\n");

    while (1) {
        int len = uart_read_bytes(UART_PORT, data, sizeof(data) - 1, pdMS_TO_TICKS(500));
        if (len > 0) {
            data[len] = '\0';
            data[strcspn((char *)data, "\r\n")] = 0;
            printf("Received UART line (%d bytes): %s\n", len, data);

            char *cmd = (char *)data;

            if (strncmp(cmd, "CMD,PRIO_MODE,INPUT", 19) == 0) {
                set_priority_mode_input();
            } else if (strncmp(cmd, "CMD,PRIO_MODE,OUTPUT", 20) == 0) {
                set_priority_mode_output();
            } else if (strncmp(cmd, "CMD,PRIO_SET,HIGH", 17) == 0) {
                priority_value = 1;
                if (priority_gpio_is_output) {
                    gpio_set_level(PRIORITY_GPIO, 1);
                }
                printf("PRIO set to HIGH\n");
            } else if (strncmp(cmd, "CMD,PRIO_SET,LOW", 16) == 0) {
                priority_value = 0;
                if (priority_gpio_is_output) {
                    gpio_set_level(PRIORITY_GPIO, 0);
                }
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
        } else {
            printf("No UART input\n");
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

            int priority = priority_gpio_is_output ? priority_value : gpio_get_level(PRIORITY_GPIO);
            uint32_t rand_value = esp_random() % 100;
            bool grant_low = (priority == 1 || (!priority_gpio_is_output && use_random && rand_value < 10));

            if (grant_low) {
                grant_count++;

                GPIO.out_w1tc.val = (1 << GRANT_GPIO) | (1 << GRANT_SWITCH);
                while (gpio_get_level(REQUEST_GPIO)) {
                    ;
                }
                GPIO.out_w1ts.val = (1 << GRANT_GPIO) | (1 << GRANT_SWITCH);

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
    printf("ESP32 STARTED\n");

    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_GPIO);

    gpio_reset_pin(GRANT_SWITCH);
    gpio_set_direction(GRANT_SWITCH, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_SWITCH);

    gpio_reset_pin(PRIORITY_GPIO);
    set_priority_mode_input();

    gpio_config_t request_input = {
        .pin_bit_mask = (1ULL << REQUEST_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_ENABLE,
        .intr_type = GPIO_INTR_DISABLE
    };
    gpio_config(&request_input);

    const uart_config_t uart_config = {
        .baud_rate = 115200,
        .data_bits = UART_DATA_8_BITS,
        .parity    = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE
    };
    uart_param_config(UART_PORT, &uart_config);
    uart_set_pin(UART_PORT, GPIO_NUM_16, GPIO_NUM_17, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
    uart_driver_install(UART_PORT, UART_BUF_SIZE * 2, 0, 0, NULL, 0);
    printf("1UART task started\n");


    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, NULL, configMAX_PRIORITIES - 1, NULL, tskNO_AFFINITY);
    printf("request_grant_task task started\n");

    xTaskCreatePinnedToCore(uart_command_task, "uart_command_task", 2048, NULL, 5, NULL, tskNO_AFFINITY);
    printf("UART task started\n");

}
