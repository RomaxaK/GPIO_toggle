void uart_command_task(void *arg) {
    uint8_t data[UART_BUF_SIZE];

    while (1) {
        // Read UART input
        int len = uart_read_bytes(UART_PORT, data, sizeof(data) - 1, pdMS_TO_TICKS(100));
        if (len > 0) {
            data[len] = '\0';  // Null-terminate
            data[strcspn((char *)data, "\r\n")] = 0;  // Strip newline (if any)

            printf("RECEIVED: %s\n", data);
            char *cmd = (char *)data;

            // Command parsing with strcmp (safer and clearer)
            if (strcmp(cmd, "CMD,PRIO_MODE,INPUT") == 0) {
                gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);
                priority_gpio_is_output = false;
                printf("PRIO_MODE set to INPUT\n");

            } else if (strcmp(cmd, "CMD,PRIO_MODE,OUTPUT") == 0) {
                gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_OUTPUT);
                priority_gpio_is_output = true;
                gpio_set_level(PRIORITY_GPIO, priority_value);  // Apply current level
                printf("PRIO_MODE set to OUTPUT\n");

            } else if (strcmp(cmd, "CMD,PRIO_SET,HIGH") == 0) {
                priority_value = 1;
                if (priority_gpio_is_output) {
                    gpio_set_level(PRIORITY_GPIO, 1);
                }
                printf("PRIO set to HIGH\n");

            } else if (strcmp(cmd, "CMD,PRIO_SET,LOW") == 0) {
                priority_value = 0;
                if (priority_gpio_is_output) {
                    gpio_set_level(PRIORITY_GPIO, 0);
                }
                printf("PRIO set to LOW\n");

            } else if (strcmp(cmd, "CMD,USE_RANDOM,ON") == 0) {
                use_random = true;
                printf("Randomness ENABLED\n");

            } else if (strcmp(cmd, "CMD,USE_RANDOM,OFF") == 0) {
                use_random = false;
                printf("Randomness DISABLED\n");

            } else {
                printf("Unknown command: %s\n", cmd);
            }
        }
    }
}
