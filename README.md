#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/uart.h"
#include "driver/gpio.h"
#include "string.h"

#define UART_PORT       UART_NUM_0
#define UART_TX_PIN     GPIO_NUM_16
#define UART_RX_PIN     GPIO_NUM_17
#define UART_BUF_SIZE   1024

void uart_task(void *arg) {
    uint8_t data[UART_BUF_SIZE];

    printf("UART task started, waiting for input...\n");

    while (1) {
        int len = uart_read_bytes(UART_PORT, data, sizeof(data) - 1, pdMS_TO_TICKS(500));
        if (len > 0) {
            data[len] = '\0';  // Null-terminate
            printf("Received UART (%d bytes): %s\n", len, data);
        }
    }
}

void app_main(void) {
    printf("ESP32 booted\n");

    // UART configuration
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

    printf("UART configured\n");

    xTaskCreate(uart_task, "uart_task", 4096, NULL, 10, NULL);
}
