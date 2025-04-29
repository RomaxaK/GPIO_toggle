#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "driver/uart.h"
#include "esp_log.h"

// UART settings
#define UART_PORT       UART_NUM_0
#define UART_TX_PIN     GPIO_NUM_16
#define UART_RX_PIN     GPIO_NUM_17
#define UART_BUF_SIZE   1024

// GPIOs
#define REQUEST_GPIO        GPIO_NUM_6
#define GRANT_GPIO          GPIO_NUM_7
#define PRIORITY_GPIO       GPIO_NUM_10
#define SWITCH_GRANT_GPIO   GPIO_NUM_11

static const char *TAG = "APP";

// Counters
volatile int request_count = 0;
volatile int grant_count = 0;

// UART command handler
void handle_uart_command(const char *cmd) {
    if (strcmp(cmd, "CMD_PRIO_MODE_INPUT") == 0) {
        gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);
        ESP_LOGI(TAG, "Set PRIORITY_GPIO to INPUT");
    } else if (strcmp(cmd, "CMD_PRIO_MODE_OUTPUT") == 0) {
        gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_OUTPUT);
        ESP_LOGI(TAG, "Set PRIORITY_GPIO to OUTPUT");
    } else if (strcmp(cmd, "CMD_PRIO_SET_HIGH") == 0) {
        gpio_set_level(PRIORITY_GPIO, 1);
        ESP_LOGI(TAG, "Set PRIORITY_GPIO HIGH");
    } else if (strcmp(cmd, "CMD_PRIO_SET_LOW") == 0) {
        gpio_set_level(PRIORITY_GPIO, 0);
        ESP_LOGI(TAG, "Set PRIORITY_GPIO LOW");
    } else {
        ESP_LOGW(TAG, "Unknown UART command: %s", cmd);
    }
}

// UART receive + response
void uart_task(void *arg) {
    uint8_t data[UART_BUF_SIZE];

    ESP_LOGI(TAG, "UART task started");

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
            ESP_LOGI(TAG, "Received UART (%d bytes): %s", len, data);
            handle_uart_command((const char *)data);
        }

        // Periodic status send (every loop ~100ms)
        char msg[64];
        snprintf(msg, sizeof(msg), "REQUESTS:%d GRANTS:%d\n", request_count, grant_count);
        uart_write_bytes(UART_PORT, msg, strlen(msg));
    }
}

// Grant logic
void request_grant_task(void *arg) {
    ESP_LOGI(TAG, "Request-Grant task started");

    int last_request = 0;

    while (1) {
        int request = gpio_get_level(REQUEST_GPIO);
        int priority = gpio_get_level(PRIORITY_GPIO);

        if (request && !last_request) request_count++;
        last_request = request;

        if (request) {
            gpio_set_level(GRANT_GPIO, 1);
            gpio_set_level(SWITCH_GRANT_GPIO, 1);
            grant_count++;
        } else {
            gpio_set_level(GRANT_GPIO, 0);
            gpio_set_level(SWITCH_GRANT_GPIO, 0);
        }

        ets_delay_us(10);  // ~10us grant reaction time
    }
}

// Init
void app_main(void) {
    ESP_LOGI(TAG, "ESP32 booted");

    // GPIO setup
    gpio_reset_pin(REQUEST_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_INPUT);
    gpio_pullup_en(REQUEST_GPIO);
    gpio_pulldown_dis(REQUEST_GPIO);

    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(GRANT_GPIO, 0);

    gpio_reset_pin(SWITCH_GRANT_GPIO);
    gpio_set_direction(SWITCH_GRANT_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(SWITCH_GRANT_GPIO, 0);

    gpio_reset_pin(PRIORITY_GPIO);
    gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);

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

    ESP_LOGI(TAG, "UART configured");

    // Tasks
    xTaskCreate(uart_task, "uart_task", 4096, NULL, 10, NULL);
    xTaskCreate(request_grant_task, "request_grant_task", 2048, NULL, 9, NULL);
}
