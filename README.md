void request_grant_task(void *arg) {
    static uint32_t request_count = 0;
    static uint32_t grant_count = 0;
    int last_request = 0;
    int grant_active = 0;

    ESP_LOGI(TAG, "Request-Grant task started");

    while (1) {
        int request = gpio_get_level(REQUEST_GPIO);
        int priority = gpio_get_level(PRIORITY_GPIO);
        bool grant = false;

        // Rising edge of REQUEST
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
                    grant = (esp_random() % 100) < 50;  // 50% chance
                    break;
            }

            if (grant) {
                gpio_set_level(GRANT_GPIO, 1);
                gpio_set_level(SWITCH_GRANT_GPIO, 1);
                grant_active = 1;
                grant_count++;
            }
        }

        // Falling edge of REQUEST — clear GRANT
        if (!request && last_request && grant_active) {
            gpio_set_level(GRANT_GPIO, 0);
            gpio_set_level(SWITCH_GRANT_GPIO, 0);
            grant_active = 0;
        }

        // Save current REQUEST state
        last_request = request;

        // Send updated stats periodically
        static uint32_t last_print = 0;
        if (esp_log_timestamp() - last_print > 1000) {  // every 1 sec
            char stats_msg[64];
            snprintf(stats_msg, sizeof(stats_msg), "STATS,%" PRIu32 ",%" PRIu32 "\n", request_count, grant_count);
            uart_write_bytes(UART_PORT, stats_msg, strlen(stats_msg));
            last_print = esp_log_timestamp();
        }

        ets_delay_us(10);  // low-latency loop
    }
}
