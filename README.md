void uart_command_task(void *arg) {
    uint8_t data[UART_BUF_SIZE];

    printf("Waiting for UART input...\n");

    while (1) {
        int len = uart_read_bytes(UART_PORT, data, sizeof(data) - 1, pdMS_TO_TICKS(500));
        if (len > 0) {
            data[len] = '\0';
            data[strcspn((char *)data, "\r\n")] = 0;
            printf("Received UART line (%d bytes): %s\n", len, data);

            char *cmd = (char *)data;

            if (strncmp(cmd, "CMD,PRIO_MODE,INPUT", 19) == 0) {
                set_priority_mode_input();
            } else if (strncmp(cmd, "CMD,PRIO_MODE,OUTPUT", 20) == 0) {
                set_priority_mode_output();
            } else if (strncmp(cmd, "CMD,PRIO_SET,HIGH", 17) == 0) {
                priority_value = 1;
                if (priority_gpio_is_output) {
                    gpio_set_level(PRIORITY_GPIO, 1);
                }
                printf("PRIO set to HIGH\n");
            } else if (strncmp(cmd, "CMD,PRIO_SET,LOW", 16) == 0) {
                priority_value = 0;
                if (priority_gpio_is_output) {
                    gpio_set_level(PRIORITY_GPIO, 0);
                }
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
        } else {
            // Optional: print something so we know it's alive
            // printf("No UART input\n");
        }
    }
}
