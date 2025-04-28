#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "driver/uart.h"
#include "esp_log.h"

#define UART_PORT UART_NUM_0
#define UART_BUF_SIZE 1024

#define REQUEST_GPIO GPIO_NUM_6
#define GRANT_GPIO   GPIO_NUM_7
#define PRIORITY_GPIO GPIO_NUM_10

static bool priority_gpio_is_output = false;
static bool use_random = true;
static int priority_value = 0; // 0: LOW, 1: HIGH

// Helper functions
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

// UART command processing
void process_uart_command(const char *cmd) {
    printf("Processing command: %s\n", cmd);

    if (strncmp(cmd, "CMD,PRIO_MODE,INPUT", 19) == 0) {
        set_priority_mode_input();
    } else if (strncmp(cmd, "CMD,PRIO_MODE,OUTPUT", 20) == 0) {
        set_priority_mode_output();
    } else if (strncmp(cmd, "CMD,PRIO_SET,HIGH", 17) == 0) {
        priority_value = 1;
        if (priority_gpio_is_output) {
            gpio_set_level(PRIORITY_GPIO, 1);
        }
        printf("Priority set to HIGH\n");
    } else if (strncmp(cmd, "CMD,PRIO_SET,LOW", 16) == 0) {
        priority_value = 0;
        if (priority_gpio_is_output) {
            gpio_set_level(PRIORITY_GPIO, 0);
        }
        printf("Priority set to LOW\n");
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

// UART reading task
void uart_command_task(void *arg) {
    uint8_t ch;
    char line_buffer[UART_BUF_SIZE];
    size_t idx = 0;

    printf("Waiting for UART input...\n");

    while (1) {
        int len = uart_read_bytes(UART_PORT, &ch, 1, pdMS_TO_TICKS(100));
        if (len > 0) {
            if (ch == '\r') continue; // Skip \r
            if (ch == '\n') {
                line_buffer[idx] = '\0';
                if (idx > 0) {
                    printf("Received UART line: %s\n", line_buffer);
                    process_uart_command(line_buffer);
                }
                idx = 0;
            } else if (idx < sizeof(line_buffer) - 1) {
                line_buffer[idx++] = ch;
            } else {
                idx = 0; // Overflow protection
            }
        }
    }
}

// Dummy grant task (replace this later)
void request_grant_task(void *pvParameter) {
    while (1) {
        vTaskDelay(pdMS_TO_TICKS(1000));
    }
}

void app_main(void) {
    printf("ESP32 STARTED\n");

    // Init grant GPIOs
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_GPIO);

    gpio_reset_pin(PRIORITY_GPIO);
    set_priority_mode_input();

    gpio_reset_pin(REQUEST_GPIO);
    gpio_config_t request_input = {
        .pin_bit_mask = (1ULL << REQUEST_GPIO),
        .mode = GPIO_MODE_INPUT,
        .pull_up_en = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_ENABLE,
        .intr_type = GPIO_INTR_DISABLE
    };
    gpio_config(&request_input);

    // UART config
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

    printf("UART configured\n");

    // Tasks
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 2048, NULL, configMAX_PRIORITIES-1, NULL, tskNO_AFFINITY);
    xTaskCreatePinnedToCore(uart_command_task, "uart_command_task", 2048, NULL, 5, NULL, tskNO_AFFINITY);
}
