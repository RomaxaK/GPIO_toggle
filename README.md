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
#include "inttypes.h"

#define UART_PORT       UART_NUM_0
#define UART_TX_PIN     GPIO_NUM_16
#define UART_RX_PIN     GPIO_NUM_17
#define UART_BUF_SIZE   1024

#define REQUEST_GPIO        GPIO_NUM_6
#define GRANT_GPIO          GPIO_NUM_7
#define SWITCH_GRANT_GPIO   GPIO_NUM_11

typedef enum {
    GRANT_MODE_ALWAYS,
    GRANT_MODE_NONE,
    GRANT_MODE_RANDOM
} grant_mode_t;

static volatile grant_mode_t current_grant_mode = GRANT_MODE_ALWAYS;
static const char *TAG = "APP";

// Global counters
static uint32_t request_count = 0;
static uint32_t grant_count = 0;

// Reset counters function
void reset_counters(void) {
    request_count = 0;
    grant_count = 0;
    printf("Counters reset\n");
}

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
    } else if (strcmp(cmd, "CMD,RESET_COUNTERS") == 0) {
        reset_counters();
    } else {
        printf("Unknown command: %s\n", cmd);
    }
}

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

void request_grant_task(void *arg) {
    int last_request = 0;
    int grant_active = 0;

    ESP_LOGI(TAG, "Request-Grant task started");

    while (1) {
        int request = gpio_get_level(REQUEST_GPIO);
        bool grant = false;

        if (request && !last_request) {
            request_count++;

            switch (current_grant_mode) {
                case GRANT_MODE_ALWAYS:
                    grant = true;
                    break;
                case GRANT_MODE_NONE:
                    grant = false;
                    break;
                case GRANT_MODE_RANDOM:
                    grant = (esp_random() % 100) < 10;
                    break;
            }

            if (grant) {
                gpio_set_level(GRANT_GPIO, 0);           
                gpio_set_level(SWITCH_GRANT_GPIO, 0);    
                grant_active = 1;
                grant_count++;
            }
        }

        if (!request && last_request && grant_active) {
            gpio_set_level(GRANT_GPIO, 1);          
            gpio_set_level(SWITCH_GRANT_GPIO, 1);
            grant_active = 0;
        }

        last_request = request;

        static uint32_t last_print = 0;
        if (esp_log_timestamp() - last_print > 1000) {
            char stats_msg[64];
            snprintf(stats_msg, sizeof(stats_msg), "STATS,%" PRIu32 ",%" PRIu32 "\n", request_count, grant_count);
            uart_write_bytes(UART_PORT, stats_msg, strlen(stats_msg));
            last_print = esp_log_timestamp();
        }

        esp_rom_delay_us(10);
    }
}

void app_main(void) {
    gpio_reset_pin(REQUEST_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_INPUT);
    gpio_pullup_en(REQUEST_GPIO);

    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(GRANT_GPIO, 1);

    gpio_reset_pin(SWITCH_GRANT_GPIO);
    gpio_set_direction(SWITCH_GRANT_GPIO, GPIO_MODE_OUTPUT);
    gpio_set_level(SWITCH_GRANT_GPIO, 1);

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

    xTaskCreatePinnedToCore(uart_task, "uart_task", 4096, NULL, 10, NULL, tskNO_AFFINITY);
    xTaskCreatePinnedToCore(request_grant_task, "request_grant_task", 4096, NULL, 9, NULL, tskNO_AFFINITY);
}
