void request_grant_task(void *arg) {
    ESP_LOGI(TAG, "Request-Grant task started");

    static int last_request = 0;

    while (1) {
        int current_request = gpio_get_level(REQUEST_GPIO);

        if (current_request && !last_request) {
            // Rising edge detected
            request_count++;

            bool grant = false;
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
                ets_delay_us(10);  // Pulse duration
                gpio_set_level(GRANT_GPIO, 0);
                gpio_set_level(SWITCH_GRANT_GPIO, 0);
                grant_count++;
            }

            // Send stats back over UART
            char stats_msg[64];
            snprintf(stats_msg, sizeof(stats_msg), "STATS,%d,%d\n", request_count, grant_count);
            uart_write_bytes(UART_PORT, stats_msg, strlen(stats_msg));
        }

        last_request = current_request;

        ets_delay_us(5);  // Small delay for edge detection stability
    }
}
