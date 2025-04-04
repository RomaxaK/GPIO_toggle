#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/gpio.h"
#include "esp_log.h"
#include "esp_rom_sys.h"
#include "soc/gpio_struct.h"
#include "esp_intr_alloc.h"

// Define GPIOs
#define REQUEST_GPIO   GPIO_NUM_4      // Input: request signal from QPG7015M
#define GRANT_GPIO     GPIO_NUM_5      // Output: grant signal to RF switch
#define PRIORITY_GPIO  GPIO_NUM_6      // Input: priority signal from QPG7015M

#define GRANT_PULSE_US 100             // Microseconds GRANT stays LOW

static const char *TAG = "ISR_GRANT";

// ISR handler
static void IRAM_ATTR request_isr_handler(void* arg) {
    // Read priority state (low latency direct call)
    int priority = gpio_get_level(PRIORITY_GPIO);

    if (priority == 1) {
        // Set GRANT low using fast register write
        GPIO.out_w1tc.val = (1 << GRANT_GPIO);
        // Short pulse (you can adjust this as needed)
        esp_rom_delay_us(GRANT_PULSE_US);
        // Set GRANT high again
        GPIO.out_w1ts.val = (1 << GRANT_GPIO);
    }
}

void app_main(void) {
    ESP_LOGI(TAG, "Initializing GPIOs and interrupt...");

    // Configure GRANT as output, initialize HIGH
    gpio_reset_pin(GRANT_GPIO);
    gpio_set_direction(GRANT_GPIO, GPIO_MODE_OUTPUT);
    GPIO.out_w1ts.val = (1 << GRANT_GPIO);  // set HIGH

    // Configure PRIORITY as input
    gpio_reset_pin(PRIORITY_GPIO);
    gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);

    // Configure REQUEST as input with rising edge interrupt
    gpio_reset_pin(REQUEST_GPIO);
    gpio_set_direction(REQUEST_GPIO, GPIO_MODE_INPUT);
    gpio_set_intr_type(REQUEST_GPIO, GPIO_INTR_POSEDGE);

    // Install ISR service (shared level 1 interrupt)
    gpio_install_isr_service(ESP_INTR_FLAG_IRAM);
    gpio_isr_handler_add(REQUEST_GPIO, request_isr_handler, NULL);

    ESP_LOGI(TAG, "Interrupt handler ready.");
}
