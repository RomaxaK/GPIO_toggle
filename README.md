#include <stdio.h>
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/uart.h"
#include "driver/gpio.h"
#include "esp_timer.h"
#include "esp_log.h"

#define UART_PORT        UART_NUM_0
#define UART_TX_PIN      GPIO_NUM_16
#define UART_RX_PIN      GPIO_NUM_17
#define UART_BUF_SIZE    1024

#define REQUEST_GPIO     GPIO_NUM_6
#define GRANT_GPIO       GPIO_NUM_7
#define PRIORITY_GPIO    GPIO_NUM_10

#define TAG "APP"

static bool random_mode = false;

// UART Task
void uart_task(void *arg) {
    uint8_t data[UART_BUF_SIZE];
    ESP_LOGI(TAG, "UART task started, waiting for input...");

    while (1) {
        int len = uart_read_bytes(UART_PORT, data, sizeof(data) - 1, pdMS_TO_TICKS(500));
        if (len > 0) {
            data[len] = '\0';  // Null-terminate

            ESP_LOGI(TAG, "Received UART (%d bytes): %s", len, data);

            if (strstr((char*)data, "CMD_PRIO_MODE_INPUT")) {
                gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);
                ESP_LOGI(TAG, "PRIORITY set to INPUT");
            }
            else if (strstr((char*)data, "CMD_PRIO_MODE_OUTPUT")) {
                gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_OUTPUT);
                ESP_LOGI(TAG, "PRIORITY set to OUTPUT");
            }
            else if (strstr((char*)data, "CMD_PRIO_SET_HIGH")) {
                gpio_set_level(PRIORITY_GPIO, 1);
                ESP_LOGI(TAG, "PRIORITY set to HIGH");
            }
            else if (strstr((char*)data, "CMD_PRIO_SET_LOW")) {
                gpio_set_level(PRIORITY_GPIO, 0);
                ESP_LOGI(TAG, "PRIORITY set to LOW");
            }
            else if (strstr((char*)data, "CMD_RANDOM_ON")) {
                random_mode = true;
                ESP_LOGI(TAG, "Random mode ENABLED");
            }
            else if (strstr((char*)data, "CMD_RANDOM_OFF")) {
                random_mode = false;
                ESP_LOGI(TAG, "Random mode DISABLED");
            }
        }
    }
}

// Grant/Request Task
void request_grant_task(void *arg) {
    ESP_LOGI(TAG, "Request-Grant task started");

    while (1) {
        int request = gpio_get_level(REQUEST_GPIO);
        int priority = gpio_get_level(PRIORITY_GPIO);

        bool grant = false;

        if (request) {
            if (random_mode) {
                // Example: 50% random grant (simple approach)
                grant = (esp_timer_get_time() % 2) == 0;
            }
            else {
                grant = (priority == 0);  // Give grant only if priority LOW
            }
        }
        else {
            grant = true;  // No request → always grant
        }

        gpio_set_level(GRANT_GPIO, grant);

        vTaskDelay(pdMS_TO_TICKS(10));  // Small delay
    }
}

void app_main(void) {
    ESP_LOGI(TAG, "ESP32 booted");

    // Configure GPIOs
    gpio_reset_pin(REQUEST_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_INPUT);
    gpio_pullup_en(REQUEST_GPIO);

    gpio_reset_pin(PRIORITY_GPIO);
    gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);
    gpio_pullup_en(PRIORITY_GPIO);

    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(GRANT_GPIO, 1);  // Default GRANT high

    // Configure UART
    const uart_config_t uart_config = {
        .baud_rate = 115200,
        .data_bits = UART_DATA_8_BITS,
        .parity    = UART_PARITY_DISABLE,
        .stop_bits = UART_STOP_BITS_1,
        .flow_ctrl = UART_HW_FLOWCTRL_DISABLE,
    };
    uart_param_config(UART_PORT, &uart_config);
    uart_set_pin(UART_PORT, UART_TX_PIN, UART_RX_PIN, UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
    uart_driver_install(UART_PORT, UART_BUF_SIZE * 2, UART_BUF_SIZE * 2, 0, NULL, 0);

    ESP_LOGI(TAG, "UART configured");

    // Start tasks
    xTaskCreate(uart_task, "uart_task", 4096, NULL, 10, NULL);
    xTaskCreate(request_grant_task, "request_grant_task", 2048, NULL, 9, NULL);
}
