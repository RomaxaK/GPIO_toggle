#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "driver/uart.h"
#include "esp_rom_sys.h"
#include "esp_log.h"

// UART configuration
#define UART_PORT       UART_NUM_0
#define UART_TX_PIN     GPIO_NUM_16
#define UART_RX_PIN     GPIO_NUM_17
#define UART_BUF_SIZE   1024

// GPIOs
#define REQUEST_GPIO        GPIO_NUM_6
#define GRANT_GPIO          GPIO_NUM_7
#define SWITCH_GRANT_GPIO   GPIO_NUM_11

// Grant mode enum
typedef enum {
    GRANT_MODE_ALWAYS,
    GRANT_MODE_NONE,
    GRANT_MODE_RANDOM
} grant_mode_t;

static volatile grant_mode_t current_grant_mode = GRANT_MODE_RANDOM;
static const char *TAG = "APP";

static uint32_t request_count = 0;
static uint32_t grant_count = 0;

// UART command handler
void handle_uart_command(const char *cmd) {
    if (strcmp(cmd, "CMD,GRANT_MODE,ALWAYS") == 0) {
        current_grant_mode = GRANT_MODE_ALWAYS;
        printf("GRANT_MODE set to ALWAYS\n");
    } else if (strcmp(cmd, "CMD,GRANT_MODE,NONE") == 0) {
        current_grant_mode = GRANT_MODE_NONE;
        printf("GRANT_MODE set to NONE\n");
    } else if (strcmp(cmd, "CMD,GRANT_MODE,RANDOM") == 0) {
        current_grant_mode = GRANT_MODE_RANDOM;
        printf("GRANT_MODE set to RANDOM\n");
    } else {
        printf("Unknown command: %s\n", cmd);
    }
}

// UART listener task
void uart_task(void *arg) {
    uint8_t data[UART_BUF_SIZE];

    while (1) {
        int len = uart_read_bytes(UART_PORT, data, sizeof(data) - 1, pdMS_TO_TICKS(100));
        if (len > 0) {
            data[len] = '\0';
            for (int i = 0; i < len; i++) {
                if (data[i] == '\r' || data[i] == '\n') {
                    data[i] = '\0';
                    break;
                }
            }
            handle_uart_command((const char *)data);
        }
    }
}

// Grant logic task
void request_grant_task(void *arg) {
    int last_request = 0;

    while (1) {
        int request = gpio_get_level(REQUEST_GPIO);
        if (request && !last_request) request_count++;
        last_request = request;

        bool grant = false;

        switch (current_grant_mode) {
            case GRANT_MODE_ALWAYS:
                grant = request;
                break;
            case GRANT_MODE_NONE:
                grant = false;
                break;
            case GRANT_MODE_RANDOM:
                grant = request && (esp_random() % 100 < 10);
                break;
        }

        if (grant) {
            gpio_set_level(GRANT_GPIO, 1);
            gpio_set_level(SWITCH_GRANT_GPIO, 1);
            grant_count++;
        } else {
            gpio_set_level(GRANT_GPIO, 0);
            gpio_set_level(SWITCH_GRANT_GPIO, 0);
        }

        // Send stats every 10 grants
        if (grant_count % 10 == 0 && grant) {
            printf("STATS,%lu,%lu\n", request_count, grant_count);
        }

        esp_rom_delay_us(10);
    }
}

// Main function
void app_main(void) {
    // GPIO init
    gpio_reset_pin(REQUEST_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_INPUT);
    gpio_pullup_en(REQUEST_GPIO);

    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(GRANT_GPIO, 0);

    gpio_reset_pin(SWITCH_GRANT_GPIO);
    gpio_set_direction(SWITCH_GRANT_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(SWITCH_GRANT_GPIO, 0);

    // UART init
    const uart_config_t uart_config = {
        .baud_rate = 115200,
        .data_bits = UART_DATA_8_BITS,
        .parity    = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
    };
    uart_param_config(UART_PORT, &uart_config);
    uart_set_pin(UART_PORT, UART_TX_PIN, UART_RX_PIN, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
    uart_driver_install(UART_PORT, UART_BUF_SIZE * 2, 0, 0, NULL, 0);

    // Start tasks
    xTaskCreatePinnedToCore(uart_task, "uart_task", 4096, NULL, 10, NULL, tskNO_AFFINITY);
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 4096, NULL, 9, NULL, tskNO_AFFINITY);
}
