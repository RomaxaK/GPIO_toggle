void uart_command_task(void *arg) {
    uint8_t data[128];

    while (1) {
        int len = uart_read_bytes(UART_PORT, data, sizeof(data) - 1, pdMS_TO_TICKS(100));
        if (len > 0) {
            data[len] = '\0';  // Null-terminate
            data[strcspn((char *)data, "\r\n")] = 0;  // Remove newline and carriage return

            printf("RECEIVED: %s\n", data);

            char *cmd = (char *)data;

            if (strncmp(cmd, "CMD,PRIO_MODE,INPUT", 19) == 0) {
                gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_INPUT);
                priority_gpio_is_output = false;
                printf("PRIO_MODE set to INPUT\n");

            } else if (strncmp(cmd, "CMD,PRIO_MODE,OUTPUT", 20) == 0) {
                gpio_set_direction(PRIORITY_GPIO, GPIO_MODE_OUTPUT);
                priority_gpio_is_output = true;
                gpio_set_level(PRIORITY_GPIO, priority_value);
                printf("PRIO_MODE set to OUTPUT\n");

            } else if (strncmp(cmd, "CMD,PRIO_SET,HIGH", 17) == 0) {
                priority_value = 1;
                if (priority_gpio_is_output)
                    gpio_set_level(PRIORITY_GPIO, 1);
                printf("PRIO set to HIGH\n");

            } else if (strncmp(cmd, "CMD,PRIO_SET,LOW", 16) == 0) {
                priority_value = 0;
                if (priority_gpio_is_output)
                    gpio_set_level(PRIORITY_GPIO, 0);
                printf("PRIO set to LOW\n");

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
    }
}
